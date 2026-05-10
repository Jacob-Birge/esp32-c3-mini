# Copilot Instructions — esp32-c3-mini

## Project Overview
LVGL-based smartwatch UI firmware for ESP32/RP2040 boards with small round/rectangular displays.
Supports multiple hardware targets (ESP32-C3, ESP32-S3, RP2040) and a native SDL2 desktop emulator.
Uses the Chronos app for BLE data (weather, notifications, navigation).

## Key Technologies
- **LVGL 9.3.0** — UI framework (C, lv_obj API)
- **PlatformIO** — Build system (`platformio.ini`)
- **Arduino framework** (ESP32 targets) / native SDL2 (desktop emulator)
- **LovyanGFX 1.1.16** — Display driver (ESP32)
- **ChronosESP32 1.9.0** — BLE watch protocol
- **ArduinoJson 7.1.0**, **Timber 2.0.0**, **Button2 2.4.1**

---

## Directory Layout
```
hal/
  esp32/          # Arduino/ESP32 HAL (app_hal.cpp, feedback.cpp, displays/)
  pico/           # RP2040 HAL
  sdl2/           # Desktop SDL2 HAL
include/
  lv_conf.h       # LVGL config
  main.h          # Shared includes/defines
  splash.h        # Splash screen image
src/
  main.cpp        # Entry point — calls hal_setup() then hal_loop()
  apps/           # Custom apps (each in own subfolder with assets/)
  games/          # Games (simon, racing)
  faces/          # Precompiled watchfaces (binary→LVGL via bin2lvgl)
  ui/             # Main UI screens (SquareLine Studio generated)
  common/
    api.h                  # IMU API (C linkage)
    app_manager.h          # REGISTER_APP macro + app registry
    app_manager.c
    generated_features.h   # Auto-generated — DO NOT EDIT MANUALLY
    input_bus/             # Encoder/button pub-sub
support/
  header_gen.py            # Pre-build script: regenerates generated_features.h
  hardware_build_extra.py
  build_firmware.py
scripts/
  app_create.py            # Scaffolds a new app from template
```

---

## Supported Board Targets (`platformio.ini` env names)
| env name | Board | Resolution |
|---|---|---|
| `elecrow_c3_1_28` | Elecrow ESP32-C3 | 240×240 |
| `elecrow_s3` | Elecrow ESP32-S3 | 240×296 |
| `m5_dial` | M5 Stack Dial | 240×240 |
| `viewe_smartring` | Viewe SmartRing | 466×466 AMOLED |
| `viewe_knob_1_5` | Viewe Touch Knob | 466×466 AMOLED |
| `viewe_2_8` | Viewe 2.8" | 240×320 |
| `lolin_c3_mini` | Lolin C3 Mini | 240×240 |
| `lolin_s3_mini_1_28` | Waveshare S3 1.28 | 240×240 |
| `lolin_s3_mini_1_69` | Waveshare S3 1.69 | 240×280 |
| `lolin_s3_1_75` | Waveshare S3 1.75 | 466×466 |
| `lolin_s3_2_06` | Waveshare S3 2.06 | 410×502 |
| `esp32_cyd` | ESP32-CYD | 240×320 |
| `rp2040_1_28` | Waveshare RP2040 1.28 | 240×240 |
| `rp2040_1_69` | Waveshare RP2040 1.69 | 240×280 |
| `windows_64` / `mac_64` / `linux_64` | SDL2 desktop emulator | configurable |

**Switching targets:** uncomment exactly one `default_envs` line in `platformio.ini`.

---

## Build Commands
```bash
pio run                        # Build
pio run --target upload        # Flash to device
pio device monitor             # Serial monitor (115200 baud)
python support/header_gen.py   # Manually regenerate generated_features.h
```

---

## Feature Toggle System
- Board defines set via `-D BOARD_NAME=1` in `platformio.ini` build_flags
- `hal/esp32/app_hal.h` — central file that enables apps/faces/games per board using `#define ENABLE_XXX`
- Each feature source file is guarded by `#ifdef ENABLE_XXX` — nothing is compiled unless enabled
- `src/common/generated_features.h` is **auto-generated** on every build by `support/header_gen.py` — it includes all `.h` files found under `src/apps/`, `src/games/`, `src/faces/`

### Key Preprocessor Defines
| Define | Meaning |
|---|---|
| `ELECROW_C3`, `ELECROW_S3` | Elecrow boards |
| `M5_STACK_DIAL` | M5 Stack Dial |
| `VIEWE_SMARTRING`, `VIEWE_KNOB_15`, `VIEWE_2_8` | Viewe boards |
| `ESPS3_1_28`, `ESPS3_1_69`, `ESPS3_1_75`, `ESPS3_2_06` | Waveshare S3 variants |
| `ESP32_CYD` | Cheap Yellow Display |
| `LV_USE_SDL` | Desktop SDL2 emulator |
| `FLASH_SIZE` | 4 / 8 / 16 MB — controls which faces are included |
| `ENABLE_RTC` | Hardware RTC present (Elecrow boards) |
| `ENABLE_CUSTOM_FACE` | Experimental installable watchfaces |
| `ENABLE_APP_XXX` | Enable a specific app |
| `ENABLE_FACE_XXX` | Enable a specific watchface |
| `ENABLE_GAME_XXX` | Enable a specific game |

---

## App Registration Pattern
Every app uses the `REGISTER_APP` macro (defined in `src/common/app_manager.h`):
```c
REGISTER_APP("App Title", &icon_image_dsc, screen_variable, screen_init_function);
```
Uses `__attribute__((constructor))` — self-registers at startup with no manual wiring needed.

---

## App File Structure (`src/apps/<name>/`)
- `<name>.h` — `ENABLE_APP_<NAME>` guard, `LV_IMAGE_DECLARE` for icon, declares `screen_init()`, `ui_app_load()`, `ui_app_exit()`
- `<name>.c` — `REGISTER_APP(...)`, screen event callback, `screen_init()` implementation
- `assets/` — PNG icons and images

### Screen Lifecycle Events (in event_cb)
| Event | When to use |
|---|---|
| `LV_EVENT_SCREEN_LOAD_START` | Before screen shown |
| `LV_EVENT_SCREEN_LOADED` | After screen shown — start timers/animations |
| `LV_EVENT_SCREEN_UNLOAD_START` | Before hidden — stop timers, save data |
| `LV_EVENT_SCREEN_UNLOADED` | After hidden — **always** call `lv_obj_delete(screen); screen = NULL;` |

Exit gesture: `LV_EVENT_GESTURE` + `lv_indev_get_gesture_dir() == LV_DIR_RIGHT` → `ui_app_exit()`

---

## Watchface File Structure (`src/faces/<name>/`)
- Generated by [bin2lvgl](https://github.com/fbiego/esp32-lvgl-watchface) tool
- `<name>.h` — `ENABLE_FACE_<NAME>` guard, declares LVGL objects and image descriptors
- `<name>.c` — image data arrays + face init function
- Enabled via `#define ENABLE_FACE_XXX` in `hal/esp32/app_hal.h`

---

## UI Screens (`src/ui/`)
- Originally generated by SquareLine Studio (LVGL 8), ported to LVGL 9
- Main screens: `ui_clockScreen`, `ui_weatherScreen`, `ui_notificationScreen`, `ui_appListScreen`, `ui_gameListScreen`
- Event handlers declared in `ui_events.h`
- Key functions: `ui_app_load(screen, init_fn)`, `ui_app_exit()`

---

## Input Bus (`src/common/input_bus/`)
Pub/sub for rotary encoder and button events:
```c
input_bus_add_encoder_sub(cb);      // subscribe
input_bus_remove_encoder_sub(cb);   // unsubscribe (do this on screen unload)
input_bus_add_button_sub(cb);
input_bus_remove_button_sub(cb);
// HAL emits events:
input_bus_emit_encoder_event(position, change);
input_bus_emit_button_event(pressed);
```

---

## HAL Display Drivers (`hal/esp32/displays/`)
- `generic.hpp` — LovyanGFX, used by most boards
- `cyd_2432.hpp`, `elecrow_s3.hpp`, `viewe.hpp`, `viewe_2_8.hpp` — board-specific
- Selected in `app_hal.cpp` via board `#ifdef` blocks

## C/C++ Interop
- All public APIs use `extern "C"` guards
- App/UI code is **C**; HAL is **C++** (Arduino)
- LVGL is **C**

---

## Common Task Playbooks

### Add a New App
1. `python scripts/app_create.py <app_name>` — scaffolds `src/apps/<app_name>/`
2. Add icon PNG to `src/apps/<app_name>/assets/`, declare with `LV_IMAGE_DECLARE`
3. Add `#define ENABLE_APP_<NAME>` in `hal/esp32/app_hal.h` under the relevant board section
4. Use `REGISTER_APP(...)` macro in the `.c` file
5. Implement all four lifecycle events in the event callback
6. `lv_obj_delete(screen); screen = NULL;` in `SCREEN_UNLOADED`
7. Right-swipe → `ui_app_exit()`

### Add a New Watchface
1. Convert binary with [bin2lvgl](https://github.com/fbiego/esp32-lvgl-watchface) → place in `src/faces/<name>/`
2. Add `#define ENABLE_FACE_<NAME>` in `hal/esp32/app_hal.h` under the correct board/flash-size guard

### Add a New Board Target
1. Add `[env:new_board]` in `platformio.ini` extending `[esp32]`
2. Set `-D MY_BOARD=1` in build_flags
3. Add display driver in `hal/esp32/displays/` if needed
4. Add pin definitions to `hal/esp32/displays/pins.h`
5. Add feature enables in `hal/esp32/app_hal.h` under `#ifdef MY_BOARD`

### Partition Tables
| File | Flash size |
|---|---|
| `partitions.csv` | 4 MB (default) |
| `partitions_8M.csv` | 8 MB |
| `partitions_16M.csv` | 16 MB (Viewe boards) |

Set via `board_build.partitions` in the env section.

### IMU (QMI8658C)
```c
imu_init();
imu_data_t data = get_imu_data();  // ax, ay, az, gx, gy, gz, temp
imu_close();
```
Only available when `ENABLE_APP_QMI8658C` is defined (Waveshare S3 boards).
