This document details the changes required for `src/android/app/build.gradle.kts` to include the pre-built `libSDL3.so` and enable SDL in the CMake build.

```diff
--- a/src/android/app/build.gradle.kts
+++ b/src/android/app/build.gradle.kts
@@ -52,7 +52,7 @@
                 arguments.addAll(
                     listOf(
                         "-DENABLE_QT=0", // Don't use QT
-                        "-DENABLE_SDL2=0", // Don't use SDL
+                        "-DENABLE_SDL2=1", // Use SDL for controller input
                         "-DENABLE_WEB_SERVICE=1", // Enable web service
                         "-DENABLE_OPENSSL=ON",
                         "-DANDROID_ARM_NEON=true", // cryptopp requires Neon to work
                         "-DYUZU_USE_CPM=ON",
                         "-DCPMUTIL_FORCE_BUNDLED=ON",
                         "-DYUZU_USE_BUNDLED_FFMPEG=ON",
@@ -66,6 +66,13 @@
 
                 abiFilters("arm64-v8a")
             }
         }
+
+    }
+
+    sourceSets {
+        getByName("main") {
+            jniLibs.srcDirs("src/main/jniLibs")
+        }
     }
 
     val keystoreFile = System.getenv("ANDROID_KEYSTORE_FILE")
```

**Explanation:**

*   We are changing `-DENABLE_SDL2=0` to `-DENABLE_SDL2=1`. Even though we are using SDL3, the existing build scripts use the `ENABLE_SDL2` flag. We will adjust this later if needed.
*   We add a `sourceSets` block to include the `src/main/jniLibs` directory. This tells Gradle to include any `.so` files from this directory in the final APK.
