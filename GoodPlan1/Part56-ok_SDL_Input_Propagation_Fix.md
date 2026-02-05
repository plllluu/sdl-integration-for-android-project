# Part 56: Fix SDL Input Propagation for Manual Mapping

## Explanation

**Problem:** The manual mapping of SDL controller buttons is failing. When the `InputDialogFragment` is active, pressing a button on an SDL device does not register, preventing the mapping from completing.

**Root Cause Analysis (as per Rule 42):
**
1.  **Event Pipeline OK, Capture is Flawed:** The event pipeline from `SDLActivity` -> `onSdlEvent` (JNI) -> `Android::HandleSdlEvent` is functioning. Inside `HandleSdlEvent`, the code correctly identifies that mapping is active (`is_mapping == true`) and populates the `captured_input` variable.

2.  **Stale Input Data:** The `Android::GetCapturedInput()` function was returning the `captured_input` object but **not clearing it**. This meant that if the UI polled for an input and it wasn't the one it wanted, the same stale data would be returned on the next poll, causing an effective lock.

3.  **Device Discovery Failure:** The `Android::GetInputDevices()` function was not advertising the connected SDL gamepads to the rest of the application. It only reported standard Android devices. This is a critical failure, as the UI layer (`InputDialogFragment`) was completely unaware that SDL devices were present and available for mapping.

4.  **Incorrect Mapping Delegation:** The `InputSubsystem` in `input_common/main.cpp` was not correctly delegating mapping operations to the `Android` driver. The `BeginMapping`, `GetNextInput`, and `StopMapping` functions were using a generic `mapping_factory` instead of calling the corresponding methods on the `android` driver instance. This completely bypassed the entire SDL event capture logic.

5.  **Incorrect `ReloadInputDevices` Delegation:** The `InputSubsystem::ReloadInputDevices` function was incorrectly calling `impl->udp_client->ReloadSockets()` on Android, instead of delegating the call to the `android` driver.

**Plan (as per Rule 41):
**
This plan will correct the entire data flow for input mapping on Android.

1.  **`D:/BuildAPK/edenMaster/src/input_common/drivers/android.cpp`:**
    *   **Modify `GetInputDevices()`:** Add logic to iterate through the `sdl_gamepad_map` and create `ParamPackage` entries for each SDL device, advertising them to the UI with the engine name `sdl_android`.
    *   **Modify `GetCapturedInput()`:** Change the function to save the `captured_input` to a temporary variable, clear the member `captured_input`, and then return the temporary variable. This ensures every call gets fresh data.

2.  **`D:/BuildAPK/edenMaster/src/input_common/main.cpp`:**
    *   **Modify `Initialize()`:** Add in Android build `RegisterEngine("sdl_android", android);` call.
    *   **Modify `GetInputEngine()`:** Remove the `|| engine == "sdl_android"` check, simplifying the logic.
    *   **Crucially, modify `BeginMapping`, `GetNextInput`, and `StopMapping`:** Implement the correct delegation to the `impl->android` driver instance when on the `ANDROID` platform. This is the key fix to ensure the native mapping functions are actually called.
    *   **Modify `ReloadInputDevices`:** Implement the correct delegation to `impl->android->ReloadInputDevices()` when on the `ANDROID` platform.

### Code Snippets

#### `D:/BuildAPK/edenMaster/src/input_common/drivers/android.cpp` - `GetInputDevices`

**Before:**
```cpp
std::vector<Common::ParamPackage> Android::GetInputDevices() const {
    std::vector<Common::ParamPackage> devices;
    auto env = Common::Android::GetEnvForThread();
    std::scoped_lock lock{input_devices_mutex};
    for (const auto& [key, value] : input_devices) {
        if (key.port == 100) {
            continue;
        }
        auto name_object = static_cast<jstring>(
                env->CallObjectMethod(value, Common::Android::GetYuzuDeviceGetName()));
        const std::string name =
                fmt::format("{} {}", Common::Android::GetJString(env, name_object), key.port);
        devices.emplace_back(Common::ParamPackage{
                {"engine", GetEngineName()},
                {"display", std::move(name)},
                {"guid", key.guid.RawString()},
                {"port", std::to_string(key.port)},
        });
    }
    return devices;
}
```

**After:**
```cpp
std::vector<Common::ParamPackage> Android::GetInputDevices() const {
    std::vector<Common::ParamPackage> devices;
    auto env = Common::Android::GetEnvForThread();
    std::scoped_lock lock{input_devices_mutex};
    for (const auto& [key, value] : input_devices) {
        if (key.port == 100) {
            continue;
        }
        auto name_object = static_cast<jstring>(
                env->CallObjectMethod(value, Common::Android::GetYuzuDeviceGetName()));
        const std::string name =
                fmt::format("{} {}", Common::Android::GetJString(env, name_object), key.port);
        devices.emplace_back(Common::ParamPackage{
                {"engine", GetEngineName()},
                {"display", std::move(name)},
                {"guid", key.guid.RawString()},
                {"port", std::to_string(key.port)},
        });
    }
    // Add SDL devices
    std::scoped_lock lock_sdl{sdl_gamepad_map_mutex};
    for(const auto& [id, gamepad] : sdl_gamepad_map) {
        const std::string name = SDL_GetGamepadName(gamepad);
        if (name.find("UDP") != std::string::npos) {
            continue;
        }
        const SDL_GUID guid = SDL_GetJoystickGUIDForID(id);
        char guid_str[33];
        SDL_GUIDToString(guid, guid_str, sizeof(guid_str));

        const std::string display_name = fmt::format("{} {}", name, id);
        devices.emplace_back(Common::ParamPackage{
                {"engine", "sdl_android"},
                {"display", display_name},
                {"guid", guid_str},
                {"port", std::to_string(id)},
        });
    }
    return devices;
}
```

#### `D:/BuildAPK/edenMaster/src/input_common/drivers/android.cpp` - `GetCapturedInput`

**Before:**
```cpp
Common::ParamPackage Android::GetCapturedInput() {
    return captured_input;
}
```

**After:**
```cpp
Common::ParamPackage Android::GetCapturedInput() {
    Common::ParamPackage input_ref = captured_input;
    captured_input = {};
    return input_ref;
}
```

#### `D:/BuildAPK/edenMaster/src/input_common/main.cpp` - `ReloadInputDevices`

**Before:**
```cpp
void InputSubsystem::ReloadInputDevices() {
    impl->udp_client->ReloadSockets();
}
```

**After:**
```cpp
void InputSubsystem::ReloadInputDevices() {
#if ANDROID
    impl->android->GetInputDevices();
#else
    impl->udp_client->ReloadSockets();
#endif
}
```
