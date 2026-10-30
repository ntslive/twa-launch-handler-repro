# Chromium bug report draft

Suggested component: Chromium > UI > Browser > Mobile > CustomTabs (Trusted Web Activity)

Fill in the `<...>` placeholders and confirm the repro results before filing.

---

## Title

TWA: launch_handler client_mode "focus-existing" is ignored when a running TWA is launched with a different URL; the existing document is navigated

## Steps to reproduce

Repro app and page: https://github.com/ntslive/twa-launch-handler-repro

1. Host the repro page at the root of an HTTPS origin, with Digital Asset Links set up for the repro app. Its web manifest declares `"launch_handler": {"client_mode": "focus-existing"}`, and the page logs every `window.launchQueue` launch, how long the document has been alive, and how many times it has loaded in the tab.
2. Install the repro app: androidbrowserhelper 2.6.2 (androidx.browser 1.9.0-alpha04) stock `LauncherActivity`, with the `android.support.customtabs.trusted.LAUNCH_HANDLER_CLIENT_MODE` meta-data set to `focus-existing`. androidbrowserhelper passes this to `TrustedWebActivityIntentBuilder.setLaunchHandlerClientMode(LaunchHandlerClientMode.FOCUS_EXISTING)`.
3. Open the app from the launcher. It runs as a verified TWA (no URL bar).
4. While it is running, launch it with a different URL, the way a notification or app link does:
   ```
   adb shell "am start -a android.intent.action.VIEW -f 0x30000000 -d 'https://twa-launch-handler-repro.web.app/?page=2' -n com.example.twalaunchhandler/com.google.androidbrowserhelper.trusted.LauncherActivity"
   ```

## Expected

The existing TWA is focused without its document being navigated, and `https://twa-launch-handler-repro.web.app/?page=2` is delivered to the page through `window.launchQueue`, as documented for [`LaunchHandlerClientMode.FOCUS_EXISTING`](https://developer.android.com/reference/androidx/browser/trusted/LaunchHandlerClientMode) and the [Launch Handler API](https://developer.chrome.com/docs/web-platform/launch-handler/).

## Actual

The existing document is navigated to `https://twa-launch-handler-repro.web.app/?page=2`: the page reloads and its state is lost. The new document then receives `?page=2` as its initial launch.

Launching the running TWA with the **same URL** it was launched with, or the same URL with only a different fragment (`https://twa-launch-handler-repro.web.app/#foo`), behaves as expected: there's no navigation, and the URL (including the fragment) is delivered to `window.launchQueue`. So launch params are delivered without navigation when the URL is unchanged, but any other URL is navigated.

logcat for step 4 in our production app shows Chrome delivering the intent to the existing `CustomTabActivity` (the same activity record before and after), then loading the URL:

```
I ActivityTaskManager: START u0 {act=android.intent.action.VIEW dat=https://... flg=0x26800000 pkg=com.android.chrome cmp=com.android.chrome/org.chromium.chrome.browser.customtabs.CustomTabActivity (has extras)}
I cr_ChromeActivity: onNewIntentWithNative: exiting picture in picture if needed
D AudioTrack: stop(31): called with 2001376 frames delivered
I MediaFocusControl: abandonAudioFocus() from uid/pid <chrome>
```

## Impact

We use a TWA for a radio streaming app. Opening a push notification or link that points anywhere other than the URL the app was launched with reloads the PWA, which stops live audio playback. `focus-existing` and `window.launchQueue` are the documented way to avoid this, but they only work for the unchanged URL.

## Questions

- Is `focus-existing` meant to be supported for Trusted Web Activities on Android? The launch handler Intent to Ship was desktop-only, but androidx.browser exposes `setLaunchHandlerClientMode` for TWAs.
- If it is, should it apply to a new launch intent for a running TWA with a different URL?

## Environment

- Chrome: 156.0.8078.73 (Stable). `<Also tested on Chrome Beta/Canary: ...>`
- Android: `<version>`, device: `<model>`
- androidbrowserhelper 2.6.2, androidx.browser 1.9.0-alpha04
- No relevant chrome://flags found ("launch handler" has no matching experiments)
