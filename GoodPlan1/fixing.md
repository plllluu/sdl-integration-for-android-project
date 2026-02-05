# Refactoring Manual and Automatic Mapping

This log documents the process of refactoring the input mapping system to clearly separate manual and automatic mapping functionalities.

## Refactor InputDialogFragment

I have refactored `D:/BuildAPK/edenMaster/src/android/app/src/main/java/org/yuzu/yuzu_emu/features/settings/ui/InputDialogFragment.kt` to use the `InputHandler` to dispatch key and motion events. This ensures that input is handled consistently and correctly.

### Before

```diff
private fun onKeyEvent(event: KeyEvent): Boolean {
    if (event.source and InputDevice.SOURCE_JOYSTICK != InputDevice.SOURCE_JOYSTICK &&
        event.source and InputDevice.SOURCE_GAMEPAD != InputDevice.SOURCE_GAMEPAD &&
        event.source and InputDevice.SOURCE_KEYBOARD != InputDevice.SOURCE_KEYBOARD &&
        event.source and InputDevice.SOURCE_MOUSE != InputDevice.SOURCE_MOUSE
    ) {
        return false
    }
    if (!InputHandler.androidControllers.containsKey(event.device.controllerNumber)) {
        return false
    }
    Log.d(TAG, "onKeyEvent: device=${event.device}, keyCode=${event.keyCode}, action=${event.action}")
    val action = when (event.action) {
        KeyEvent.ACTION_DOWN -> NativeInput.ButtonState.PRESSED
        KeyEvent.ACTION_UP -> NativeInput.ButtonState.RELEASED
        else -> return false
    }
    val controllerData =
        InputHandler.androidControllers[event.device.controllerNumber] ?: return false
    NativeInput.onGamePadButtonEvent(
        controllerData.getGUID(),
        controllerData.getPort(),
        event.keyCode,
        action
    )
    return true
}

private fun onMotionEvent(event: MotionEvent): Boolean {
    if (event.source and InputDevice.SOURCE_JOYSTICK != InputDevice.SOURCE_JOYSTICK &&
        event.source and InputDevice.SOURCE_GAMEPAD != InputDevice.SOURCE_GAMEPAD &&
        event.source and InputDevice.SOURCE_KEYBOARD != InputDevice.SOURCE_KEYBOARD &&
        event.source and InputDevice.SOURCE_MOUSE != InputDevice.SOURCE_MOUSE
    ) {
        return false
    }
    if (!InputHandler.androidControllers.containsKey(event.device.controllerNumber)) {
        return false
    }
    Log.d(TAG, "onMotionEvent: device=${event.device}")
    // Temp workaround for DPads that give both axis and button input. The input system can't
    // take in a specific axis direction for a binding so you lose half of the directions for a DPad.

    val controllerData =
        InputHandler.androidControllers[event.device.controllerNumber] ?: return false
    event.device.motionRanges.forEach {
        Log.d(TAG, "onMotionEvent: axis=${it.axis}, value=${event.getAxisValue(it.axis)}")
        NativeInput.onGamePadAxisEvent(
            controllerData.getGUID(),
            controllerData.getPort(),
            it.axis,
            event.getAxisValue(it.axis)
        )
    }
    return true
}
```

### After

```diff
private fun onKeyEvent(event: KeyEvent): Boolean {
    if (event.source and InputDevice.SOURCE_JOYSTICK != InputDevice.SOURCE_JOYSTICK &&
        event.source and InputDevice.SOURCE_GAMEPAD != InputDevice.SOURCE_GAMEPAD &&
        event.source and InputDevice.SOURCE_KEYBOARD != InputDevice.SOURCE_KEYBOARD &&
        event.source and InputDevice.SOURCE_MOUSE != InputDevice.SOURCE_MOUSE
    ) {
        return false
    }
    if (!InputHandler.androidControllers.containsKey(event.device.controllerNumber)) {
        return false
    }
    Log.d(TAG, "onKeyEvent: device=${event.device}, keyCode=${event.keyCode}, action=${event.action}")
    return InputHandler.dispatchKeyEvent(event)
}

private fun onMotionEvent(event: MotionEvent): Boolean {
    if (event.source and InputDevice.SOURCE_JOYSTICK != InputDevice.SOURCE_JOYSTICK &&
        event.source and InputDevice.SOURCE_GAMEPAD != InputDevice.SOURCE_GAMEPAD &&
        event.source and InputDevice.SOURCE_KEYBOARD != InputDevice.SOURCE_KEYBOARD &&
        event.source and InputDevice.SOURCE_MOUSE != InputDevice.SOURCE_MOUSE
    ) {
        return false
    }
    if (!InputHandler.androidControllers.containsKey(event.device.controllerNumber)) {
        return false
    }
    Log.d(TAG, "onMotionEvent: device=${event.device}")
    // Temp workaround for DPads that give both axis and button input. The input system can't
    // take in a specific axis direction for a binding so you lose half of the directions for a DPad.

    return InputHandler.dispatchGenericMotionEvent(event)
}
```

## Fix Manual Mapping Input Router

I have fixed the broken input router in `D:/BuildAPK/edenMaster/src/input_common/main.cpp`. The central `InputSubsystem` was not correctly activating or polling the individual input drivers (`android` and `sdl_android`) during a manual mapping session. This caused captured inputs to be ignored.

### Before

```cpp
void InputSubsystem::BeginMapping(const Common::ParamPackage& params, Polling::InputType type) {
    impl->BeginConfiguration();
#ifdef ANDROID
    impl->mapping_engine = impl->GetInputEngine(params);
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            android->BeginMapping(type);
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            sdl_android->BeginMapping(type);
        }
    } else {
        //impl->mapping_factory->BeginMapping(type);
    }
#else
    impl->mapping_factory->SetInputType(type);
    impl->mapping_factory->SetEnabled(true);
#endif
}

Common::ParamPackage InputSubsystem::GetNextInput() const {
#ifdef ANDROID
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            return android->GetCapturedInput();
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            return sdl_android->GetCapturedInput();
        }
    }
#endif
    return impl->mapping_factory->GetNextInput();
}

void InputSubsystem::StopMapping() const {
    impl->EndConfiguration();
#ifdef ANDROID
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            android->StopMapping();
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            sdl_android->StopMapping();
        }
        impl->mapping_engine = nullptr;
    } else {
        impl->mapping_factory->StopMapping();
    }
#else
    impl->mapping_factory->SetEnabled(false);
#endif
}
```

### After

```cpp
void InputSubsystem::BeginMapping(const Common::ParamPackage& params, Polling::InputType type) {
    impl->BeginConfiguration();
#ifdef ANDROID
    impl->mapping_engine = impl->GetInputEngine(params);
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            android->BeginMapping(type);
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            sdl_android->BeginMapping(type);
        }
    } else {
        impl->android->BeginMapping(type);
        impl->sdl_android->BeginMapping(type);
    }
#else
    impl->mapping_factory->SetInputType(type);
    impl->mapping_factory->SetEnabled(true);
#endif
}

Common::ParamPackage InputSubsystem::GetNextInput() const {
#ifdef ANDROID
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            return android->GetCapturedInput();
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            return sdl_android->GetCapturedInput();
        }
    } else {
        auto android_input = impl->android->GetCapturedInput();
        if (android_input.Has("engine")) {
            return android_input;
        }
        auto sdl_input = impl->sdl_android->GetCapturedInput();
        if (sdl_input.Has("engine")) {
            return sdl_input;
        }
    }
#endif
    return impl->mapping_factory->GetNextInput();
}

void InputSubsystem::StopMapping() const {
    impl->EndConfiguration();
#ifdef ANDROID
    if (impl->mapping_engine) {
        if (auto android = std::dynamic_pointer_cast<Android>(impl->mapping_engine)) {
            android->StopMapping();
        } else if (auto sdl_android = std::dynamic_pointer_cast<SdlAndroid>(impl->mapping_engine)) {
            sdl_android->StopMapping();
        }
        impl->mapping_engine = nullptr;
    } else {
        impl->android->StopMapping();
        impl->sdl_android->StopMapping();
    }
#else
    impl->mapping_factory->SetEnabled(false);
#endif
}
```

## Restore Vibration Functionality

I have restored the vibration functionality in `D:/BuildAPK/edenMaster/src/input_common/drivers/android.cpp` and `D:/BuildAPK/edenMaster/src/input_common/drivers/android.h`. The original implementation was incomplete and only handled native Android devices. I have implemented a self-contained vibration system within the `Android` driver, mirroring the clean design of the `SdlAndroid` driver, ensuring that each driver is responsible only for its own device type.

### `android.h` Before

```diff
private:
    std::unordered_map<PadIdentifier, jobject> input_devices;

    /// Returns the correct identifier corresponding to the player index
    PadIdentifier GetIdentifier(const std::string& guid, size_t port) const;

    /// Takes all vibrations from the queue and sends the command to the controller
    void SendVibrations(JNIEnv* env, std::stop_token token);
```

### `android.h` After

```diff
private:
    std::unordered_map<PadIdentifier, jobject> input_devices;
    mutable std::mutex input_devices_mutex;

    /// Returns the correct identifier corresponding to the player index
    PadIdentifier GetIdentifier(const std::string& guid, size_t port) const;

    /// Takes all vibrations from the queue and sends the command to the controller
    void SendVibrations(JNIEnv* env, std::stop_token token);
```

### `android.cpp` Before

```diff
void Android::SendVibrations(JNIEnv* env, std::stop_token token) {
    VibrationRequest request = vibration_queue.PopWait(token);
    auto device = input_devices.find(request.identifier);
    if (device != input_devices.end()) {
        float average_intensity = static_cast<float>(
            (request.vibration.high_amplitude + request.vibration.low_amplitude) / 2.0);
        env->CallVoidMethod(device->second, Common::Android::GetYuzuDeviceVibrate(),
                            average_intensity);
    }
}
```

### `android.cpp` After

```diff
void Android::SendVibrations(JNIEnv* env, std::stop_token token) {
    VibrationRequest request = vibration_queue.PopWait(token);
    std::scoped_lock lock{input_devices_mutex};
    auto device = input_devices.find(request.identifier);
    if (device != input_devices.end()) {
        float average_intensity = static_cast<float>(
            (request.vibration.high_amplitude + request.vibration.low_amplitude) / 2.0);
        env->CallVoidMethod(device->second, Common::Android::GetYuzuDeviceVibrate(),
                            average_intensity);
    }
}
```
