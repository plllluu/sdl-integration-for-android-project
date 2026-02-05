This document details the plan to restore the essential axis, keycode, and VID constants to `src/input_common/drivers/android.h`.

```diff
--- a/src/input_common/drivers/android.h
+++ b/src/input_common/drivers/android.h
@@ -53,6 +53,60 @@
     // Mapping Members
     std::atomic<bool> is_mapping = false;
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
 };
 
 } // namespace InputCommon
```

**Explanation:**

*   Restores the `AXIS_*`, `KEYCODE_*`, and `*_vid` constants that are essential for mapping raw Android input to the emulator's virtual controller.
