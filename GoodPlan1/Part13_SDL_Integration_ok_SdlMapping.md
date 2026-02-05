This file provides the complete code for the new `src/android/app/src/main/java/org/yuzu/yuzu_emu/features/input/model/SdlMapping.kt` file.

```kotlin
package org.yuzu.yuzu_emu.features.input.model

import android.os.Parcelable
import kotlinx.parcelize.Parcelize

@Parcelize
data class SdlMapping(
    val guid: String,
    val button: Int,
    val axis: Int,
    val direction: String
) : Parcelable
```

**Explanation:**

*   This new data class is needed to pass the captured SDL input data from the native layer to the UI. The `nativeGetCapturedSdlInput` JNI function in `main.cpp` will create an instance of this class.
