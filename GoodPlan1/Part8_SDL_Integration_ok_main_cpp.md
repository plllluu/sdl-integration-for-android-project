This file provides the complete code for the new `src/android/app/src/main/jni/main.cpp` file.

```cpp
// Copyright 2026 Eden Emulator Project
// SPDX-License-Identifier: GPL-3.0-or-later

#include "main.h"
#include "native.h"
#include "input_common/drivers/android.h"

std::atomic<bool> g_service_running = true;

// --- JNI Functions ---
namespace {

void HandleSdlEvent(const SDL_Event& event) {
    auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
    if (!android_driver) {
        return;
    }

    switch(event.type) {
        case SDL_EVENT_GAMEPAD_ADDED:
            android_driver->InitSdlGamepad(event.gdevice.which);
            break;
        case SDL_EVENT_GAMEPAD_REMOVED:
            android_driver->CloseSdlGamepad(event.gdevice.which);
            break;
        default:
            // For all other relevant events, let the driver process them
            android_driver->HandleSdlEvent(event);
            break;
    }
}

void InitializeSdl() {
    SDL_SetHint(SDL_HINT_APP_NAME, "yuzu");
    SDL_SetHint(SDL_HINT_JOYSTICK_ALLOW_BACKGROUND_EVENTS, "1");
    SDL_SetHint(SDL_HINT_JOYSTICK_HIDAPI_XBOX, "1");
    if (!SDL_Init(SDL_INIT_GAMEPAD)) { // Use SDL_INIT_GAMEPAD for SDL3
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Failed to initialize SDL: %s", SDL_GetError());
    }
}

} // anonymous namespace

// --- JNI Exports ---
extern "C" {

JNIEXPORT int SDL_main(int argc, char* argv[]) {
    SDL_SetLogPriority(SDL_LOG_CATEGORY_APPLICATION, SDL_LOG_PRIORITY_INFO);
    InitializeSdl();

    while (g_service_running) {
        SDL_Event event;
        while (SDL_PollEvent(&event)) {
            HandleSdlEvent(event);
        }
        SDL_Delay(8); // ~120 FPS
    }

   // SDL_Quit(); do nothing, dont quit
    return 0;
}

JNIEXPORT void JNICALL Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeShutdownSdlLoop(JNIEnv* env, jobject /* this */) {
    g_service_running = false;
}

JNIEXPORT void JNICALL Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeStartListeningForInput(JNIEnv* env, jclass clazz) {
    auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
    if (android_driver) {
        android_driver->BeginMapping();
    }
}

JNIEXPORT jobject JNICALL Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeGetCapturedSdlInput(JNIEnv* env, jclass clazz) {
    auto* android_driver = EmulationSession::GetInstance().GetInputSubsystem().GetAndroid();
    if (android_driver) {
        Common::ParamPackage params = android_driver->GetCapturedInput();
        if (!params.Has("guid")) {
            return nullptr;
        }

        jclass mapping_class = env->FindClass("org/yuzu/yuzu_emu/features/input/model/SdlMapping");
        jmethodID constructor = env->GetMethodID(mapping_class, "<init>", "(Ljava/lang/String;IILjava/lang/String;)V");

        jstring guid = env->NewStringUTF(params.Get("guid", "").c_str());
        jint button = params.Get("button", -1);
        jint axis = params.Get("axis", -1);
        jstring direction = env->NewStringUTF(params.Get("invert", "").c_str());

        return env->NewObject(mapping_class, constructor, guid, button, axis, direction);
    }
    return nullptr;
}

JNIEXPORT void JNICALL Java_org_yuzu_yuzu_1emu_activities_SdlService_nativeLoadSdlMappings(JNIEnv* env, jclass clazz) {
    // This can be re-implemented if needed, but for now, the driver handles mappings internally.
}

}
```
