This document details linking `input_common` to our new global `SDL3::SDL3` target.

```diff
---
 a/src/input_common/CMakeLists.txt
+++ b/src/input_common/CMakeLists.txt
@@ -53,7 +53,7 @@
         helpers/joycon_protocol/rumble.cpp
         helpers/joycon_protocol/rumble.h
     )
-    target_link_libraries(input_common PRIVATE)
+    target_link_libraries(input_common PRIVATE SDL3::SDL3)
     target_compile_definitions(input_common PRIVATE HAVE_SDL2)
 endif()
 

```

**Explanation:**

*   Links the `input_common` library to `SDL3::SDL3`, giving it access to the SDL functions and ensuring the compiler knows about the headers.
