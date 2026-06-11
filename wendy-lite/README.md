# Wendy Lite

Wendy Lite is the WebAssembly runtime layer for ESP32 microcontrollers. It runs WASM guest applications on device and exposes the hardware via host-imported functions from the `"wendy"` module.

## Role in the Wendy Platform

The broader Wendy platform targets Linux/macOS edge devices (Raspberry Pi, Jetson, Mac) via WendyOS and wendy-agent. Wendy Lite covers the other end: bare-metal MCUs where containers and a full OS are not viable. The same hardware APIs (BLE, WiFi, sockets, TLS, OTel) are available from WASM guests as are exposed by wendy-agent to containerised apps, making the programming model portable across device classes.

## Supported Targets

| Target | Status |
|--------|--------|
| ESP32-C6 | CI-built, nightly releases |
| ESP32-C5 | CI-built, nightly releases |
| ESP32-P4 (Waveshare LCD-4B) | CI-built, nightly releases |
| ESP32-P4 (DFRobot FireBeetle 2) | CI-built, nightly releases |

Firmware is built with [ESP-IDF v5.5.1](https://docs.espressif.com/projects/esp-idf/en/v5.5.1/esp32c6/index.html) via the official Espressif Docker image.

## Repository Layout

```
wendy-lite/
  Sources/
    CWendyLite/
      include/wendy.h        Host-function declarations (WASM import attributes)
      shim.c                 Thin C shim for Swift interop
    WendyLite/               Swift SDK (SwiftPM library target)
      WendyLite.swift        Re-exports CWendyLite
      WendyLiteApp.swift     @main protocol + async runtime bootstrap
      WendyClock.swift       Embedded-Swift async clock + TimerHub
      CallbackDispatch.swift Handler registry + wendy_handle_callback export
      GPIO.swift             GPIO types and wrapper enum
      I2C.swift              I2C wrapper
      SPI.swift              SPI wrapper
      UART.swift             UART wrapper
      RMT.swift              RMT (remote control) wrapper
      NeoPixel.swift         NeoPixel/WS2812 wrapper
      Timer.swift            Low-level timer wrapper
      Network.swift          WiFi, Net (sockets), DNS, TLS wrappers
      BLE.swift              BLE, GATTS, GATTC wrappers
      OTel.swift             OpenTelemetry log/metrics/tracing wrapper
      Storage.swift          NVS key-value wrapper
      System.swift           System + Console wrappers
      USB.swift              USB CDC + HID wrapper
  src/lib.rs                 Rust crate (no_std FFI + safe wrappers)
  CMakeLists.txt             ESP-IDF component CMake (firmware build)
  Package.swift              SwiftPM package (Swift SDK)
  Cargo.toml                 Cargo package (Rust crate)
  boards/                    Per-board overlays for ESP32-P4 variants
    waveshare_lcd_4b.cfg     idf.py argfile for Waveshare LCD-4B
    waveshare_lcd_4b.defaults  sdkconfig overlay (32 MB flash, partition table)
    dfr1172_firebeetle.cfg   idf.py argfile for DFRobot FireBeetle 2
    dfr1172_firebeetle.defaults  sdkconfig overlay (16 MB flash, partition table)
  .github/workflows/
    build.yml                Matrix firmware build + nightly/release publishing
```

## CI and Releases

Every push to `main` and every pull request against `main` runs a matrix build using `espressif/idf:v5.5.1`. The matrix covers `esp32c5`, `esp32c6`, and two ESP32-P4 board variants (`esp32p4_waveshare_lcd_4b` and `esp32p4_dfr1172_firebeetle`). On merge to `main`, a `nightly` pre-release is created (or replaced) on GitHub containing all four merged firmware `.bin` files. On a `v*` tag push, the firmware is attached to the corresponding GitHub release.

## Guest Languages

| Language | Entry point | Library |
|----------|-------------|---------|
| Swift | `@main` on `WendyLiteApp` | SwiftPM: `WendyLite` (this repo) |
| Rust | `#[no_mangle] pub extern "C" fn _start()` | Cargo: `wendy-lite` (this repo) |
| C / C++ | `void _start(void)` | Include `Sources/CWendyLite/include/wendy.h` |
| AssemblyScript | `export function _start()` | `@external("wendy", ...)` declarations |
| WAT | `(export "_start" ...)` | Direct import from `"wendy"` module |

See [`host-api.md`](host-api.md) for the full function reference, [`swift-sdk.md`](swift-sdk.md) for Swift-specific internals, and [`deploying.md`](deploying.md) for the full build-to-device flow including OTA updates.

## Deploying a WASM App to Device

1. Build your app to `.wasm` (see the README in the repository root for per-language build commands).
2. Convert the binary to a C header array: `./wasm_apps/wasm2header.sh my_app.wasm main/demo_wasm.h`
3. Rebuild the firmware: `idf.py build`
4. Flash: `idf.py flash`

For ESP32-P4 targets, step 3 requires selecting a board overlay first — see [ESP32-P4 notes](#esp32-p4-notes) below.

## Building the Firmware

```bash
idf.py set-target esp32c5      # for the C5 DevKit
# or
idf.py set-target esp32c6      # for the C6 DevKit
# or, for ESP32-P4, select a board overlay:
idf.py @boards/waveshare_lcd_4b.cfg   set-target esp32p4
idf.py @boards/dfr1172_firebeetle.cfg set-target esp32p4
```

Then build and flash:

```bash
idf.py build
idf.py flash
idf.py monitor
```

## Supported Hardware

| Target | Reference Boards | Connectivity | Flash | PSRAM | Allocator |
|---|---|---|---|---|---|
| `esp32c5` | Espressif ESP32-C5-DevKitC | Native (2.4 / 5 GHz Wi-Fi 6 + BLE) | 4 MB | - | system allocator |
| `esp32c6` | Espressif ESP32-C6-DevKitC | Native | 4 MB | - | system allocator |
| `esp32p4` | Waveshare ESP32-P4-WIFI6-Touch-LCD-4B, DFRobot DFR1172 FireBeetle 2, Espressif P4-Function-EV-Board, other P4+C6 boards (see notes) | Via on-board ESP32-C6 over SDIO (ESP-Hosted) | 16–32 MB | 32 MB | 24 MiB pre-allocated from PSRAM |

The three targets share the same source tree. Per-target overrides live in `sdkconfig.defaults.<target>`. The `esp32p4` target additionally selects a per-board overlay from `boards/` (see [ESP32-P4 notes](#esp32-p4-notes) below). Guest WASM binaries are interchangeable between targets.

## ESP32-P4 notes

The P4 build assumes the on-board co-processor is an ESP32-C6 wired the same way as Espressif's ESP32-P4-Function-EV-Board (SDIO 4-bit on GPIO14-19 plus reset on GPIO54). This is the official Espressif reference layout that the currently supported boards copy verbatim.

### Picking a P4 board

P4 boards differ in flash size and partition layout, so the build selects a board-specific overlay from `boards/`. The overlay is chosen with an `idf.py` argfile passed *before* `set-target`:

```bash
# Waveshare ESP32-P4-WIFI6-Touch-LCD-4B (32 MB flash, 4" 720x720 MIPI-DSI panel)
idf.py @boards/waveshare_lcd_4b.cfg set-target esp32p4

# DFRobot DFR1172 FireBeetle 2 ESP32-P4 (16 MB flash, headless)
idf.py @boards/dfr1172_firebeetle.cfg set-target esp32p4
```

A bare `idf.py set-target esp32p4` with no board overlay is rejected at configure time with an error pointing to the available overlays.

### Other P4 boards

For a new board, copy one of the files in `boards/` (the `.defaults` overlay, its `.cfg` argfile, and a partition CSV) and adjust the flash size + partition table. If the C6 is on different pins or talks over SPI instead of SDIO, also override `CONFIG_ESP_HOSTED_P4_DEV_BOARD_FUNC_BOARD` and set the transport pins via `idf.py menuconfig` -> *Component config -> ESP-Hosted config*. If you have rev 3.0+ silicon, drop `CONFIG_ESP32P4_SELECTS_REV_LESS_V3` / `CONFIG_ESP32P4_REV_MIN_100` from `sdkconfig.defaults.esp32p4`.
