# AURO Music — lightweight ad‑free YouTube music player for Android

<p align="center"><img src="docs/icon.png" width="96" alt="AURO Music icon"></p>

A ~1.7 MB Android app that searches online, streams the **audio‑only** track
(no ads, no video decoding) and plays it in the background with a proper
media notification.

| | |
|---|---|
| Release APK size | **≈1.7 MB** (R8‑minified, resources shrunk) |
| Min Android | 7.0 (API 24) → target 14 (API 34) |
| RAM while playing | typically 35–60 MB (no artwork, no WebView, no Compose) |
| CPU | hardware AAC decode + audio offload when EQ is off (CPU sleeps between buffer refills) |

## Features

- 🔍 **Search** songs (YouTube Music "songs" filter, falls back to videos) — **double‑tap** a result to add it to the queue and play it; **+** adds it to the queue without interrupting playback
- ⏮ ▶ ⏸ ⏭ **Prev / Play‑Pause / Next** — also from the notification, lock screen, headset buttons, Bluetooth
- 📊 **Progress bar** with seeking + buffered indicator
- 🎚 **Equalizer** — device presets, per‑band sliders, bass boost (settings persist)
- 🔁 **Autoplay** — related songs are appended only in the last 30 s of the final queued song (toggle); they are marked "autoplay" in the queue
- 🔀 **Shuffle**, 🔂 **Repeat one / all / off**
- ⬇ **Download** to `Music/AURO Music/` (long‑press a result, or the ⬇ button for the current song) — shows progress in a notification
- 📴 **Offline mode** — 📁 button lists downloaded songs (plus any other music on the phone); double‑tap to play, long‑press to delete. Long‑press the 📁 button (or tap the status line) to **add your own folders** via the system picker — phone or SD card, scanned recursively, remembered across restarts. Folders whose name starts with `ring`, `record`, `audio`, `whatsapp` or `voice`, and anything under `Android/media`, are never scanned. Songs you've downloaded play from disk automatically even when picked from search/autoplay, and with no internet the player skips to the next offline song instead of failing
- 😴 **Sleep timer** — presets 5 min–1.5 h or custom, optional "finish current song", 1.5 s fade‑out; runs inside the service so it works with the screen off
- 📋 Queue view (tap the list icon): tap to jump, long‑press to remove, `+` on a result to enqueue
- Pauses on headphone unplug, respects audio focus (calls, other apps)

## Install

Download the latest `AURO-Music-vX.Y-release.apk` from the **Releases** page, then
```
adb install AURO-Music-v1.11-release.apk
```
or copy it to your phone and open it (allow "install unknown apps").

> The release build is signed with the debug keystore so it can be side‑loaded straight away.
> For distribution, add your own `signingConfig` in `app/build.gradle.kts`.

## Build from source

Requirements: JDK 17, Android SDK (platform 34, build‑tools 34). Then:

```
./gradlew assembleRelease
```

Or open the folder in Android Studio (Hedgehog or newer).

## How it stays light

| Decision | Why |
|---|---|
| **No video** — only `MediaCodecAudioRenderer` is instantiated | Video renderers / surfaces are the biggest RAM+CPU consumer in players |
| **Audio‑only stream selection** (M4A/AAC 128 kbps preferred) | Hardware decoders on every SoC; Opus is the fallback |
| **Audio offload** enabled when EQ is off | Decoding happens in the DSP; app process idles |
| **Small buffers** — 15 s min / 45 s max / 2 MB cap | Default ExoPlayer buffers are 50 s+ and tens of MB |
| **Platform widgets only** (`Activity`, `ListView`, `SeekBar`) | No AppCompat/Material/Compose → ~4 MB less dex, far less heap |
| **No image loading** — no thumbnails, no artwork | Bitmaps are the #1 source of memory bloat in music apps |
| **`HttpURLConnection`** instead of OkHttp | Saves ~1.5 MB of code and extra thread pools |
| **NewPipeExtractor** for stream URLs | Pure‑Java, no WebView, no Google Play Services, no tracking; gives ad‑free direct CDN URLs |
| Single low‑priority worker thread for all network extraction | No contention with the audio thread |
| R8 full‑mode + resource shrinking, English resources only | 6.8 MB debug → 1.7 MB release |
| 1 Hz UI progress ticker, only while the screen is visible | Zero UI work in the background |

## Project layout

```
app/src/main/java/com/ytlite/player/
├─ App.kt            – initialises NewPipeExtractor
├─ YtDownloader.kt   – tiny HttpURLConnection client for the extractor
├─ YouTube.kt        – search(), resolve() (cached), related() for autoplay
├─ Song.kt           – model ⇄ MediaItem
├─ PlayerService.kt  – Media3 MediaSessionService: ExoPlayer, EQ, autoplay
├─ Downloader.kt     – MediaStore download with progress notification
└─ MainActivity.kt   – single‑screen UI via MediaController
```

## Notes / limitations

- The extractor calls `URLEncoder.encode(String, Charset)` (Java 10 / Android 13 API). The project uses
  `desugar_jdk_libs_nio` 2.1.x, which back-ports it to Android 7–12 — don't downgrade it to the
  plain `desugar_jdk_libs` artifact or search will crash on devices below Android 13.

- YouTube changes its site regularly; if searching/playback suddenly breaks, bump
  `com.github.TeamNewPipe:NewPipeExtractor` in `app/build.gradle.kts` to the newest tag
  (`v0.26.5` at the time of writing).
- Stream URLs are valid for a few hours; the app caches them for 3 h and re‑resolves after that.
- Equalizer availability depends on the device's audio HAL; a toast is shown if unsupported.
- Downloads are for personal/offline use — please respect the content owners' rights and
  YouTube's Terms of Service in your jurisdiction.

## License

GPL‑3.0 — see [LICENSE](LICENSE). This app depends on
[NewPipeExtractor](https://github.com/TeamNewPipe/NewPipeExtractor) (GPL‑3.0), so the
app itself must be distributed under the GPL as well.
