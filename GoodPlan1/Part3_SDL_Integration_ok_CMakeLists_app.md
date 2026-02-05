This document details the changes required for `src/android/app/src/main/jni/CMakeLists.txt` to link against the pre-built `libSDL3.so` library and include the new `main.cpp`.

```diff
--- a/src/android/app/src/main/jni/CMakeLists.txt
+++ b/src/android/app/src/main/jni/CMakeLists.txt
@@ -3,6 +3,7 @@
 
 add_library(yuzu-android SHARED
     emu_window/emu_window.cpp
     emu_window/emu_window.h
+    main.cpp
     native.cpp
     native.h
     native_config.cpp
@@ -14,7 +15,13 @@
     native_input.cpp
 )
 
-set_property(TARGET yuzu-android PROPERTY IMPORTED_LOCATION ${FFmpeg_LIBRARY_DIR})
-
 target_link_libraries(yuzu-android PRIVATE audio_core common core input_common frontend_common video_core)
-target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log)
+target_link_libraries(yuzu-android PRIVATE android camera2ndk EGL glad jnigraphics log)
+
+add_library(SDL3 SHARED IMPORTED)
+set_target_properties(SDL3 PROPERTIES IMPORTED_LOCATION
+    ${CMAKE_SOURCE_DIR}/app/src/main/jniLibs/${ANDROID_ABI}/libSDL3.so)
+
+target_link_libraries(yuzu-android PRIVATE SDL3)
+
 if (ARCHITECTURE_arm64)
     target_link_libraries(yuzu-android PRIVATE adrenotools)
 endif()
```

**Explanation:**

*   We add an `IMPORTED` shared library target named `SDL3`.
*   We set the `IMPORTED_LOCATION` of this target to the path of our pre-built `libSDL3.so` file, using the `${ANDROID_ABI}` variable to ensure the correct architecture is used.
*   We link our main `yuzu-android` library against `SDL3`.
*   We add `main.cpp` to the list of source files for the `yuzu-android` library.
