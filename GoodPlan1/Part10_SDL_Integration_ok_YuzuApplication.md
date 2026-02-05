This document details the changes required for `src/android/app/src/main/java/org/yuzu/yuzu_emu/YuzuApplication.kt` to create a notification channel for the `SdlService`.

```diff
--- a/src/android/app/src/main/java/org/yuzu/yuzu_emu/YuzuApplication.kt
+++ b/src/android/app/src/main/java/org/yuzu/yuzu_emu/YuzuApplication.kt
@@ -24,6 +24,15 @@
         foregroundService.setSound(null, null)
         foregroundService.vibrationPattern = null
 
+        val sdlService = NotificationChannel(
+            "SDL_SERVICE_CHANNEL",
+            "SDL Service",
+            NotificationManager.IMPORTANCE_LOW
+        )
+        sdlService.description = "Notification for the SDL background service"
+        sdlService.setSound(null, null)
+        sdlService.vibrationPattern = null
+
         val noticeChannel = NotificationChannel(
             getString(R.string.notice_notification_channel_id),
             getString(R.string.notice_notification_channel_name),
@@ -36,6 +45,7 @@
         val notificationManager = getSystemService(NotificationManager::class.java)
         notificationManager.createNotificationChannel(noticeChannel)
         notificationManager.createNotificationChannel(foregroundService)
+        notificationManager.createNotificationChannel(sdlService)
     }
 
     override fun onCreate() {

```

**Explanation:**

*   We add a new `NotificationChannel` with the ID `SDL_SERVICE_CHANNEL`. This is the channel that the `SdlService` will use to display its persistent notification.
