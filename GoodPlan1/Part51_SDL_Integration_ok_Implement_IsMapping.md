This document provides the definitive plan to fix the `Unresolved reference: isMapping` error by correctly implementing the missing JNI function.

### 1. Rationale

The `InputDialogFragment` requires a way to check if the native input system is currently in "mapping mode." The blueprint accomplishes this with a JNI function, `NativeInput.isMapping()`. This function was called by the fragment but was never declared in the Kotlin `NativeInput` object or implemented in the C++ backend. This plan adds the missing declaration and implementation, completing the feature.

### 2. Implementation Steps

#### 2.1. `src/android/app/src/main/java/org/yuzu/yuzu_emu/features/input/NativeInput.kt` - Declare JNI Function

**Action:** Add the `isMapping` function declaration to the `NativeInput` object.

```diff
--- a/src/android/app/src/main/java/org/yuzu/yuzu_emu/features/input/NativeInput.kt
+++ b/src/android/app/src/main/java/org/yuzu/yuzu_emu/features/input/NativeInput.kt
@@ -124,4 +124,7 @@
     fun getIsConnected(playerIndex: Int): Boolean = getIsConnectedImpl(playerIndex)
 
     fun resetControllerMappings(playerIndex: Int) = resetControllerMappingsImpl(playerIndex)
+
+    fun isMapping(): Boolean = isMappingImpl()
+    private external fun isMappingImpl(): Boolean
 }

```

#### 2.2. `src/android/app/src/main/jni/native_input.cpp` - Implement JNI Function

**Action:** Add the C++ implementation for the `isMapping` JNI function.

```diff
--- a/src/android/app/src/main/jni/native_input.cpp
+++ b/src/android/app/src/main/jni/native_input.cpp
@@ -493,4 +493,9 @@
     }
 }
 
+jboolean Java_org_yuzu_yuzu_1emu_features_input_NativeInput_isMappingImpl(JNIEnv* env, jobject) {
+    return EmulationSession::GetInstance().GetInputSubsystem().IsMapping();
+}
+
 } // extern "C"

```

**Explanation:**

*   The new `isMapping` function in `NativeInput.kt` provides the UI with a safe way to call the native backend.
*   The new `Java_org_yuzu_yuzu_1emu_features_input_NativeInput_isMappingImpl` C++ function bridges the call, delegating it to the `InputSubsystem::IsMapping()` method. This method correctly queries the state of our `Android` driver, returning `true` if it is in mapping mode and `false` otherwise, resolving the `Unresolved reference` error.
