# Xantrex Freedom XC Pro 2000 RV-C decoding reference

This reference covers the Freedom XC Pro 2000 communication card on a 250 kbit/s RV-C/J1939 bus. It distinguishes vendor release-note facts, RV-C definitions, observations from captures, and interpretations that remain provisional.

## Confidence and provenance

| Label | Meaning |
|---|---|
| **Xantrex-confirmed** | Stated in Xantrex firmware release notes or update instructions |
| **RV-C-defined** | Defined by the RV-C data-group/parameter encoding |
| **Observed** | Seen in captures from a Freedom XC Pro 2000 |
| **Provisional** | Reasonable but not yet validated across controlled states/units |

The capture-derived values below used inverter/DC instance `1`, AC output instance `0x41`, and source address `0x42` on older firmware. Those are not universal constants.

## Bus and identifier handling

| Parameter | Value |
|---|---|
| Bit rate | 250 kbit/s |
| Frame format | 29-bit extended |
| Protocol | RV-C over J1939 conventions |
| Multi-byte order | Little-endian |

Extract the full 18-bit DGN, including the J1939 data-page bit:

```cpp
uint8_t pf = (can_id >> 16) & 0xff;
uint32_t dgn = (can_id >> 8) & 0x3ffff;
if (pf < 0xf0) dgn &= 0x3ff00; // PDU1: PS is destination
uint8_t source = can_id & 0xff;
```

Earlier notes/tools called `0x1FEE8` simply `FEE8` because they discarded the data-page bit. This project now uses the full RV-C DGN.

### Source address

**Observed:** old captures used `0x42`.

**Xantrex-confirmed:** U3 2.14 changed the default node address to decimal 143 (`0x8F`) and states that the Freedom XC Pro supports dynamic addressing. A bootloader update may also reset configuration.

Therefore a filter must mask the low source byte, and application code should either discover the source from inverter-specific traffic or allow it to be configured. The supplied YAML defaults to auto-discovery, latches only on inverter-specific DGNs, and permits rediscovery after 30 seconds of silence. Pin a source address when multiple inverters share a bus.

### Instances

**Observed:** inverter/DC status instance `1`; AC output instance `0x41` (AC line in the high nibble, instance in the low nibble).

**Xantrex-confirmed:** U3 2.16 introduced DC Instance configuration and only reacts to BMS `DC_SRC_STS` messages whose DC instance matches the configured value. Do not assume the inverter instance, DC-source instance, and AC instance are interchangeable.

## Common encodings

```text
voltage (uint16)       = raw * 0.05 V       = raw / 20
current (uint16)       = (raw - 32000) * 0.05 A
frequency (uint16)     = raw / 128 Hz
temperature (uint16)   = raw / 32 - 273 °C
percentage (uint8)     = raw / 2 %
```

Reserved/not-available values must not be scaled. The code rejects the reserved upper range for each parameter type and range-checks physical values. Consult the applicable definition before reusing a helper for transmit code. Power calculated as volts × amps is apparent power (VA) unless a DGN explicitly supplies real power.

## Status DGNs

### `0x1FEE8` – Inverter DC Status

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 | Inverter instance | exact match | Observed/RV-C-defined |
| 1–2 | DC input voltage | `raw / 20 V` | Observed |
| 3–4 | DC input current | `(raw - 32000) / 20 A` | Observed |
| 5–7 | unavailable in captures | — | Observed |

Positive current meant inverter discharge in the captures. During pass-through/charging, this inverter-input current may be near zero and is not a battery-shunt measurement.

### `0x1FFC7` – Charger Status 1

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 | Charger/inverter instance | exact match | Observed/RV-C-defined |
| 1–2 | Desired charge voltage | `raw / 20 V` | RV-C-defined; observed |
| 3–4 | Desired charge current | `(raw - 32000) / 20 A` | RV-C-defined; observed |
| 5 | Charge current percent of max | `raw / 2 %` | Xantrex-confirmed in U3 2.14; RV-C-defined |
| 6 | Charger operating state | enum below | RV-C-defined; observed |

U3 2.15 explicitly separated desired charge current reported here from the user-configured maximum current reported in `CHARGER_CONFIGURATION_STATUS` (`0x1FFC6`). Do not present `0x1FFC7` current as an independent measured battery current.

| Value | Charger state |
|---:|---|
| 0 | Undefined |
| 1 | Do not charge |
| 2 | Bulk |
| 3 | Absorption |
| 4 | Overcharge |
| 5 | Equalize |
| 6 | Float |
| 7 | Constant voltage/current |

### `0x1FEA3` – Charger Status 2

**Xantrex-confirmed:** U3 2.14 implemented the transmit handler and added this DGN to the default transmit list. U3 2.16 corrected its temperature reporting.

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 low nibble | Instance | compare to configured inverter instance | RV-C-defined |
| 0 high nibble | Charger type | enum not exposed yet | RV-C-defined |
| 3–4 | Measured output voltage | `raw / 20 V` | RV-C-defined/observed |
| 5–6 | Measured output current | `(raw - 32000) / 20 A` | RV-C-defined; sign needs broader validation |
| 7 | Charger temperature | `raw - 40 °C` | RV-C-defined; Xantrex fix in 2.16 |

This is the preferred measured charger-output frame on U3 2.14+.

### `0x1FFCA` – AC Input Status

| Byte(s) | Field | Decode |
|---|---|---|
| 0 | Inverter instance | exact match |
| 1–2 | RMS voltage | `raw / 20 V` |
| 3–4 | RMS current | `(raw - 32000) / 20 A` |
| 5–6 | Frequency | `raw / 128 Hz` |

Observed examples included 112.3 V, 14.8 A, and 59.9 Hz (`0x1DF3 / 128`). Zero frequency is valid when AC is absent.

### `0x1FFD7` – AC Output Status

Same measurement layout as `0x1FFCA`, but byte 0 is the AC instance. Captures used `0x41`. Output frequency tracked input frequency during pass-through.

### `0x1FFD5` – AC Output Power

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 | AC instance | exact match | Observed |
| 2–3 | Real output power | little-endian uint16 W | RV-C-defined; consistent with captures |
| 4–5 | Reactive output power | retained as raw uint16 | Provisional signedness/scaling |

The repository originally treated bytes 2 and 4 as unrelated single-byte fields and hypothesized temperature or duty cycle. That interpretation was incorrect: each is the low byte of a 16-bit power field. The apparent square-wave behavior is plausible for a changing load, not a thermal signal.

### `0x1FEBD` – Inverter Temperature Status

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 | Inverter instance | exact match | Observed |
| 1–2 | FET1 temperature | `raw / 32 - 273 °C` | RV-C-defined; U3 2.14 fix confirmed by Xantrex |
| 3–4 | Transformer temperature | same; may be unavailable | RV-C-defined/observed |

### `0x1FFD4` – Inverter Status

The low nibble of byte 1 is decoded as:

| Value | State |
|---:|---|
| 0 | Off |
| 1 | Inverting |
| 2 | Pass-through |
| 3 | APS only |
| 4 | Load sense |
| 5 | Waiting to invert |
| 6 | Short circuit |
| 7 | Overtemperature |
| 8 | Undervoltage |
| 9 | Overvoltage |
| 10 | Internal fault |
| 11 | Communication lost |

## Firmware behavior relevant to RV-C

These items are **Xantrex-confirmed** in the supplied release notes:

| U3 | Relevant change |
|---:|---|
| 2.11 | Reduced reaction time to inverter and charger On/Off RV-C commands; confirms command support |
| 2.14 | Added/fixed `0x1FEA3`, FET1 temperature in `0x1FEBD`, charge-current-percent in `0x1FFC7`; default address changed to 143 with dynamic addressing |
| 2.15 | Added `CHARGER_PROPERTIES` (`0x1FDAA`); separated desired current from configured maximum; corrected RV-C encodings/ranges |
| 2.16 | Added DC Instance configuration/matching for BMS messages; corrected `0x1FEA3` temperature and BMS/user max-current coordination |
| 2.17 | Applies a valid CC/CV Control Current received via `CHARGER_COMMAND` (`0x1FFC5`) using a volatile parameter that does not overwrite MAX_CHARGE_CURRENT setting #24 |

`0x1FDAA` is documented as supported from 2.15, but this repository does not expose fields because no authoritative byte-level decode was validated against captures.

## U3 2.17 migration paths

Both current packages install communication-card application U3 2.17 and segmented bootloader 3.00, but the files and migration paths are not interchangeable.

| Package | Starting firmware | Board assembly |
|---|---|---|
| `150-0324-01-05` | U3 `<= 1.09` | `10-0431-01-01` |
| `150-0317-01-07` | U3 `>= 2.08` | `10-0431-01-02` |

For Freedom XC Pro 2000 part `818-2010`, the Xantrex compatibility table requires U1 `>= 2.96` for U3 2.17 charging control. A unit observed at U1 2.87 and U3 1.06 belongs on the 1.0x U3 migration path, but its U1 prerequisite remains unresolved; monitoring support does not imply charging-control readiness.

Always identify the installed U3 version and board assembly, use the matching complete package, and follow the vendor work instructions. Never combine files from the two packages.

## Commands: documented, not enabled

| DGN | Purpose | Status here |
|---:|---|---|
| `0x1FFD3` | Inverter enable, pass-through, load sense | Documented; no transmitter shipped |
| `0x1FFC5` | Charger enable/disable/actions and CC/CV control current | Xantrex support confirmed; control-current layout/use still requires validation |
| `0x1FFC4` | Charger configuration | Can alter persistent charge settings; no transmitter shipped |
| `0x1FFD0` / `0x1FFCF` | Inverter configuration | Documented; no transmitter shipped |

U3 2.17's volatile Control Current is specifically for CC/CV mode and does not modify setting #24. Tests in a three-stage charging configuration found candidate `0x1FFC5` control-current payloads inert, which is consistent with the command being mode-dependent; it is not proof of a general-purpose current limiter.

A responsible transmitter must be opt-in and should:

1. claim a non-conflicting RV-C address;
2. validate the identified Xantrex source and configured instance;
3. distinguish volatile control current from persistent maximum-charge configuration;
4. rate-limit and range-check every command;
5. fill unrelated fields with the correct reserved/not-available values; and
6. confirm the result through status/configuration readback, failing closed on mismatch.

## Remaining validation work

- Validate `0x1FEA3` current sign across charge/discharge transitions.
- Confirm the reactive-power field's signedness and engineering units in `0x1FFD5`.
- Capture `0x1FDAA` (`CHARGER_PROPERTIES`) from U3 2.15+ and map it against an authoritative RV-C definition.
- Collect ISO Address Claim NAMEs from multiple XC Pro models before using manufacturer/function identity as a public auto-discovery rule.
- Confirm status/instance layouts on a second unit and on pre-2.14 firmware.

## Capture setup

- ESP32-S3 with SN65HVD230-compatible transceiver
- 250 kbit/s, extended frames
- Xantrex Freedom XC Pro 2000, part `818-2010`
- Operating states included inverter-only, AC pass-through, charging, and changing load

Contributions with raw timestamped frames, U1/U3 versions, part number, source address, configured instances, and a simultaneous reference measurement are especially useful.
