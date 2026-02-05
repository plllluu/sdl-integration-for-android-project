This file details the changes required for `src/android/app/src/main/jni/native_input.cpp` to forward SDL events from the Android UI layer to the input driver.

```diff
--- a/src/android/app/src/main/jni/native_input.cpp
+++ b/src/android/app/src/main/jni/native_input.cpp
@@ -8,6 +8,7 @@
 #include <android/keycodes.h>
 #include <android/log.h>
 #include <input_common/drivers/android.h>
+#include <input_common/main.h>
 #include "native.h"
 #include "native_input.h"
 
@@ -52,3 +53,11 @@
     input_driver->SetMotionState(guid, port, timestamp, gyro_x, gyro_y, gyro_z, accel_x, accel_y,
                                  accel_z);
 }
+
+extern "C" JNIEXPORT void JNICALL Java_org_yuzu_yuzu_1emu_NativeLibrary_onSdlEvent(
+    JNIEnv* env, jclass, jobject event) {
+    if (g_input_driver) {
+        const auto input_driver = static_cast<InputCommon::Android*>(g_input_driver->get());
+        input_driver->HandleSdlEvent(*static_cast<const SDL_Event*>(env->GetDirectBufferAddress(event)));
+    }
+}
```

**Explanation:**

*   A new JNI function, `onSdlEvent`, is added. This function will be called from the Android frontend whenever an SDL event occurs.
*   It retrieves the native `SDL_Event` from the Java `DirectBuffer` and passes it to the `Android::HandleSdlEvent` function, which we will implement as part of the `android.cpp` changes.
