# Part 59: Add Vibration Support for SDL Controllers

## Explanation

**Problem:** SDL controllers connected through the `SdlAndroid` driver do not currently support vibration (rumble).

**Root Cause Analysis:** The `SdlAndroid` driver, created during the refactoring in `Part57`, lacks the necessary logic to handle vibration requests. It does not have a mechanism to receive vibration commands from the input subsystem or a process to send those commands to the physical SDL devices.

**Plan:** This plan will add full vibration support to the `SdlAndroid` driver, mirroring the implementation in the standard `Android` driver. This involves:
1.  Adding a thread-safe queue for vibration requests.
2.  Creating a dedicated thread to process this queue.
3.  Implementing the `SetVibration` and `IsVibrationEnabled` methods.
4.  Updating the `InputSubsystem` to correctly delegate vibration calls to the `SdlAndroid` driver.

## Code Snippets

### 1. `src/input_common/drivers/sdl_android.h`

**Action:** Add the vibration request struct, queue, and thread members to the `SdlAndroid` class definition. Also, add the `SetVibration` and `IsVibrationEnabled` method overrides.

**Before:**
```cpp
class SdlAndroid final : public InputEngine {
public:
    // ...
private:
    // ...
    // Mapping Members
    std::atomic<bool> is_mapping = false;
    Common::ParamPackage captured_input;
};
```

**After:**
```cpp
class SdlAndroid final : public InputEngine {
public:
    // ...
    Common::Input::DriverResult SetVibration(
        const PadIdentifier& identifier, const Common::Input::VibrationStatus& vibration) override;

    bool IsVibrationEnabled(const PadIdentifier& identifier) override;
    // ...
private:
    // ...
    // Mapping Members
    std::atomic<bool> is_mapping = false;
    Common::ParamPackage captured_input;

    struct VibrationRequest {
        PadIdentifier identifier;
        Common::Input::VibrationStatus vibration;
    };

    Common::SPSCQueue<VibrationRequest> vibration_queue;
    std::jthread vibration_thread;
};
```

### 2. `src/input_common/drivers/sdl_android.cpp`

**Action:** Implement the constructor to start the vibration thread, and implement the `SetVibration` and `IsVibrationEnabled` methods.

**Code:**
```cpp
// In the constructor
SdlAndroid::SdlAndroid(std::string input_engine_) : InputEngine(std::move(input_engine_)) {
    vibration_thread = std::jthread([this](std::stop_token token) {
        Common::SetCurrentThreadName("SdlAndroid_Vibration");
        while (!token.stop_requested()) {
            VibrationRequest request = vibration_queue.PopWait(token);
            if (token.stop_requested()) {
                break;
            }
            std::scoped_lock lock{sdl_gamepad_map_mutex};
            if (auto it = sdl_gamepad_map.find(static_cast<SDL_JoystickID>(request.identifier.port));
                it != sdl_gamepad_map.end()) {
                const u16 low_freq = static_cast<u16>(request.vibration.low_amplitude * 0xFFFF);
                const u16 high_freq = static_cast<u16>(request.vibration.high_amplitude * 0xFFFF);
                SDL_RumbleGamepad(it->second, low_freq, high_freq, 100);
            }
        }
    });
}

// New methods
Common::Input::DriverResult SdlAndroid::SetVibration(const PadIdentifier& identifier, const Common::Input::VibrationStatus& vibration) {
    vibration_queue.Push(VibrationRequest{
        .identifier = identifier,
        .vibration = vibration,
    });
    return Common::Input::DriverResult::Success;
}

bool SdlAndroid::IsVibrationEnabled(const PadIdentifier& identifier) {
    std::scoped_lock lock{sdl_gamepad_map_mutex};
    if (auto it = sdl_gamepad_map.find(static_cast<SDL_JoystickID>(identifier.port)); it != sdl_gamepad_map.end()) {
        return SDL_RumbleGamepad(it->second, 0, 0, 0) == 0;
    }
    return false;
}
```

### 3. `src/input_common/main.cpp`

**Action:** Update the `InputSubsystem::Impl::GetInputEngine` function to correctly return the `sdl_android` driver instance, allowing the core to call `SetVibration` on it.

**Rationale:** The core emulation engine retrieves a pointer to the correct driver via `GetInputEngine` and calls the virtual `SetVibration` method on that pointer. This change ensures that for devices with the `sdl_android` engine, the correct driver is returned.

**Before:**
```cpp
// ... in GetInputEngine
#ifdef ANDROID
if (engine == android->GetEngineName()) {
    return android;
}
#endif
// ...
```

**After:**
```cpp
// ... in GetInputEngine
#ifdef ANDROID
if (engine == android->GetEngineName()) {
    return android;
} else if (engine == sdl_android->GetEngineName()) {
    return sdl_android;
}
#endif
// ...
```
