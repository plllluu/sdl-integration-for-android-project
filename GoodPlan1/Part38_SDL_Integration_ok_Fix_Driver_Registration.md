This document details the critical fix to `src/input_common/main.cpp` to correctly register the `Android` driver for SDL-based devices.

```diff
--- a/src/input_common/main.cpp
+++ b/src/input_common/main.cpp
@@ -79,6 +79,7 @@
         RegisterEngine("camera", camera);
 #ifdef ANDROID
         RegisterEngine("android", android);
+        RegisterEngine("sdl_android", android);
 #endif
         RegisterEngine("virtual_amiibo", virtual_amiibo);
         RegisterEngine("virtual_gamepad", virtual_gamepad);

```

**Explanation:**

*   This adds the line `RegisterEngine("sdl_android", android);`.
*   The blueprint uses the engine name `sdl_android` for devices discovered via the native SDL backend. By registering the `Android` driver with this second name, the input subsystem will now know how to correctly create and manage these devices, resolving the `Unknown engine name: sdl_android` error.
