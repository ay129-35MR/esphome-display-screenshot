# display_screenshot
Remote screenshots for ESPHome displays over HTTP.

Adds `GET /screenshot` to any ESP32 with a display and `web_server`. Fetch a pixel-perfect BMP of the live framebuffer, switch pages remotely with `?page=N`, and discover available pages with a JSON info endpoint.

```bash
curl -o screenshot.bmp http://<device-ip>/screenshot
```

---

## Why?

If you're building display UIs on ESPHome, the dev cycle is painful: edit YAML, compile, upload, walk over, squint at a small TFT, walk back, repeat. This component lets you see what's on screen from anywhere with `curl`.

**Let your coding agent close the loop.** After every display lambda change, Claude Code / Codex / Gemini / your coding agent of choice can `curl` a screenshot, view the BMP, and verify the layout looks right -- without you ever looking at the device. This is the use case that prompted the component: AI-assisted display development where the agent can check its own work.

**Remote device monitoring.** Expose the endpoint through ngrok or a Cloudflare tunnel and see what your device is displaying from anywhere. Useful for devices mounted on walls, inside enclosures, or at a different site entirely. No VPN needed.

**Home Assistant integration.** Fire a webhook that fetches the screenshot and posts it to a notification, Lovelace card, or Telegram bot. "What does the controller screen say right now?" -- answered without leaving the couch.

**Auto-generated documentation.** Script a loop that hits `/screenshot/info` to discover all pages, captures each one, and dumps them into a docs folder. Re-run after every UI change and your docs stay current.

**Visual regression testing.** Capture baseline screenshots, make changes, capture again, diff. Catch layout breakage before it ships.

**Remote debugging.** "The display looks wrong" -- now you can see exactly what they see without asking them to photograph their screen.

---

## What it supports

Three page modes, covering every common ESPHome display pattern:

| Mode | Config | How it works |
|------|--------|-------------|
| **Single screen** | Just `display_id` | Captures whatever's on screen. No page switching. |
| **Native pages** | `pages: [page_main, ...]` | Uses ESPHome's built-in `pages:` system. Switches with `?page=N`. |
| **Global-based pages** | `page_global: current_page` | For UIs that track the current page with a `globals` int. |

Two HTTP endpoints:

| Endpoint | Returns |
|----------|---------|
| `GET /screenshot[?page=N]` | 24-bit BMP image of the display |
| `GET /screenshot/info` | JSON with page count, dimensions, mode, and page names |

```bash
# Grab the current screen
curl -o screenshot.bmp http://192.168.1.100/screenshot

# Capture a specific page
curl -o page2.bmp "http://192.168.1.100/screenshot?page=2"

# Discover what's available
curl http://192.168.1.100/screenshot/info
# {"pages":3,"width":320,"height":240,"mode":"native_pages","page_names":["Main","Graph","Settings"]}
```

---

## Requirements

- **ESP32 with PSRAM** -- ESP32-S3, ESP32-S2, or ESP32 WROVER. The ~225 KB BMP buffer is allocated in PSRAM. Regular ESP32 without PSRAM won't work.
- **Display using RGB565** -- any `DisplayBuffer` subclass in `BITS_16` colour mode (ILI9XXX, ST7789V, ILI9341, ILI9488, etc.)
- **`web_server` component enabled** -- the screenshot endpoint hooks into ESPHome's built-in web server

---

## Quick Start

### 1. Get the component

**Option A -- Clone this repo into your ESPHome config directory:**

```bash
cd /path/to/your/esphome/config
git clone https://github.com/YOUR_USERNAME/display_capture.git components/display_capture
```

**Option B -- Reference it directly from GitHub in your YAML:**

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/YOUR_USERNAME/display_capture
    components: [display_capture]
```

**Option C -- Copy the files manually:**

Download the repo and copy the `display_capture` folder into your ESPHome `components/` directory:

```
your-esphome-config/
  components/
    display_capture/
      __init__.py
      display_capture.h
      display_capture.cpp
  your-device.yaml
```

### 2. Make sure you have `web_server` enabled

If you don't already have this in your YAML, add it:

```yaml
web_server:
  port: 80
```

### 3. Tell ESPHome where to find the component

If you used Option A or C (local files), add this to your YAML:

```yaml
external_components:
  - source:
      type: local
      path: components
```

If you used Option B (git), you already did this in step 1.

### 4. Add the `display_capture` block

Pick the config that matches your setup (see [Which page mode do I need?](#which-page-mode-do-i-need) below):

```yaml
# Simplest -- just capture whatever's on screen
display_capture:
  display_id: my_display
```

### 5. Compile, upload, and test

```bash
esphome run your-device.yaml
```

Once it's running:

```bash
# Grab a screenshot
curl -o screenshot.bmp http://<device-ip>/screenshot

# Open it
open screenshot.bmp        # macOS
xdg-open screenshot.bmp    # Linux
start screenshot.bmp        # Windows
```

That's it. You should see a pixel-perfect BMP of your display.

---

## Which page mode do I need?

Look at how your display is set up in YAML and pick the matching config:

### "I have a single screen with a `lambda:` -- no pages"

```yaml
# Your display config probably looks like:
display:
  - platform: ili9xxx
    id: my_display
    lambda: |-
      it.printf(10, 10, id(font), "Hello World");

# Just add this:
display_capture:
  display_id: my_display
```

`?page=N` is ignored in this mode -- there's only one screen to capture.

### "I use ESPHome's built-in `pages:` system"

```yaml
# Your display config probably looks like:
display:
  - platform: ili9xxx
    id: my_display
    pages:
      - id: page_main
        lambda: |-
          it.printf(10, 10, id(font), "Main");
      - id: page_graph
        lambda: |-
          // graph code...
      - id: page_settings
        lambda: |-
          it.printf(10, 10, id(font), "Settings");

# List the same page IDs here:
display_capture:
  display_id: my_display
  pages:
    - page_main
    - page_graph
    - page_settings
  page_names: ["Main", "Graph", "Settings"]  # optional, shows up in /info
```

Now `?page=0` captures page_main, `?page=1` captures page_graph, etc.

### "I track the current page with a `globals` int"

This is common in complex UIs where a rotary encoder or button sets an integer and the display lambda switches on it.

```yaml
# Your globals probably look like:
globals:
  - id: current_page
    type: int
    restore_value: no
    initial_value: '0'

# And your display lambda does something like:
# if (id(current_page) == 0) { ... } else if (id(current_page) == 1) { ... }

# Point display_capture at the global:
display_capture:
  display_id: my_display
  page_global: current_page
  sleep_global: is_sleeping     # optional -- if you have a sleep/screensaver global
  page_names: ["Main", "History", "Settings"]  # optional
```

**Note:** `pages` and `page_global` are mutually exclusive -- ESPHome will reject your config if you specify both.

---

## Endpoints

Once running, your device exposes two new HTTP endpoints:

### `GET /screenshot`

Returns a 24-bit BMP image of the current display.

```bash
curl -o screenshot.bmp http://192.168.1.100/screenshot
```

### `GET /screenshot?page=N`

Switches to page N (0-indexed), captures it, then switches back. The physical display flashes briefly (~50ms).

```bash
# Capture page 2
curl -o page2.bmp "http://192.168.1.100/screenshot?page=2"

# Capture all pages in a loop
for p in 0 1 2 3; do
  curl -s -o "page${p}.bmp" "http://192.168.1.100/screenshot?page=${p}"
done
```

### `GET /screenshot/info`

Returns JSON metadata -- useful for scripts that need to discover pages automatically.

```bash
curl http://192.168.1.100/screenshot/info
```

```json
{
  "pages": 3,
  "width": 320,
  "height": 240,
  "mode": "native_pages",
  "page_names": ["Main", "Graph", "Settings"]
}
```

### Response Codes

| Code | Meaning |
|------|---------|
| 200 | Success -- BMP or JSON returned |
| 500 | PSRAM allocation failed (device out of memory) |
| 504 | Main loop didn't respond in 5 seconds (device too busy) |

---

## Configuration Reference

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `display_id` | ID | **Yes** | Your display component's `id` |
| `pages` | list of IDs | No | `DisplayPage` IDs -- for ESPHome native pages |
| `page_global` | ID | No | `globals` int that tracks the current page |
| `sleep_global` | ID | No | `globals` bool -- wakes display before capture |
| `page_names` | list of strings | No | Human-readable names for the `/screenshot/info` endpoint |

---

## Troubleshooting

### Linker error: undefined reference to vtable

PlatformIO's CMake cache doesn't know about the new `.cpp` file. Clear the cache (one-time fix):

```bash
rm -rf .esphome/build/<device>/.pioenvs/<device>/CMakeCache.txt \
       .esphome/build/<device>/.pioenvs/<device>/CMakeFiles/
```

Then compile again -- it'll pick up the file and won't happen again.

### 504 timeout on `/screenshot`

The main ESPHome loop didn't respond within 5 seconds. This usually means:
- The device is very busy (heavy sensor polling, large display updates)
- The `display_id` doesn't match your actual display component's ID

### 500 error on `/screenshot`

PSRAM allocation failed. Check that your board actually has PSRAM and it's enabled in your board config. For ESP32-S3, you may need:

```yaml
esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: arduino
psram:
  mode: octal  # or quad, depending on your board
```

### Screenshot is all black

If you're using `sleep_global`, make sure the global ID matches the bool your display lambda checks. The component sets it to `false` before capture, captures, then restores it.

If you're not using sleep, check that your display lambda is actually drawing something (add a test `it.fill(Color(255, 0, 0));` to confirm).

### Screenshot colours look wrong

The component assumes RGB565 (BITS_16) buffer format, which is the default for ILI9XXX displays. If your display uses a different colour mode, the output will be garbled.

---

## How It Works (for the curious)

### Thread Safety

ESPHome's web server runs on a separate FreeRTOS task from the main loop. The display buffer can only be safely accessed from the main loop. The component uses a binary semaphore to coordinate:

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

### Protected Buffer Access

`DisplayBuffer::buffer_` is `protected` in ESPHome -- there's no public API to read pixels back. The component uses `#define protected public` in a separate `.cpp` translation unit. This is the standard approach for accessing ESPHome internals without forking the framework.

### Rotation Handling

The output BMP always matches what you see on the physical display, regardless of rotation setting. The component applies the inverse of ESPHome's rotation transform when reading pixels back from the buffer.

---

## Compatibility

| | |
|---|---|
| **Tested on** | ST7789V 240x320 @ rotation 90, ESP32-S3 |
| **ESPHome** | 2025.11.x and later |
| **Should work with** | Any `DisplayBuffer` subclass in BITS_16 mode on any PSRAM-equipped ESP32 |

## License

MIT
