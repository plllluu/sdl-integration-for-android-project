This document details the definitive fix for the `'SDL_hidapi.h' file not found` error by preventing the `joycon` and `sdl_driver` sources from being compiled on Android.

```diff
--- a/src/input_common/CMakeLists.txt
+++ b/src/input_common/CMakeLists.txt
@@ -35,7 +35,7 @@
     )
 endif()
 
-if (ENABLE_SDL2)
+if (ENABLE_SDL2 AND NOT ANDROID)
     target_sources(input_common PRIVATE
         drivers/joycon.cpp
         drivers/joycon.h

```

**Explanation:**

*   This changes the condition for compiling the generic SDL drivers from `if (ENABLE_SDL2)` to `if (ENABLE_SDL2 AND NOT ANDROID)`.
*   This correctly reflects the project blueprint's architecture: on Android, all SDL logic is self-contained within the `Android` driver, and the separate `joycon` and `sdl_driver` are not used.
*   By preventing these files from being compiled on Android, we completely avoid the `'SDL_hidapi.h' not found` error, as the file that includes it is no longer part of the build.
