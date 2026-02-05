# Part 55: Propagate SDL Events to InputDialogFragment

## Explanation

This part implements the native JNI function `onSdlEvent` which was added to `NativeInput.kt`. This function will act as the bridge to pass SDL events from the Android UI layer down to the native `input_common` driver.

The errors from the build log indicate that `SDL_Event` is an unknown type and `InputCommon::Android` is an incomplete type. This is because the necessary headers (`SDL.h` and `android.h`) are not included in `native.cpp`.

This change will:
1.  Include `SDL3/SDL.h` to define `SDL_Event`.
2.  Include `input_common/drivers/android.h` to provide the full definition of the `InputCommon::Android` class.
3.  Implement the `Java_org_yuzu_yuzu_1emu_features_input_NativeInput_onSdlEvent` function to deserialize the SDL event and pass it to the `Android` input driver.

## Code

Here is the implementation of the `onSdlEvent` function to be added to `native.cpp`.

```cpp
void Java_org_yuzu_yuzu_1emu_features_input_NativeInput_onSdlEvent(JNIEnv* env, jclass clazz, jbyteArray j_event) {
    jbyte* event_bytes = env->GetByteArrayElements(j_event, nullptr);
    if (!event_bytes) {
        return;
    }

    SDL_Event event;
    std::memcpy(&event, event_bytes, sizeof(SDL_Event));

    auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
    if (android_driver) {
        android_driver->HandleSdlEvent(event);
    }

    env->ReleaseByteArrayElements(j_event, event_bytes, JNI_ABORT);
}
```
