# DeckLink AV Test Pattern Generator

Web interface to send video test patterns and audio tones to a Blackmagic DeckLink SDI output.

Built for macOS. Requires a custom FFmpeg build with DeckLink support — see installation below.

---

![Test pattern output](docs/test-pattern-output.png)
*Color bars with logo overlay, NTP-synced clock and settings overlay*

![Web interface](docs/web-interface-main.png)
*Main controls: background, text, logo, animations, video format, DeckLink device*

![Audio section](docs/web-interface-audio.png)
*8-channel audio: per-channel enable, ID cycle, 1-frame flash, 400Hz force*

---

## What it does

- Outputs test patterns to a DeckLink card via SDI
- Video formats: 1080i50, 1080i60, 1080p25/30, 720p50/60, 576i50
- Test sources: SMPTE bars, PAL bars, color bars, gradients, fractals, custom image upload
- 8-channel audio with configurable tone (frequency + level), ID cycling, 1-frame flash
- On-screen text, logo overlay (PNG upload), NTP-synchronized clock
- Preset save/load, real-time FFmpeg log, web monitor (HLS preview)
- Auto-start on server launch via env var

---

## Requirements

- **macOS** (Catalina or later, Apple Silicon supported)
- **Blackmagic Desktop Video 14.x drivers** — [download here](https://www.blackmagicdesign.com/support/family/capture-and-playback)
- **Blackmagic DeckLink SDK 14.x** — same support page
- **A DeckLink card** connected (UltraStudio, Mini Monitor, etc.)
- **FFmpeg compiled with `--enable-decklink`** — see below
- **Node.js >= 18**
- **Homebrew packages:** `freetype`, `fontconfig`, `harfbuzz`

---

## FFmpeg

The Homebrew FFmpeg does not include DeckLink support. You need to compile it yourself.

### 1. Install Homebrew dependencies

```bash
brew install freetype fontconfig harfbuzz
```

### 2. Download the Blackmagic DeckLink SDK

Download **SDK 14.x** from the [Blackmagic support page](https://www.blackmagicdesign.com/support/family/capture-and-playback) (free, same page as Desktop Video drivers).


### 3. Run configure

Set `PKG_CONFIG_PATH` to include all three libraries, then run configure. Replace `/path/to/decklink-sdk/include` with the actual path to your SDK's `include` directory (e.g. `~/Library/DecklinkSDK/include`).

```bash
PKG_CONFIG_PATH="/opt/homebrew/opt/freetype/lib/pkgconfig:/opt/homebrew/opt/fontconfig/lib/pkgconfig:/opt/homebrew/opt/harfbuzz/lib/pkgconfig" \
./configure \
  --enable-gpl \
  --enable-nonfree \
  --enable-decklink \
  --enable-libfreetype \
  --enable-libharfbuzz \
  --enable-libfontconfig \
  --extra-cflags="-I/path/to/decklink-sdk/include" \
  --extra-cxxflags="-I/path/to/decklink-sdk/include" \
  --extra-ldflags="-F/Library/Frameworks -framework DeckLinkAPI -framework CoreFoundation"
```

### 4. Patch `libavdevice/decklink_common.h`

After configure, before running `make`, apply this patch. `IID_IUnknown` is a Windows COM identifier and is not defined on macOS. Open `libavdevice/decklink_common.h` and add the following block immediately after the `#include <DeckLinkAPIVersion.h>` line:

```cpp
#ifndef IID_IUnknown
static const REFIID IID_IUnknown = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};
#endif
```

Without this patch, compilation fails with:
```
error: use of undeclared identifier 'IID_IUnknown'
```

### 6. Compile

```bash
make -j$(sysctl -n hw.ncpu)
```

Compilation takes 15–30 minutes. Put the binary somewhere stable (e.g. `~/ffmpeg-decklink/bin/ffmpeg`) and set `FFMPEG_PATH` in `.env`.

### 7. Verify DeckLink support

```bash
/path/to/your/ffmpeg -hide_banner -sinks decklink
# Should list your connected card, e.g.:
# [0] UltraStudio Monitor 3G
```

---

## Install

```bash
git clone https://github.com/stephanebhiri/decklink-AV-test-pattern-generator.git
cd decklink-AV-test-pattern-generator
bash install.sh
```

The script checks Node.js, installs npm dependencies, verifies FFmpeg and drivers, and creates a `.env` template.

---

## Configuration

Edit `web-interface/.env` (created by `install.sh`):

```env
# Path to your FFmpeg binary (required, must have DeckLink support)
FFMPEG_PATH=/path/to/ffmpeg

# Path to your logo PNG (optional, transparent background recommended)
LOGO_PATH=/path/to/logo.png

# Server port (default: 3000)
PORT=3000

# Auto-start broadcast when the server starts
AUTO_START_BROADCAST=false
AUTO_START_DELAY_MS=5000
```

> **`FFMPEG_PATH` must point to your custom-compiled binary**, not the Homebrew one. The Homebrew FFmpeg has no DeckLink support and will return no devices.

---

## DeckLink device name

The app detects devices by running `ffmpeg -sinks decklink`. The name shown there must be used exactly as-is in the UI.

Verify your device name:
```bash
/path/to/your/ffmpeg -hide_banner -sinks decklink
```

Use the name shown in brackets, e.g. `UltraStudio Monitor 3G`, not a generic or shortened form.

> **An active SDI connection is required.** The DeckLink device must have an SDI cable connected to a display or downstream device before FFmpeg can configure its output. Without an active connection, FFmpeg will fail with:
> ```
> Could not write header: Device not configured
> ```

---

## Run

```bash
cd web-interface
node server.js
```

Open [http://localhost:3000](http://localhost:3000).

---

## Notes

- Tested with UltraStudio Monitor 3G, UltraStudio Mini Monitor, and UltraStudio 4K
- NTP sync runs on startup and every 15 minutes (Cloudflare, Google, NIST)
- Uploaded logos and backgrounds persist in `web-interface/uploads/`
- FFmpeg logs stream live in the web UI
- `SIGINT` and `SIGTERM` both cleanly stop the FFmpeg child process

---

## Troubleshooting

**`error: use of undeclared identifier 'IID_IUnknown'`** — Apply the `decklink_common.h` patch described in step 4 of the FFmpeg section above.

**`DeckLinkAPIVersion.h not found` during `make`** — `--extra-cflags` alone is not enough. You must also pass `--extra-cxxflags` with the same SDK include path.

**`No such filter: 'drawtext'` / `Error: Filter not found`** — FFmpeg was compiled without harfbuzz support. Recompile with `--enable-libharfbuzz` and ensure `harfbuzz` is in `PKG_CONFIG_PATH`.

**`Unknown input format: decklink`** — FFmpeg was compiled without `--enable-decklink`.

**No devices in UI dropdown** — Verify that `FFMPEG_PATH` in `.env` points to your custom FFmpeg binary, not the Homebrew one. The Homebrew FFmpeg has no DeckLink support and will return no devices.

**`Could not write header: Device not configured`** — Check that an SDI cable is connected and the device name in the UI matches exactly what `ffmpeg -sinks decklink` reports.

**Blank output / crash on start** — Check the FFmpeg log in the web UI. Usually a format mismatch with the card.

**No DeckLink sinks detected** — Check that Desktop Video is installed, the card is connected, and run `ffmpeg -hide_banner -sinks decklink`.
