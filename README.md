# Remote Screenshots for ESPHome Displays

**TL;DR** â€” An ESPHome external component that adds `GET /screenshot` to any ESP32 with a display and `web_server`. Fetch a pixel-perfect BMP of the live framebuffer over HTTP, switch pages remotely with `?page=N`, and iterate on display UIs without ever walking over to the device.

---

## The Problem

If you've built a display UI in ESPHome, you know the loop: edit the YAML lambda, compile, upload, walk over to the device, squint at a small TFT, realise the layout is off by a few pixels, walk back. Repeat for every change, every page.

There's no built-in way to see what the display is rendering without physically looking at it. The web server dashboard shows entity states and buttons, but not what's actually on the screen.

## The Solution

`display_screenshot` is a drop-in external component that hooks into ESPHome's existing web server and registers an HTTP endpoint. One `curl` command gives you a 24-bit BMP of exactly what's on the display:

```bash
curl -o screenshot.bmp http://<device-ip>/screenshot
```

Want to see a specific page without touching the rotary encoder?

```bash
curl -o page4.bmp "http://<device-ip>/screenshot?page=4"
```

Want to capture every page in one go?

```bash
for p in 0 1 2 3 4 5 6; do
  curl -s -o "page${p}.bmp" "http://<device-ip>/screenshot?page=${p}"
done
```

The images below were all captured remotely â€” no one touched the device. The display was asleep at the time, and the component woke it up automatically for each capture.

### Page 0 â€” Main (Hot Water Temperature)
> Current temperature with rate of change, target slider, and heater status.

### Page 1 â€” Graph
> 24-hour temperature trend graph.

### Page 2 â€” Settings
> Target temperature adjustment with rotary encoder.

### Page 3 â€” Prana Ventilation
> Prana recuperator status showing front/back airflow.

### Page 4 â€” Network Info
> WiFi SSID, signal strength, IP address, uptime.

### Page 5 â€” About / Version
> Firmware version, ESPHome version, device info.

### Page 6 â€” Cleaning Schedule
> Upcoming cleaning dates pulled from Home Assistant calendar.

---

## Setup

Three lines in your YAML (plus pointing at the component source):

```yaml
external_components:
  - source:
      type: local
      path: components

display_screenshot:
  display_id: my_display
  page_global: current_page    # optional â€” enables ?page=N
  sleep_global: is_sleeping    # optional â€” wakes display for capture
```

`display_id` is the only required field. The two optional globals let you switch pages remotely and wake a sleeping display for capture.

### Prerequisites

- ESP32 with **PSRAM** (ESP32-S3, ESP32-S2, ESP32 WROVER)
- A `display` component using **BITS_16** (RGB565) color mode
- The built-in `web_server` component enabled

---

## How It Works

The interesting engineering challenge here isn't the BMP encoding â€” it's safely reading the display buffer from a different FreeRTOS task.

### The Threading Problem

ESPHome's HTTP server (`web_server_idf`) runs on its own FreeRTOS task. The display buffer is written by the main ESPHome loop task during `display.update()`. Reading the buffer from the HTTP task while the main loop is mid-render would be a data race.

### The Semaphore Solution

The component uses a binary semaphore to hand off work to the main loop:

```
HTTP task                                Main loop
â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€                               â”€â”€â”€â”€â”€â”€â”€â”€â”€
handleRequest()
  set request_pending_ = true
  xSemaphoreTake(5s timeout)
  ... blocks ...                         loop() sees request_pending_
                                           wake display if sleeping
                                           switch to requested page
                                           display_->update()
                                           read buffer â†’ BMP in PSRAM
                                           restore original page + sleep
                                           xSemaphoreGive() â”€â”€â”€â”
  semaphore acquired  <â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
  send BMP response
  free PSRAM buffer
```

All buffer access happens on the main loop task, which is the only task that should touch display state. The HTTP task just blocks until the data is ready, then sends it.

### Accessing the Protected Buffer

ESPHome's `DisplayBuffer::buffer_` is a `protected` member. There's no public API to read raw pixels back.

The workaround: the component uses a **split header/implementation** pattern. The `.h` file uses only forward declarations and public APIs â€” it compiles cleanly alongside everything else. The `.cpp` file applies `#define protected public` as its very first line, before any `#include`, so when `display_buffer.h` is parsed in that translation unit, `buffer_` is accessible.

This is admittedly a hack, but it's the standard approach for accessing internal ESPHome members without forking the framework. Each `.cpp` is its own translation unit, so the macro only affects that one file.

### Rotation-Correct Output

ESPHome's `draw_pixel_at()` applies a rotation transform when writing pixels: screen coordinates get mapped to buffer coordinates. The screenshot must apply the **inverse** transform to read them back in screen order:

| Rotation | Screen (sx, sy) â†’ Buffer (bx, by) |
|----------|-----------------------------------|
| 0Â° | `bx = sx, by = sy` |
| 90Â° | `bx = w_native - 1 - sy, by = sx` |
| 180Â° | `bx = w_native - 1 - sx, by = h_native - 1 - sy` |
| 270Â° | `bx = sy, by = h_native - 1 - sx` |

The output BMP always matches what you'd see looking at the physical display, regardless of the panel's native orientation.

### RGB565 â†’ 24-bit BMP

ILI9XXX in `BITS_16` mode stores each pixel as two bytes:

```
buffer[pos]     = RRRRRGGG   (high byte)
buffer[pos + 1] = GGGBBBBB   (low byte)
```

These get expanded to 8-bit-per-channel BGR (BMP's native byte order), with rows stored bottom-to-top and padded to 4-byte boundaries. For a 320x240 display, the output is exactly 230,454 bytes.

The full BMP is assembled in PSRAM (`heap_caps_malloc` with `MALLOC_CAP_SPIRAM`), allocated per-request and freed immediately after the HTTP response is sent.

---

## Configuration Reference

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `display_id` | ID | **Yes** | Reference to a `DisplayBuffer` component |
| `page_global` | ID | No | `globals` int tracking the current page â€” enables `?page=N` |
| `sleep_global` | ID | No | `globals` bool tracking sleep state â€” wakes display for capture |

## HTTP Endpoint

```
GET /screenshot[?page=N]
```

| Response | |
|----------|---|
| Content-Type | `image/bmp` |
| Format | 24-bit uncompressed BMP |
| Size | 54 + (width Ã— height Ã— 3) bytes |
| Cache-Control | `no-cache` |

| Status | Meaning |
|--------|---------|
| 200 | BMP image returned |
| 500 | PSRAM allocation failed |
| 504 | Main loop did not respond within 5 seconds |

---

## Limitations

- **RGB565 only** â€” assumes BITS_16 buffer format. Grayscale and indexed modes are not supported.
- **PSRAM required** â€” the ~225 KB BMP buffer won't fit in internal SRAM.
- **Single concurrent request** â€” one screenshot at a time (shared semaphore and buffer).
- **Brief visual flash** â€” when `?page=N` switches pages, the physical display shows the requested page for ~50ms before restoring.

## Build Gotcha: CMake GLOB Caching

When first adding this component to an existing project, PlatformIO's CMake cache may not discover the new `.cpp` file. If you get a linker error about undefined vtable references:

```bash
rm -rf .esphome/build/<device>/.pioenvs/<device>/CMakeCache.txt \
       .esphome/build/<device>/.pioenvs/<device>/CMakeFiles/
```

One-time fix â€” subsequent compiles work automatically.

---

## Use Cases

- **UI iteration**: See your display lambda changes without walking to the device
- **CI/visual regression**: Capture screenshots as part of a build pipeline, diff against a known-good set
- **Remote debugging**: Check what a deployed device is actually showing
- **Documentation**: Generate display page images for project docs
- **AI-assisted development**: Let Claude Code (or similar) see the display and suggest layout changes

## Compatibility

Tested on ST7789V 240x320 at rotation 90 (320x240 landscape) on ESP32-S3, with ESPHome 2025.11.x and 2025.12.x. Should work with any `DisplayBuffer` subclass in BITS_16 mode on any PSRAM-equipped ESP32.

---

## File Structure

```
components/display_screenshot/
  __init__.py               ESPHome Python codegen (config schema + to_code)
  display_screenshot.h      C++ class declaration (forward-declaration safe)
  display_screenshot.cpp    Implementation (buffer access, BMP generation)
  README.md                 Technical reference documentation
```

## Source

The component is designed as a self-contained external component directory. Copy `components/display_screenshot/` into your ESPHome config and add the `external_components` + `display_screenshot` blocks to your YAML.
