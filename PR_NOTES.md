# Vital 100S & Vital 200S Fixes — Commit & PR Notes

Branch: `fix/vital-100s-200s` (rebased on `upstream/main` @ `b6f1cd9`)

---

## 1. `949b70d` — Remove leftover boot-time filter LED blinking and bogus Vital `02 06 55` command

**Files changed:**
- `components/levoit/levoit.h`
- `components/levoit/levoit.cpp`
- `components/levoit/vital_commands.cpp`

**What & Why:**
- During every ESP boot/Wi-Fi reconnect, `Levoit::loop()` toggled `setFilterLedOn` / `setFilterLedOff` in a debug loop left over from early development.
- Worse, in `vital_commands.cpp`, `setFilterLedOn` and `setFilterLedOff` were mapped to `CMD 02 06 55` with a 1-byte payload (`{0x01, 0x01, 0x01}` / `{0x01, 0x01, 0x00}`), whereas `02 06 55` in the V2 protocol is actually the 2-byte Sprout AQI scale command (`{0x01, 0x02, lo, hi}`).
- Removed the boot-time filter LED blink loop from `Levoit` and removed the bogus `setFilterLedOn` / `setFilterLedOff` cases from `vital_commands.cpp`.

---

## 2. `7c5afd5` — Stop Vital timer polling on empty ACK and restrict dedup to status pushes

**Files changed:**
- `components/levoit/vital_status.cpp`
- `components/levoit/decoder.cpp`

**What & Why:**
- **Infinite timer polling fix (`vital_status.cpp`):** `Levoit` starts with `timer_active_ = true` at boot so it polls `requestTimerStatus` (`02 1A 50`) once to check if a hardware timer is running. However, `decode_vital_timer()` returned early when `payload_len == 0` (before calling `self->stop_timer()`), and when a 12-byte response returned `initial == 0 && remaining == 0`, it only cleared `timer_stop_pending_` without calling `self->stop_timer()`. As a result, `timer_active_` remained `true` forever and the ESP polled `02 1A 50` every 10 seconds indefinitely. Fixed both paths to call `self->stop_timer()`.
- **Restrict payload deduplication to periodic status pushes (`decoder.cpp`):** `dispatch_decoder()` previously ran FNV-1a payload deduplication on *all* incoming packet types, including command replies like `1A 50` (timer status). If two `1A 50` replies had the same payload (e.g. both `0`), the second response was dropped before `decode_vital_timer()` could act on it. Restricted `payload_changed_()` deduplication to periodic status broadcasts (`0x00 0x55`, `0x60 0x40`, `0x30 0x40`, `0xB0 0x40`, `0x40 0x41`, `0x61 0x40`).

---

## 3. `d8b9b10` — Recompute filter stats immediately on `filter_lifetime_months` change

**Files changed:**
- `components/levoit/levoit.cpp`

**What & Why:**
- Changing the `Filter Lifetime Months` number slider in Home Assistant updated the number entity state, but `Filter %`, `Remaining Filter Life`, and `Total Air Filtered` were not recalculated until the next 60-second `track_cadr_usage()` tick (and only while the fan was actively running).
- Added a `NumberType::FILTER_LIFETIME_MONTHS` case in `Levoit::on_number_command()` that calls `this->publish_filter_stats_now()` immediately.

---

## 4. `9a7c8f9` — Remove 40MHz flash limit on Vital 100S and align Vital 200S stock C3 config

**Files changed:**
- `devices/levoit-vital100s/levoit-vital100s{,-builder,_dev}.yaml`
- `docs/esphome-devices/Levoit-Vital-100s/config.yaml`
- `devices/levoit-vital200s/levoit-vital200s{,-builder,_dev}.yaml`
- `docs/esphome-devices/Levoit-Vital-200s/config.yaml`

**What & Why:**
- Upstream PR #105 (`3f17899`) ported the old ignored `platformio_options` (`board_build.flash_mode: dio` + `board_build.f_flash: 40000000L`) into native `esp32: flash_mode: dio` + `flash_frequency: 40MHz` on Vital 100S, and missed the Vital 200S stock configs.
- The stock ESP32-C3-SOLO-1 firmware runs at **DIO 80 MHz** (and both Vital 100S and Vital 200S Pro run cleanly at the default 80 MHz DIO), so throttling to 40 MHz is unnecessary.
- Removed `flash_frequency: 40MHz` from Vital 100S stock configs (keeping `flash_mode: dio`) and aligned Vital 200S stock configs to match (`min_version: 2026.6.0`, `variant: esp32c3`, `flash_size: 4MB`, `flash_mode: dio`, `cpu_frequency: 160Mhz`, `CONFIG_FREERTOS_UNICORE: y`, and fixed the `ESP32-SOLO-1C` comment typo to `ESP32-C3-SOLO-1`).

---

## 5. `dbd1023` — Align Vital 100S and 200S common YAML entities, room size bounds, and logger level

**Files changed:**
- `devices/levoit-vital100s/common.yaml`
- `devices/levoit-vital100s/levoit-vital100s-builder{,-c3,-s3}.yaml`
- `devices/levoit-vital200s/common.yaml`
- `devices/levoit-vital200s/levoit-vital200s-builder{,-c3,-s3}.yaml`

**What & Why:**
- **Room size bounds (`efficiency_room_size` & `auto_profile_room_size_input`):** Vital 100S (`common.yaml`) didn't set model-specific `min_value` / `max_value`, falling back to the component's EverestAir range (`9–296 m²`), while Vital 200S only overrode `efficiency_room_size` (`17–90 m²`) and forgot `auto_profile_room_size_input`. Set both number entities to `10–52 m²` on Vital 100S and `17–90 m²` on Vital 200S.
- **Entity parity:** Added the missing `auto_profile_room_size_input` and writable custom sleep / quick-clean / daytime preference entities (`quick_clean`, `quick_clean_minutes`, `quick_clean_fan_level`, `daytime_enabled`, `daytime_fan_mode`, `daytime_fan_level`, `sleep_fan_level`, `sleep_minutes`, `sleep_preference`) to Vital 100S `common.yaml`, and added `fan_operating_mode` select to Vital 200S `common.yaml`.
- **Builder YAML sync & logger level:** Synchronized all six `*-builder*.yaml` files with their corresponding `common.yaml` + hardware configs (verified against `devices/make-builder-yaml.ps1`) and lowered the default Vital 100S logger level from `VERBOSE` to `DEBUG` to avoid UART/CPU overhead warnings on the single-core ESP32-C3.

---

## 6. `f9bec29` — Fix remaining `uint32_t` format warning in `superior_status.cpp`

**Files changed:**
- `components/levoit/superior_status.cpp`

**What & Why:**
- Upstream PR #106 (`4df4673`) fixed the `uint32_t` `%u` format warnings in `levoit.cpp` using `PRIu32`, but missed `ESP_LOGV(TAG_SUP, "DryRemaining=%u sec, ...", dry_remaining, ...)` in `superior_status.cpp` (only compiled when `logger: level: VERBOSE`).
- Switched it to `%" PRIu32 "` with `#include <cinttypes>` to match `levoit.cpp`.

---

## 7. `f111a4e` — Remove inverted `'light'` `device_class` from `dark_detected` binary sensor

**Files changed:**
- `components/levoit/binary_sensor/__init__.py`

**What & Why:**
- The MCU sets TLV `0x17` (`dark_detected`) to `1` (`ON`) when the room is **dark** and `0` (`OFF`) when the room is **bright**.
- Because `dark_detected` was assigned `device_class: "light"` in `binary_sensor/__init__.py`, Home Assistant translated `ON` into **"Light detected"** when it was actually dark, and **"No light"** when the room was bright.
- Removed `CONF_DEVICE_CLASS: "light"` so Home Assistant displays `Dark Detected: On / Off` accurately (matching the `mdi:weather-night` icon).

---

## 8. `8d31793` — Derate CADR and filter usage when Auto mode idles at `FanLevel 0`

**Files changed:**
- `components/levoit/levoit.h`
- `components/levoit/levoit.cpp`
- `devices/levoit-vital100s/README.md`

**What & Why:**
- On-device `VERBOSE` logs on the Vital 100S (`MCU 1.0.5`) confirmed that in **Auto** mode (`FanMode = 2`) with Very Good air quality (`AirQualityLevel = 1`), the MCU drops `0x04 FanLevel` to `0` (sleep speed) while `0x05 FanSpeed` stays at `1`.
- Because ESPHome's 4-speed `Fan` entity clamps `speed = 0` up to `1` (`25%`) so the fan entity remains `ON` in Home Assistant, `Levoit::calculate_current_cadr_per_hour()` previously saw `speed == 1` with `preset == "Auto"` and failed to apply the `0.63f` sleep-speed derating (which only checked `preset == "Sleep" && speed == 1`). That caused `Current CADR` and accumulated filter wear (`used_cadr_`) to be overcounted by ~58% (`55 m³/h` instead of `34 m³/h` on Vital 100S) whenever Auto mode idled in clean air.
- Added `fan_sleep_speed_` tracking in `Levoit::apply_fan_status()` (before clamping) and `Levoit::on_fan_command()` so `calculate_current_cadr_per_hour()` applies the `0.63f` derating whenever `speed == 1` and either `fan_sleep_speed_` is true or the preset is `"Sleep"`.
- Updated the "Known issue" section in `devices/levoit-vital100s/README.md` with the verified wire behavior.

---

## 9. `e6820ca` — Remove `Unknown3`/`Unknown4` placeholder options from `daytime_fan_mode` select

**Files changed:**
- `components/levoit/levoit.cpp`
- `components/levoit/select/levoit_select.cpp`

**What & Why:**
- The Vital Daytime Preset fan mode (`TLV 0x22`, `SelectType::DAYTIME_FAN_MODE`) uses the same raw mode enum as `TLV 0x03`: `0 = Manual`, `1 = Sleep`, `2 = Auto`, `5 = Pet` (while `3 = Humidity` and `4 = Turbo` belong to other Levoit models).
- Previously, `DAYTIME_FAN_MODE` exposed dummy `"Unknown3"` and `"Unknown4"` options in Home Assistant just so direct array indexing would land `"Pet"` at index `5`.
- Updated `DAYTIME_FAN_MODE` to use `{"Manual", "Sleep", "Auto", "Pet"}` and reused the `{"Manual", "Sleep", "Auto", "Turbo", "Pet"}` ↔ `{0, 1, 2, 4, 5}` mapping in `Levoit::publish_select()`, `Levoit::try_get_select_index()`, and `LevoitSelect::control()`.

---

## 10. `0605bd3` — Fix Wi-Fi LED staying stuck blinking after boot/connect

**Files changed:**
- `components/levoit/levoit.h`
- `components/levoit/levoit.cpp`

**What & Why:**
- **Boot race condition:** `Levoit::setup()` scheduled an unconditional 2-second `"wifi_led_init"` timeout that sent `setWifiLedBlinking` without checking whether the network was already connected. When ESP32-C3 connected in under 2 seconds, `loop()` set `wifi_led_solid_ = true` and sent `setWifiLedOn`, and then `"wifi_led_init"` fired at `t = 2s` and put the MCU back into blinking mode — while `wifi_led_solid_` stayed `true`, so `loop()` never corrected it.
- **0 ms back-to-back UART commands:** On non-Core200S models, `loop()` sent `setWifiLedOff` and `setWifiLedOn` back-to-back in the same tick with 0 ms delay, even though the MCU takes ~13–32 ms to process and ACK a `02 18 50` frame.
- **Cold-boot MCU readiness:** If the network connected before the MCU finished booting on AC power-on, the initial `setWifiLedOn` could be missed with no retry.
- Replaced `wifi_led_solid_` / `wifi_led_blink_sent_` with a `WifiLedState` state machine (`UNKNOWN`, `OFF`, `BLINKING`, `SOLID`) compatible with `network::is_connected()` / `network::is_disabled()`, removed the unconditional `"wifi_led_init"` timeout from `setup()`, spaced `setWifiLedOff` → `setWifiLedOn` by 500 ms across all models (keeping Core 200S's extra 500 ms retry) with cancellation on state changes, and triggered a one-shot Wi-Fi LED re-sync on the first non-Wi-Fi-LED-ACK frame received from the MCU if an earlier command was sent before hearing from it.
