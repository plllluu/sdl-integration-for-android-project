This document details the fix for the likely build error related to `std::jthread` by explicitly adding the necessary compiler flag to the `input_common` library.

```diff
--- a/src/input_common/CMakeLists.txt
+++ b/src/input_common/CMakeLists.txt
@@ -29,6 +29,10 @@
     main.h
 )
 
+if (ANDROID)
+    target_compile_options(input_common PRIVATE -fexperimental-library)
+endif()
+
 if (MSVC)
     target_compile_options(input_common PRIVATE
         /we4242 # 'identifier': conversion from 'type1' to 'type2', possible loss of data

```

**Explanation:**

*   This adds the `-fexperimental-library` flag directly and exclusively to the `input_common` target when building for Android.
*   This flag is required by the Android NDK's Clang compiler to enable experimental C++20 features like `std::jthread`, which is used for the vibration thread in `android.cpp`.
*   Applying it directly to the target is more robust than relying on global flag inheritance from the parent `CMakeLists.txt`.
