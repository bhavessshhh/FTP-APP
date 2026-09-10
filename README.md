# FTP — Future Trading Price Calculator (Android)

A native Android wrapper around the FTP calculator, with the calculator's state
stored on the device and a Data panel for inspecting and deleting it.

- **Package** `com.ftpcalc.app`
- **minSdk** 24 (Android 7.0) · **targetSdk / compileSdk** 34
- **Language** Kotlin · **Build** Gradle 8.7 + AGP 8.5.2

---

## Build it

There are two ways. The cloud one needs nothing installed on your machine.

### A. Let GitHub build it (no Android Studio, no SDK)

`.github/workflows/build-apk.yml` is already in this project. GitHub installs the
JDK and the Android SDK on a throwaway machine, runs the build, and hands you the
APK as a download.

1. Create a new **private** repository on github.com. Do not add a README.
2. On the empty repo page, click **uploading an existing file**, then drag in
   everything from *inside* this folder — `app`, `gradle`, `gradlew`,
   `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, and the
   hidden `.github` folder. The important part is that `gradlew` ends up at the
   top level of the repo, not nested inside another folder, and that `.github`
   comes along, or the build will not trigger.
3. Commit. Open the **Actions** tab; a run called *Build APK* starts by itself
   and takes roughly five minutes.
4. When the green tick appears, open the run and download **FTP-app-debug-apk**
   from the Artifacts section at the bottom.
5. Unzip it, copy `app-debug.apk` to your phone, and tap it. Android will ask you
   to allow installs from unknown sources the first time.

The APK is signed with Android's standard debug key, which is fine for your own
devices. For the Play Store you would need a release key instead.

### B. Android Studio (if you want to edit the code)

The project has not been compiled here — the Android SDK is not installable in
the environment it was written in — so the first build on your machine is also
the first real compile. Everything it needs is in the repo.

1. **File → Open**, pick the `FTPCalculator` folder.
2. Accept the SDK-download prompts when it offers them.
3. Press **Run**.

### C. Command line

Point the build at your SDK, then assemble:

```bash
echo "sdk.dir=$HOME/Android/Sdk" > local.properties   # or wherever yours lives
./gradlew assembleDebug
```

The APK lands at `app/build/outputs/apk/debug/app-debug.apk`. Install it with
`./gradlew installDebug`, or `adb install -r <path>`.

For a Play-ready build you will need a signing config; `./gradlew assembleRelease`
produces an unsigned APK until you add one.

---

## How it is put together

The calculator is a single self-contained HTML file. Rather than rewrite ~2,500
lines of pricing maths in Kotlin, the app hosts it and gives it real native
capabilities.

```
MainActivity ──► WebView ──► https://appassets.androidplatform.net/assets/index.html
                   │                          (WebViewAssetLoader → app/src/main/assets)
                   └──► FtpStore  ◄── JavaScript calls `FtpStore.save(...)`
                          │
                          └──► filesDir/ftp_state.json
```

**Why a synthetic https origin and not `file://`.** A `file://` page is treated
as an opaque origin, and DOM storage behaves inconsistently across OEM WebView
builds. `WebViewAssetLoader` serves the same assets over
`https://appassets.androidplatform.net`, which gives the page ordinary web
semantics and keeps the JavaScript bridge reachable only from our own content.

**Storage.** `FtpStore.kt` writes one JSON file to `filesDir` — app-private
internal storage. No permission is required, no other app can read it, and it
disappears on uninstall or on *Settings → Apps → FTP → Storage → Clear storage*.
Writes go to a temp file and are then renamed, because the page saves on a
debounce while you type and a process death mid-write would otherwise leave
unparseable JSON to be read back on next launch.

**What gets saved:** every input (symbol, spot, strike, rate, dividend, both
dates, T), Call/Put, Market/Manual IV, typed per-strike IVs, the selected theme,
and any option chain imported from CSV. The bundled sample chain is deliberately
*not* saved — it is already in the page, and freezing a copy of it would mean
app updates could never change it.

**Deleting it.** The header has a **Data** button. It shows where data lives, how
many bytes are used, which chain is loaded and how many typed IVs exist, and
offers three actions: clear typed IVs, restore the sample chain, and delete
everything. The last one wipes the file *and* resets the running page, so the
result is visible immediately rather than only after a restart.

**Theming.** The page has twelve themes across light and dark, so a fixed status
bar would clash with most of them. `setTheme()` in the page passes the active
background colour to `FtpStore.themeColor()`, which tints the status and
navigation bars and flips the icons to light or dark based on relative
luminance.

**CSV import.** `<input type="file">` needs a host-side picker;
`WebChromeClient.onShowFileChooser` opens the system picker and hands the URI
back. The callback is always answered, including on cancel — dropping it leaves
the import button permanently dead.

**External links.** The NSE and TradingView links open in the user's browser via
an intent; everything on the asset origin stays in the WebView.

---

## Icon

`FTP` in DejaVu Sans Mono Bold on a navy-to-azure diagonal, matching the app's
default Azure accent. Supplied as an adaptive icon (separate background and
foreground layers, plus a monochrome layer for themed icons on Android 13+) with
legacy square and round PNGs for older launchers, at all five densities.
`ic_launcher-playstore.png` in the project root is the 512px store listing icon.

To restyle it, edit and re-run the icon generator described in the delivery
notes, or replace the PNGs in `app/src/main/res/mipmap-*/`.

---

## Things worth knowing

- **Fonts.** The page pulls Space Grotesk and JetBrains Mono from Google Fonts,
  so first launch on a new device wants a network connection to look exactly
  right. Offline it falls back to the system sans and mono, which is a visible
  but not harmful change. To make it fully offline, drop the `.woff2` files into
  `assets/fonts/` and swap the `<link>` in `index.html` for a local
  `@font-face` block.
- **`INTERNET` permission** is only there for those fonts and the two outbound
  links. All pricing, all storage, and the CSV import are local.
- **Backup rules** include `ftp_state.json` in cloud backup and device transfer,
  so a saved session follows the user to a new phone. Remove it from
  `res/xml/data_extraction_rules.xml` and `res/xml/backup_rules.xml` if you would
  rather it did not.
- **`isMinifyEnabled = false`** for release. There is very little Kotlin to
  shrink, and R8 would need keep rules for the `@JavascriptInterface` methods —
  `proguard-rules.pro` already has them if you turn it on.
- **Updating the calculator.** `app/src/main/assets/index.html` is the whole app.
  Replace it and rebuild; nothing in the Kotlin needs to change unless you rename
  the element IDs the bridge touches.
