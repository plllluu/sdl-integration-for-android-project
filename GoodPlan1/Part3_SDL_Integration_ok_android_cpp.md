This file provides the complete, updated code for `input_common/drivers/android.cpp`.

```cpp
// SPDX-FileCopyrightText: Copyright 2024 yuzu Emulator Project
// SPDX-License-Identifier: GPL-3.0-or-later

#include <set>
#include <common/settings_input.h>
#include <common/thread.h>
#include <jni.h>
#include <android/log.h>
#include "common/android/android_common.h"
#include "common/android/id_cache.h"
#include "common/logging/log.h"
#include "input_common/drivers/android.h"

namespace InputCommon {

    namespace {
// Normalize to the [-1, 1] range
        constexpr float AXIS_MAX = 32767.0f;

        PadIdentifier GetSdlIdentifier(SDL_JoystickID instance_id) {
            const SDL_GUID guid = SDL_GetJoystickGUIDForID(instance_id);
            std::array<u8, 16> data{};
            std::memcpy(data.data(), guid.data, sizeof(data));
            return {
                    .guid = Common::UUID{data},
                    .port = static_cast<std::size_t>(instance_id), // Use instance_id as port for uniqueness
                    .pad = 0,
            };
        }
    } // Anonymous namespace

    Android::Android(std::string input_engine_) : InputEngine(std::move(input_engine_)) {
        vibration_thread = std::jthread([this](std::stop_token token) {
            Common::SetCurrentThreadName("Android_Vibration");
            auto env = Common::Android::GetEnvForThread();
            using namespace std::chrono_literals;
            while (!token.stop_requested()) {
                SendVibrations(env, token);
            }
        });
    }

    Android::~Android() = default;

    void Android::RegisterController(jobject j_input_device) {
        auto env = Common::Android::GetEnvForThread();
        const std::string name = Common::Android::GetJString(env, static_cast<jstring>(env->CallObjectMethod(j_input_device, Common::Android::GetYuzuDeviceGetName())));
        if (name.find("UDP") != std::string::npos) {
            return;
        }

        const std::string guid = Common::Android::GetJString(
                env, static_cast<jstring>(
                        env->CallObjectMethod(j_input_device, Common::Android::GetYuzuDeviceGetGUID())));
        const s32 port = env->CallIntMethod(j_input_device, Common::Android::GetYuzuDeviceGetPort());
        const auto identifier = GetIdentifier(guid, static_cast<size_t>(port));
        PreSetController(identifier);

        std::scoped_lock lock{input_devices_mutex};
        if (auto it = input_devices.find(identifier); it != input_devices.end()) {
            env->DeleteGlobalRef(it->second);
            input_devices.erase(it);
        }
        input_devices.emplace(identifier, env->NewGlobalRef(j_input_device));
    }

    void Android::UnregisterController(const PadIdentifier& identifier) {
        std::scoped_lock lock{input_devices_mutex};
        if (auto it = input_devices.find(identifier); it != input_devices.end()) {
            Common::Android::GetEnvForThread()->DeleteGlobalRef(it->second);
            input_devices.erase(it);
        }
    }

    void Android::InitSdlGamepad(SDL_JoystickID instance_id) {
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

    void Android::CloseSdlGamepad(SDL_JoystickID instance_id) {
        std::scoped_lock lock{sdl_gamepad_map_mutex};
        if (auto it = sdl_gamepad_map.find(instance_id); it != sdl_gamepad_map.end()) {
            LOG_INFO(Input, "Closing gamepad: {}", SDL_GetGamepadName(it->second));
            SDL_CloseGamepad(it->second);
            sdl_gamepad_map.erase(it);
        }
    }

    void Android::HandleSdlEvent(const SDL_Event& event) {
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
                    captured_input.Set("engine", "sdl_android");
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
                        captured_input.Set("engine", "sdl_android");
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

    void Android::SetButtonState(std::string guid, size_t port, int button_id, bool value) {
        const auto identifier = GetIdentifier(guid, port);
        SetButton(identifier, button_id, value);
    }

    void Android::SetAxisPosition(std::string guid, size_t port, int axis_id, float value) {
        const auto identifier = GetIdentifier(guid, port);
        SetAxis(identifier, axis_id, value);
    }

    void Android::SetMotionState(std::string guid, size_t port, u64 delta_timestamp, float gyro_x,
                                 float gyro_y, float gyro_z, float accel_x, float accel_y,
                                 float accel_z) {
        const auto identifier = GetIdentifier(guid, port);
        const BasicMotion motion_data{
                .gyro_x = gyro_x,
                .gyro_y = gyro_y,
                .gyro_z = gyro_z,
                .accel_x = accel_x,
                .accel_y = accel_y,
                .accel_z = accel_z,
                .delta_timestamp = delta_timestamp,
        };
        SetMotion(identifier, 0, motion_data);
    }

    Common::Input::DriverResult Android::SetVibration(
            [[maybe_unused]] const PadIdentifier& identifier,
            [[maybe_unused]] const Common::Input::VibrationStatus& vibration) {
        vibration_queue.Push(VibrationRequest{
                .identifier = identifier,
                .vibration = vibration,
        });
        return Common::Input::DriverResult::Success;
    }

    bool Android::IsVibrationEnabled(const PadIdentifier& identifier) {
        auto device = input_devices.find(identifier);
        if (device != input_devices.end()) {
            return Common::Android::RunJNIOnFiber<bool>([&](JNIEnv* env) {
                return static_cast<bool>(env->CallBooleanMethod(
                        device->second, Common::Android::GetYuzuDeviceGetSupportsVibration()));
            });
        }

        std::scoped_lock lock{sdl_gamepad_map_mutex};
        if (auto it = sdl_gamepad_map.find(static_cast<SDL_JoystickID>(identifier.port)); it != sdl_gamepad_map.end()) {
            return SDL_RumbleGamepad(it->second, 0, 0, 0) == 0;
        }

        return false;
    }

    std::vector<Common::ParamPackage> Android::GetInputDevices() const {
        std::vector<Common::ParamPackage> devices;
        auto env = Common::Android::GetEnvForThread();
        std::scoped_lock lock{input_devices_mutex};
        for (const auto& [key, value] : input_devices) {
            if (key.port == 100) {
                continue;
            }
            auto name_object = static_cast<jstring>(
                    env->CallObjectMethod(value, Common::Android::GetYuzuDeviceGetName()));
            const std::string name =
                    fmt::format("{} {}", Common::Android::GetJString(env, name_object), key.port);
            devices.emplace_back(Common::ParamPackage{
                    {"engine", GetEngineName()},
                    {"display", std::move(name)},
                    {"guid", key.guid.RawString()},
                    {"port", std::to_string(key.port)},
            });
        }
        // Add SDL devices
        std::scoped_lock lock_sdl{sdl_gamepad_map_mutex};
        for(const auto& [id, gamepad] : sdl_gamepad_map) {
            const std::string name = SDL_GetGamepadName(gamepad);
            if (name.find("UDP") != std::string::npos) {
                continue;
            }
            const SDL_GUID guid = SDL_GetJoystickGUIDForID(id);
            char guid_str[33];
            SDL_GUIDToString(guid, guid_str, sizeof(guid_str));

            const std::string display_name = fmt::format("{} {}", name, id);
            devices.emplace_back(Common::ParamPackage{
                    {"engine", "sdl_android"},
                    {"display", display_name},
                    {"guid", guid_str},
                    {"port", std::to_string(id)},
            });
        }
        return devices;
    }

    void Android::BeginMapping() {
        is_mapping = true;
        captured_input = {};
    }

    void Android::StopMapping() {
        is_mapping = false;
    }

    bool Android::IsMapping() const {
        return is_mapping;
    }

    Common::ParamPackage Android::GetCapturedInput() {
        return captured_input;
    }

    std::set<s32> Android::GetDeviceAxes(JNIEnv* env, jobject& j_device) const {
        auto j_axes = static_cast<jobjectArray>(
                env->CallObjectMethod(j_device, Common::Android::GetYuzuDeviceGetAxes()));
        std::set<s32> axes;
        for (int i = 0; i < env->GetArrayLength(j_axes); ++i) {
            jobject axis = env->GetObjectArrayElement(j_axes, i);
            axes.insert(env->GetIntField(axis, Common::Android::GetIntegerValueField()));
        }
        return axes;
    }

    Common::ParamPackage Android::BuildParamPackageForAnalog(PadIdentifier identifier, int axis_x,
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

        // Invert Y-Axis by default
        params.Set("invert_y", "-");
        return params;
    }

    Common::ParamPackage Android::BuildAnalogParamPackageForButton(PadIdentifier identifier, s32 axis,
                                                                   bool invert) const {
        Common::ParamPackage params{};
        params.Set("engine", GetEngineName());
        params.Set("port", static_cast<int>(identifier.port));
        params.Set("guid", identifier.guid.RawString());
        params.Set("axis", axis);
        params.Set("threshold", "0.5");
        params.Set("invert", invert ? "-" : "+");
        return params;
    }

    Common::ParamPackage Android::BuildButtonParamPackageForButton(PadIdentifier identifier,
                                                                   s32 button) const {
        Common::ParamPackage params{};
        params.Set("engine", GetEngineName());
        params.Set("port", static_cast<int>(identifier.port));
        params.Set("guid", identifier.guid.RawString());
        params.Set("button", button);
        return params;
    }

    bool Android::MatchVID(Common::UUID device, const std::vector<std::string>& vids) const {
        for (size_t i = 0; i < vids.size(); ++i) {
            auto guid_str = device.RawString();
            if (guid_str.find(vids[i]) != std::string::npos) {
                return true;
            }
        }
        return false;
    }

    AnalogMapping Android::GetAnalogMappingForDevice(const Common::ParamPackage& params) {
        const std::string engine = params.Get("engine", "");
        if (engine == "sdl_android") {
            AnalogMapping mapping = {};
            const auto id =  static_cast<SDL_JoystickID>(std::stoi(params.Get("port", "0")));
            const auto identifier = GetSdlIdentifier(id);
            auto build_analog = [&](int axis_x, int axis_y) {
                auto package = BuildParamPackageForAnalog(identifier, axis_x, axis_y);
                package.Set("engine", "sdl_android");
                return package;
            };

            mapping.insert_or_assign(Settings::NativeAnalog::LStick, build_analog(SDL_GAMEPAD_AXIS_LEFTX, SDL_GAMEPAD_AXIS_LEFTY));
            mapping.insert_or_assign(Settings::NativeAnalog::RStick, build_analog(SDL_GAMEPAD_AXIS_RIGHTX, SDL_GAMEPAD_AXIS_RIGHTY));
            return mapping;
        }

        if (!params.Has("guid") || !params.Has("port")) {
            return {};
        }

        auto identifier =
                GetIdentifier(params.Get("guid", ""), static_cast<size_t>(params.Get("port", 0)));
        std::scoped_lock lock(input_devices_mutex);
        auto it = input_devices.find(identifier);
        if (it == input_devices.end()) {
            return {};
        }
        auto& j_device = it->second;

        auto env = Common::Android::GetEnvForThread();
        std::set<s32> axes = GetDeviceAxes(env, j_device);
        if (axes.empty()) {
            return {};
        }

        AnalogMapping mapping = {};
        if (axes.count(AXIS_X) && axes.count(AXIS_Y)) {
            mapping.insert_or_assign(Settings::NativeAnalog::LStick,
                                     BuildParamPackageForAnalog(identifier, AXIS_X, AXIS_Y));
        }

        if (axes.count(AXIS_RX) && axes.count(AXIS_RY)) {
            mapping.insert_or_assign(Settings::NativeAnalog::RStick,
                                     BuildParamPackageForAnalog(identifier, AXIS_RX, AXIS_RY));
        } else if (axes.count(AXIS_Z) && axes.count(AXIS_RZ)) {
            mapping.insert_or_assign(Settings::NativeAnalog::RStick,
                                     BuildParamPackageForAnalog(identifier, AXIS_Z, AXIS_RZ));
        }
        return mapping;
    }

    ButtonMapping Android::GetButtonMappingForDevice(const Common::ParamPackage& params) {
        const std::string engine = params.Get("engine", "");
        if (engine == "sdl_android") {
            ButtonMapping mapping = {};
            const auto id =  static_cast<SDL_JoystickID>(std::stoi(params.Get("port", "0")));
            const auto identifier = GetSdlIdentifier(id);
            auto build_button = [&](s32 button) {
                auto package = BuildButtonParamPackageForButton(identifier, button);
                package.Set("engine", "sdl_android");
                return package;
            };
            auto build_analog_button = [&](s32 axis, bool invert) {
                auto package = BuildAnalogParamPackageForButton(identifier, axis, invert);
                package.Set("engine", "sdl_android");
                return package;
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

        if (!params.Has("guid") || !params.Has("port")) {
            return {};
        }

        auto identifier =
                GetIdentifier(params.Get("guid", ""), static_cast<size_t>(params.Get("port", 0)));
        std::scoped_lock lock(input_devices_mutex);
        auto it = input_devices.find(identifier);
        if (it == input_devices.end()) {
            return {};
        }
        auto& j_device = it->second;

        auto env = Common::Android::GetEnvForThread();
        jintArray j_keys = env->NewIntArray(static_cast<int>(keycode_ids.size()));
        env->SetIntArrayRegion(j_keys, 0, static_cast<int>(keycode_ids.size()), keycode_ids.data());
        auto j_has_keys_object = static_cast<jbooleanArray>(
                env->CallObjectMethod(j_device, Common::Android::GetYuzuDeviceHasKeys(), j_keys));
        jboolean isCopy = false;
        jboolean* j_has_keys = env->GetBooleanArrayElements(j_has_keys_object, &isCopy);

        std::set<s32> available_keys;
        for (size_t i = 0; i < keycode_ids.size(); ++i) {
            if (j_has_keys[i]) {
                available_keys.insert(keycode_ids[i]);
            }
        }

        // Some devices use axes instead of buttons for certain controls so we need all the axes here
        std::set<s32> axes = GetDeviceAxes(env, j_device);

        ButtonMapping mapping = {};
        if (axes.count(AXIS_HAT_X) && axes.count(AXIS_HAT_Y)) {
            mapping.insert_or_assign(Settings::NativeButton::DUp,
                                     BuildAnalogParamPackageForButton(identifier, AXIS_HAT_Y, true));
            mapping.insert_or_assign(Settings::NativeButton::DDown,
                                     BuildAnalogParamPackageForButton(identifier, AXIS_HAT_Y, false));
            mapping.insert_or_assign(Settings::NativeButton::DLeft,
                                     BuildAnalogParamPackageForButton(identifier, AXIS_HAT_X, true));
            mapping.insert_or_assign(Settings::NativeButton::DRight,
                                     BuildAnalogParamPackageForButton(identifier, AXIS_HAT_X, false));
        } else if (available_keys.count(KEYCODE_DPAD_UP) &&
                   available_keys.count(KEYCODE_DPAD_DOWN) &&
                   available_keys.count(KEYCODE_DPAD_LEFT) &&
                   available_keys.count(KEYCODE_DPAD_RIGHT)) {
            mapping.insert_or_assign(Settings::NativeButton::DUp,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_DPAD_UP));
            mapping.insert_or_assign(Settings::NativeButton::DDown,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_DPAD_DOWN));
            mapping.insert_or_assign(Settings::NativeButton::DLeft,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_DPAD_LEFT));
            mapping.insert_or_assign(Settings::NativeButton::DRight,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_DPAD_RIGHT));
        }

        if (axes.count(AXIS_LTRIGGER)) {
            mapping.insert_or_assign(Settings::NativeButton::ZL, BuildAnalogParamPackageForButton(
                    identifier, AXIS_LTRIGGER, false));
        } else if (available_keys.count(KEYCODE_BUTTON_L2)) {
            mapping.insert_or_assign(Settings::NativeButton::ZL,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_L2));
        }

        if (axes.count(AXIS_RTRIGGER)) {
            mapping.insert_or_assign(Settings::NativeButton::ZR, BuildAnalogParamPackageForButton(
                    identifier, AXIS_RTRIGGER, false));
        } else if (available_keys.count(KEYCODE_BUTTON_R2)) {
            mapping.insert_or_assign(Settings::NativeButton::ZR,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_R2));
        }

        if (available_keys.count(KEYCODE_BUTTON_A)) {
            if (MatchVID(identifier.guid, flipped_ab_vids)) {
                mapping.insert_or_assign(Settings::NativeButton::B, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_A));
            } else {
                mapping.insert_or_assign(Settings::NativeButton::A, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_A));
            }
        }
        if (available_keys.count(KEYCODE_BUTTON_B)) {
            if (MatchVID(identifier.guid, flipped_ab_vids)) {
                mapping.insert_or_assign(Settings::NativeButton::A, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_B));
            } else {
                mapping.insert_or_assign(Settings::NativeButton::B, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_B));
            }
        }
        if (available_keys.count(KEYCODE_BUTTON_X)) {
            if (MatchVID(identifier.guid, flipped_xy_vids)) {
                mapping.insert_or_assign(Settings::NativeButton::Y, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_X));
            } else {
                mapping.insert_or_assign(Settings::NativeButton::X, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_X));
            }
        }
        if (available_keys.count(KEYCODE_BUTTON_Y)) {
            if (MatchVID(identifier.guid, flipped_xy_vids)) {
                mapping.insert_or_assign(Settings::NativeButton::X, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_Y));
            } else {
                mapping.insert_or_assign(Settings::NativeButton::Y, BuildButtonParamPackageForButton(
                        identifier, KEYCODE_BUTTON_Y));
            }
        }

        if (available_keys.count(KEYCODE_BUTTON_L1)) {
            mapping.insert_or_assign(Settings::NativeButton::L,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_L1));
        }
        if (available_keys.count(KEYCODE_BUTTON_R1)) {
            mapping.insert_or_assign(Settings::NativeButton::R,
                                     BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_R1));
        }

        if (available_keys.count(KEYCODE_BUTTON_THUMBL)) {
            mapping.insert_or_assign(
                    Settings::NativeButton::LStick,
                    BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_THUMBL));
        }
        if (available_keys.count(KEYCODE_BUTTON_THUMBR)) {
            mapping.insert_or_assign(
                    Settings::NativeButton::RStick,
                    BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_THUMBR));
        }

        if (available_keys.count(KEYCODE_BUTTON_START)) {
            mapping.insert_or_assign(
                    Settings::NativeButton::Plus,
                    BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_START));
        }
        if (available_keys.count(KEYCODE_BUTTON_SELECT)) {
            mapping.insert_or_assign(
                    Settings::NativeButton::Minus,
                    BuildButtonParamPackageForButton(identifier, KEYCODE_BUTTON_SELECT));
        }

        return mapping;
    }

    Common::Input::ButtonNames Android::GetUIName(
            [[maybe_unused]] const Common::ParamPackage& params) const {
        return Common::Input::ButtonNames::Value;
    }

    PadIdentifier Android::GetIdentifier(const std::string& guid, size_t port) const {
        return {
                .guid = Common::UUID{guid},
                .port = port,
                .pad = 0,
        };
    }

    void Android::SendVibrations([[maybe_unused]] JNIEnv* env, std::stop_token token) {
        VibrationRequest request = vibration_queue.PopWait(token);

        {
            std::scoped_lock lock{input_devices_mutex};
            auto device = input_devices.find(request.identifier);
            if (device != input_devices.end()) {
                float average_intensity =
                    (request.vibration.high_amplitude + request.vibration.low_amplitude) / 2.0f;
                Common::Android::GetEnvForThread()->CallVoidMethod(
                    device->second, Common::Android::GetYuzuDeviceVibrate(), average_intensity);
                return;
            }
        }

        std::scoped_lock lock{sdl_gamepad_map_mutex};
        if (auto it = sdl_gamepad_map.find(static_cast<SDL_JoystickID>(request.identifier.port));
            it != sdl_gamepad_map.end()) {
            const u16 low_freq = static_cast<u16>(request.vibration.low_amplitude * 0xFFFF);
            const u16 high_freq = static_cast<u16>(request.vibration.high_amplitude * 0xFFFF);
            SDL_RumbleGamepad(it->second, low_freq, high_freq, 100);
        }
    }

} // namespace InputCommon
```
