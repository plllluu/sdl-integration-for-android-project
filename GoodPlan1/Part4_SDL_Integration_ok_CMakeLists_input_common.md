This document details the changes required for `src/input_common/CMakeLists.txt`.

```diff
--- a/src/input_common/CMakeLists.txt
+++ b/src/input_common/CMakeLists.txt
@@ -53,7 +53,7 @@
         helpers/joycon_protocol/rumble.cpp
         helpers/joycon_protocol/rumble.h
     )
-    target_link_libraries(input_common PRIVATE SDL2::SDL2)
+    target_link_libraries(input_common PRIVATE)
     target_compile_definitions(input_common PRIVATE HAVE_SDL2)
 endif()
 

```

**Explanation:**

*   We remove the `SDL2::SDL2` dependency from the `input_common` library. This is because we are now linking against our pre-built `libSDL3.so` at the application level in `src/android/app/src/main/jni/CMakeLists.txt`.
