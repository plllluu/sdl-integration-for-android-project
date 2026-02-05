This document outlines the plan to re-integrate SDL-based controller input into the Android input driver. The goal is to restore the functionality present in the `src-v1old` version of the code, which allows for robust controller support through the SDL library, and to run this in a headless foreground service for background input processing.

#### Plan:

1.  **Modify `src/android/app/build.gradle.kts`:** Add the `jniLibs` directory to the `sourceSets` and enable SDL in the CMake build.
2.  **Modify `src/android/app/src/main/jni/CMakeLists.txt`:** Add `SDL3` as an imported library and link it to the `yuzu-android` native library. Also, add the new `main.cpp` to the `add_library` command.
3.  **Modify `src/input_common/CMakeLists.txt`:** do not link against the `SDL2::SDL2` target.
4.  **Modify `src/input_common/drivers/android.h`:** Update the header to include SDL headers and declare the necessary member variables and functions for SDL integration.
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
