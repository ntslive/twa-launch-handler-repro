# Chromium bug report draft

Suggested component: Chromium > UI > Browser > Mobile > CustomTabs (Trusted Web Activity)

Fill in the `<...>` placeholders before filing.

---

## Title

TWA: launch_handler client_mode "focus-existing" is ignored when a running TWA is launched with a URL with a different path; the existing document is navigated

## Steps to reproduce

Repro app and page: https://github.com/ntslive/twa-launch-handler-repro (page hosted at https://twa-launch-handler-repro.web.app)

1. The repro page's web manifest declares `"launch_handler": {"client_mode": "focus-existing"}`. The page shows how long its document has been alive, how many times a document has loaded in the tab, and every `window.launchQueue` launch it receives.
2. Install the repro app: a TWA using androidbrowserhelper 2.6.2's (androidx.browser 1.9.0-alpha04) stock `LauncherActivity`, no custom code, with the `android.support.customtabs.trusted.LAUNCH_HANDLER_CLIENT_MODE` meta-data set to `focus-existing`. androidbrowserhelper passes this to `TrustedWebActivityIntentBuilder.setLaunchHandlerClientMode(LaunchHandlerClientMode.FOCUS_EXISTING)`.
3. Open the app from the launcher. It runs as a verified TWA (no URL bar) at `https://twa-launch-handler-repro.web.app/`.
4. While it is running, launch it with a URL with a different path, the way a notification or app link does:
   ```
   adb shell "am start -a android.intent.action.VIEW -f 0x30000000 -d 'https://twa-launch-handler-repro.web.app/page/2' -n com.example.twalaunchhandler/com.google.androidbrowserhelper.trusted.LauncherActivity"
   ```

## Expected

The existing TWA is focused without its document being navigated, and `https://twa-launch-handler-repro.web.app/page/2` is delivered to the page through `window.launchQueue`, as documented for [`LaunchHandlerClientMode.FOCUS_EXISTING`](https://developer.android.com/reference/androidx/browser/trusted/LaunchHandlerClientMode) and the [Launch Handler API](https://developer.chrome.com/docs/web-platform/launch-handler/).

## Actual

The existing document is navigated to `https://twa-launch-handler-repro.web.app/page/2`: the page reloads and its state is lost.

The same launch with only the query or fragment changed works as expected. There's no navigation, and the URL is delivered to `window.launchQueue`:

| Launched while running at `/` | Result |
|---|---|
| `/` | Focused, delivered to `launchQueue` |
| `/#foo` | Focused, delivered to `launchQueue` |
| `/?page=2` | Focused, delivered to `launchQueue` |
| `/page/2` | **Existing document navigated (reloads)** |

Chrome seems to navigate only when the launched URL's path differs from both the path the TWA was launched with and the current document's path. After the page changed its URL to `/page/3` with `history.pushState` (no navigation), launching `/?page=9` (the launch path) and then `/page/3?x=1` (the document's path) were both focused and delivered to `launchQueue`.

We first saw this in our production app, on a phone with Chrome 156.0.8078.73. logcat there shows Chrome delivering the intent to the existing `CustomTabActivity` (the same activity record before and after), then loading the URL:

```
I ActivityTaskManager: START u0 {act=android.intent.action.VIEW dat=https://... flg=0x26800000 pkg=com.android.chrome cmp=com.android.chrome/org.chromium.chrome.browser.customtabs.CustomTabActivity (has extras)}
I cr_ChromeActivity: onNewIntentWithNative: exiting picture in picture if needed
D AudioTrack: stop(31): called with 2001376 frames delivered
I MediaFocusControl: abandonAudioFocus() from uid/pid <chrome>
```

## Impact

We use a TWA for a radio streaming app. Opening a push notification or link to another page of the app reloads the PWA, which stops live audio playback. `focus-existing` and `window.launchQueue` are the documented way to avoid this, and they work for query or fragment changes, but deep links to other pages nearly always change the path.

## Environment

- Chrome 156.0.8078.75 (Stable) on an Android 15 emulator (Pixel 9, API 35), and Chrome 156.0.8078.73 on Pixel 5, Android 14
- androidbrowserhelper 2.6.2, androidx.browser 1.9.0-alpha04
- No relevant chrome://flags found ("launch handler" has no matching experiments)
