# sdl-integration-for-android-project
The plan to re-integrate SDL-based controller input into the Android input driver. Allows for robust controller support through the SDL library to run this in a headless foreground service for background input processing.

# Instroduction: Overview of the Refactored Android Input System

## Overview

**Simple DirectMedia Layer** (SDL) It is a cross-platform development library designed to provide low-level access to hardware through graphical, audio, and input device interfaces.
This document provides an overview of the successfully integration SDL in Android input handling system. The original implementation, which standard Android input events in a single driver, has been replaced by a cleaner, more robust dual-driver architecture.

This integration has significantly improved the modularity, maintainability, and correctness of the input subsystem. The final implementation consists of two specialized drivers:

1.  **`android` driver:** Responsible *only* for standard Android `KeyEvent` and `MotionEvent` handling, including native vibration.
2.  **`sdl_android` driver:** Responsible *only* for events originating from the SDL library, including SDL-based rumble.

This document, along with the subsequent parts, will serve as the definitive record of the final implementation.

---

## Key Architectural Changes

- **Driver Separation:** The monolithic `android` driver was split. All SDL-related logic was moved into the new `SdlAndroid` driver, leaving the original `Android` driver to handle only native inputs.
- **Unified Manual Mapping:** The central `InputSubsystem` was corrected to handle manual input mapping from either driver. It now polls both drivers simultaneously and accepts the first valid input, providing a seamless user experience.
- **Correct JNI Routing:** The JNI layer was updated to be a simple, direct router. Native Android events are passed to the `Android` driver, and SDL events are passed to the `SdlAndroid` driver, eliminating any ambiguity.

This series of documents will provide the full, final code for each of these components.

#### Plan:

1.  **Modify `src/android/app/build.gradle.kts`:** Add the `jniLibs` directory to the `sourceSets` and enable SDL in the CMake build.
2.  **Modify `src/android/app/src/main/jni/CMakeLists.txt`:** Add `SDL3` as an imported library and link it to the `yuzu-android` native library. Also, add the new `main.cpp` to the `add_library` command.
3.  **Modify `src/input_common/CMakeLists.txt`:** Do not link against the `SDL2::SDL2` target.
4.  **Modify `src/input_common/drivers/android.h`:** Update the header to include SDL headers and declare the necessary member variables and functions for SDL integration, using `src-v1old` as a reference.
5.  **Modify `src/input_common/drivers/android.cpp`:** Implement the SDL-related functionality in the `Android` driver, mirroring the logic from `src-v1old`.
6.  **Modify `src/android/app/src/main/jni/native_input.cpp`:** Add a JNI function to pass SDL events from the Java layer to the native C++ code.
7.  **Modify `src/input_common/main.cpp`:** Ensure the `InputSubsystem` is aware of and correctly utilizes the SDL-enabled `Android` driver.
8.  **Create `src/android/app/src/main/jni/main.cpp`:** This new file will contain the native `SDL_main` entry point and related JNI functions.
9.  **Modify `src/android/app/src/main/AndroidManifest.xml`:** Add the necessary permissions and declare the `SdlService`.
10. **Modify `src/android/app/src/main/java/org/yuzu/yuzu_emu/YuzuApplication.kt`:** Create a notification channel for the `SdlService`.
11. **Create `src/android/app/src/main/java/org/yuzu/yuzu_emu/activities/SdlService.kt`:** Implement the headless SDL service.
12. **Modify `src/android/app/src/main/java/org/yuzu/yuzu_emu/ui/main/MainActivity.kt`:** Start the `SdlService`.
13. **Create `src/android/app/src/main/java/org/yuzu/yuzu_emu/features/input/model/SdlMapping.kt`:** Create the data class for passing captured SDL input data to the UI.
14. **Refine `src/android/app/src/main/jni/CMakeLists.txt`:** Use a modern CMake target for SDL3 and add the header include directory.
