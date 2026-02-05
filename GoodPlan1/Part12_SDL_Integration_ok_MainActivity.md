This document details the changes required for `src/android/app/src/main/java/org/yuzu/yuzu_emu/ui/main/MainActivity.kt` to start the `SdlService`.

```diff
--- a/src/android/app/src/main/java/org/yuzu/yuzu_emu/ui/main/MainActivity.kt
+++ b/src/android/app/src/main/java/org/yuzu/yuzu_emu/ui/main/MainActivity.kt
@@ -29,6 +29,7 @@
 import org.yuzu.yuzu_emu.model.TaskViewModel
 import org.yuzu.yuzu_emu.utils.*
 import org.yuzu.yuzu_emu.utils.ViewUtils.setVisible
+import org.yuzu.yuzu_emu.activities.SdlService
 import java.io.BufferedInputStream
 import java.io.BufferedOutputStream
 import java.util.zip.ZipEntry
@@ -107,6 +108,8 @@
              checkForUpdates()
         }
         setInsets()
+
+        startSdlService()
     }
 
     private fun checkForUpdates() {
@@ -279,6 +282,11 @@
         }.start()
     }
 
+    private fun startSdlService() {
+        val intent = Intent(this, SdlService::class.java)
+        startService(intent)
+    }
+
     override fun onResume() {
         ThemeHelper.setCorrectTheme(this)
         super.onResume()

```

**Explanation:**

*   We add a call to `startSdlService()` in the `onCreate` method of `MainActivity`. This will ensure that the `SdlService` is started when the application launches.
