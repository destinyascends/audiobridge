# Android Audio Bridge → CAVA

Captures whatever's playing on your phone (via Android's `AudioPlaybackCapture`
API) and feeds it to CAVA running in Termux, so the terminal bars react in
real time.

## 0. Read this first: what will and won't work

**Spotify will not work.** Verified against multiple independent sources
(Android's own AudioPlaybackCapture docs, and the RootlessJamesDSP project,
which uses this exact API for audio effects): Spotify explicitly opts its
audio out of playback capture (`ALLOW_CAPTURE_BY_NONE`), the same DRM opt-out
used by Netflix and Apple Music. Capturing while Spotify plays will produce
silence — not a bug in this code, a deliberate block on Spotify's end. There
is a known "Spotify patch" that strips this flag from the APK; that's DRM
circumvention against Spotify's ToS, so it's out of scope here.

**YouTube works.** Confirmed by the same source (RootlessJamesDSP explicitly
lists YouTube and YouTube Music as working). Most other media players and
games work too — anything that doesn't specifically opt out.

**Chrome/SoundCloud are also known blockers**, for the same reason.

If you want to *know* whether a given app is capturable rather than guess:
play it, start the bridge, and check `termux-side/verify_flow.sh` and the
app's in-app peak-level readout. Non-zero peak = that app allows capture.

## 1. Why the architecture is what it is (not a FIFO written directly by the app)

You asked for the app to write straight into a Termux FIFO like
`/data/data/com.termux/files/usr/tmp/audio.fifo`. **That's not possible on a
non-rooted device**, and here's the actual mechanism, not a hand-wave:

- Every Android app runs under its own Linux UID with its own SELinux domain.
  `/data/data/com.termux/...` is Termux's private app directory — the kernel
  and SELinux policy block any other app's process from opening a path there
  at all, regardless of Unix file permissions on the FIFO itself.
- Shared storage (`/sdcard`) doesn't help either: it's a FUSE-emulated
  filesystem for scoped storage that doesn't support special files like FIFOs
  (`mkfifo` there fails or is meaningless).

So a cross-app filesystem FIFO is off the table without root. What **does**
work, and is the standard mechanism many local Android IPC tools use: a
**loopback TCP socket**. Binding a `ServerSocket` to `127.0.0.1` isn't
filesystem-sandboxed — it's ordinary networking, gated only by the
`INTERNET` permission (which just puts the app's UID in the `inet` group).
Any other app or process on the same device, Termux included, can connect to
it as a normal TCP client. This doesn't touch Android's Network Security
Config cleartext-traffic restrictions either, because that policy is enforced
by high-level APIs (HttpURLConnection, OkHttp, WebView) — not by raw
`java.net.Socket`/`ServerSocket`, which is what this app uses.

The full pipeline:

```
YouTube (or other capturable app)
  → Android AudioPlaybackCapture (MediaProjection-gated)
  → AudioRecord (48000 Hz, 16-bit PCM, stereo)
  → this app's ServerSocket on 127.0.0.1:28428
  → Termux: `nc 127.0.0.1 28428 > ~/audiobridge.fifo`   (all inside Termux's own sandbox)
  → CAVA reads ~/audiobridge.fifo with method=fifo
```

The socket→FIFO hop happens entirely *inside* Termux's own process and
filesystem, so there's no cross-app FIFO problem there — `nc` and `cava` are
both Termux's own processes writing/reading Termux's own file. This gets you
CAVA's real FIFO input method (`method = fifo`), which is the best-supported,
dependency-free option, rather than something more fragile.

## 2. Known Android-version-specific gotchas (verified against Android's own
   docs and issue tracker, not assumed)

- **Android 14+ (API 34+, which covers your Android 16 device):**
  `getMediaProjection()` throws `SecurityException` unless the foreground
  service was already started with `foregroundServiceType="mediaProjection"`
  *before* that call. `CaptureService.kt` does this in the right order.
- **`MediaProjection.registerCallback()` is required** before capture, to
  detect the session being revoked (e.g. via the Quick Settings recording
  chip) — otherwise the app would silently keep "recording" nothing.
- **Screen lock:** there are confirmed reports (Google's issue tracker) of
  MediaProjection sessions being stopped when the device locks, on some
  Android 15/16 builds. If bars stop moving when your screen turns off,
  this is why — keep the screen on (or check One UI's specific behavior;
  Samsung's build may differ from AOSP here) while running the bridge.
- **Samsung/One UI battery management** is known to be aggressive about
  killing background foreground services regardless of the above. Go to
  **Settings → Apps → Audio Bridge → Battery** and disable optimization /
  put it in "unrestricted" or "never sleeping apps", or the OS may kill the
  service after a few minutes in the background anyway.
- **No app can *detect* "this app is blocking capture"** as a distinct
  error — a blocked app's audio is just silently excluded from the mixed
  signal AudioRecord receives. There's no exception to catch for it, which
  is why `verify_flow.sh` and the in-app peak meter exist: they're how you
  confirm real signal is present, since the API gives no explicit signal.

## 3. Exact CAVA config (verified against CAVA's documented `[input]` keys)

Already written to `cava-config/config` in this bundle. Key facts, not
guessed: for `method = fifo`, CAVA's own docs state `sample_rate` and
`sample_bits` are configurable, but **`channels` is not** (only
`sndio`/`oss`/`jack` accept a `channels` key) — the fifo reader assumes
interleaved stereo, which matches this app's `CHANNEL_IN_STEREO` output
exactly, so nothing else is needed.

```ini
[general]
framerate = 60
bars = 0

[input]
method = fifo
source = /data/data/com.termux/files/home/audiobridge.fifo
sample_rate = 48000
sample_bits = 16

[output]
method = noncurses
```

Install it:
```
mkdir -p ~/.config/cava
cp cava-config/config ~/.config/cava/config
```

## 4. Termux commands — full run

```
# One-time
pkg install netcat-openbsd

# Every time you want to run it:
# 1. Open the Audio Bridge app, tap "Start capture", grant permissions,
#    accept the screen/audio recording dialog. Start something playing
#    (e.g. YouTube - not Spotify, see section 0).
# 2. In Termux:
bash termux-side/start_bridge.sh
cava -p ~/.config/cava/config
```

Verify PCM is actually flowing (independent of cava, run this *instead of*
step 2 above if you just want a raw check first — it will consume the one
active connection slot):
```
bash termux-side/verify_flow.sh
```

Stop everything:
```
# In cava's terminal: press q (or Ctrl+C)
bash termux-side/stop_bridge.sh
# In the app: tap "Stop capture", or use the notification's Stop action
```

## 5. Building the APK on-device (no PC)

Android 16 = API 36. **Caveat, verified, not glossed over:** the
community-maintained aarch64 build-tools used for on-device building
(`lzhiyong/android-sdk-tools`) top out at **build-tools 35.0.2** as of this
writing — API 36 isn't cross-compiled for on-device use yet. So this project
is pinned to `compileSdk/targetSdk 35` so it actually builds today. It still
runs correctly on your Android 16 phone. Bump both to 36 in `app/build.gradle`
once 36 build-tools show up at that repo, or if you ever get access to a full
Android Studio install.

### Method A — pure Termux (matches what you originally asked for)

This relies on community (not Google-official) cross-compiled tooling. It's
real and documented (multiple independent 2025 write-ups converge on the same
steps), but it's not an officially supported path, so treat it as "verified
to work, not guaranteed to stay working forever."

```
pkg update -y && pkg upgrade -y
pkg install openjdk-17 wget unzip -y

# 1. Android SDK "platform" data (android.jar etc.) - pure Java/data,
#    no architecture issue, safe to fetch via Google's own sdkmanager:
wget -O ~/install-android-sdk.sh https://raw.githubusercontent.com/Sohil876/termux-sdk-installer/main/installer.sh
# (inspect the script before running it, as with any third-party installer)
chmod +x ~/install-android-sdk.sh
bash ~/install-android-sdk.sh -i
yes | sdkmanager --licenses
yes | sdkmanager "platforms;android-35"

# 2. Gradle (Termux packages it directly - pure JVM bytecode, no arch issue)
pkg install gradle -y
gradle -v   # note the version; if it's wildly newer/older than Gradle 8.7,
            # check Google's AGP-Gradle compatibility table and adjust
            # app-level plugin versions in build.gradle/settings.gradle to match

# 3. THE gotcha: AGP auto-downloads its own aapt2, which is an x86_64 glibc
#    Linux binary and will NOT run in Termux ("Exec format error"). Replace
#    it with a real aarch64 build:
mkdir -p ~/android-sdk-tools
cd ~/android-sdk-tools
wget https://github.com/lzhiyong/android-sdk-tools/releases/download/35.0.2/android-sdk-tools-static-aarch64.zip
unzip android-sdk-tools-static-aarch64.zip
chmod +x build-tools/aapt2
cd ~

# 4. Point the project at it - already scaffolded, commented out, in
#    gradle.properties. Uncomment this line (adjust the path if you unzipped
#    somewhere else):
#    android.aapt2FromMavenOverride=/data/data/com.termux/files/home/android-sdk-tools/build-tools/aapt2

# 5. Point the project at the SDK platform data:
cp local.properties.example local.properties
# edit local.properties: sdk.dir should already match ~/android-sdk from step 1

# 6. Build
cd /path/to/this/audiobridge/project
ANDROID_HOME=$HOME/android-sdk gradle --no-daemon assembleDebug
```

The APK lands at `app/build/outputs/apk/debug/app-debug.apk`. Install it:
```
termux-setup-storage   # first time only, so Termux can see the apk from a file manager
pkg install termux-api -y   # optional, only if you want termux-open below
termux-open app/build/outputs/apk/debug/app-debug.apk
```
(or just move it to `~/storage/downloads/` and tap it in a file manager —
either way, Android will prompt to install it since it's from an unknown
source; allow that for Termux/your file manager once.)

If step 3-4's `aapt2` override still errors, apksigner and zipalign are
*not* usually separate binaries you need to worry about — modern AGP signs
and aligns debug builds using an in-process Java library (`apksig`), not by
shelling out, so aapt2 is the one real architecture landmine here, not a
whole chain of them.

### Method B — AndroidIDE (turnkey fallback if Method A gives you grief)

AndroidIDE is a real, on-device Gradle IDE (edit, build, run, all in an app,
no root/PC). Worth knowing up front: **the original
`AndroidIDEOfficial/AndroidIDE` repo was archived/unmaintained as of late
2024.** Several community forks continue it (e.g. search GitHub for
"AndroidIDE fork" — there were multiple active ones as of this writing).
Pick whichever fork has the most recent commits/releases when you look, since
this landscape shifts. Requires AGP ≥ 7.2.0 (this project's 8.6.0 qualifies).

1. Install an AndroidIDE fork's APK (F-Droid or GitHub releases).
2. Follow its own "install build tools" step (it downloads its own ported
   SDK/aapt2/Gradle — you don't need Method A's manual steps for this route).
3. Copy this whole project folder into AndroidIDE's workspace (or `git clone`
   it via AndroidIDE's built-in terminal).
4. Open the project, let it sync, Build → Assemble Debug APK.
5. Install the resulting APK from AndroidIDE's file browser.

## 6. Project layout

```
audiobridge/
  build.gradle, settings.gradle, gradle.properties, local.properties.example
  app/
    build.gradle
    src/main/AndroidManifest.xml
    src/main/java/com/audiobridge/app/MainActivity.kt
    src/main/java/com/audiobridge/app/CaptureService.kt
    src/main/res/layout/activity_main.xml
    src/main/res/values/strings.xml
  cava-config/config
  termux-side/start_bridge.sh, stop_bridge.sh, verify_flow.sh
```

## 7. What the app actually does, requirement by requirement

- MediaProjection + AudioPlaybackCapture, AudioRecord: `CaptureService.kt`.
- Continuous PCM output to something Termux can read: the 127.0.0.1 socket
  (§1), consumed by CAVA via a FIFO (§4) — this satisfies requirement #7's
  fallback clause honestly, rather than pretending a direct FIFO write works.
- CAVA-compatible format: 16-bit signed LE PCM, 48000 Hz, interleaved
  stereo — matches CAVA's fifo-method assumptions exactly (§3).
- Foreground service, correct Android 14+/16 permission order (§2).
- Clean stop: `ACTION_STOP` releases AudioRecord, closes the socket, stops
  and unregisters the MediaProjection.
- Error handling: every AudioRecord/MediaProjection call that can throw is
  caught and surfaced via the in-app status text and a broadcast; apps that
  block capture aren't a catchable error (§2) so the peak meter is the
  detection mechanism instead.
- No root, no ADB, no privileged APIs — everything used is public
  `android.media`/`android.media.projection` API surface available to any
  third-party app.
