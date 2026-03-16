# faraday-rescue — Claude Context

## Project Summary

Open-source replacement battery for Faraday e-bikes (now-defunct). The original battery had ~300uA quiescent draw that bricked packs in 2–3 weeks when left idle. This design achieves ~3uA using a low-power STM32L0 MCU and TI BQ76972 battery management IC.

**Hardware:** 12S 2P, 24× Samsung 35E 18650 cells, custom PCB fits in a 41mm tube.
**Software:** STM32 HAL C firmware (STM32CubeIDE), communicates with the bike over RS485.

---

## Repository Layout

```
faraday-rescue/
├── build/
│   ├── README.md          # Full build guide + BOM
│   ├── cad/               # STEP files, 3D-print STLs, lasercut/machining drawings
│   └── pcb/
│       ├── faraday-rescue-bms/   # KiCad project + JLCPCB production files
│       └── battery-simulator/
├── reference/
│   ├── design/README.md   # Full design doc (cell selection, IC choices, sleep/wake, PCB)
│   ├── official/          # Original Faraday docs, firmware utilities, dealer handbook
│   ├── rebuild/           # Cell-only rebuild guides (PDFs)
│   └── teardowns/         # Photos/notes on BMS, mode selector, rear controller
├── software/
│   ├── bms/               # STM32CubeIDE project (the active firmware)
│   │   ├── Core/Src/main.c          # ← main firmware (build this one)
│   │   ├── Core/Src/BQ769x2.c      # BQ76972 driver
│   │   ├── Core/Inc/BQ769x2.h
│   │   ├── Core/Inc/utils.h
│   │   └── Release/faraday_stm32_bms.bin  # pre-built binary
│   └── bus_snooper/
│       ├── parse_raw_log.py         # decode RS485 bus captures
│       └── *.log                    # captured RS485 sessions
└── test/
    ├── bms_test_log.md
    └── realtime/
        └── faraday_bms_stmstudio_config.tsc   # STMStudio config for live data
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| `software/bms/Core/Src/main.c` | Active firmware — sleep/wake FSM, RS485 comms, BQ config |
| `software/bms/Core/Src/BQ769x2.c` | BQ76972 I2C driver |
| `software/bms/Core/Inc/BQ769x2Header.h` | BQ76972 register map |
| `software/bms/Core/Src/utils.c` | Shared helpers |
| `software/bus_snooper/parse_raw_log.py` | Python RS485 log parser |

> **Note:** `Core/Src/` also contains several archived `main_*.c` snapshots (e.g. `main_working_sleep.c`, `main_flashed_to_battery 26JAN25.c`). These are reference copies — only `main.c` is compiled.

---

## Firmware Build Flags (in main.c)

| Flag | Production value | Notes |
|------|-----------------|-------|
| `DEBUG` | `0` | Enables printf over RS485 at 115200 baud when `1` |
| `LEDS` | `0` | Enables CHG/DSG LEDs when `1`; costs power and heats linear reg |
| `WATCHDOG` | `1` | IWDG enabled |
| `RESET_3V3` | `0` | Set `1` only when needing hard STM32 reset during debug |

---

## Hardware Architecture

- **BQ76972** — handles OV/UV/OC/balancing/temp, high-side back-to-back NFET drive, DEEPSLEEP/SHUTDOWN states, wake-on-charger (CD pin)
- **STM32L010K8** — manages RS485 comms + sleep FSM; wakes on button (active-low EXTI in STOP mode) or BQ ALERT pin
- **RS485** — 3.3V bus, matches original Faraday pinout; battery talks to rear controller + mode selector
- **Sleep power:** ~3uA (BQ DEEPSLEEP + STM32 STOP); falls to <50uA UVLO SHUTDOWN to prevent bricking

### STMStudio Live Memory Addresses (useful for debug)

| Value | Address | Type |
|-------|---------|------|
| Current | `0x200003fe` | int16 mA |
| Temp 1–3 | `0x200003e8/ec/f0` | float °C |
| Temp 4 (BQ die) | `0x200003f4` | float °C |
| Min Cell | `0x200003e4` | uint16 mV |
| Max Cell | `0x200003e6` | uint16 mV |

---

## Development Environment

- **IDE:** STM32CubeIDE (Eclipse-based), project root at `software/bms/`
- **Programmer:** ST-Link V2 via 5-pin green Julet connector → SWD
- **RS485 debug:** Waveshare RS485-USB adapter, forced to COM3 (Windows); use Faraday BMS Tester utility
- **Bus analysis:** Capture with logic analyzer → `software/bus_snooper/parse_raw_log.py`

---

## Cell Protection Thresholds (Samsung 35E)

| Parameter | Value |
|-----------|-------|
| Cell OV | 4199 mV |
| Cell UV | 2631 mV |
| Shutdown UV | 2400 mV |
| Safety UV (permanent fail) | 1900 mV |
| Max charge temp | 45°C |
| Min charge temp | 0°C |
| Max discharge temp | 60°C |

---

## Design Principles

1. **Delegate all safety-critical functions to BQ76972** — STM32 is communications + sleep only.
2. **Reset at every wake** — avoids long-term state bugs; power button causes full MCU reset.
3. **When in doubt, reset** — watchdog always on in production.
4. **No switching regulators** — linear reg simplicity preferred over efficiency.
