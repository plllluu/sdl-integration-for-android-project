This document provides the definitive plan to correctly implement the hybrid `android` and `sdl_android` engine logic within the `Android` driver, as per the `src-v1old` blueprint.

### 1. Rationale

The previous strategy of unifying all button inputs under a single engine name was a flawed deviation from the project's blueprint. The blueprint correctly uses a single `Android` driver to handle two distinct input paths, identified by the engine names `android` (for native NDK inputs) and `sdl_android` (for inputs from the SDL service). This plan restores that original, correct architecture, resolving the logical errors and aligning the project with its blueprint.

### 2. Implementation Steps

#### 2.1. `src/input_common/drivers/android.cpp` - Restore Blueprint Logic

**Action:** This change restores the `android.cpp` file to its pristine blueprint state. This re-introduces the conditional logic based on the `engine` name, ensuring that SDL devices are handled separately and correctly.

```diff
--- a/src/input_common/drivers/android.cpp
+++ b/src/input_common/drivers/android.cpp
@@ -204,9 +204,10 @@
             char guid_str[33];
             SDL_GUIDToString(guid, guid_str, sizeof(guid_str));
 
-            devices.emplace_back(Common::ParamPackage{
-                    {"engine", "android"},
-                    {"display", fmt::format("{} {}", name, id)},
+            const std::string display_name = fmt::format("{} {}", name, id);
+            devices.emplace_back(Common::ParamPackage{
+                    {"engine", "sdl_android"},
+                    {"display", display_name},
                     {"guid", guid_str},
                     {"port", std::to_string(id)},
             });
@@ -328,6 +329,19 @@
     }
 
     AnalogMapping Android::GetAnalogMappingForDevice(const Common::ParamPackage& params) {
+        const std::string engine = params.Get("engine", "");
+        if (engine == "sdl_android") {
+            AnalogMapping mapping = {};
+            const auto id =  static_cast<SDL_JoystickID>(std::stoi(params.Get("port", "0")));
+            const auto identifier = GetSdlIdentifier(id);
+            auto build_analog = [&](int axis_x, int axis_y) {
+                auto package = BuildParamPackageForAnalog(identifier, axis_x, axis_y);
+                package.Set("engine", "sdl_android");
+                return package;
+            };
+
+            mapping.insert_or_assign(Settings::NativeAnalog::LStick, build_analog(SDL_GAMEPAD_AXIS_LEFTX, SDL_GAMEPAD_AXIS_LEFTY));
+            mapping.insert_or_assign(Settings::NativeAnalog::RStick, build_analog(SDL_GAMEPAD_AXIS_RIGHTX, SDL_GAMEPAD_AXIS_RIGHTY));
+            return mapping;
+        }
+
         if (!params.Has("guid") || !params.Has("port")) {
             return {};
         }
@@ -357,6 +370,42 @@
     }
 
     ButtonMapping Android::GetButtonMappingForDevice(const Common::ParamPackage& params) {
+        const std::string engine = params.Get("engine", "");
+        if (engine == "sdl_android") {
+            ButtonMapping mapping = {};
+            const auto id =  static_cast<SDL_JoystickID>(std::stoi(params.Get("port", "0")));
+            const auto identifier = GetSdlIdentifier(id);
+            auto build_button = [&](s32 button) {
+                auto package = BuildButtonParamPackageForButton(identifier, button);
+                package.Set("engine", "sdl_android");
+                return package;
+            };
+            auto build_analog_button = [&](s32 axis, bool invert) {
+                auto package = BuildAnalogParamPackageForButton(identifier, axis, invert);
+                package.Set("engine", "sdl_android");
+                return package;
+            };
+            mapping.insert_or_assign(Settings::NativeButton::A, build_button(SDL_GAMEPAD_BUTTON_SOUTH));
+            mapping.insert_or_assign(Settings::NativeButton::B, build_button(SDL_GAMEPAD_BUTTON_EAST));
+            mapping.insert_or_assign(Settings::NativeButton::X, build_button(SDL_GAMEPAD_BUTTON_WEST));
+            mapping.insert_or_assign(Settings::NativeButton::Y, build_button(SDL_GAMEPAD_BUTTON_NORTH));
+            mapping.insert_or_assign(Settings::NativeButton::LStick, build_button(SDL_GAMEPAD_BUTTON_LEFT_STICK));
+            mapping.insert_or_assign(Settings::NativeButton::RStick, build_button(SDL_GAMEPAD_BUTTON_RIGHT_STICK));
+            mapping.insert_or_assign(Settings::NativeButton::L, build_button(SDL_GAMEPAD_BUTTON_LEFT_SHOULDER));
+            mapping.insert_or_assign(Settings::NativeButton::R, build_button(SDL_GAMEPAD_BUTTON_RIGHT_SHOULDER));
+            mapping.insert_or_assign(Settings::NativeButton::ZL, build_analog_button(SDL_GAMEPAD_AXIS_LEFT_TRIGGER, false));
+            mapping.insert_or_assign(Settings::NativeButton::ZR, build_analog_button(SDL_GAMEPAD_AXIS_RIGHT_TRIGGER, false));
+            mapping.insert_or_assign(Settings::NativeButton::Plus, build_button(SDL_GAMEPAD_BUTTON_START));
+            mapping.insert_or_assign(Settings::NativeButton::Minus, build_button(SDL_GAMEPAD_BUTTON_BACK));
+            mapping.insert_or_assign(Settings::NativeButton::DUp, build_button(SDL_GAMEPAD_BUTTON_DPAD_UP));
+            mapping.insert_or_assign(Settings::NativeButton::DDown, build_button(SDL_GAMEPAD_BUTTON_DPAD_DOWN));
+            mapping.insert_or_assign(Settings::NativeButton::DLeft, build_button(SDL_GAMEPAD_BUTTON_DPAD_LEFT));
+            mapping.insert_or_assign(Settings::NativeButton::DRight, build_button(SDL_GAMEPAD_BUTTON_DPAD_RIGHT));
+            return mapping;
+        }
+
         if (!params.Has("guid") || !params.Has("port")) {
             return {};
         }

```

#### 2.2. `src/input_common/main.cpp` - Restore Blueprint Registration

**Action:** Re-register the `sdl_android` engine and restore the logic that routes requests to it.

```diff
--- a/src/input_common/main.cpp
+++ b/src/input_common/main.cpp
@@ -79,6 +79,7 @@
         RegisterEngine("camera", camera);
 #ifdef ANDROID
         RegisterEngine("android", android);
+        RegisterEngine("sdl_android", android);
 #endif
         RegisterEngine("virtual_amiibo", virtual_amiibo);
         RegisterEngine("virtual_gamepad", virtual_gamepad);
@@ -156,9 +157,6 @@
 #ifdef ANDROID
         if (engine == android->GetEngineName()) {
             return android;
-        }
-        if (engine == "sdl_android") {
-            return android;
         }
 #endif
 #ifdef ENABLE_LIBUSB
@@ -207,9 +205,6 @@
 #ifdef ANDROID
         if (engine == android->GetEngineName()) {
             return true;
-        }
-        if (engine == "sdl_android") {
-            return true;
         }
 #endif
 #ifdef ENABLE_LIBUSB

```
