This document details the necessary refinement to `src/android/app/src/main/jni/CMakeLists.txt` to correctly link the pre-built `libSDL3.so` library and its headers. This supersedes the previous CMake changes for this file.

```diff
--- a/src/android/app/src/main/jni/CMakeLists.txt
+++ b/src/android/app/src/main/jni/CMakeLists.txt
@@ -15,13 +15,13 @@
 )
 
 target_link_libraries(yuzu-android PRIVATE audio_core common core input_common frontend_common video_core)
-target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log)
 
-add_library(SDL3 SHARED IMPORTED)
-set_target_properties(SDL3 PROPERTIES IMPORTED_LOCATION
-    ${CMAKE_SOURCE_DIR}/app/src/main/jniLibs/${ANDROID_ABI}/libSDL3.so)
+add_library(SDL3::SDL3 SHARED IMPORTED)
+set_target_properties(SDL3::SDL3 PROPERTIES
+    IMPORTED_LOCATION "${CMAKE_SOURCE_DIR}/src/android/app/src/main/jniLibs/${CMAKE_ANDROID_ARCH_ABI}/libSDL3.so"
+    INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_SOURCE_DIR}/externals/SDL3/include"
+)
 
-target_link_libraries(yuzu-android PRIVATE SDL3)
+target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log SDL3::SDL3)
 
 if (ARCHITECTURE_arm64)
     target_link_libraries(yuzu-android PRIVATE adrenotools)

```

**Explanation:**

*   Replaces the previous `add_library(SDL3 ...)` block with the more robust `add_library(SDL3::SDL3 ...)`.
*   Crucially, it adds the `INTERFACE_INCLUDE_DIRECTORIES` property to tell the compiler where to find the SDL headers.
*   It links `yuzu-android` against the new `SDL3::SDL3` target in a single `target_link_libraries` call for better readability.
