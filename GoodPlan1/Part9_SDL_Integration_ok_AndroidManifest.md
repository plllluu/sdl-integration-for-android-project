This document details the changes required for `src/android/app/src/main/AndroidManifest.xml` to add the necessary permissions and declare the `SdlService`.

```diff
--- a/src/android/app/src/main/AndroidManifest.xml
+++ b/src/android/app/src/main/AndroidManifest.xml
@@ -13,6 +13,7 @@
     <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
     <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
     <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
+    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />
     <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
     <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
     <uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" tools:ignore="ScopedStorage" />
@@ -69,6 +70,11 @@
             <meta-data android:name="android.nfc.action.TECH_DISCOVERED" android:resource="@xml/nfc_tech_filter" />
         </activity>
 
+        <service
+            android:name=".activities.SdlService"
+            android:exported="false"
+            android:foregroundServiceType="connectedDevice" />
+
         <service android:name="org.yuzu.yuzu_emu.utils.ForegroundService" android:foregroundServiceType="specialUse">
             <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="Keep emulation running in background"/>
         </service>

```

**Explanation:**

*   We add the `FOREGROUND_SERVICE_CONNECTED_DEVICE` permission, which is required for foreground services that access connected devices like game controllers.
*   We declare the `SdlService`, setting its `foregroundServiceType` to `connectedDevice`.
