---
title: "How to Open a Specified YouTube Video in the YouTube App on Android TV"
description: "Launch YouTube videos with Kotlin intents that prioritize the TV app and fall back to other handlers, accounting for package visibility and non-Activity contexts."
pubDatetime: 2026-01-28T15:20:07.638Z
---

This article clearly explains **how to open a YouTube video in the YouTube app on Android TV**, with sample Kotlin code.
On Android TV, the package name and behavior differ from the standard smartphone YouTube app, so implementing a step-by-step fallback strategy is important.

## YouTube App Package Name for Android TV

The package name for the Android TV version of the YouTube app is as follows:

* `com.google.android.youtube.tv`

By sending an `Intent` that specifies this package, you can directly launch the YouTube app on Android TV.

## Kotlin Example: Opening a Specific Video

Below is an example of an extension function that launches YouTube with a specified video ID.

```kotlin
import android.content.ActivityNotFoundException
import android.content.Context
import android.content.Intent
import android.net.Uri

fun Context.openYouTubeVideoOnTv(videoId: String) {
    val pm = packageManager

    // 1) Directly target the Android TV YouTube app (vnd.youtube: is the simplest)
    val tvAppIntent = Intent(Intent.ACTION_VIEW, Uri.parse("vnd.youtube:$videoId"))
        .setPackage("com.google.android.youtube.tv")

    // 2) Watch URL for the TV YouTube app (fallback for device/version differences)
    val tvUrlIntent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.youtube.com/watch?v=$videoId"))
        .setPackage("com.google.android.youtube.tv")

    // 3) Standard YouTube app (for phones/tablets)
    val mobileAppIntent = Intent(Intent.ACTION_VIEW, Uri.parse("vnd.youtube:$videoId"))
        .setPackage("com.google.android.youtube")

    // 4) Final fallback: browser
    val browserIntent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.youtube.com/watch?v=$videoId"))

    val candidates = listOf(tvAppIntent, tvUrlIntent, mobileAppIntent, browserIntent)

    val intentToLaunch = candidates.firstOrNull { it.resolveActivity(pm) != null }
        ?: throw ActivityNotFoundException("No app can handle YouTube video intent")

    // Handle cases where this is called from a non-Activity Context
    intentToLaunch.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)

    startActivity(intentToLaunch)
}
```

### Implementation Notes

* **Try Intents in order of priority**
  Safely fall back in this order: Android TV YouTube → URL-based launch → standard YouTube app → browser.
* **Check availability with `resolveActivity()`**
  Prevents crashes when the specified app is not installed.
* **Add `FLAG_ACTIVITY_NEW_TASK`**
  Ensures compatibility when called from a Service or Application context.

## (Important) Package Visibility on Android 11 and Later

On Android 11 (API 30) and later, when resolving specific packages with `setPackage()`, **you may need to declare them in `queries` in the Manifest**.

```xml
<manifest ...>
    <queries>
        <package android:name="com.google.android.youtube.tv" />
        <package android:name="com.google.android.youtube" />
    </queries>
</manifest>
```

### Why This Is Necessary

* Due to package visibility restrictions, other apps are no longer visible by default.
* Prevents `resolveActivity()` from always returning `null`.

## Summary

* On Android TV, it’s important to explicitly specify the **package name of the TV-specific YouTube app**.
* To account for device and OS differences, a **design that tries multiple Intents sequentially** is safest.
* On Android 11 and later, don’t forget the **`queries` configuration in the Manifest**.

Using this approach, you can reliably play specific YouTube videos from an Android TV app.
