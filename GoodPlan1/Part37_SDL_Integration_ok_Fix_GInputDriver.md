This document details the definitive fix for the `use of undeclared identifier 'g_input_driver'` error by using the correct method to access the input driver in `native_input.cpp`.

```diff
--- a/src/android/app/src/main/jni/native_input.cpp
+++ b/src/android/app/src/main/jni/native_input.cpp
@@ -215,9 +215,9 @@
 }
 
 extern "C" JNIEXPORT void JNICALL Java_org_yuzu_yuzu_1emu_NativeLibrary_onSdlEvent(
     JNIEnv* env, jclass, jobject event) {
-    if (g_input_driver) {
-        const auto input_driver = static_cast<InputCommon::Android*>(g_input_driver->get());
+    auto* input_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
+    if (input_driver) {
         input_driver->HandleSdlEvent(*static_cast<const SDL_Event*>(env->GetDirectBufferAddress(event)));
     }
 }

```

**Explanation:**

*   This fix removes the use of the non-existent global variable `g_input_driver`.
*   It replaces it with the project's standard method for accessing the Android input driver singleton: `EmulationSession::GetInstance().GetInputSubsystem().GetAndroid()`.
*   This is the correct pattern used by all other input-related JNI functions in this file and will resolve the final build error.
