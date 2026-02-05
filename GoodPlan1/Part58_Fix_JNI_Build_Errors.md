# Part 58: Fix JNI Build Errors by Correcting Driver Usage

## Explanation

**Problem:** The build is failing with multiple "no member named..." errors in `src/android/app/src/main/jni/main.cpp`.
**Problem:** The build is failing with an "incomplete type" error in `src/android/app/src/main/jni/native.cpp`.

**Root Cause Analysis:** The errors are a direct result of the refactoring to separate the `Android` and `SdlAndroid` drivers. The JNI layer in `jni/main.cpp`, which handles the main SDL event loop, was not updated after the SDL-related methods (`InitSdlGamepad`, `HandleSdlEvent`, `BeginMapping`, etc.) were moved from the `Android` class to the new `SdlAndroid` class. It is still attempting to call these non-existent methods on the wrong driver instance.
**Root Cause Analysis:** The error `member access into incomplete type 'InputCommon::SdlAndroid'` occurs because `native.cpp` is attempting to call the `HandleSdlEvent` method on the `SdlAndroid` class, but it only has a forward declaration of that class from `input_common/main.h`. The compiler does not have the full class definition and therefore does not know about its methods.
**Plan:** This plan will fix the build errors by updating `src/android/app/src/main/jni/main.cpp` to retrieve and use the correct driver (`SdlAndroid`) for all SDL and input mapping operations, as intended by the two-driver architecture from `Part57`.
**Plan:** This plan will fix the build error by including the `sdl_android.h` header in `native.cpp`. This will provide the full definition of the `SdlAndroid` class to the compiler, resolving the incomplete type error.
## Code Snippets

### `src/android/app/src/main/jni/main.cpp`

**Action:** Modify the `HandleSdlEvent` function and the JNI mapping functions to retrieve and use the `SdlAndroid` driver instead of the `Android` driver.

**Before:**
```cpp
// ... in HandleSdlEvent
auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
if (!android_driver) {
    return;
}
switch(event.type) {
    case SDL_EVENT_GAMEPAD_ADDED:
        android_driver->InitSdlGamepad(event.gdevice.which);
        break;
    // ... other cases calling android_driver
}

// ... in Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeStartListeningForInput
auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
if (android_driver) {
    android_driver->BeginMapping();
}

// ... in Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeGetCapturedSdlInput
auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
if (android_driver) {
    Common::ParamPackage params = android_driver->GetCapturedInput();
    // ...
}
```

**After:**
```cpp
// ... in HandleSdlEvent
auto* sdl_driver = EmulationSession::GetInstance().GetInputSubsystem().GetSdlAndroid();
if (!sdl_driver) {
    return;
}
switch(event.type) {
    case SDL_EVENT_GAMEPAD_ADDED:
        sdl_driver->InitSdlGamepad(event.gdevice.which);
        break;
    case SDL_EVENT_GAMEPAD_REMOVED:
        sdl_driver->CloseSdlGamepad(event.gdevice.which);
        break;
    default:
        sdl_driver->HandleSdlEvent(event);
        break;
}

// ... in Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeStartListeningForInput
auto* sdl_driver = EmulationSession::GetInstance().GetInputSubsystem().GetSdlAndroid();
if (sdl_driver) {
    sdl_driver->BeginMapping();
}

// ... in Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeGetCapturedSdlInput
auto* sdl_driver = EmulationSession::GetInstance().GetInputSubsystem().GetSdlAndroid();
if (sdl_driver) {
    Common::ParamPackage params = sdl_driver->GetCapturedInput();
    // ...
}
```

### `src/android/app/src/main/jni/native.cpp`

**Action:** Add an `#include` directive for `input_common/drivers/sdl_android.h`.

**Before:**
```cpp
// ... other includes
#include "input_common/drivers/virtual_amiibo.h"
#include "jni/native.h"
// ...
```

**After:**
```cpp
// ... other includes
#include "input_common/drivers/virtual_amiibo.h"
#include "input_common/drivers/sdl_android.h"
#include "jni/native.h"
// ...
```