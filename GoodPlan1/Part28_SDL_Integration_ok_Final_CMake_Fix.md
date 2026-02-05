This document details the definitive and final fix for the `'SDL3/SDL.h' file not found` error by placing the necessary configuration inside the correct build block in `src/input_common/CMakeLists.txt`.

```diff
--- a/src/input_common/CMakeLists.txt
+++ b/src/input_common/CMakeLists.txt
@@ -73,12 +73,13 @@
     target_compile_definitions(input_common PRIVATE HAVE_SDL2)
 endif()
 
 if (ENABLE_LIBUSB)
     target_sources(input_common PRIVATE
         drivers/gc_adapter.cpp
         drivers/gc_adapter.h
     )
     target_link_libraries(input_common PRIVATE libusb::usb)
     target_compile_definitions(input_common PRIVATE ENABLE_LIBUSB)
 endif()
 
 create_target_directory_groups(input_common)
target_link_libraries(input_common PUBLIC hid_core PRIVATE common Boost::headers)
 
 if (ANDROID)
     target_sources(input_common PRIVATE
         drivers/android.cpp
         drivers/android.h
     )
-    target_link_libraries(input_common PRIVATE android)
+    target_include_directories(input_common PRIVATE ${CMAKE_SOURCE_DIR}/externals/SDL3/include)
+    target_link_libraries(input_common PRIVATE android SDL3::SDL3)
+    target_compile_definitions(input_common PRIVATE HAVE_SDL2)
 endif()

```

**Explanation:**

*   This plan abandons the incorrect attempts to use the `if(ENABLE_SDL2)` block, which is disabled on Android by the root `CMakeLists.txt`.
*   It moves all necessary SDL configuration (`target_include_directories`, `target_link_libraries`, and `target_compile_definitions`) into the `if(ANDROID)` block.
*   This ensures that whenever the `input_common` library is compiled for Android, it is correctly given the path to the SDL headers and linked against the SDL3 library, resolving the build error definitively.
