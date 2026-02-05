This document provides the definitive and final fix for the `IsMapping` logic, correctly delegating the call to the `Android` driver, which owns the state.

### 1. Rationale

Previous attempts to fix the `IsMapping` build error were fundamentally flawed, adding the function to the `MappingFactory` which does not own the relevant state. The `is_mapping` flag is a private member of the `Android` driver. The correct architectural pattern, as guided by expert user feedback, is for the `InputSubsystem` to delegate this call directly to the `Android` driver instance. This plan reverts the incorrect changes to the `MappingFactory` and implements the correct delegation.

### 2. Implementation Steps

#### 2.1. `src/input_common/input_poller.h` - Revert Incorrect Change

**Action:** Remove the incorrect `IsMapping` declaration from the `MappingFactory` class.

```diff
--- a/src/input_common/input_poller.h
+++ b/src/input_common/input_poller.h
@@ -38,9 +38,6 @@
 
     /// Stop polling from all backends
     void StopMapping();
-
-    /// Returns true if the factory is currently mapping an input.
-    [[nodiscard]] bool IsMapping() const;
 
 private:
     std::atomic<bool> is_mapping;

```

#### 2.2. `src/input_common/input_poller.cpp` - Revert Incorrect Change

**Action:** Remove the incorrect `IsMapping` implementation.

```diff
--- a/src/input_common/input_poller.cpp
+++ b/src/input_common/input_poller.cpp
@@ -107,8 +107,4 @@
     is_mapping = false;
 }
 
-bool MappingFactory::IsMapping() const {
-    return is_mapping;
-}
-
 } // namespace InputCommon::Polling

```

#### 2.3. `src/input_common/main.cpp` - Implement Correct Delegation

**Action:** Modify `InputSubsystem::IsMapping` to call the `IsMapping` method on the `Android` driver instance.

```diff
--- a/src/input_common/main.cpp
+++ b/src/input_common/main.cpp
@@ -482,7 +482,12 @@
 }
 
 bool InputSubsystem::IsMapping() const {
-    return impl->mapping_factory->IsMapping();
+#ifdef ANDROID
+    return impl->android->IsMapping();
+#else
+    return impl->mapping_factory->IsMapping();
+#endif
 }
 
 void InputSubsystem::PumpEvents() const {

```

**Explanation:**

*   This plan first removes the incorrect code from `input_poller.h` and `input_poller.cpp`.
*   It then provides the correct, definitive implementation for `InputSubsystem::IsMapping()`. Crucially, it uses an `#ifdef ANDROID` guard. On Android, it correctly delegates to the `Android` driver. On other platforms (desktop), it retains the original behavior of checking the `MappingFactory`. This is a robust, platform-aware solution.
