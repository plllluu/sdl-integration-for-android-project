This document provides the definitive fix for the `no member named 'IsMapping'` error by correctly exposing the `IsMapping` function through the `InputSubsystem` class, as per the blueprint.

### 1. Rationale

The JNI layer needs to query whether the input system is in mapping mode. This is accomplished by calling `InputSubsystem::IsMapping()`. This function was missing from the `InputSubsystem` class declaration and definition, causing a build failure. This plan restores the function by adding it to the public interface in `main.h` and implementing its logic in `main.cpp`, perfectly mirroring the `src-v1old` blueprint.

### 2. Implementation Steps

#### 2.1. `src/input_common/main.h` - Add Public Function Declaration

**Action:** Declare the `IsMapping` function in the `InputSubsystem` class interface.

```diff
--- a/src/input_common/main.h
+++ b/src/input_common/main.h
@@ -171,6 +171,9 @@
     /// Stop polling from all backends.
     void StopMapping() const;
 
+    /// Returns true if the input system is currently mapping an input.
+    [[nodiscard]] bool IsMapping() const;
+
     /// Signals SDL driver for new input events
     void PumpEvents() const;
 

```

#### 2.2. `src/input_common/main.cpp` - Add Function Implementation

**Action:** Implement the `IsMapping` function to delegate the call to the underlying factory.

```diff
--- a/src/input_common/main.cpp
+++ b/src/input_common/main.cpp
@@ -482,6 +482,10 @@
     impl->mapping_factory->StopMapping();
 }
 
+bool InputSubsystem::IsMapping() const {
+    return impl->mapping_factory->IsMapping();
+}
+
 void InputSubsystem::PumpEvents() const {
     impl->PumpEvents();
 }

```

**Explanation:**

*   By adding `IsMapping()` to the public interface of `InputSubsystem` and implementing it to call the underlying `mapping_factory`, we correctly expose the state to the JNI layer.
*   This resolves the `'IsMapping' is not a member of 'InputCommon::InputSubsystem'` error and completes the manual mapping feature.
