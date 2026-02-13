# Remote Screenshots for ESPHome Displays (Generic Component)

**TL;DR**  An ESPHome external component that adds `GET /screenshot` and `GET /screenshot/info` to any ESP32 with a display and `web_server`. Fetch a pixel-perfect BMP of the live framebuffer over HTTP, switch pages remotely with `?page=N`, and discover available pages with a JSON endpoint. Supports ESPHome's native page system, custom global-based paging, and single-screen devices.

---

## The Problem

If you've built a display UI in ESPHome, you know the loop: edit the YAML lambda, compile, upload, walk over to the device, squint at a small TFT, realise the layout is off by a few pixels, walk back. Repeat for every change, every page.

There's no built-in way to see what the display is rendering without physically looking at it. The web server dashboard shows entity states and buttons, but not what's actually on the screen.

## The Solution

`display_capture` is a drop-in external component that hooks into ESPHome's existing web server and registers HTTP endpoints. One `curl` command gives you a 24-bit BMP of exactly what's on the display:

```bash
curl -o screenshot.bmp http://<device-ip>/screenshot
```

Want to see a specific page without touching the device?

```bash
curl -o page2.bmp "http://<device-ip>/screenshot?page=2"
```

Want to discover what's available programmatically?

```bash
curl http://<device-ip>/screenshot/info
# {"pages":3,"width":320,"height":240,"mode":"native_pages","page_names":["Main","Graph","Settings"]}
```

---

## Three Page Modes

The component supports every common ESPHome display pattern:

### 1. Minimal No Pages

Just capture whatever's on screen. Three lines of config:

```yaml
display_capture:
  display_id: my_display
```

### 2. Native ESPHome Pages

If your display uses ESPHome's built-in page system (`pages:` in the display config), reference them directly:

```yaml
display:
  - platform: ili9xxx
    id: my_display
    pages:
      - id: page_main
        lambda: |-
          it.printf(10, 10, id(font), "Main");
      - id: page_graph
        lambda: |-
          // graph rendering...
      - id: page_settings
        lambda: |-
          it.printf(10, 10, id(font), "Settings");

display_capture:
  display_id: my_display
  pages:
    - page_main
    - page_graph
    - page_settings
  page_names: ["Main", "Graph", "Settings"]
```

The component uses `show_page()` / `get_active_page()` to switch and restore the same mechanism ESPHome uses internally.

### 3. Global-Based Pages

For devices that track the current page with an int global (common in complex UIs with custom page logic):

```yaml
globals:
  - id: current_page
    type: int
    restore_value: no
    initial_value: '0'

display_capture:
  display_id: my_display
  page_global: current_page
  sleep_global: is_sleeping     # optional, wakes display for capture
  page_names: ["Main", "History", "Guest Info", "Camera", "Network", "Kill Switch", "Cleaning"]
```

`pages` and `page_global` are mutually exclusive ESPHome config validation enforces this.

---

## The Info Endpoint

`GET /screenshot/info` returns JSON metadata so callers can discover what's available without trial and error:

```json
{
  "pages": 7,
  "width": 320,
  "height": 240,
  "mode": "global_pages",
  "page_names": ["Main", "History", "Guest Info", "Camera", "Network", "Kill Switch", "Cleaning"]
}
```

This runs synchronously on the HTTP task no semaphore needed since it only reads immutable setup-time data.

---

## Setup

```yaml
external_components:
  - source:
      type: local
      path: components

web_server:
  port: 80

display_capture:
  display_id: my_display
  # Plus one of: pages, page_global, or neither (for single-screen)
```

### Prerequisites

- ESP32 with **PSRAM** (ESP32-S3, ESP32-S2, ESP32 WROVER)
- A `display` component using **BITS_16** (RGB565) color mode
- The built-in `web_server` component enabled

---

## How It Works

### The Threading Problem

ESPHome's HTTP server (`web_server_idf`) runs on its own FreeRTOS task. The display buffer is written by the main ESPHome loop task during `display.update()`. Reading the buffer from the HTTP task while the main loop is mid-render would be a data race.

### The Semaphore Solution

The component uses a binary semaphore to hand off work to the main loop:

```
HTTP task                                Main loop
----------                               ---------
handleRequest()
  set request_pending_ = true
  xSemaphoreTake(5s timeout)
  ... blocks ...                         loop() sees request_pending_
                                           wake display if sleeping
                                           switch to requested page
                                           display_->update()
                                           read buffer -> BMP in PSRAM
                                           restore original page + sleep
                                           xSemaphoreGive() ---+
  semaphore acquired  <------------------------------------|
  send BMP response
  free PSRAM buffer
```

All buffer access happens on the main loop task. The HTTP task just blocks until the data is ready, then sends it.

### Protected Buffer Access

ESPHome's `DisplayBuffer::buffer_` is `protected` no public API to read raw pixels. The component uses `#define protected public` in a separate `.cpp` translation unit (the standard approach for accessing internal ESPHome members without forking the framework).

### Runtime Safety

Uses `dynamic_cast<DisplayBuffer*>` to safely access the pixel buffer, with graceful error handling if the display isn't a compatible subclass.

### Rotation-Correct Output

The output BMP always matches what you'd see looking at the physical display, regardless of the panel's native orientation. Handles all four rotations (0/90/180/270).

---

## Configuration Reference

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `display_id` | ID | **Yes** | Reference to any `Display` component |
| `pages` | list of IDs | No | `DisplayPage` IDs for native page mode |
| `page_global` | ID | No | `globals` int for global page mode |
| `sleep_global` | ID | No | `globals` bool for sleep-aware capture |
| `page_names` | list of strings | No | Human-readable names for the info endpoint |

## HTTP Endpoints

| Endpoint | Content-Type | Description |
|----------|-------------|-------------|
| `GET /screenshot` | `image/bmp` | Capture current screen |
| `GET /screenshot?page=N` | `image/bmp` | Capture specific page (0-indexed) |
| `GET /screenshot/info` | `application/json` | Page count, dimensions, mode, names |

---

## Limitations

- **RGB565 only** assumes BITS_16 buffer format
- **PSRAM required** ~225 KB BMP buffer won't fit in internal SRAM
- **Single concurrent request** one screenshot at a time
- **Brief visual flash** when `?page=N` switches pages, the physical display shows the requested page for ~50ms before restoring

## Build Gotcha: CMake GLOB Caching

When first adding this component, PlatformIO's CMake cache may not discover the new `.cpp` file. If you get a linker error:

```bash
rm -rf .esphome/build/<device>/.pioenvs/<device>/CMakeCache.txt \
       .esphome/build/<device>/.pioenvs/<device>/CMakeFiles/
```

One-time fix subsequent compiles work automatically.

---

## Use Cases

- **UI iteration**: See your display lambda changes without walking to the device
- **CI/visual regression**: Capture screenshots as part of a build pipeline, diff against a known-good set
- **Remote debugging**: Check what a deployed device is actually showing
- **Documentation**: Generate display page images for project docs
- **AI-assisted development**: Let Claude Code (or similar) see the display and suggest layout changes
- **Programmatic discovery**: Use the info endpoint to build tools that enumerate and capture all pages automatically

## Compatibility

Tested on ST7789V 240x320 at rotation 90 (320x240 landscape) on ESP32-S3, with ESPHome 2025.11.x and later. Should work with any `DisplayBuffer` subclass in BITS_16 mode on any PSRAM-equipped ESP32.

---

## File Structure

```
components/display_capture/
  __init__.py               ESPHome Python codegen (config schema + to_code)
  display_capture.h         C++ class declaration (forward-declaration safe)
  display_capture.cpp       Implementation (buffer access, BMP generation, info endpoint)
  README.md                 Technical reference documentation
```

## Source

The component is designed as a self-contained external component directory. Copy `components/display_capture/` into your ESPHome config and add the `external_components` + `display_capture` blocks to your YAML.
