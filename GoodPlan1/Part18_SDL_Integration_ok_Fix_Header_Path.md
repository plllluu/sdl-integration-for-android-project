This document details the fix for the `'SDL_events.h' file not found` error by correcting the include path in `android.h`.

```diff
--- a/src/input_common/drivers/android.h
+++ b/src/input_common/drivers/android.h
@@ -1,8 +1,5 @@
 #pragma once
 
 #include <jni.h>
-#include <SDL_events.h>
-#include <SDL_gamepad.h>
 #include <memory>
 #include <mutex>
 #include <string>
@@ -15,6 +12,9 @@
 #include "input_common/main.h"
 
 namespace Common {
+// SDL3
+#include <SDL3/SDL.h>
+
 class ParamPackage;
 }
 

```

**Explanation:**

*   Replaces the incorrect includes for `<SDL_events.h>` and `<SDL_gamepad.h>` with the single, correct include `<SDL3/SDL.h>`, which matches the blueprint and the include directory we are about to configure.
