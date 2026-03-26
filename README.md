# ESP32-CAM - Hellboy's WatchTower v1.0

### ESP32-CAM Complete Surveillance Dashboard System

A full-featured WiFi surveillance system for the **AI Thinker ESP32-CAM** (HW-381 board, OV2640 sensor). Streams live video, captures photos, records AVI video, detects motion, and sends alerts via Telegram, all controlled from a built-in web dashboard or Telegram bot commands.

---

## Table of Contents

1. [Hardware Requirements](#hardware-requirements)
2. [Required Libraries](#required-libraries)
3. [Arduino IDE Board Settings](#arduino-ide-board-settings)
4. [First-Time Setup](#first-time-setup)
5. [Access Point (AP) Mode](#access-point-ap-mode)
6. [Web Dashboard](#web-dashboard)
7. [Live Stream](#live-stream)
8. [Photo Capture](#photo-capture)
9. [Video Recording](#video-recording)
10. [Motion Detection](#motion-detection)
11. [Telegram Bot](#telegram-bot)
12. [SD Card & File Management](#sd-card--file-management)
13. [REST API Reference](#rest-api-reference)
14. [Configuration Reference](#configuration-reference)
15. [Serial Monitor / Debugging](#serial-monitor--debugging)
16. [Troubleshooting](#troubleshooting)
17. [Architecture Notes](#architecture-notes)

---

## Hardware Requirements

| Component | Details |
|-----------|---------|
| Board | AI Thinker ESP32-CAM (HW-381) |
| Camera | OV2640 sensor (included on board) |
| SD Card | MicroSD card (FAT32 formatted, up to 32 GB recommended) |
| Programmer | FTDI adapter or USB-to-Serial (3.3 V logic) for flashing |
| Power | 5 V / 2 A USB supply (the camera is power-hungry) |

> **Wiring for upload:** Connect GPIO0 to GND before powering on to enter flash mode. Remove that wire before normal use.

---

## Required Libraries

Install both via **Arduino IDE → Sketch → Include Library → Manage Libraries**:

| Library | Author | Version |
|---------|--------|---------|
| Universal Arduino Telegram Bot | Brian Lough | latest |
| ArduinoJson | Benoit Blanchon | **v6.x** (not v7) |

The ESP32 camera libraries (`esp_camera.h`, `img_converters.h`) are included automatically with the ESP32 board package.

---

## Arduino IDE Board Settings

Open **Tools** and set every option exactly as shown:

| Setting | Value |
|---------|-------|
| Board | **AI Thinker ESP32-CAM** |
| Partition Scheme | **Huge APP (3MB No OTA / 1MB SPIFFS)** |
| CPU Frequency | **240 MHz** |
| Upload Speed | **115200** |
| Port | Your FTDI COM / tty port |

> ⚠️ The **Huge APP** partition scheme is mandatory. The standard partition is too small and the firmware will not fit.

---

## First-Time Setup

Because credentials are stored in non-volatile storage (NVS) rather than hardcoded, the first boot works like this:

### Step 1 — Flash the firmware
Upload `HellboysWatchTower_v1.8.ino` with the board settings above.

### Step 2 — Connect to the AP
On first boot (or whenever no saved WiFi is found), the device starts as a **WiFi Access Point**:

| Field | Value |
|-------|-------|
| SSID | `ESP32HBWT` |
| Password | `watchtower` |
| Dashboard URL | `http://192.168.4.1` |

Connect your phone or laptop to that WiFi network, then open the URL in a browser.

### Step 3 - Enter your credentials via the Settings panel
In the dashboard, open the **Settings** tab and fill in:

- **WiFi SSID** - your home/office router name
- **WiFi Password** - your router password
- **Telegram Bot Token** - from `@BotFather` (see [Telegram Bot](#telegram-bot))
- **Telegram Chat ID** - from `@userinfobot`

Click **Save**. The device will restart, connect to your router, and send a confirmation message to Telegram.

> Credentials are saved to flash (NVS) and survive reboots. You only need to do this once.

---

## Access Point (AP) Mode

If the saved WiFi credentials are missing or the router is unreachable, the device automatically falls back to AP mode. All dashboard features work in AP mode except:

- Telegram alerts (requires internet)
- NTP time sync (filenames will use `notime_<millis>` format)

---

## Web Dashboard

The dashboard is served on **port 80** (`http://<device-ip>/`). It is a single-page app with the following tabs:

### Live Tab
- Starts/stops the MJPEG stream
- Resolution selector (QVGA → UXGA)
- FPS selector (1–30)
- **Capture Photo** button (with optional flash toggle)
- **Record Video** button (5–120 s)

### Motion Tab
- **Arm / Disarm** motion detection
- Sensitivity slider (1 = least sensitive, 100 = most sensitive)
- Cooldown slider (seconds between repeated alerts)
- Real-time status showing warm-up progress

### Events Tab
- Paginated list of all photos and videos on the SD card
- Thumbnail preview for photos
- Download and delete per-file buttons
- Delete all files button

### Settings Tab
- WiFi SSID / Password
- Telegram Bot Token / Chat ID
- Save button (triggers automatic reboot to apply)

### Status Bar (always visible)
Shows: uptime, WiFi signal, IP address, CPU temperature, free heap, free PSRAM, photo/video counts, and recording indicator.

---

## Live Stream

The MJPEG stream runs on a **separate server on port 81**:

```
http://<device-ip>:81/stream
```

You can open this URL directly in any browser, VLC, or embed it in an `<img>` tag. The main dashboard stream viewer connects to this endpoint automatically.

**Supported resolutions for streaming:**

| Label | Size |
|-------|------|
| QVGA | 320×240 |
| VGA | 640×480 (default) |
| SVGA | 800×600 |
| XGA | 1024×768 |
| SXGA | 1280×1024 |
| UXGA | 1600×1200 |

> Higher resolutions reduce frame rate significantly due to WiFi bandwidth limits. VGA @ 10 fps is the recommended balance.

---

## Photo Capture

Photos are captured at **XGA resolution (1024×768)** with JPEG quality 6 (high detail). The capture process:

1. Stops the stream temporarily
2. Reinitialises the camera at photo resolution
3. Discards 3 warm-up frames so auto-exposure settles
4. Captures the frame (with or without flash)
5. Saves to `/photos/YYYY-MM-DD_HH-MM-SS.jpg` on the SD card
6. Reinitialises the camera back to stream mode
7. Optionally sends the photo to Telegram

**Flash behaviour:** When flash is enabled, the pre-flash frame is discarded automatically so the saved image is always flash-lit.

> If XGA fails (PSRAM limitation), the firmware automatically falls back to VGA.

---

## Video Recording

Videos are saved as **AVI files with MJPEG codec**, directly compatible with VLC, Windows Media Player, and most video players.

**File naming:** `/videos/YYYY-MM-DD_HH-MM-SS_<RES>_<DUR>s.avi`

Example: `/videos/2024-11-15_14-30-22_VGA_30s.avi`

**Specs:**
- Maximum frames: 3000 per file
- Duration: 5–120 seconds (selectable in dashboard or via API)
- Frame rate: auto-calculated from actual capture timing
- Dimensions: auto-detected from the first JPEG frame (correct aspect ratio)
- The stream stays active during recording so you can monitor while recording

---

## Motion Detection

Motion detection is **OFF by default** and must be explicitly armed.

### Algorithm
Uses **Mean Absolute Difference (MAD)** on grayscale frames downscaled to 80×60 pixels. This approach is robust against JPEG compression noise, which would cause false triggers with simple pixel-count methods.

### Trigger logic
1. After arming, the first **20 frames** are discarded as a warm-up (auto-exposure settling)
2. Each frame is compared to the previous one; the MAD is calculated
3. Motion is only confirmed after **2 consecutive frames** exceed the threshold
4. Once triggered, a **cooldown period** (default 8 s) prevents repeat alerts

### What happens on detection
1. A brief flash fires to illuminate the scene (80 ms)
2. A hi-res photo is captured and saved to SD
3. The photo is sent to Telegram with a motion alert caption

### Sensitivity tuning
Open Serial Monitor at 115200 baud and watch the `[MD] MAD=x.xx` output:
- **Idle scene** - MAD should read 0.5–3.0
- **Motion present** - MAD spikes to 10+

The sensitivity slider maps as follows:
- Slider **1** → threshold **25.0** (almost never triggers)
- Slider **50** → threshold **8.0** (default, good for indoor use)
- Slider **100** → threshold **2.0** (triggers on very slight movement)

---

## Telegram Bot

### Creating your bot
1. Open Telegram and search for `@BotFather`
2. Send `/newbot` and follow the prompts
3. Copy the **Bot Token** (looks like `123456789:ABCdef...`)
4. Search for `@userinfobot`, start it, and copy your **Chat ID** (a number)
5. Enter both in the dashboard Settings tab

### Available commands

| Command | Action |
|---------|--------|
| `/start` or `/help` | Show the full command menu |
| `/photo` | Capture a hi-res photo (no flash) |
| `/photoflash` | Capture a hi-res photo **with** flash |
| `/video` | Record a 10-second AVI video |
| `/record30` | Record a 30-second AVI video |
| `/status` | Full system status (WiFi, camera, motion, SD) |
| `/arm` | Arm motion detection |
| `/disarm` | Disarm motion detection |
| `/storage` | SD card usage summary |
| `/restart` | Restart the ESP32 remotely |

### Security
Only messages from the configured Chat ID are processed. All others are silently ignored.

---

## SD Card & File Management

### Folder structure on the card
```
/
├── photos/
│   ├── 2024-11-15_14-30-22.jpg
│   └── ...
└── videos/
    ├── 2024-11-15_14-35-10_VGA_30s.avi
    └── ...
```

### Recommendations
- Use a **Class 10 / UHS-I** card for smooth video recording
- FAT32 format (cards above 32 GB may need manual formatting to FAT32)
- The SD card uses the **1-bit MMC** bus (GPIO2 is used for data, not available as a GPIO while the card is mounted)

### File management
All files can be viewed, downloaded, and deleted individually from the **Events tab** of the dashboard, or deleted all at once with the "Delete All" button.

---

## REST API Reference

All endpoints are on **port 80**. Responses are JSON unless noted.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Dashboard HTML |
| GET | `/api/status` | System status (uptime, WiFi, heap, counts, armed state) |
| GET | `/api/sdstats` | SD card type, total/used/free MB |
| GET | `/api/photo?flash=0` | Queue a photo capture (`flash=1` for flash) |
| GET | `/api/stream/start?res=8&fps=10` | Start stream (optional res/fps override) |
| GET | `/api/stream/stop` | Stop stream |
| GET | `/api/stream/quality?res=8&fps=15` | Change resolution/fps while streaming |
| GET | `/api/motion?armed=1` | Arm (`armed=1`) or disarm (`armed=0`) motion detection |
| GET | `/api/record?secs=30` | Queue a video recording (5–120 s) |
| GET | `/api/events?offset=0&limit=20` | Paginated file list from SD |
| GET | `/api/download?f=/photos/file.jpg` | Download a file from SD |
| GET | `/api/thumb?f=/photos/file.jpg` | Serve a photo as inline image |
| DELETE | `/api/delete?f=/photos/file.jpg` | Delete one file |
| DELETE | `/api/deleteall` | Delete all photos and videos |
| GET | `/api/config` | Read current config (sensitivity, cooldown, WiFi SSID) |
| POST | `/api/config` | Update config (JSON body) |
| GET | `/api/restart` | Restart the ESP32 |

**Resolution codes for `res` parameter:**

| Code | Resolution |
|------|-----------|
| 5 | QVGA 320×240 |
| 8 | VGA 640×480 |
| 9 | SVGA 800×600 |
| 10 | XGA 1024×768 |
| 12 | SXGA 1280×1024 |
| 13 | UXGA 1600×1200 |

---

## Configuration Reference

These `#define` values can be changed at the top of the `.ino` file before flashing:

| Define | Default | Description |
|--------|---------|-------------|
| `AP_SSID` | `ESP32HBWT` | Access point SSID when no router found |
| `AP_PASS` | `watchtower` | Access point password |
| `AP_IP_STR` | `192.168.4.1` | Access point IP address |
| `MD_THRESHOLD_DEFAULT` | `8.0` | MAD motion trigger level |
| `MOTION_SENSITIVITY_DEFAULT` | `50` | Default slider position (1–100) |
| `MOTION_COOLDOWN_DEFAULT_S` | `8` | Seconds between repeat motion alerts |
| `MD_WARMUP_FRAMES` | `20` | Frames discarded on arm before detection starts |
| `NTP_SERVER` | `pool.ntp.org` | NTP time server |
| `GMT_OFFSET_SEC` | `3600` | UTC offset in seconds (3600 = UTC+1) |
| `DST_OFFSET_SEC` | `3600` | Daylight saving offset (3600 = +1 h DST) |
| `STREAM_JPEG_QUALITY` | `12` | Stream JPEG quality (lower = better, larger) |
| `PHOTO_JPEG_QUALITY` | `6` | Photo JPEG quality |
| `PHOTO_FRAMESIZE` | `FRAMESIZE_XGA` | Photo capture resolution |

> **Timezone:** Adjust `GMT_OFFSET_SEC` and `DST_OFFSET_SEC` for your region. For UTC+0 with no DST, set both to `0`. For UTC+5:30 (India), set `GMT_OFFSET_SEC=19800` and `DST_OFFSET_SEC=0`.

---

## Serial Monitor / Debugging

Set baud rate to **115200**. Key log prefixes:

| Prefix | Meaning |
|--------|---------|
| `[CAM]` | Camera init, mode changes, captures |
| `[SD]` | SD card mount, file operations |
| `[MD]` | Motion detection — MAD values, warm-up progress |
| `[AVI]` | Video recording start, frame count, finish |
| `[TG]` | Telegram send/receive |
| `[WEB]` | Web server ready |
| `[STREAM]` | Stream server ready |
| `[RTOS]` | FreeRTOS task creation |
| `[JOB]` | Pending job execution (photo/record) |
| `[FATAL]` | Camera failed to init — check wiring |

---

## Troubleshooting

**Camera init fails (`[FATAL] Camera failed`)**
- Check that GPIO0 is NOT grounded (only needed during flashing)
- Ensure power supply is at least 2 A — the camera draws significant current on init
- Reseat the camera ribbon cable

**Blank/white photos**
- This is an AE/AWB settling issue — the firmware already discards 3 warm-up frames
- If still occurring, increase the `delay(350)` after `initCameraMode` in `captureAndSavePhoto()`

**SD card not detected**
- Format the card as FAT32
- Try a different card; some cheap cards are not compatible with the ESP32-CAM's MMC interface
- Ensure the card is fully seated

**No Telegram messages received**
- Double-check Bot Token and Chat ID in Settings
- Make sure the ESP32 is in STA mode (connected to a router), not AP mode
- Verify your bot is not blocked — send `/start` to it from your Telegram account

**Stream is laggy or disconnecting**
- Reduce resolution (try QVGA or VGA) or lower FPS
- Move the device closer to the router
- Only one browser tab should be viewing the stream at a time

**Motion detection firing constantly**
- Lower the sensitivity slider (move it toward 1)
- Check Serial Monitor — if idle MAD is above 5.0, lighting is too inconsistent (flickering lights, wind moving objects)
- Increase the cooldown period

**Dashboard not loading after settings save**
- The device reboots after credential changes — wait 10–15 seconds then reconnect to your router and navigate to the new IP address

---

## Architecture Notes

The firmware uses a **non-blocking architecture** so the dashboard is always responsive:

- **Telegram FreeRTOS task** runs on Core 0 with a 12 KB stack. All Telegram I/O (text, photo upload, video upload) is queued and executed off the main loop. The main loop on Core 1 never blocks waiting for TLS.
- **Pending-job state machine** — photo capture and video recording are never executed inside HTTP handlers. The handler sets a flag and returns immediately; `loop()` picks up and executes the job on the next iteration.
- **SD mutex** (`sdMutex`) guards every `SD_MMC` call across both cores to prevent concurrent access corruption.
- **Camera resolution changes** always use a full `esp_camera_deinit()` + `esp_camera_init()` cycle — the only safe method on this hardware.
- **CAMERA_GRAB_LATEST** with 2 PSRAM frame buffers prevents frame-buffer overflow during streaming.
- **AVI index arrays** (offsets + sizes for up to 3000 frames) are allocated in PSRAM to preserve internal SRAM for WiFi and HTTP buffers.
- **Dashboard HTML** is gzip-compressed in PROGMEM (31 KB → 8.5 KB, 73% reduction) and served with `Content-Encoding: gzip`.

## Security Assessment: 

## Security Assessment - Hellboy's WatchTower v1.0

The short answer is: **no, this solution does not use NVS Encryption, Flash Encryption, or any equivalent hardening.** Here is a complete picture of the security posture, both where it is reasonable and where it is genuinely exposed.

---

### What the firmware does well

**Telegram communication is encrypted in transit.** All communication with Telegram's API uses `WiFiClientSecure` with the Telegram root CA certificate pinned via `setCACert(TELEGRAM_CERTIFICATE_ROOT)`. This means the TLS session is authenticated — the ESP32 verifies it is talking to a real Telegram server and not an impersonator. Traffic cannot be read or tampered with by anyone watching the network.

**Telegram commands are chat-ID gated.** The bot handler checks every incoming message against the configured Chat ID and silently ignores anything that does not match. This means a random person who finds your bot token cannot issue commands — they would need both the token and your specific Chat ID.

**No hardcoded credentials in the firmware binary by default.** WiFi credentials and the Telegram token are stored in NVS via the `Preferences` library and loaded at boot. They are not compiled into the sketch as string literals that would appear in a strings dump of the binary.

---

### Where the solution is exposed

#### 1. NVS is plaintext on the flash chip

This is the most significant issue. The `Preferences` library writes to a standard NVS partition with no encryption. The WiFi password, Telegram bot token, and Chat ID sit in plaintext on the ESP32's flash chip.

Anyone with physical access to the device and a flash programmer — or even just an FTDI cable and `esptool.py` — can dump the entire flash image in a few minutes and extract every credential from the NVS partition without any resistance. For a device that will be mounted somewhere semi-accessible, like a doorframe, a garage, or an outdoor enclosure, this is a real threat.

**NVS Encryption** (available in the ESP-IDF and exposed through Arduino) would protect this by encrypting the NVS partition with a key stored in eFuses. Without it, the partition is an open book.

#### 2. Flash Encryption is not enabled

Without Flash Encryption, the entire flash contents — firmware, PROGMEM data, and NVS — can be read off the chip directly. Flash Encryption would make the raw flash contents unreadable without the key burned into the chip's eFuses. Once enabled, `esptool.py` can no longer dump a meaningful image.

It is worth noting that Flash Encryption on ESP32 is a one-way operation with limited re-flash cycles in its production mode, which makes development more complex. That is probably why it was not used here, and it is a reasonable trade-off for a personal home project — but it is the right recommendation for any deployment where physical access cannot be controlled.

#### 3. The web dashboard has no authentication

The HTTP server on port 80 serves the full dashboard — live stream controls, photo capture, recording, motion arming, SD card deletion, and credential configuration — to **any device on the same network** with zero authentication. There is no login, no token, no IP allowlist.

In a home network where the ESP32 and your own devices share the same WiFi, this is mostly fine in practice. But it means any other device on that network — a guest's phone, a compromised smart TV, anything — can access and control the camera in full. The dashboard also allows writing new WiFi credentials and rebooting the device, which are high-impact operations.

#### 4. The dashboard is served over plain HTTP

The web server runs on HTTP, not HTTPS. This means the dashboard traffic, including any credentials you type into the WiFi or Telegram fields and submit, travels in plaintext over the WiFi network. Someone running a packet capture on the same network segment could read those submissions.

Implementing HTTPS on an ESP32 WebServer is non-trivial because it requires generating and storing a certificate, and the TLS handshake is expensive in terms of RAM and time on this hardware — which is why virtually no ESP32-CAM projects do it. That does not make it any less of a genuine exposure.

#### 5. AP mode credentials are hardcoded

The fallback access point uses hardcoded credentials compiled into the binary (`ESP32HBWT` / `watchtower`). Anyone who reads the source code — or who has seen another device running the same firmware — knows the AP password. For the brief window when the device is in AP mode during first setup, the entire dashboard is accessible to anyone within WiFi range who knows those credentials.

#### 6. No stream authentication

The MJPEG stream on port 81 (`/stream`) requires no authentication. Anyone who knows or guesses the IP address and port can pull the live video feed directly, bypassing the dashboard entirely. The stream URL is `http://<ip>:81/stream` with no token or session check.

---

### Threat model summary

|Threat|Protected?|Notes|
|---|---|---|
|Network eavesdropping on Telegram traffic|✅ Yes|TLS with CA pinning|
|Unauthorised Telegram bot commands|✅ Yes|Chat ID gating|
|Physical flash dump / credential extraction|❌ No|NVS and flash are plaintext|
|LAN access to dashboard controls|❌ No|No authentication|
|LAN eavesdropping on credential submission|❌ No|HTTP only|
|LAN access to live video stream|❌ No|No stream authentication|
|Brute-force or token guessing for Telegram|✅ Partially|Token is long; Chat ID filter adds a second factor|

---

### Practical risk assessment

For a **personal home project** on a trusted home network, the risk profile is acceptable for most people. The realistic attacker is someone with physical access to the device, and for that scenario enabling NVS Encryption and Flash Encryption would be the correct remediation. The lack of dashboard authentication is the other meaningful gap, and the simplest mitigation there — without touching the firmware — is network segmentation: putting the ESP32 on an IoT VLAN that your other devices cannot reach.

For any deployment where the device is accessible to untrusted parties, or where the network cannot be fully controlled, the absence of Flash Encryption, NVS Encryption, and dashboard authentication would each be considered a meaningful security deficiency.

