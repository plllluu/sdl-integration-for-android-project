This document details moving the `SDL3::SDL3` target definition to the root `CMakeLists.txt` to make it globally available.

```diff
---
 a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -345,6 +345,14 @@
     set_target_properties(Qt6::Platform PROPERTIES INTERFACE_COMPILE_FEATURES "")
 endif()
 
+if (ANDROID)
+    add_library(SDL3::SDL3 SHARED IMPORTED GLOBAL)
+    set_target_properties(SDL3::SDL3 PROPERTIES
+        IMPORTED_LOCATION "${CMAKE_SOURCE_DIR}/src/android/app/src/main/jniLibs/${CMAKE_ANDROID_ARCH_ABI}/libSDL3.so"
+        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_SOURCE_DIR}/externals/SDL3/include"
+    )
+endif()
+
 if (NOT (YUZU_USE_BUNDLED_FFMPEG OR YUZU_USE_EXTERNAL_FFMPEG))
     # Use system installed FFmpeg
     find_package(FFmpeg REQUIRED QUIET COMPONENTS ${FFmpeg_COMPONENTS})

```

**Explanation:**

*   Moves the `SDL3::SDL3` definition from the app-level `CMakeLists.txt` to the root `CMakeLists.txt`.
*   Adds the `GLOBAL` keyword to `add_library` to make the target visible in all subdirectories.
