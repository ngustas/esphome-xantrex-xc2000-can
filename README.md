# ESPHome – Xantrex Freedom XC Pro 2000 RV-C monitor

Monitor a Xantrex Freedom XC Pro 2000 inverter/charger from Home Assistant with an ESP32 and CAN transceiver. The configuration is receive-only: it decodes RV-C/J1939 status traffic but does not transmit commands or change inverter settings.

The decoder supports older U3 1.x communication cards as well as U3 2.14+ cards whose CAN source address may differ. It auto-discovers the inverter from inverter-specific DGNs instead of assuming the address is always `0x42`.

## Sensors

| Home Assistant value | RV-C DGN | Notes |
|---|---:|---|
| DC input volts, amps, watts | `0x1FEE8` | Inverter DC input |
| Charger target volts, amps, watts | `0x1FFC7` | Desired/control values on newer firmware |
| Charge-current percent and charger state | `0x1FFC7` | Percent is `raw / 2` |
| Measured charger volts, amps, watts, temperature | `0x1FEA3` | Added to the default transmit list in U3 2.14 |
| AC input volts, amps, VA, frequency | `0x1FFCA` | Watts label is retained for compatibility; mathematically V×A is apparent power |
| AC output volts, amps, VA, frequency | `0x1FFD7` | Observed AC instance `0x41` |
| AC output real power | `0x1FFD5` | Corrected from the original single-byte hypothesis |
| AC output reactive field | `0x1FFD5` | Raw/provisional pending signedness and scale validation |
| FET and transformer temperatures | `0x1FEBD` | `raw / 32 - 273 °C`; FET1 reporting fixed in U3 2.14 |
| Inverter and charger states | `0x1FFD4`, `0x1FFC7` | RV-C enum labels, including CC/CV state 7 |
| Xantrex source address and bus frame rate | — | Diagnostics |

See [CAN_DECODING.md](CAN_DECODING.md) for byte layouts, evidence levels, firmware behavior, and command safety notes.

## Hardware

Any ESP32/ESP32-S3 with a TWAI controller works. The example targets a LilyGo T-Display S3 / ESP32-S3-DevKitC-1 and an SN65HVD230 3.3 V CAN transceiver.

```text
ESP32-S3        SN65HVD230       Xantrex CAN port
GPIO11  ------> TX
GPIO12  <------ RX
3.3 V   ------  VCC
GND     ------  GND
                CANH  <-------> CANH (DB9 pin 7)
                CANL  <-------> CANL (DB9 pin 2)
                GND   <-------> GND  (DB9 pin 3 or 6)
```

The bus uses 250 kbit/s and 29-bit extended IDs. Add 120 ohm termination only when this node is physically at a bus end. Classic ESP32/WROOM-32 boards must not use GPIO11/12; GPIO21/22 or GPIO16/17 are common alternatives.

## Setup

1. Copy `xantrex_xc2000_can.yaml` into the ESPHome configuration directory.
2. Add `api_encryption_key`, `ota_password`, `ap_fallback_password`, `wifi_ssid`, and `wifi_password` to `secrets.yaml`.
3. Adjust `tx_pin` and `rx_pin` for the board.
4. Flash with `esphome run xantrex_xc2000_can.yaml` or the ESPHome Dashboard.
5. Add the discovered ESPHome device to Home Assistant.

Generate an API key with `esphome generate-api-key`.

### Address and instance configuration

The recommended defaults are:

```yaml
substitutions:
  xantrex_source_address: "0xFF"  # auto-discover
  inv_instance: "1"              # observed inverter/DC instance
  ac_instance: "0x41"            # observed AC output instance
```

U3 2.14 changed the communication card's default CAN node address to decimal 143 (`0x8F`) and supports dynamic addressing. Older captures used decimal 66 (`0x42`). Auto-discovery is therefore safer than a fixed filter. If two compatible inverters share one bus, set `xantrex_source_address` explicitly after checking the diagnostic sensor/log.

U3 2.16 added a separately configurable DC instance for matching BMS `DC_SRC_STS` traffic. This monitor does not transmit BMS messages, but integrations that do must match the configured DC instance.

### DC current calibration

Use `g_dc_current_gain` and `g_dc_current_offset` if a reference meter shows a repeatable error. `g_invert_dc_current` changes the sign convention; the default follows the Xantrex observation where positive inverter DC current means discharge.

## Firmware update warning

The two current U3 2.17 packages reach the same application (`U3 2.17`) and segmented bootloader (`3.00`) through different, non-interchangeable migration paths:

| Starting communication card | Board | Required package |
|---|---|---|
| U3 `<= 1.09` | `10-0431-01-01` | `150-0324-01-05` |
| U3 `>= 2.08` | `10-0431-01-02` | `150-0317-01-07` |

Never mix files between these packages. Follow the Xantrex work instructions for the exact starting U3 version and board assembly.

For Freedom XC Pro 2000 part `818-2010`, Xantrex lists U1 `>= 2.96` as the prerequisite for U3 2.17 charging-control behavior. A unit at U1 2.87 / U3 1.06 follows the U3 1.0x migration package, but the U1 prerequisite must be resolved before relying on charging control.

Firmware updates can reset configuration and the CAN address. Recheck model, instances, battery type, charge-current limit, and the discovered source address afterward.

## Transmit controls

This repository intentionally remains monitor-only. Xantrex release notes confirm that U3 2.11 improved inverter/charger on/off command handling and U3 2.17 accepts a valid CC/CV Control Current through `CHARGER_COMMAND` (`0x1FFC5`) without overwriting non-volatile MAX_CHARGE_CURRENT setting #24. That does not make arbitrary transmit payloads safe.

Documented command DGNs include:

- `0x1FFD3` – inverter enable, pass-through, and load-sense command
- `0x1FFC5` – charger enable/disable/actions and CC/CV control current
- `0x1FFC4` – charger configuration
- `0x1FFD0` / `0x1FFCF` – inverter configuration

Any future transmit implementation should be opt-in, claim a non-conflicting source address, validate target source and instance, rate-limit writes, use reserved/not-available encodings correctly, and require status/readback confirmation. Do not treat untested control fields as proven safe.

## Evidence labels

- **Xantrex-confirmed:** local Xantrex work instructions and release notes.
- **RV-C-defined:** field names/encodings from the RV-C standard.
- **Observed:** bus captures from a Freedom XC Pro 2000.
- **Provisional:** plausible interpretations that still need controlled validation.

## License

MIT — use freely, no warranty. Not affiliated with or endorsed by Xantrex.
