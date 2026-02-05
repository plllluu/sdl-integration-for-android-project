This document details cleaning up the now-redundant code from the app-level `CMakeLists.txt`.

```diff
---
 a/src/android/app/src/main/jni/CMakeLists.txt
+++ b/src/android/app/src/main/jni/CMakeLists.txt
@@ -15,13 +15,7 @@
 )
 
 target_link_libraries(yuzu-android PRIVATE audio_core common core input_common frontend_common video_core)
-target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log SDL3::SDL3)
-
-add_library(SDL3::SDL3 SHARED IMPORTED)
-set_target_properties(SDL3::SDL3 PROPERTIES
-    IMPORTED_LOCATION "${CMAKE_SOURCE_DIR}/src/android/app/src/main/jniLibs/${CMAKE_ANDROID_ARCH_ABI}/libSDL3.so"
-    INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_SOURCE_DIR}/externals/SDL3/include"
-)
+target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log)
 
 if (ARCHITECTURE_arm64)
     target_link_libraries(yuzu-android PRIVATE adrenotools)

```

**Explanation:**

*   Removes the `SDL3::SDL3` definition and the explicit link from the app-level CMake file, as this is now handled globally.
