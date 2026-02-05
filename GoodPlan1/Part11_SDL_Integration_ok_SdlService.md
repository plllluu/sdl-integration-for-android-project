This file provides the complete code for the new `src/android/app/src/main/java/org/yuzu/yuzu_emu/activities/SdlService.kt` file.

```kotlin
package org.yuzu.yuzu_emu.activities

import android.app.Service
import android.content.Intent
import android.os.IBinder
import android.util.Log
import androidx.core.app.NotificationCompat
import org.libsdl.app.HIDDeviceManager
import org.libsdl.app.SDL
import org.libsdl.app.SDLActivity
import org.libsdl.app.SDLAudioManager
import org.libsdl.app.SDLControllerManager
import org.yuzu.yuzu_emu.R
import org.yuzu.yuzu_emu.features.input.model.SdlMapping

class SdlService : Service() {

    private var sdlThread: Thread? = null
    private val NOTIFICATION_ID = 1
    private val CHANNEL_ID = "SDL_SERVICE_CHANNEL"
    private var mHIDDeviceManager: HIDDeviceManager? = null

    inner class SDLMain : Runnable {
        override fun run() {
            val libraryPath = applicationInfo.nativeLibraryDir + "/libyuzu-android.so"
            try {
                SDLActivity.nativeRunMain(libraryPath, "SDL_main", arrayOf<String>())
                Log.v("SdlService", "SDL_main finished.")
            } catch (e: Exception) {
                Log.e("SdlService", "Error in SDL_main", e)
            } finally {
                // When the native code finishes, stop the service.
                stopSelf()
            }
        }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        Log.v("SdlService", "Service command received.")

        // Promote the service to the foreground
        val notification = createNotification()
        startForeground(NOTIFICATION_ID, notification)
        Log.i("SdlService", "Service started in foreground.")

        if (sdlThread == null || !sdlThread!!.isAlive) {
            startSdlMain()
        }

        // We want the service to continue running until it is explicitly stopped.
        return START_STICKY
    }

    private fun startSdlMain() {
        try {
            // Correct Initialization Order:
            // 1. Set the context for the SDL helper classes.
            SDL.setContext(this)

            // 2. Load native libraries first.
            for (lib in getLibraries()) {
                SDL.loadLibrary(lib)
            }

            // 3. Initialize Java-side handlers.
            SDLActivity.initialize()
            SDLAudioManager.initialize()
            SDLControllerManager.initialize()

            // 4. Acquire and initialize the HID device manager.
            if (mHIDDeviceManager == null) {
                mHIDDeviceManager = HIDDeviceManager.acquire(this)
                mHIDDeviceManager?.initialize(true, true)
            }

            // 5. Set up the JNI bridge.
            SDL.setupJNI()

            // 6. Start the native entry point on its own thread.
            sdlThread = Thread(SDLMain(), "SDLThread")
            sdlThread?.start()
            Log.v("SdlService", "SDL thread started.")

        } catch (e: Exception) {
            Log.e("SdlService", "Error starting SDL thread", e)
            stopSelf() // Stop the service if initialization fails
        }
    }

    private fun createNotification() = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("SDL Service Active")
            .setContentText("Processing controller inputs.")
            .setSmallIcon(R.mipmap.ic_launcher) // Replace with your app's icon
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .build()

    private fun getLibraries(): Array<String> {
        return arrayOf("SDL3", "yuzu-android")
    }

    override fun onDestroy() {
        Log.v("SdlService", "Service being destroyed.")
        // Ensure the native code is signaled to quit
        if (sdlThread != null && sdlThread!!.isAlive) {
            nativeShutdownSdlLoop()
            try {
                sdlThread?.join(500)
            } catch (e: InterruptedException) { /* Ignore */ }
        }

        if (mHIDDeviceManager != null) {
            HIDDeviceManager.release(mHIDDeviceManager)
            mHIDDeviceManager = null
        }

        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? = null

    private external fun nativeShutdownSdlLoop()
    
    companion object {
        @JvmStatic
        external fun nativeStartListeningForInput()

        @JvmStatic
        external fun nativeGetCapturedSdlInput(): SdlMapping?

        @JvmStatic
        external fun nativeLoadSdlMappings()
    }
}
```
