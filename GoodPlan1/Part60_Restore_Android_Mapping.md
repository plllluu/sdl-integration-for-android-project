# Part 60 (Revised): Restore Android Driver Mapping and Fix Build

You are right, my previous fixes were insufficient and created new problems. My apologies. I have re-analyzed the situation based on the new compiler errors. The initial change to support the Android driver mapping correctly required changes in `input_common/main.h`, which then caused cascading build errors in other files because their function signatures no longer matched.

This revised plan will not only restore the Android driver mapping but also fix the two resulting compilation errors, ensuring a clean build.

### Part 1: Update `input_common/main.h` (No Change from Original Plan)

**Action:** Modify the `InputSubsystem::BeginMapping` signature to accept device parameters.
**Rationale:** To allow the system to identify which input driver to use for mapping.

```diff
// in class InputSubsystem
    /// Start polling from all backends for a desired input type.
-   void BeginMapping(Polling::InputType type);
+   void BeginMapping(const Common::ParamPackage& params, Polling::InputType type);
```

### Part 2: Update `input_common/drivers/android.h` (No Change from Original Plan)

**Action:** Add mapping-related method declarations to the `Android` driver.
**Rationale:** To equip the `Android` driver with the necessary functions for the mapping process.

```diff
class Android final : public InputEngine {
public:
    // ...
+   void BeginMapping(Polling::InputType type);
+   void StopMapping();
+   bool IsMapping() const;
+   Common::ParamPackage GetCapturedInput();
+
private:
    // ...
+   // Mapping Members
+   std::atomic<bool> is_mapping = false;
+   Common::ParamPackage captured_input;
    // ...
};
```

### Part 3: Update `input_common/drivers/android.cpp` (No Change from Original Plan)

**Action:** Implement the mapping methods and input capture logic.
**Rationale:** To enable the `Android` driver to capture input events when mapping is active.

```cpp
// Add these method implementations
void Android::BeginMapping(Polling::InputType type) {
    is_mapping = true;
    captured_input = {};
}

void Android::StopMapping() {
    is_mapping = false;
}

bool Android::IsMapping() const {
    return is_mapping;
}

Common::ParamPackage Android::GetCapturedInput() {
    Common::ParamPackage input_ref = captured_input;
    captured_input = {};
    return input_ref;
}

// Modify existing SetButtonState and SetAxisPosition...
```

### Part 4: Fix `SdlAndroid` Driver Signature (`input_common/drivers/sdl_android.h`)

**Action:** Update the `BeginMapping` function signature in the `SdlAndroid` driver.
**Rationale:** To fix the first compiler error (`too many arguments to function call`). The signature must match the new logic that will be implemented in `main.cpp`.

```diff
// in class SdlAndroid
-   void BeginMapping();
+   void BeginMapping(Polling::InputType type);
```

### Part 5: Update `SdlAndroid` Driver Implementation (`input_common/drivers/sdl_android.cpp`)

**Action:** Update the implementation of `BeginMapping` in the `SdlAndroid` driver.
**Rationale:** To match the new header signature defined in the previous step.

```diff
-void SdlAndroid::BeginMapping() {
+void SdlAndroid::BeginMapping(Polling::InputType type) {
     is_mapping = true;
     captured_input = {};
 }
```

### Part 6: Fix `MappingFactory` (`input_common/input_poller.h`)

**Action:** Add a public `IsMapping()` method to the `MappingFactory` class.
**Rationale:** To fix the second compiler error (`no member named 'IsMapping'`). This allows the `InputSubsystem` to query if the generic mapping factory is active.

```diff
class MappingFactory final : public Polling::Subsystem {
public:
    void BeginMapping(Polling::InputType type) override;
    void StopMapping() override;
+   bool IsMapping() const;
    Common::ParamPackage GetNextInput() const;
    void RegisterInput(const MappingData& data);
private:
    Common::ParamPackage captured_input;
    std::atomic<bool> is_mapping = false;
};
```

### Part 7: Update `MappingFactory` Implementation (`input_common/input_poller.cpp`)

**Action:** Implement the new `IsMapping()` method and other missing `MappingFactory` methods.
**Rationale:** To provide the function bodies for the declarations in the header file. These appear to have been missing from the file entirely.

```cpp
// Add these new function implementations at the end of the file

void MappingFactory::BeginMapping(Polling::InputType type) {
    is_mapping = true;
    captured_input = {};
}

void MappingFactory::StopMapping() {
    is_mapping = false;
}

bool MappingFactory::IsMapping() const {
    return is_mapping;
}

Common::ParamPackage MappingFactory::GetNextInput() const {
    return captured_input;
}

void MappingFactory::RegisterInput(const MappingData& data) {
    if (is_mapping) {
        captured_input = data.params;
        is_mapping = false;
    }
}
```

### Part 8: Update `InputSubsystem` Logic (`input_common/main.cpp`)

**Action:** Implement the final mapping logic in `InputSubsystem`.
**Rationale:** This centralizes the mapping logic, correctly delegating to a specific driver (`android` or `sdl_android`) if one is provided, or falling back to the generic `mapping_factory` if not. This logic works now because the dependency issues are resolved in the preceding steps.

```cpp
// In input_common/main.cpp
void InputSubsystem::BeginMapping(const Common::ParamPackage& params, Polling::InputType type) {
    impl->BeginConfiguration();
#ifdef ANDROID
    impl->mapping_engine = impl->GetInputEngine(params);
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            android->BeginMapping(type);
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            sdl_android->BeginMapping(type);
        }
    } else {
        impl->mapping_factory->BeginMapping(type);
    }
#else
    impl->mapping_factory->BeginMapping(type);
#endif
}

Common::ParamPackage InputSubsystem::GetNextInput() const {
#ifdef ANDROID
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            return android->GetCapturedInput();
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            return sdl_android->GetCapturedInput();
        }
    }
    return impl->mapping_factory->GetNextInput();
#else
    return impl->mapping_factory->GetNextInput();
#endif
}

void InputSubsystem::StopMapping() const {
    impl->EndConfiguration();
#ifdef ANDROID
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            android->StopMapping();
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            sdl_android->StopMapping();
        }
        impl->mapping_engine = nullptr;
    } else {
        impl->mapping_factory->StopMapping();
    }
#else
    impl->mapping_factory->StopMapping();
#endif
}

bool InputSubsystem::IsMapping() const {
#ifdef ANDROID
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            return android->IsMapping();
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            return sdl_android->IsMapping();
        }
    }
    return impl->mapping_factory->IsMapping();
#else
    return impl->mapping_factory->IsMapping();
#endif
}
```

### Part 9: Fix JNI `beginMapping` Call (`native_input.cpp`)

**Action:** Update `beginMapping` to pass the device parameter string.
**Rationale:** The JNI call signature was changed. This adapts the call from the Java side, passing the device parameters which are needed by the new `InputSubsystem` logic.

```diff
 void Java_org_yuzu_yuzu_1emu_features_input_NativeInput_beginMapping(JNIEnv* env, jobject j_obj,
-                                                                     jint jtype) {
-    EmulationSession::GetInstance().GetInputSubsystem().BeginMapping(
-        static_cast<InputCommon::Polling::InputType>(jtype));
+                                                                     jstring j_device_params, jint jtype) {
+    EmulationSession::GetInstance().GetInputSubsystem().BeginMapping(
+        Common::ParamPackage(Common::Android::GetJString(env, j_device_params)),
+        static_cast<InputCommon::Polling::InputType>(jtype));
 }
```
