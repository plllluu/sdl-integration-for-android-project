This document provides the definitive and final fix for the `'SDL3/SDL.h' file not found` error by adding the necessary include directory directly to the final `yuzu-android` shared library target.

```diff
--- a/src/android/app/src/main/jni/CMakeLists.txt
+++ b/src/android/app/src/main/jni/CMakeLists.txt
@@ -15,7 +15,9 @@
 )
 
 target_link_libraries(yuzu-android PRIVATE audio_core common core input_common frontend_common video_core)
target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log)
+target_include_directories(yuzu-android PRIVATE ${CMAKE_SOURCE_DIR}/externals/SDL3/include)
+
 
 if (ARCHITECTURE_arm64)
     target_link_libraries(yuzu-android PRIVATE adrenotools)

```

**Explanation:**

*   This plan abandons all previous attempts to fix this issue through dependency propagation, which have failed due to the complexities of linking static libraries.
*   It adds `target_include_directories(yuzu-android PRIVATE ${CMAKE_SOURCE_DIR}/externals/SDL3/include)`.
*   This directly and explicitly tells the compiler where to find the SDL headers when compiling the sources for the final `yuzu-android` shared library, including `native_input.cpp` which includes the problematic header. This is the most direct and robust solution to the error.
