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

Confirmed on hardware: a unit updated from U3 1.06 to 2.17 moved from `0x42`
to `0x8F` as part of the update, and auto-discovery followed it without
intervention.

#### Observed ISO Address Claim

The card transmits `ADDRESS_CLAIM` (`0xEE00`) once per second, which is
unusual — RV-C specifies "on request only" broadcast, and most nodes answer
only when asked. One unit produced:

```text
src 0x8F (143)  NAME 803C81080EFE6696
  manufacturer code 119      function 129 (inverter/charger)
  function instance 1        identity 0x1E6696
  arbitrary address capable  SET
```

The arbitrary-address-capable bit being set is the device stating outright
that it does dynamic addressing, which is why no fixed address is safe.

Identifying the unit by manufacturer + function is more robust than inferring
it from DGN traffic, because the NAME is burned into the device and survives
the configuration reset described below. **This is a single sample** — do not
treat manufacturer 119 as a universal Xantrex constant until it is confirmed
on other models and build dates. The supplied YAML therefore still
discovers by DGN signature.

#### Surveying a bus

Listening passively for `ADDRESS_CLAIM` will miss most nodes, for the reason
above. To enumerate a bus, log every distinct source address seen on any
frame, or send a global DGN request for `0xEE00`.

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

Positive current meant inverter discharge in the captures. During
pass-through or charging this field reads **0.0 A**, because it is the
inverter's DC input rather than a battery measurement. Confirmed on hardware:
while the battery was being charged at 97 A, FEE8 still read `0.0`.

Measured accuracy while inverting is poor. Against a SmartShunt and a Fluke
clamp meter on the same conductor, the unit reported 7.53 A where both
references read 9.77 A, and 7.04 A against 9.33 A — ratios of 1.30 and 1.33.
Charge current from `0x1FFFD` / `0x1FEA3` on the same unit is accurate to
within 1%, so this is specific to the inverting path and not a global scale
error. A plausible cause is that inverter DC draw pulsates at twice line
frequency and the unit's averaging under-reads it. Apply `g_dc_current_gain`
to the discharge direction only.

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
| 5–6 | Measured output current | `(raw - 32000) / 20 A` | RV-C-defined; observed |
| 7 | Charger temperature | `raw - 40 °C` | RV-C-defined; Xantrex fix in 2.16 |

Sign convention is **positive = charging**, confirmed against a shunt during a
bulk charge: this field and `0x1FFFD` agreed to within 0.16 A across a
45–100 A range, and the shunt tracked both to within ~1%. Note that this is
the opposite polarity to `0x1FFFD`, which needs negating.

It reads `0.00 A` whenever the charger is idle, including while the inverter
is running — it reports charger output, not net battery current. Use
`0x1FFFD` for battery flow.

This is the preferred measured charger-output frame on U3 2.14+.

### `0x1FFFD` – DC Source Status 1

**Observed.** The only measured net DC current this unit publishes, and the
figure to trust for battery flow. It is transmitted every 500 ms; U3 2.16/2.17
added `DCSrcSts1` to the default transmit list.

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 | DC source instance | compare to configured DC instance | RV-C-defined |
| 1 | Device priority | enum (100 = inverter/charger) | RV-C-defined |
| 2–3 | DC voltage | `raw / 20 V` | RV-C-defined; observed |
| 4–7 | DC current | `(raw - 0x77359400) * 0.001 A` | RV-C-defined; observed |

The current field is **uint32 with 0.001 A resolution and 0 A at `0x77359400`**
(RV-C Table 5.3) — a different width, scale and offset from the uint16
`0.05 A` fields used elsewhere in this document. Reusing the uint16 helper here
produces nonsense. The "not available" code is `0xFFFFFFFF`; an unpopulated
field reads as roughly ±2,000,000 A rather than zero, which makes a decode
error obvious.

Two observed quirks:

- **Sign is inverted** relative to a battery shunt. Measured against a
  SmartShunt during a charge ramp, the shunt peaked at `+9.019 A` while this
  field read `-8.960 A` — 0.7% apart on two independent instruments. Negate to
  get the conventional "+ = charging".
- **Effective resolution is 0.128 A**, not the 0.001 A the encoding allows. All
  observed values are multiples of 128 counts; the low seven bits are always
  zero.

When the charger is idle and the unit is not inverting this field reads exactly
`0.000 A` — a valid encoded zero, not an unpopulated field. Sampling it only in
that state can easily create the false impression that it is hardcoded.

### `0x1FFFC` – DC Source Status 2

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 | DC source instance | compare to configured DC instance | RV-C-defined |
| 1 | Device priority | enum | RV-C-defined |
| 2–3 | Source temperature | `raw / 32 - 273 °C` | RV-C-defined; observed |
| 4 | State of charge | `raw / 2 %` | RV-C-defined; **placeholder on this unit** |
| 5–6 | Time remaining | minutes | RV-C-defined; reported unavailable |

**State of charge is not real on the observed unit.** It emits a constant
`200` (= 100%) regardless of actual battery state. That is a *valid* encoding,
not the `255` "data not available" code, so it cannot be filtered out — it
reads as a confident, wrong 100%. Three independent findings explain why the
unit cannot produce a real figure:

1. No battery capacity is configured anywhere. The USB configuration file has
   no capacity field, and `0x1FFC6` reports battery bank size as `0xFFFF`
   ("data not available").
2. `0x1FFFB` (DC Source Status 3), which carries state of health, capacity
   remaining and relative capacity, is **not implemented** — four separate
   requests drew no response at all.
3. The unit measures only its own DC current. Coulomb counting a bank requires
   seeing all current in and out.

The likely intent is that a BMS on the RV-C bus supplies state of charge and
the inverter republishes it. With no BMS present there is no source, and the
unit emits a placeholder rather than marking the field unavailable.

Time remaining is reported as `0xFFFF` (unavailable), which is at least honest.

### `0x1FFC6` – Charger Configuration Status

**Not broadcast.** This DGN only answers a DGN request (see below). U3 2.15
reports the user-configured maximum charge current here, separate from the
desired current in `0x1FFC7`.

| Byte(s) | Field | Decode | Confidence |
|---|---|---|---|
| 0 | Instance | low nibble | Observed |
| 1 | Charging algorithm | `2` = 3-stage | Observed |
| 2 | Charger mode | `0` = stand-alone | Observed |
| 3 high nibble | Battery type | `3` = LiFePO4 | Observed |
| 4–5 | Battery bank size | Ah; reads `0xFFFF` on this unit | Observed |
| 6–7 | Maximum charging current | `(raw - 32000) / 20 A` | Observed |

An observed frame, cross-checked against the unit's own configuration:

```text
01 02 00 30 FF FF D0 84   -> 3-stage, LiFePO4, bank n/a, max 100.00 A
01 02 00 30 FF FF E8 80   -> same, max  50.00 A
01 02 00 30 FF FF 84 80   -> same, max  45.00 A
```

Note the asymmetry with the write side: maximum charging current is **uint16 at
bytes 6–7** in this status message but **uint8 at byte 7** in
`CHARGER_CONFIGURATION_COMMAND` (`0x1FFC4`), and battery type moves from byte 3
to byte 6. The two layouts are not interchangeable.

### Requesting a non-broadcast DGN

`0x1FFC6` and other request-only DGNs are retrieved with a standard DGN request
(`0xEA00`, RV-C 3.3.2a): three data bytes carrying the wanted DGN LSB-first,
addressed to the target node.

```text
CAN ID  0x18EA<dest><src>      e.g. 0x18EA8FA0 = to 0x8F from 0xA0
Data    C6 FF 01               request DGN 1FFC6
Data    FB FF 01               request DGN 1FFFB  (no response from this unit)
```

Requests are read-only and change nothing on the unit.

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

### What the update resets

The bootloader update reinitialises the communication card's flash file
system. **The entire `CAN` section of the card's configuration returns to
defaults while the `InvChg` settings survive untouched.** Observed across a
1.06 → 2.17 update:

| Setting | Before | After |
|---|---|---|
| `NodeAddr` | 66 | 143 |
| `WakeMask` | `0x0000ff00` | `0x0FFFFFF00` |
| `ReceiveTimers` | `IsoAddrClaim, 1500` | `DiagMsg1, 11000` |
| `DCSrcInst` | absent | 1 |
| `TransmitTimers` | 12 entries | 14 — adds `ChgSts2`, `DCSrcSts1` |

Battery type, charge current, absorption/float voltages, LBCO thresholds and
transfer mode all came through unchanged. The event log also restarts, so
pre-update history is lost.

Practical consequence: an integration keyed to the old node address goes
silent after a firmware update even though the inverter is working normally.

### Partial networking

The card uses ISO 11898-6 selective wake-up via an Infineon TLE9255W
transceiver — U3 2.08 added support for Infineon and NXP partial-networking
parts, and 2.17 "corrected the setup of the TLE9255W ... for wakeup via CAN
frame". Observed configuration after a 2.17 update:

```text
Enable  true        Frame  extended
SleepId 0x01234200  SleepMask 0x01FFFF00
WakeId  0x00004200  WakeMask  0x0FFFFFF00
```

Decoded as J1939 identifiers, both patterns carry PS = `0x42` — destination
address 66, the *old* default node address, which did not follow `NodeAddr`
to 143. U3 2.15 widened `WakeMask` from `0x0000ff00` "for compatibility with
the XGW", making the wake pattern far more selective: only a frame matching
`0x00004200` across the whole identifier qualifies, which forces priority to
0 and leaves just the source byte free.

The card's event log records a `Sleep Command` roughly four seconds after the
inverter reports `DEV_MODE: Standby`.

**Whether this ever takes the node off the bus is unverified.** Four days of
continuous capture with the bus connected — about 34,000 samples per day —
showed no gap of 60 seconds or longer, with frame rate steady at 38–50 fps
including while the inverter was idle. Do not assume a dropout is the card
sleeping; see the troubleshooting note in the README.

## Commands: documented, not enabled

| DGN | Purpose | Result on hardware |
|---|---|---|
| `0x1FFC4` | Charger configuration (max charge current) | **Works.** Layout validated across five values; `0xFF` no-change fill proven non-destructive by full-byte readback; applied in ~2 s. Writes non-volatile setting #24 |
| `0x1FFC5` | Charger enable / disable | **Works.** State changes within ~2 s, confirmed by charger state and by AC and DC current |
| `0x1FFC5` | CC/CV control current | **Inert** in 3-stage mode; two payload variants tried (status `0xFF` and `0x01`) |
| `0xEE00` | ISO address claim | **Works.** Address claimed and held; the node also answers `0xEA00` requests for its own claim |
| `0xEA00` | DGN request | **Works.** `0x1FFC6` answers on request; `0x1FFFB` never responds |
| `0x1FFD3` | Inverter enable | **Untested here.** RV-C 6.20.9 defines byte 1 bits 0–1 (`00` off, `01` on); Xantrex marks the field supported and U3 2.11 reduced its reaction time, so it is expected to work |
| `0x1FFD3` | Pass-through enable | **Untested here, and expected NOT to work.** RV-C defines byte 1 bits 4–5, but Xantrex marks this field unsupported on the Freedom SW. Taking the documentation at face value, the unit should accept the frame and ignore the bit |
| `0x00004200` | Partial-networking wake frame | **Untested.** Matches the configured `WakeId`/`WakeMask`, and U3 2.17 fixed TLE9255W frame wake-up, so it should wake a sleeping card — but whether the card ever sleeps is itself unverified |
| `0x1FFD0` / `0x1FFCF` | Inverter configuration | **Not attempted.** RV-C-defined; same no-change fill discipline as `0x1FFC4` would apply |

The untested rows are untested because exercising them means taking a
production inverter out of service — switching the inverter off, or moving
the AC load onto the battery — rather than because they are expected to
fail. Where the RV-C standard and the Xantrex documentation agree on a
field's behaviour, this table reports that expectation and labels it
untested. Expectations are not results: treat them as a starting point for
your own validation, not as proven behaviour.

### `0x1FFC4` write: validated layout

A maximum-charge-current write was exercised on hardware and **confirmed**,
with a full-byte `0x1FFC6` readback after every attempt. The layout differs
from the status message:

| Byte | Field | Encoding |
|---|---|---|
| 0 | Instance | uint8 |
| 1 | Charging algorithm | uint8, `0xFF` = no change |
| 2 | Charger mode | uint8, `0xFF` = no change |
| 3 | Battery sensor / installation line | `0xFF` = no change |
| 4–5 | Battery bank size | uint16, `0xFFFF` = no change |
| 6 | Battery type | uint4, `0xFF` = no change |
| 7 | Maximum charging current | **uint8, 1 A/bit** |

Filling every unrelated field with the RV-C "data not available" value
leaves them untouched. Verified across writes of 100, 50, 10, 30 and 45 A:
only bytes 6–7 of the `0x1FFC6` readback ever changed, with charging
algorithm (`02`, 3-stage) and battery type (`3`, LiFePO4) stable throughout.
The unit applied each change within about two seconds, and `0x1FFC7`
desired-current followed. Writes were accepted while actively charging,
which is looser than U3 2.16's note about accepting changes only when the
BMS is not controlling or the unit is not charging.

This writes **non-volatile setting #24** and survives a power cycle. Setting
it low and forgetting is a real hazard: 10 A into a 300 Ah bank is a very
long charge.

### `0x1FFC5` volatile control current: inert outside CC/CV

U3 2.17's volatile Control Current is specifically for CC/CV mode and does
not modify setting #24. Two candidate payloads were tested on a unit in
**3-stage** mode, with the current as uint16 `0.05 A/bit` at bytes 3–4
(mirroring `0x1FFC7`, where the status counterpart carries charge current):

| Variant | Byte 1 (status) | Result |
|---|---|---|
| 1 | `0xFF` (no change) | no effect |
| 2 | `0x01` (enable) | no effect |

Both frames were accepted without complaint and nothing changed, while the
same unit responded to `0x1FFC4` in about two seconds. The second variant
rules out "the unit needs a valid status byte before parsing the rest".

The remaining explanation is the release note's own scoping: the parameter
applies to **CC/CV mode**, which is BMS-directed. A unit running its own
3-stage algorithm has nothing for a control current to attach to. Reaching
CC/CV requires `DC_SRC_STS4`, whose byte layout is not in any RV-C revision
or library consulted here. **The byte 3–4 offset was never confirmed and is
not the reason these tests failed** — do not treat it as disproven either.

A responsible transmitter must be opt-in and should:

1. claim a non-conflicting RV-C address;
2. validate the identified Xantrex source and configured instance;
3. distinguish volatile control current from persistent maximum-charge configuration;
4. rate-limit and range-check every command;
5. fill unrelated fields with the correct reserved/not-available values; and
6. confirm the result through status/configuration readback, failing closed on mismatch.

## Remaining validation work

- Re-measure the ~1.30 discharge current ratio at a heavier inverter load; it
  rests on a single ~7.5 A point and may not be a constant if the cause is a
  waveform-averaging effect.
- Confirm `0x1FFD3` pass-through enable (byte 1 bits 4–5) does anything. The
  Xantrex Freedom SW DGN guide marks that field unsupported while leaving
  inverter enable and load sense unmarked; untested on the XC Pro.
- Establish whether partial networking ever actually takes the card off the
  bus, and if so whether a `0x00004200` wake frame revives it. Four days of
  continuous capture showed no sleep-related gap.
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
