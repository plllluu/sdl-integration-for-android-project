This document details the necessary changes to `src/android/app/src/main/jni/CMakeLists.txt` to find and link the SDL library.

```diff
--- a/src/android/app/src/main/jni/CMakeLists.txt
+++ b/src/android/app/src/main/jni/CMakeLists.txt
@@ -14,7 +14,7 @@
     native_input.cpp
 )
 
-set_property(TARGET yuzu-android PROPERTY IMPORTED_LOCATION ${FFmpeg_LIBRARY_DIR})
-
 target_link_libraries(yuzu-android PRIVATE audio_core common core input_common frontend_common video_core)
-target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log)
+target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log SDL3::SDL3)
 if (ARCHITECTURE_arm64)
     target_link_libraries(yuzu-android PRIVATE adrenotools)
 endif()

```

**Note:** This change assumes that SDL has been added to the project's external dependencies and is available to be found by CMake. The name `SDL3::SDL3` is a placeholder for the actual target name provided by the SDL CMake configuration.
