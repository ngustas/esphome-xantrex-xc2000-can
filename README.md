# ESPHome – Xantrex XC 2000 Pro CAN Bus Monitor

Monitor your Xantrex XC 2000 Pro inverter/charger from Home Assistant using an ESP32 and a cheap CAN transceiver. All values are decoded directly from the unit's J1939 CAN bus — no serial adapter, no Modbus, no proprietary gateway required.

**Sensors published to Home Assistant:**

| Sensor | Source PGN | Unit |
|---|---|---|
| DC In Volts | FEE8 | V |
| DC In Amps | FEE8 | A |
| DC In Watts | FEE8 | W |
| Charger Volts | FFC7 | V |
| Charger Amps | FFC7 | A |
| Charger Watts | FFC7 | W |
| AC In Volts | FFCA | V |
| AC In Amps | FFCA | A |
| AC In Watts | FFCA | W |
| AC In Frequency | FFCA | Hz |
| AC Out Volts | FFD7 | V |
| AC Out Amps | FFD7 | A |
| AC Out Watts | FFD7 | W |
| AC Out Frequency | FFD7 | Hz |
| FFD5 Field A | FFD5 | raw |
| FFD5 Field B | FFD5 | raw |
| CAN Frames/s | — | fps |

> **FFD5 — decode unknown:** FFD5 carries two single-byte fields (d[2] and d[4]). The FFD7 voltage/current formula does not apply (produces 1212 V / −1595 A). A temperature hypothesis was proposed based on static captures (values 70–99) but ruled out by live data: Field B oscillates in large square-wave transitions every ~20–30 seconds during pass-through mode, which is inconsistent with any thermal sensor. Current best guess is charger duty-cycle or internal state signalling. Both fields are logged as raw integers. See [CAN_DECODING.md](CAN_DECODING.md) for full analysis.

---

## Hardware

### ESP32 board

Any ESP32 or ESP32-S3 with a built-in TWAI (CAN) controller works. Tested on a **LilyGo T-Display S3** (ESP32-S3-DevKitC-1 compatible).

> **Classic ESP32 (WROOM-32) users:** GPIOs 11 and 12 are tied to the SPI flash on that module. Use a different pair — GPIO21/GPIO22 or GPIO16/GPIO17 are common safe choices. Update `tx_pin` and `rx_pin` in the YAML accordingly.

### CAN transceiver

The ESP32 TWAI peripheral is a controller only — it needs a physical-layer transceiver to drive the bus. A **SN65HVD230** (3.3 V) module is the most common choice and costs a few dollars.

```
ESP32-S3        SN65HVD230       Xantrex CAN port
─────────       ──────────       ────────────────
GPIO11  ──────► TX               
GPIO12  ◄─────  RX               
3.3 V   ──────  VCC              
GND     ──────  GND              
                CANH  ◄────────► CANH
                CANL  ◄────────► CANL
```

The Xantrex XC 2000 Pro has a standard 9-pin DB9 CAN port (CANopen/J1939 pinout):
- Pin 2 → CANL  
- Pin 7 → CANH  
- Pin 3 / Pin 6 → GND  

> Add a 120 Ω termination resistor across CANH/CANL if the ESP32 node is at either end of the bus. Many SN65HVD230 breakout boards have a solder-jumper for this.

---

## Setup

### 1. Copy the YAML

Copy `xantrex_xc2000_can.yaml` into your ESPHome config directory (the folder where your other `.yaml` devices live).

### 2. Add secrets

Add the following entries to your `secrets.yaml`:

```yaml
api_encryption_key: "generate with: esphome generate-api-key"
ota_password: "your-ota-password"
ap_fallback_password: "your-fallback-ap-password"
wifi_ssid: "your-wifi-ssid"
wifi_password: "your-wifi-password"
```

Generate a fresh API encryption key with:
```bash
esphome generate-api-key
```

### 3. Adjust pins (if needed)

If you are not using an ESP32-S3 or your board uses different CAN pins, update these two lines:

```yaml
canbus:
  - platform: esp32_can
    tx_pin: GPIO11    # ← change to your TX pin
    rx_pin: GPIO12    # ← change to your RX pin
```

### 4. Flash

```bash
esphome run xantrex_xc2000_can.yaml
```

Or use the ESPHome Dashboard — add the file and click **Install**.

### 5. Add to Home Assistant

Once the device is online, Home Assistant will discover it automatically via the ESPHome integration. Accept the prompt and enter the API key you set in `secrets.yaml`.

---

## Configuration options

### DC current calibration

If the reported DC current is consistently off by a fixed ratio, adjust the gain global. For example, if the unit reports 7 A but a clamp meter reads 9.3 A:

```yaml
globals:
  - id: g_dc_current_gain
    initial_value: '1.33'    # 9.3 / 7 ≈ 1.33
```

A fixed offset (in amps) can also be dialled in with `g_dc_current_offset`.

### DC current sign convention

By default, positive current means the battery is discharging (Xantrex convention). Set `g_invert_dc_current` to `true` if you want positive to mean charging (Victron convention):

```yaml
globals:
  - id: g_invert_dc_current
    initial_value: 'true'
```

### Averaging window

All sensors use a `throttle_average` filter. The default window is 2 seconds. Change the substitution to suit your dashboard refresh rate:

```yaml
substitutions:
  AVG_WIN: "5s"    # slower, smoother
```

---

## CAN bus wiring tips

- Keep CAN wiring twisted-pair and away from AC lines.
- The Xantrex runs at **250 kbps** — do not change `bit_rate`.
- If you see `esp32_can: E` errors in the log, check termination, wire length, and that CANH/CANL are not swapped.
- The `CAN Frames/s` sensor is useful for confirming the bus is active. A healthy idle typically shows 10–30 fps.

---

## Technical reference

See [CAN_DECODING.md](CAN_DECODING.md) for the full byte-level breakdown of each PGN, scaling formulas, and notes from the original CAN sniffing session.

---

## License

MIT — use freely, no warranty. Not affiliated with or endorsed by Xantrex / Schneider Electric.
