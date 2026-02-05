# Part 57: Refactor Android Input Drivers

## Overview

This plan details the systematic refactoring of the Android input handling system. The current implementation mixes standard Android input events and SDL events within a single driver (`android.cpp`/`.h`). This creates tight coupling and potential for conflicts.

The goal is to separate this logic into two distinct, specialized drivers:
1.  **`android` driver:** Will be responsible *only* for standard Android `KeyEvent` and `MotionEvent` handling.
2.  **`sdl_android` driver:** A new driver responsible *only* for events originating from the SDL library.

This refactoring will significantly improve the modularity and maintainability of the input subsystem.

--- 

## Part 1: Create `sdl_android.h` Header File

**File:** `D:/BuildAPK/edenMaster/src/input_common/drivers/sdl_android.h` (New File)

**Action:** Create the header file for the new `SdlAndroid` driver. Its class definition will be derived from the SDL-specific parts of the old `android.h`.

**Rationale:** This establishes the public interface for the new driver. It will handle SDL events and the input mapping process, as mapping is triggered by SDL controller inputs.

**Code (New File):**
```cpp
#pragma once

#include <memory>
#include <mutex>
#include <string>
#include <unordered_map>

#include <SDL3/SDL.h>

#include "common/common_types.h"
#include "common/uuid.h"
#include "input_common/input_engine.h"

namespace Common {
class ParamPackage;
}

namespace InputCommon {

class SdlAndroid final : public InputEngine {
public:
    explicit SdlAndroid(std::string input_engine_);
    ~SdlAndroid() override;

    void InitSdlGamepad(SDL_JoystickID instance_id);
    void CloseSdlGamepad(SDL_JoystickID instance_id);

    void HandleSdlEvent(const SDL_Event& event);

    std::vector<Common::ParamPackage> GetInputDevices() const override;

    void BeginMapping();
    void StopMapping();
    bool IsMapping() const;
    Common::ParamPackage GetCapturedInput();

    AnalogMapping GetAnalogMappingForDevice(const Common::ParamPackage& params) override;
    ButtonMapping GetButtonMappingForDevice(const Common::ParamPackage& params) override;
    Common::Input::ButtonNames GetUIName(const Common::ParamPackage& params) const override;

private:
    PadIdentifier GetSdlIdentifier(SDL_JoystickID instance_id) const;
    Common::ParamPackage BuildAnalogParamPackageForButton(PadIdentifier identifier, s32 axis,
                                                           bool invert) const;
    Common::ParamPackage BuildButtonParamPackageForButton(PadIdentifier identifier,
                                                           s32 button) const;
    Common::ParamPackage BuildParamPackageForAnalog(PadIdentifier identifier, int axis_x,
                                                     int axis_y) const;

    // SDL Members
    mutable std::mutex sdl_gamepad_map_mutex;
    std::unordered_map<SDL_JoystickID, SDL_Gamepad*> sdl_gamepad_map;

    // Mapping Members
    std::atomic<bool> is_mapping = false;
    Common::ParamPackage captured_input;
};

} // namespace InputCommon
```

---

## Part 2: Create `sdl_android.cpp` Implementation File

**File:** `D:/BuildAPK/edenMaster/src/input_common/drivers/sdl_android.cpp` (New File)

**Action:** Create the implementation file for `SdlAndroid`. The logic will be moved from `android.cpp`.

**Rationale:** This file will contain the complete implementation for SDL event handling, device discovery, and input mapping.

**Code (New File):**
```cpp
#include <SDL3/SDL.h>
#include "common/android/android_common.h"
#include "common/logging/log.h"
#include "common/settings_input.h"
#include "input_common/drivers/sdl_android.h"

namespace InputCommon {

namespace {
// Normalize to the [-1, 1] range
constexpr float AXIS_MAX = 32767.0f;
} // Anonymous namespace

SdlAndroid::SdlAndroid(std::string input_engine_) : InputEngine(std::move(input_engine_)) {}

SdlAndroid::~SdlAndroid() = default;

PadIdentifier SdlAndroid::GetSdlIdentifier(SDL_JoystickID instance_id) const {
    const SDL_GUID guid = SDL_GetJoystickGUIDForID(instance_id);
    std::array<u8, 16> data{};
    std::memcpy(data.data(), guid.data, sizeof(data));
    return {
        .guid = Common::UUID{data},
        .port = static_cast<std::size_t>(instance_id),
        .pad = 0,
    };
}

void SdlAndroid::InitSdlGamepad(SDL_JoystickID instance_id) {
    SDL_Gamepad* gamepad = SDL_OpenGamepad(instance_id);
    if (!gamepad) {
        LOG_ERROR(Input, "Could not open gamepad: {}", SDL_GetError());
        return;
    }
    std::scoped_lock lock{sdl_gamepad_map_mutex};
    sdl_gamepad_map[instance_id] = gamepad;
    PreSetController(GetSdlIdentifier(instance_id));
    LOG_INFO(Input, "Initialized gamepad: {}", SDL_GetGamepadName(gamepad));
}

void SdlAndroid::CloseSdlGamepad(SDL_JoystickID instance_id) {
    std::scoped_lock lock{sdl_gamepad_map_mutex};
    if (auto it = sdl_gamepad_map.find(instance_id); it != sdl_gamepad_map.end()) {
        LOG_INFO(Input, "Closing gamepad: {}", SDL_GetGamepadName(it->second));
        SDL_CloseGamepad(it->second);
        sdl_gamepad_map.erase(it);
    }
}

void SdlAndroid::HandleSdlEvent(const SDL_Event& event) {
    if (is_mapping) {
        switch (event.type) {
        case SDL_EVENT_GAMEPAD_BUTTON_DOWN: {
            const auto& gbutton = event.gbutton;
            const auto id = gbutton.which;
            std::scoped_lock lock{sdl_gamepad_map_mutex};
            const auto it = sdl_gamepad_map.find(id);
            std::string name;
            if (it != sdl_gamepad_map.end()) {
                name = SDL_GetGamepadName(it->second);
            }
            captured_input = BuildButtonParamPackageForButton(GetSdlIdentifier(id), gbutton.button);
            captured_input.Set("engine", GetEngineName());
            captured_input.Set("display", fmt::format("{} {}", name, id));
            StopMapping();
            break;
        }
        case SDL_EVENT_GAMEPAD_AXIS_MOTION: {
            if (static_cast<float>(std::abs(event.gaxis.value)) > AXIS_MAX * 0.5f) {
                const auto& gaxis = event.gaxis;
                const auto id = gaxis.which;
                std::scoped_lock lock{sdl_gamepad_map_mutex};
                const auto it = sdl_gamepad_map.find(id);
                std::string name;
                if (it != sdl_gamepad_map.end()) {
                    name = SDL_GetGamepadName(it->second);
                }
                captured_input = BuildAnalogParamPackageForButton(GetSdlIdentifier(id), gaxis.axis, gaxis.value < 0);
                captured_input.Set("engine", GetEngineName());
                captured_input.Set("display", fmt::format("{} {}", name, id));
                StopMapping();
            }
            break;
        }
        default:
            break;
        }
        return;
    }

    switch (event.type) {
    case SDL_EVENT_GAMEPAD_BUTTON_DOWN:
    case SDL_EVENT_GAMEPAD_BUTTON_UP: {
        const auto& gbutton = event.gbutton;
        const auto identifier = GetSdlIdentifier(gbutton.which);
        SetButton(identifier, gbutton.button, event.type == SDL_EVENT_GAMEPAD_BUTTON_DOWN);
        break;
    }
    case SDL_EVENT_GAMEPAD_AXIS_MOTION: {
        const auto& gaxis = event.gaxis;
        const auto identifier = GetSdlIdentifier(gaxis.which);
        const float value = static_cast<float>(gaxis.value) / AXIS_MAX;
        SetAxis(identifier, gaxis.axis, value);
        break;
    }
    default:
        break;
    }
}

std::vector<Common::ParamPackage> SdlAndroid::GetInputDevices() const {
    std::vector<Common::ParamPackage> devices;
    std::scoped_lock lock_sdl{sdl_gamepad_map_mutex};
    for (const auto& [id, gamepad] : sdl_gamepad_map) {
        const std::string name = SDL_GetGamepadName(gamepad);
        if (name.find("UDP") != std::string::npos) {
            continue;
        }
        const SDL_GUID guid = SDL_GetJoystickGUIDForID(id);
        char guid_str[33];
        SDL_GUIDToString(guid, guid_str, sizeof(guid_str));

        const std::string display_name = fmt::format("{} {}", name, id);
        devices.emplace_back(Common::ParamPackage{{"engine", GetEngineName()},
                                                {"display", display_name},
                                                {"guid", guid_str},
                                                {"port", std::to_string(id)}});
    }
    return devices;
}

void SdlAndroid::BeginMapping() {
    is_mapping = true;
    captured_input = {};
}

void SdlAndroid::StopMapping() {
    is_mapping = false;
}

bool SdlAndroid::IsMapping() const {
    return is_mapping;
}

Common::ParamPackage SdlAndroid::GetCapturedInput() {
    Common::ParamPackage input_ref = captured_input;
    captured_input = {};
    return input_ref;
}

Common::ParamPackage SdlAndroid::BuildParamPackageForAnalog(PadIdentifier identifier, int axis_x,
                                                          int axis_y) const {
    Common::ParamPackage params;
    params.Set("engine", GetEngineName());
    params.Set("port", static_cast<int>(identifier.port));
    params.Set("guid", identifier.guid.RawString());
    params.Set("axis_x", axis_x);
    params.Set("axis_y", axis_y);
    params.Set("offset_x", 0);
    params.Set("offset_y", 0);
    params.Set("invert_x", "+");
    params.Set("invert_y", "-");
    return params;
}

Common::ParamPackage SdlAndroid::BuildAnalogParamPackageForButton(PadIdentifier identifier, s32 axis, bool invert) const {
    Common::ParamPackage params{};
    params.Set("engine", GetEngineName());
    params.Set("port", static_cast<int>(identifier.port));
    params.Set("guid", identifier.guid.RawString());
    params.Set("axis", axis);
    params.Set("threshold", "0.5");
    params.Set("invert", invert ? "-" : "+");
    return params;
}

Common::ParamPackage SdlAndroid::BuildButtonParamPackageForButton(PadIdentifier identifier, s32 button) const {
    Common::ParamPackage params{};
    params.Set("engine", GetEngineName());
    params.Set("port", static_cast<int>(identifier.port));
    params.Set("guid", identifier.guid.RawString());
    params.Set("button", button);
    return params;
}

AnalogMapping SdlAndroid::GetAnalogMappingForDevice(const Common::ParamPackage& params) {
    AnalogMapping mapping = {};
    const auto id = static_cast<SDL_JoystickID>(std::stoi(params.Get("port", "0")));
    const auto identifier = GetSdlIdentifier(id);
    auto build_analog = [&](int axis_x, int axis_y) {
        return BuildParamPackageForAnalog(identifier, axis_x, axis_y);
    };

    mapping.insert_or_assign(Settings::NativeAnalog::LStick, build_analog(SDL_GAMEPAD_AXIS_LEFTX, SDL_GAMEPAD_AXIS_LEFTY));
    mapping.insert_or_assign(Settings::NativeAnalog::RStick, build_analog(SDL_GAMEPAD_AXIS_RIGHTX, SDL_GAMEPAD_AXIS_RIGHTY));
    return mapping;
}

ButtonMapping SdlAndroid::GetButtonMappingForDevice(const Common::ParamPackage& params) {
    ButtonMapping mapping = {};
    const auto id = static_cast<SDL_JoystickID>(std::stoi(params.Get("port", "0")));
    const auto identifier = GetSdlIdentifier(id);
    auto build_button = [&](s32 button) {
        return BuildButtonParamPackageForButton(identifier, button);
    };
    auto build_analog_button = [&](s32 axis, bool invert) {
        return BuildAnalogParamPackageForButton(identifier, axis, invert);
    };
    mapping.insert_or_assign(Settings::NativeButton::A, build_button(SDL_GAMEPAD_BUTTON_SOUTH));
    mapping.insert_or_assign(Settings::NativeButton::B, build_button(SDL_GAMEPAD_BUTTON_EAST));
    mapping.insert_or_assign(Settings::NativeButton::X, build_button(SDL_GAMEPAD_BUTTON_WEST));
    mapping.insert_or_assign(Settings::NativeButton::Y, build_button(SDL_GAMEPAD_BUTTON_NORTH));
    mapping.insert_or_assign(Settings::NativeButton::LStick, build_button(SDL_GAMEPAD_BUTTON_LEFT_STICK));
    mapping.insert_or_assign(Settings::NativeButton::RStick, build_button(SDL_GAMEPAD_BUTTON_RIGHT_STICK));
    mapping.insert_or_assign(Settings::NativeButton::L, build_button(SDL_GAMEPAD_BUTTON_LEFT_SHOULDER));
    mapping.insert_or_assign(Settings::NativeButton::R, build_button(SDL_GAMEPAD_BUTTON_RIGHT_SHOULDER));
    mapping.insert_or_assign(Settings::NativeButton::ZL, build_analog_button(SDL_GAMEPAD_AXIS_LEFT_TRIGGER, false));
    mapping.insert_or_assign(Settings::NativeButton::ZR, build_analog_button(SDL_GAMEPAD_AXIS_RIGHT_TRIGGER, false));
    mapping.insert_or_assign(Settings::NativeButton::Plus, build_button(SDL_GAMEPAD_BUTTON_START));
    mapping.insert_or_assign(Settings::NativeButton::Minus, build_button(SDL_GAMEPAD_BUTTON_BACK));
    mapping.insert_or_assign(Settings::NativeButton::DUp, build_button(SDL_GAMEPAD_BUTTON_DPAD_UP));
    mapping.insert_or_assign(Settings::NativeButton::DDown, build_button(SDL_GAMEPAD_BUTTON_DPAD_DOWN));
    mapping.insert_or_assign(Settings::NativeButton::DLeft, build_button(SDL_GAMEPAD_BUTTON_DPAD_LEFT));
    mapping.insert_or_assign(Settings::NativeButton::DRight, build_button(SDL_GAMEPAD_BUTTON_DPAD_RIGHT));
    return mapping;
}

Common::Input::ButtonNames SdlAndroid::GetUIName(const Common::ParamPackage& params) const {
    return Common::Input::ButtonNames::Value;
}

} // namespace InputCommon
```

---

## Part 3: Refactor `android.h`

**File:** `D:/BuildAPK/edenMaster/src/input_common/drivers/android.h`

**Action:** Remove all SDL and mapping-related code.

**Rationale:** This leaves the driver with a single responsibility: handling standard Android input. It will no longer be aware of SDL or the mapping process.

**Code:**

**Before:**
```diff
#pragma once

#include <jni.h>
#include <memory>
#include <mutex>
#include <string>
#include <thread>
#include <vector>
#include <set>
#include <unordered_map>
#include <common/threadsafe_queue.h>
#include "common/android/android_common.h"
#include "common/common_types.h"
#include "common/uuid.h"
#include "input_common/input_engine.h"
#include "input_common/main.h"

// SDL3 must be included outside of any namespace
#include <SDL3/SDL.h>

// ...
class Android final : public InputEngine {
public:
    // ...
    void InitSdlGamepad(SDL_JoystickID instance_id);
    void CloseSdlGamepad(SDL_JoystickID instance_id);
    void HandleSdlEvent(const SDL_Event& event);
    // ...
    void BeginMapping();
    void StopMapping();
    bool IsMapping() const;
    Common::ParamPackage GetCapturedInput();
    // ...
private:
    // ...
    // SDL Members
    mutable std::mutex sdl_gamepad_map_mutex;
    std::unordered_map<SDL_JoystickID, SDL_Gamepad*> sdl_gamepad_map;

    // Mapping Members
    std::atomic<bool> is_mapping = false;
    Common::ParamPackage captured_input;
    // ...
};
```

**After:**
```diff
#pragma once

#include <jni.h>
#include <memory>
#include <mutex>
#include <string>
#include <thread>
#include <vector>
#include <set>
#include <unordered_map>
#include <common/threadsafe_queue.h>
#include "common/android/android_common.h"
#include "common/common_types.h"
#include "common/uuid.h"
#include "input_common/input_engine.h"

namespace Common {
class ParamPackage;
}

namespace InputCommon {

class Android final : public InputEngine {
public:
    explicit Android(std::string input_engine_);
    ~Android() override;

    void RegisterController(jobject j_input_device);
    void UnregisterController(const PadIdentifier& identifier);

    void SetButtonState(std::string guid, size_t port, int button_id, bool value);
    void SetAxisPosition(std::string guid, size_t port, int axis_id, float value);
    void SetMotionState(std::string guid, size_t port, u64 delta_timestamp, float gyro_x,
                        float gyro_y, float gyro_z, float accel_x, float accel_y, float accel_z);

    Common::Input::DriverResult SetVibration(const PadIdentifier& identifier,
                                             const Common::Input::VibrationStatus& vibration) override;

    bool IsVibrationEnabled(const PadIdentifier& identifier) override;
    std::vector<Common::ParamPackage> GetInputDevices() const override;
    AnalogMapping GetAnalogMappingForDevice(const Common::ParamPackage& params) override;
    ButtonMapping GetButtonMappingForDevice(const Common::ParamPackage& params) override;
    Common::Input::ButtonNames GetUIName(const Common::ParamPackage& params) const override;

private:
    PadIdentifier GetIdentifier(const std::string& guid, size_t port) const;
    void SendVibrations(JNIEnv* env, std::stop_token token);
    std::set<s32> GetDeviceAxes(JNIEnv* env, jobject& j_device) const;
    bool MatchVID(Common::UUID device, const std::vector<std::string>& vids) const;

    mutable std::mutex input_devices_mutex;
    std::unordered_map<PadIdentifier, jobject> input_devices;

    // ... (rest of the constants and vibration struct)
};

}
```

---

## Part 4: Refactor `android.cpp`

**File:** `D:/BuildAPK/edenMaster/src/input_common/drivers/android.cpp`

**Action:** Remove all SDL and mapping-related logic from the implementation.

**Rationale:** This completes the separation, leaving this file with only the logic required to handle standard Android inputs passed down from the JNI layer.

**Code:** The changes involve deleting large blocks of code. The most significant changes are:
-   Deleting `InitSdlGamepad`, `CloseSdlGamepad`, and `HandleSdlEvent`.
-   Deleting `BeginMapping`, `StopMapping`, `IsMapping`, and `GetCapturedInput`.
-   Removing the SDL-related loop from `GetInputDevices`.
-   Removing the `engine == "sdl_android"` checks from `GetAnalogMappingForDevice` and `GetButtonMappingForDevice`.

---

## Part 5: Update `input_common/CMakeLists.txt`

**File:** `D:/BuildAPK/edenMaster/src/input_common/CMakeLists.txt`

**Action:** Add the new `sdl_android.cpp` and `sdl_android.h` files to the library sources.

**Rationale:** This ensures the new driver is compiled and linked into the `input_common` library.

**Code:**

**Before:**
```cmake
# ...
if (ANDROID)
    target_sources(input_common PRIVATE
        drivers/android.cpp
        drivers/android.h
    )
# ...
```

**After:**
```cmake
# ...
if (ANDROID)
    target_sources(input_common PRIVATE
        drivers/android.cpp
        drivers/android.h
        drivers/sdl_android.cpp
        drivers/sdl_android.h
    )
# ...
```

---

## Part 6: Integrate in `input_common/main.cpp`

**File:** `D:/BuildAPK/edenMaster/src/input_common/main.cpp`

**Action:** Register the new `SdlAndroid` driver and delegate all mapping-related calls to it.

**Rationale:** This correctly wires the new driver into the input subsystem and fixes a pre-existing bug where `ReloadInputDevices` was incorrectly delegated.

**Code:**

```diff
// ...
#ifdef ANDROID
#include "input_common/drivers/android.h"
#include "input_common/drivers/sdl_android.h"
#endif

// ...
struct InputSubsystem::Impl {
    // ...
#ifdef ANDROID
        RegisterEngine("android", android);
-       RegisterEngine("sdl_android", android);
+       RegisterEngine("sdl_android", sdl_android);
#endif
    // ...
    [[nodiscard]] std::shared_ptr<InputEngine> GetInputEngine(
        const Common::ParamPackage& params) const {
        // ...
#ifdef ANDROID
        if (engine == android->GetEngineName()) {
            return android;
+       } else if (engine == sdl_android->GetEngineName()) {
+           return sdl_android;
        }
#endif
        // ...
    }
    // ...
#if defined(HAVE_SDL2) && !defined(ANDROID)
    std::shared_ptr<SDLDriver> sdl;
    std::shared_ptr<Joycons> joycon;
#endif

#ifdef ANDROID
    std::shared_ptr<Android> android;
+   std::shared_ptr<SdlAndroid> sdl_android;
#endif
};

// ...
void InputSubsystem::ReloadInputDevices() {
    impl->udp_client->ReloadSockets();
}

void InputSubsystem::BeginMapping(Polling::InputType type) {
    impl->BeginConfiguration();
#ifdef ANDROID
-   impl->android->BeginMapping();
+   impl->sdl_android->BeginMapping();
#else
    impl->mapping_factory->BeginMapping(type);
#endif
}

Common::ParamPackage InputSubsystem::GetNextInput() const {
#ifdef ANDROID
-   return impl->android->GetCapturedInput();
+   return impl->sdl_android->GetCapturedInput();
#else
    return impl->mapping_factory->GetNextInput();
#endif
}

void InputSubsystem::StopMapping() const {
    impl->EndConfiguration();
#ifdef ANDROID
-   impl->android->StopMapping();
+   impl->sdl_android->StopMapping();
#else
    impl->mapping_factory->StopMapping();
#endif
}
// ...
bool InputSubsystem::IsMapping() const {
    #ifdef ANDROID
-   return impl->android->IsMapping();
+   return impl->sdl_android->IsMapping();
    #else
    return impl->mapping_factory->IsMapping();
    #endif
}
```

---

## Part 7: Update JNI Layer

**File:** `D:/BuildAPK/edenMaster/src/android/app/src/main/jni/native.h`, `D:/BuildAPK/edenMaster/src/input_common/main.h`, `D:/BuildAPK/edenMaster/src/input_common/main.cpp`, `D:/BuildAPK/edenMaster/src/android/app/src/main/jni/native.cpp`

**Action:** Add a getter for the `SdlAndroid` driver and update the `onSdlEvent` JNI function to call it.

**Rationale:** This ensures that SDL events from the Java/Kotlin layer are routed to the new, correct driver.

**Code:**

**`jni/native.h`:**
```diff
class EmulationSession {
    // ...
    InputCommon::SdlAndroid* GetSdlAndroid() { return m_input_subsystem.GetSdlAndroid(); }
    // ...
}
```

**`input_common/main.h`:**
```diff
class InputSubsystem {
    // ...
#ifdef ANDROID
    Android* GetAndroid();
    const Android* GetAndroid() const;
+   SdlAndroid* GetSdlAndroid();
+   const SdlAndroid* GetSdlAndroid() const;
#endif
    // ...
}
```

**`input_common/main.cpp`:**
```diff
#ifdef ANDROID
Android* InputSubsystem::GetAndroid() {
    return impl->android.get();
}
const Android* InputSubsystem::GetAndroid() const {
    return impl->android.get();
}
+SdlAndroid* InputSubsystem::GetSdlAndroid() {
+    return impl->sdl_android.get();
+}
+const SdlAndroid* InputSubsystem::GetSdlAndroid() const {
+    return impl->sdl_android.get();
+}
#endif
```

**`native.cpp`:**
```diff
void Java_org_yuzu_yuzu_1emu_features_input_NativeInput_onSdlEvent(JNIEnv* env, jclass clazz, jbyteArray j_event) {
    jbyte* event_bytes = env->GetByteArrayElements(j_event, nullptr);
    if (!event_bytes) {
        return;
    }

    SDL_Event event;
    std::memcpy(&event, event_bytes, sizeof(SDL_Event));

-   auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
-   if (android_driver) {
-       android_driver->HandleSdlEvent(event);
-   }
+   auto* sdl_driver = EmulationSession::GetInstance().GetSdlAndroid();
+   if (sdl_driver) {
+       sdl_driver->HandleSdlEvent(event);
+   }

    env->ReleaseByteArrayElements(j_event, event_bytes, JNI_ABORT);
}
```
This completes the refactoring plan. Each step is designed to be a discrete, verifiable action, leading to the desired final state of two separate and specialized Android input drivers.
