# TWA launch handler repro

Minimal reproduction for: a running Trusted Web Activity launched with a URL with a **different
path** is navigated (the document reloads), even though both the app and the web manifest set
`launch_handler` `client_mode` to `focus-existing`. A different query or fragment works as expected.

Chromium issue: https://issues.chromium.org/issues/579632124

- `web/` is a static page that shows how long its document has been alive, how many times the
  document has loaded in this tab, and every launch received by `window.launchQueue`.
- `android/` is a TWA using androidbrowserhelper's stock `LauncherActivity`, no custom code, with
  `LAUNCH_HANDLER_CLIENT_MODE` set to `focus-existing`.

## Setup

1. Host `web/` at the **root** of an HTTPS origin, so `/.well-known/assetlinks.json` is served.
   The page must be served at any path. `firebase.json` sets this up for Firebase Hosting,
   including serving the `.well-known` directory, which Firebase ignores by default:

   ```
   firebase deploy --only hosting --project <project id>
   ```

   It deploys to the hosting site named in `firebase.json`; change `site` to one in your project.
   The hosted copy is at https://twa-launch-handler-repro.web.app.

   Any other static host works too, e.g. a GitHub Pages user site `https://<user>.github.io`.
2. Put the signing certificate's SHA-256 fingerprint in `web/.well-known/assetlinks.json`, replacing
   the one there. For a debug build:

   ```
   keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android | grep SHA256
   ```

3. Build and install the app for that origin:

   ```
   cd android
   ./gradlew assembleDebug -Porigin=https://<your origin>
   adb install -r app/build/outputs/apk/debug/app-debug.apk
   ```

4. Open the app from the launcher. It should show no URL bar, i.e. it is running as a verified TWA.

## Steps

Each step launches the app with a URL while it is running, the same way a notification or app link
does (`VIEW` intent to the launcher activity). Replace `ORIGIN` with your origin.

```
launch() {
  adb shell "am start -a android.intent.action.VIEW -f 0x30000000 -d '$1' \
    -n com.example.twalaunchhandler/com.google.androidbrowserhelper.trusted.LauncherActivity"
}
```

1. Open the app from the launcher. Wait a few seconds so the uptime counter is clearly non-zero.
2. `launch 'ORIGIN/'` - the URL the app was launched with.
3. `launch 'ORIGIN/#foo'` - a different fragment.
4. `launch 'ORIGIN/?page=2'` - a different query.
5. `launch 'ORIGIN/page/2'` - a different path.

After each step, note whether "Seconds since this document loaded" kept counting and
"Document loads in this tab" stayed the same (no reload), and what was added to the launches list.

## Expected

With `focus-existing`, every step focuses the existing document without navigating it, and appends
the launched URL to the launches list.

## Observed (Chrome 156.0.8078.75, Android 15 emulator)

- Steps 2, 3 and 4: as expected. No reload, and the launched URL is delivered to `launchQueue`.
- Step 5: the existing document is navigated to the new URL. The uptime counter resets and the load
  count goes up.
