# Performance Improvement Tasks

Tasks are ordered by estimated impact-to-effort ratio. Test after each change.

---

## CPU & Battery

- [x] **1. Cache clock label updates** — Only call `lv_label_set_text` when the displayed second/minute actually changes, not every 5ms loop iteration. (`hal/esp32/app_hal.cpp` — clock label update block ~line 2210)

- [x] **2. Throttle watchface updates** — Cache the last-rendered time; skip `update_faces()` when nothing has changed. (`hal/esp32/app_hal.cpp` — `update_faces` call ~line 2462)

- [x] **3. Reduce LVGL refresh rate** — Increase `LV_DEF_REFR_PERIOD` from 33ms (30 FPS) to 50ms (20 FPS) or 100ms (10 FPS) for a clock UI that barely moves. (`include/lv_conf.h` line 72)

---

## Power (Idle / Sleep)

- [x] **4. Implement light sleep on screen-off** — After the screen timeout fires, call `esp_light_sleep_start()` and wake via touch INT pin or button interrupt instead of running the full 5ms loop. This can reduce idle current from ~15 mA to <1 mA. (`hal/esp32/app_hal.cpp` — screen timeout handler ~line 2340)

- [x] **5. Reduce default backlight brightness** — Lower the default PWM value in `screenBrightness()`. The backlight is the single largest current draw. (`hal/esp32/app_hal.cpp` ~line 532)

- [x] **6. Replace touch polling with INT pin interrupt** — The CST816S has an interrupt pin. Subscribe to it instead of reading I2C every 5ms. (`hal/esp32/app_hal.cpp` — `my_touchpad_read` ~line 206; `hal/esp32/displays/generic.hpp` — touch config)

---

## Flash & RAM

- [x] **7. Disable unused watchfaces** — Comment out `#define ENABLE_FACE_XXX` entries for faces not used on the target board. Each face is a large compiled-in image array. (`hal/esp32/app_hal.h` — face enable section)

---

## Optional / Low Priority

- [ ] **8. Disable feedback tasks when not needed** — The speaker and vibration FreeRTOS tasks run even at idle. If the target hardware has no speaker or motor, remove the `ENABLE_TONE` / `ENABLE_VIBRATION` defines. (`hal/esp32/app_hal.h`, `hal/esp32/feedback.cpp`)

- [ ] **9. Reduce IMU poll rate** — The QMI8658C app polls every 100ms. Change to 250ms if smooth real-time display is not required. (`src/apps/qmi8658c/qmi8658c.c` line 73)

- [x] **10. Replace 5ms delay with FreeRTOS yield** — `delay(5)` in `hal_loop` is not a true sleep. Use `vTaskDelay(pdMS_TO_TICKS(5))` so the FreeRTOS scheduler can run other tasks properly. (`hal/esp32/app_hal.cpp` ~line 2181)
