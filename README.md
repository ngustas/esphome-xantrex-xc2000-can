# ESPHome – Xantrex Freedom XC Pro 2000 RV-C monitor

Monitor a Xantrex Freedom XC Pro 2000 inverter/charger from Home Assistant with an ESP32 and CAN transceiver. The configuration is receive-only: it decodes RV-C/J1939 status traffic but does not transmit commands or change inverter settings.

The decoder supports older U3 1.x communication cards as well as U3 2.14+ cards whose CAN source address may differ. It auto-discovers the inverter from inverter-specific DGNs instead of assuming the address is always `0x42`.

## Sensors

| Home Assistant value | RV-C DGN | Notes |
|---|---:|---|
| **Net DC volts, amps, watts** | `0x1FFFD` | **Measured battery flow; + = charging.** The figure to trust |
| DC source temperature | `0x1FFFC` | |
| DC source SOC, time remaining | `0x1FFFC` | Diagnostic only — not populated by the observed unit, see below |
| DC input volts, amps, watts | `0x1FEE8` | Inverter DC *input*; reads 0 while charging or in pass-through |
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

This unit measures **charge** current accurately and **discharge** current
poorly. Against a SmartShunt and a Fluke clamp meter on the same conductor:

| Direction | Unit reports | Reference | Ratio |
|---|---|---|---|
| Discharging (inverting) | 7.53 A | 9.77 A | 1.30 |
| Discharging (inverting) | 7.04 A | 9.33 A | 1.33 |
| Charging, 45–100 A | — | — | within 1% |

`g_dc_current_gain` is therefore applied to the **discharge direction only**
on `0x1FFFD`, and to `0x1FEE8` (which is only non-zero while inverting). It is
deliberately *not* applied to `0x1FEA3` charger output. A blanket gain would
corrupt the charge readings, which are already correct.

The default 1.30 matches the observed unit at roughly 7.5 A of draw — a single
calibration point. Check it against a reference meter at your own load before
relying on it. `g_dc_current_offset` handles a fixed offset;
`g_invert_dc_current` flips the `0x1FEE8` sign convention.

### State of charge is not real

The SOC field in `0x1FFFC` reads a constant **100%** on the observed unit. It
has no battery capacity configured (`0x1FFC6` reports bank size as "not
available", and `0x1FFFB` is not implemented at all), and it measures only its
own current, so it cannot compute one. The value is a valid encoding rather
than the "unavailable" code, so it cannot be filtered out — it is exposed as a
diagnostic entity named *unverified*. Use a battery shunt or BMS instead.

## Firmware update warning

The two current U3 2.17 packages reach the same application (`U3 2.17`) and segmented bootloader (`3.00`) through different, non-interchangeable migration paths:

| Starting communication card | Board | Required package |
|---|---|---|
| U3 `<= 1.09` | `10-0431-01-01` | `150-0324-01-05` |
| U3 `>= 2.08` | `10-0431-01-02` | `150-0317-01-07` |

Never mix files between these packages. Follow the Xantrex work instructions for the exact starting U3 version and board assembly.

For Freedom XC Pro 2000 part `818-2010`, Xantrex lists U1 `>= 2.96` as the prerequisite for U3 2.17 charging-control behavior. A unit at U1 2.87 / U3 1.06 follows the U3 1.0x migration package, but the U1 prerequisite must be resolved before relying on charging control.

The bootloader update reinitialises the card's flash file system and **resets
the entire CAN configuration section to defaults while leaving the inverter
and charger settings untouched**. Observed on a 1.06 → 2.17 update: node
address 66 → 143, the partial-networking wake mask widened, the receive
heartbeat changed from `IsoAddrClaim` to `DiagMsg1`, and `DCSrcSts1` /
`ChgSts2` were added to the transmit list. Battery type, charge current and
voltage setpoints all survived.

The practical effect is that an integration keyed to the old node address goes
silent after the update even though the inverter is fine. Auto-discovery in
this config handles that automatically. See
[CAN_DECODING.md](CAN_DECODING.md) for the full before/after table.

### If RV-C traffic stops

Check wiring, connectors and termination first.

The card has ISO 11898-6 partial networking enabled and logs a sleep command
when the inverter reaches standby, which makes "it went to sleep" a tempting
explanation for a dropout. It is not a verified one: over four days of
continuous monitoring with the bus connected, the frame rate never dropped
for even 60 seconds.

The two causes look different. A genuine sleep would follow the inverter into
standby and recover when it wakes. A wiring fault stops abruptly, at one
instant, and stays stopped regardless of what the inverter is doing.

## Transmit controls

This repository intentionally remains monitor-only. Transmit was nonetheless
exercised on hardware to establish what actually works, and the results are
documented so that anyone adding a transmitter starts from evidence rather
than from release-note prose:

| DGN | Result on hardware |
|---|---|
| `0x1FFC4` max charge current | **Works.** Layout validated, non-destructive with correct no-change fill, applied in ~2 s |
| `0x1FFC5` charger enable/disable | **Works.** State changes within ~2 s |
| `0x1FFC5` CC/CV control current | **Inert** in 3-stage mode; two payload variants tried |
| `0x1FFD3` inverter enable | Untested |
| `0x1FFD3` pass-through enable | Untested; Xantrex marks the equivalent field unsupported on the Freedom SW |

`0x1FFC4` writes **non-volatile setting #24**, which survives a power cycle —
setting a low charge limit and forgetting is a genuine hazard. None of this
makes arbitrary transmit payloads safe.

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

## Known gaps and open decisions

Everything below is deliberately unresolved rather than overlooked. Detail and
evidence for each is in [CAN_DECODING.md](CAN_DECODING.md).

### Decisions a maintainer should revisit

- **Should the SOC entity exist at all?** `DC Source SOC (unverified)` is a
  constant 100% placeholder on the one unit tested, and it is a valid encoding
  rather than the "unavailable" code, so it cannot be filtered out. It is kept
  as a diagnostic because a different unit — or one with a BMS on the bus —
  may populate it honestly. If that never materialises, delete it: a sensor
  that reads a confident wrong value is worse than no sensor.
- **Should manufacturer code 119 drive discovery?** Identifying the inverter
  from its Address Claim NAME (manufacturer 119, function 129) would be more
  robust than the current DGN-signature inference, because the NAME survives
  the configuration reset a firmware update performs. It rests on **one**
  sample, so the code still discovers by DGN signature. Confirm 119 on another
  model or build date before changing that.

### Untested or single-sample

| Item | Status |
|---|---|
| Discharge current gain (1.30) | One calibration point at ~7.5 A; may not hold at heavier loads |
| `0x1FFD3` pass-through enable | Untested; Xantrex marks the equivalent field unsupported on the Freedom SW |
| `0x1FFD3` inverter enable | Untested |
| Partial-networking sleep/wake | Enabled and logged, but never observed to interrupt the bus; wake frame untested |
| `DC_SRC_STS4` | Byte layout not in any RV-C revision or library consulted; required for CC/CV |
| `0x1FFC5` control-current offset | Bytes 3–4 inferred from `0x1FFC7`; neither confirmed nor disproven |
| `0x1FFD5` reactive power | Raw; signedness and units unvalidated |
| `0x1FDAA` CHARGER_PROPERTIES | Supported from U3 2.15 but never captured |

Captures from a second unit are the single most useful contribution — see the
capture-setup notes at the end of CAN_DECODING.md.

## License

MIT — use freely, no warranty. Not affiliated with or endorsed by Xantrex.
