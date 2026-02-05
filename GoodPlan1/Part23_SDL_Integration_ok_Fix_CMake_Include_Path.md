This document details the definitive fix for the `'SDL3/SDL.h' file not found` error by directly providing the include path to the `input_common` target.

```diff
--- a/src/input_common/CMakeLists.txt
+++ b/src/input_common/CMakeLists.txt
@@ -53,6 +53,7 @@
         helpers/joycon_protocol/rumble.cpp
         helpers/joycon_protocol/rumble.h
     )
+    target_include_directories(input_common PRIVATE ${CMAKE_SOURCE_DIR}/externals/SDL3/include)
     target_link_libraries(input_common PRIVATE SDL3::SDL3)
     target_compile_definitions(input_common PRIVATE HAVE_SDL2)
 endif()

```

**Explanation:**

*   Adds `target_include_directories` to the `input_common` library definition.
*   This explicitly tells the compiler to look in `${CMAKE_SOURCE_DIR}/externals/SDL3/include` when compiling the `input_common` sources, directly resolving the path issue for `<SDL3/SDL.h>`.
