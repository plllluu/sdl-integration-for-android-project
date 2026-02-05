This document details the critical fix to the `SDL_main` loop in `src/android/app/src/main/jni/main.cpp` to correctly handle controller connection events, as per the blueprint.

```diff
--- a/src/android/app/src/main/jni/main.cpp
+++ b/src/android/app/src/main/jni/main.cpp
@@ -5,12 +5,24 @@
 
 std::atomic<bool> g_service_running = true;
 
-// --- JNI Functions ---
 namespace {
 
 void HandleSdlEvent(const SDL_Event& event) {
     auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
     if (!android_driver) {
         return;
     }
 
+    switch(event.type) {
+        case SDL_EVENT_GAMEPAD_ADDED:
+            android_driver->InitSdlGamepad(event.gdevice.which);
+            break;
+        case SDL_EVENT_GAMEPAD_REMOVED:
+            android_driver->CloseSdlGamepad(event.gdevice.which);
+            break;
+        default:
+            android_driver->HandleSdlEvent(event);
+            break;
+    }
 }
 
 void InitializeSdl() {
@@ -22,18 +34,13 @@
     }
 }
 
 } // anonymous namespace
 
-// --- JNI Exports ---
 extern "C" {
 
 JNIEXPORT int SDL_main(int argc, char* argv[]) {
     SDL_SetLogPriority(SDL_LOG_CATEGORY_APPLICATION, SDL_LOG_PRIORITY_INFO);
     InitializeSdl();
 
     while (g_service_running) {
         SDL_Event event;
-        while (SDL_PollEvent(&event)) {
-            HandleSdlEvent(event);
-        }
+        SDL_PollEvent(&event);
+        HandleSdlEvent(event);
         SDL_Delay(8); // ~120 FPS
     }
 

```

**Explanation:**

*   This adds the `switch` statement to the `HandleSdlEvent` function. This logic is critical for managing the lifecycle of controllers.
*   When a new gamepad is added (`SDL_EVENT_GAMEPAD_ADDED`), it now calls the `InitSdlGamepad` function on our `Android` driver.
*   When a gamepad is removed, it calls `CloseSdlGamepad`.
*   All other events are passed to the generic `HandleSdlEvent` for processing button and axis inputs.
*   It also corrects the event polling loop to process one event at a time, which is more robust.
