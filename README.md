# FoxESS T-Series — ESPHome / Home Assistant

Read your FoxESS T-Series solar inverter into Home Assistant in real time using an ESP32-C3 and RS485.

## Hardware

| Component | Detail |
|-----------|--------|
| Microcontroller | ESP32-C3 DevKitM-1 |
| Protocol | RS485 (FoxESS proprietary, 9600 baud) |
| UART TX | GPIO0 |
| UART RX | GPIO10 |
| RS485 flow control (DE/RE) | GPIO4 |
| Status LED | GPIO7 (active low) |

Wire the RS485 adapter to the inverter's RS485 port (A/B terminals). FoxCloud and RS485 monitoring coexist safely — installer access is unaffected.

## Software

- **ESPHome** (tested on 2026.4.3+)
- **Home Assistant** (any recent version)

## Folder structure

```
esphome/
├── components/
│   └── foxess_solar/        # Local external component (vendored from PR #26)
│       ├── __init__.py
│       ├── foxess_solar.h
│       ├── foxess_solar.cpp
│       └── sensor.py
├── foxess-inverter.yaml     # Main ESPHome config
├── secrets.yaml             # Credentials (not committed)
└── old/                     # Archived — no longer needed
```

## Setup

### 1. secrets.yaml

Create `secrets.yaml` with these keys (not committed to git):

```yaml
wifi_ssid: "your-wifi"
wifi_password: "your-wifi-password"
esphome_ota_pass: "your-ota-password"
esphome_api_encryption_key: "base64-32-byte-key"   # generate: esphome generate-encryption-key
foxess_ap_ssid: "FoxESS Inverter Fallback"
foxess_ap_pass: "your-fallback-ap-password"
```

Generate an API encryption key:
```bash
esphome generate-encryption-key
```

### 2. Flash

```bash
esphome run foxess-inverter.yaml
```

Subsequent updates are via OTA (no USB needed).

### 3. Add to Home Assistant

ESPHome will be discovered automatically. Accept the device and enter your API encryption key when prompted.

## Sensors exposed to Home Assistant

### Power (live, 1 s update)

| Entity | Unit | Notes |
|--------|------|-------|
| T-Series Grid Power | W | Signed: + = import, − = export |
| T-Series Generation Power | W | Total inverter output |
| T-Series Loads Power | W | House consumption |
| T-Series Grid Power R/S/T | W | Per-phase |

### Voltage / Current / Frequency (throttled to 1 h)

Per-phase R, S, T — voltage (V), current (A), frequency (Hz).

### PV Strings

PV1–PV4 voltage, current, and power (calculated as V×I).  
PV3/PV4 only relevant on T8dual–T12dual and T15–T25 models.

### Temperature (throttled to 10 min)

Boost, Inverter, and Ambient temperatures in °C.

### Energy (HA Energy Dashboard)

| Entity | Unit | state_class |
|--------|------|-------------|
| T-Series Today Yield | kWh | `total_increasing` |
| T-Series Generation Total | kWh | `total_increasing` |

Both are pre-configured for the **HA Energy Dashboard** — just add "T-Series Generation Total" as your solar production source.

### Status

| Entity | Type | Values |
|--------|------|--------|
| T-Series Inverter Mode | Text sensor | Offline / Online / Error / Waiting… |

## Component notes

The `components/foxess_solar/` folder is a vendored copy of [PR #26](https://github.com/assembly12/Foxess-T-series-ESPHome-Home-Assistant/pull/26) by [@SibrenVasse](https://github.com/SibrenVasse). It replaces the original `platform: custom` approach (removed in ESPHome 2026.4.3) with a proper `external_components` architecture.

Key improvements over the original `foxess_t_series.h`:

| Issue | Old | New |
|-------|-----|-----|
| ESPHome compatibility | Broken (`platform: custom` removed) | ✅ `external_components` |
| `delay()` calls | 30+ blocking delays (≥50 ms each) | ✅ Non-blocking, event-driven |
| CRC validation | None — corrupt frames accepted silently | ✅ CRC16 checked |
| Grid power sign | Unsigned (import/export indistinguishable) | ✅ Signed int16 |
| Buffer | Growing `std::vector` → heap fragmentation | ✅ Fixed 256-byte `std::array` |
| Sensor allocation | 37× `new Sensor` — heap never freed | ✅ Stack-allocated structs |
| AP credentials | Hardcoded in YAML | ✅ `!secret` references |
| API security | Unencrypted | ✅ Encrypted with key |

## Troubleshooting

**Inverter shows "Waiting for response…"**  
Check RS485 wiring polarity (swap A/B if needed). Verify GPIO4 is your DE/RE pin.

**CRC errors in logs**  
Cable too long (>4 m can be marginal). Try ferrite bead on RS485 cable.

**Unexpected message length warning (expected 163)**  
Older inverter firmware sends a shorter frame. The component will still parse what it can.

**HA Energy Dashboard shows wrong values**  
Ensure "T-Series Generation Total" is added as the solar source (not "Today Yield" — that resets daily).
