This file details the required changes to the `input_common/drivers/android.h` header file, using `src-v1old` as a blueprint.

```diff
--- a/src/input_common/drivers/android.h
+++ b/src/input_common/drivers/android.h
@@ -1,13 +1,18 @@
 // SPDX-FileCopyrightText: Copyright 2024 yuzu Emulator Project
 // SPDX-License-Identifier: GPL-3.0-or-later
 
 #pragma once
 
-#include <jni.h>
-#include <SDL_events.h>
-#include <SDL_gamepad.h>
+#include <set>
 #include <memory>
 #include <mutex>
+#include <common/threadsafe_queue.h>
 #include <string>
 #include <thread>
 #include <vector>
+
 #include <concurrentqueue.h>
+#include <jni.h>
+
 #include "common/android/android_common.h"
 #include "common/common_types.h"
 #include "common/uuid.h"
@@ -15,6 +20,9 @@
 #include "input_common/main.h"
 
 namespace Common {
+// SDL3
+#include <SDL3/SDL.h>
+
 class ParamPackage;
 }
 
@@ -53,20 +61,84 @@
     void SendVibrations(JNIEnv* env, std::stop_token token);
     std::set<s32> GetDeviceAxes(JNIEnv* env, jobject& j_device) const;
 
-    Common::ParamPackage BuildParamPackageForAnalog(PadIdentifier identifier, int axis_x,
-                                                     int axis_y) const;
-
-    Common::ParamPackage BuildAnalogParamPackageForButton(PadIdentifier identifier, s32 axis,
-                                                           bool invert) const;
-
-    Common::ParamPackage BuildButtonParamPackageForButton(PadIdentifier identifier,
-                                                           s32 button) const;
     bool MatchVID(Common::UUID device, const std::vector<std::string>& vids) const;
 
-    moodycamel::ConcurrentQueue<VibrationRequest> vibration_queue;
-    std::jthread vibration_thread;
     mutable std::mutex input_devices_mutex;
-    std::map<PadIdentifier, jobject> input_devices;
+
+    std::unordered_map<PadIdentifier, jobject> input_devices;
+
+    std::unordered_map<SDL_JoystickID, SDL_Gamepad*> sdl_gamepad_map;
 
     // SDL Members
     mutable std::mutex sdl_gamepad_map_mutex;
-    std::map<SDL_JoystickID, SDL_Gamepad*> sdl_gamepad_map;
 
     // Mapping Members
-    std::atomic<bool> is_mapping = false;
+    bool is_mapping = false;
     Common::ParamPackage captured_input;
+    
+    static constexpr s32 AXIS_X = 0;
+    static constexpr s32 AXIS_Y = 1;
+    static constexpr s32 AXIS_Z = 11;
+    static constexpr s32 AXIS_RX = 12;
+    static constexpr s32 AXIS_RY = 13;
+    static constexpr s32 AXIS_RZ = 14;
+    static constexpr s32 AXIS_HAT_X = 15;
+    static constexpr s32 AXIS_HAT_Y = 16;
+    static constexpr s32 AXIS_LTRIGGER = 17;
+    static constexpr s32 AXIS_RTRIGGER = 18;
+
+    static constexpr s32 KEYCODE_DPAD_UP = 19;
+    static constexpr s32 KEYCODE_DPAD_DOWN = 20;
+    static constexpr s32 KEYCODE_DPAD_LEFT = 21;
+    static constexpr s32 KEYCODE_DPAD_RIGHT = 22;
+    static constexpr s32 KEYCODE_BUTTON_A = 96;
+    static constexpr s32 KEYCODE_BUTTON_B = 97;
+    static constexpr s32 KEYCODE_BUTTON_X = 99;
+    static constexpr s32 KEYCODE_BUTTON_Y = 100;
+    static constexpr s32 KEYCODE_BUTTON_L1 = 102;
+    static constexpr s32 KEYCODE_BUTTON_R1 = 103;
+    static constexpr s32 KEYCODE_BUTTON_L2 = 104;
+    static constexpr s32 KEYCODE_BUTTON_R2 = 105;
+    static constexpr s32 KEYCODE_BUTTON_THUMBL = 106;
+    static constexpr s32 KEYCODE_BUTTON_THUMBR = 107;
+    static constexpr s32 KEYCODE_BUTTON_START = 108;
+    static constexpr s32 KEYCODE_BUTTON_SELECT = 109;
+    const std::vector<s32> keycode_ids{
+        KEYCODE_DPAD_UP,       KEYCODE_DPAD_DOWN,     KEYCODE_DPAD_LEFT,    KEYCODE_DPAD_RIGHT,
+        KEYCODE_BUTTON_A,      KEYCODE_BUTTON_B,      KEYCODE_BUTTON_X,     KEYCODE_BUTTON_Y,
+        KEYCODE_BUTTON_L1,     KEYCODE_BUTTON_R1,     KEYCODE_BUTTON_L2,    KEYCODE_BUTTON_R2,
+        KEYCODE_BUTTON_THUMBL, KEYCODE_BUTTON_THUMBR, KEYCODE_BUTTON_START, KEYCODE_BUTTON_SELECT,
+    };
+
+    const std::string sony_vid{"054c"};
+    const std::string nintendo_vid{"057e"};
+    const std::string razer_vid{"1532"};
+    const std::string redmagic_vid{"3537"};
+    const std::string backbone_labs_vid{"358a"};
+    const std::string xbox_vid{"045e"};
+    const std::vector<std::string> flipped_ab_vids{sony_vid,     nintendo_vid,      razer_vid,
+                                                   redmagic_vid, backbone_labs_vid, xbox_vid};
+    const std::vector<std::string> flipped_xy_vids{sony_vid, razer_vid, redmagic_vid,
+                                                   backbone_labs_vid, xbox_vid};
+
+    Common::SPSCQueue<VibrationRequest> vibration_queue;
+    std::jthread vibration_thread;
 };
 
 } // namespace InputCommon
```
