This document details the definitive and final correction to `src/input_common/drivers/android.h` to resolve the `unknown type name` build errors.

```diff
--- a/src/input_common/drivers/android.h
+++ b/src/input_common/drivers/android.h
@@ -10,12 +10,12 @@
 #include "input_common/input_engine.h"
 #include "input_common/main.h"
 
+// SDL3 must be included outside of any namespace
+#include <SDL3/SDL.h>
+
 namespace Common {
-// SDL3
-#include <SDL3/SDL.h>
-
 class ParamPackage;
 }
 
 namespace InputCommon {

```

**Explanation:**

*   This plan corrects the fundamental error of placing an `#include` for a C library inside a C++ namespace.
*   It moves `#include <SDL3/SDL.h>` from within the `Common` namespace to the global scope at the top of the file.
*   This ensures that all SDL types, such as `SDL_JoystickID` and `SDL_Event`, are correctly declared in the global namespace and will be found by the compiler, resolving all related build errors.
