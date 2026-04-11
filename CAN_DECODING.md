# Xantrex XC 2000 Pro – CAN Bus Decoding Reference

This document describes the J1939 CAN frames broadcast by the Xantrex XC 2000 Pro inverter/charger, as captured via CAN sniffing. It is intended for developers integrating this device into their own firmware, logging tools, or monitoring platforms.

All decodes have been verified against raw capture data across four operating states: inverting (battery only), AC input charging, AC input + charging under load, and utility power with higher-accuracy capture.

---

## Bus parameters

| Parameter | Value |
|---|---|
| Bit rate | 250 kbps |
| Frame format | Extended (29-bit arbitration ID) |
| Protocol layer | J1939 / SAE J1939 |
| Byte order | Little-endian (Intel byte order) within data fields |
| Source address (SA) | 0x42 (observed on all frames) |

---

## J1939 arbitration ID structure

All frames use 29-bit extended IDs. The J1939 encoding is:

```
Bits 28–26  Priority     (3 bits)
Bit  25     Reserved     (1 bit, always 0)
Bit  24     Data Page    (1 bit, usually 0)
Bits 23–16  PF – PDU Format (8 bits)
Bits 15–8   PS – PDU Specific (8 bits)
Bits 7–0    SA – Source Address (8 bits)
```

**PGN extraction:**

- If `PF >= 0xF0` (PDU2 — broadcast): `PGN = (PF << 8) | PS`
- If `PF < 0xF0` (PDU1 — addressed): `PGN = PF << 8` (PS is destination address, not part of PGN)

All Xantrex PGNs observed are PDU2 (`PF >= 0xF0`), so the full PGN is always encoded in the upper two bytes of the arbitration ID, and the source address (SA, low byte) varies and should be masked when filtering.

**Example — FEE8 frame on the wire:**

```
Arbitration ID:  0x18FEE842
Priority:        6  (bits 28–26 = 0b110)
PF:              0xFE
PS:              0xE8
SA:              0x42
PGN:             0xFEE8
```

### Filter mask for a single PGN

To receive all frames for PGN `0xFEE8` regardless of source address:

```
ID:   0x18FEE800
Mask: 0x1FFFFF00    ← zeros out the SA byte
```

---

## Data field conventions

All multi-byte integer fields are **little-endian** (low byte first).

### Voltage scaling

```
Voltage [V] = raw_uint16 / 20.0
```

Valid range: 9–64 V DC, 0–300 V AC.
`0xFFFF` = not available — discard.

### Current scaling

```
Current [A] = (raw_uint16 - 32000) / 20.0
```

Offset 32000 centres the unsigned 16-bit field around zero:
- `32000` → 0.0 A
- `> 32000` → positive (discharge on DC, draw on AC input)
- `< 32000` → negative (charge on DC)

`0xFFFF` = not available — discard.

### Frequency scaling

```
Frequency [Hz] = raw_uint16 / 128.0
```

Present in bytes 5–6 of FFCA and FFD7. Reads 0 when no AC is present.
Verified: `0x1E00` = 7680 → 60.00 Hz; `0x1DF3` = 7667 → 59.9 Hz (sub-Hz precision).

### Power (derived)

Power is not transmitted directly. It is calculated as:

```
Watts = Volts × Amps
```

---

## PGN reference

### FEE8 – DC Inverter Input

Describes the DC bus as seen from the inverter input. Broadcast continuously in both inverting and charging modes.

| Byte(s) | Field | Scaling | Notes |
|---|---|---|---|
| 0 | Status/mode | — | Not fully decoded; observed values: 0x01 |
| 1–2 | DC Voltage | `raw / 20.0` V | Battery / DC bus voltage |
| 3–4 | DC Current | `(raw - 32000) / 20.0` A | Positive = discharging (inverting); reads ~0 during charging |
| 5–7 | Not decoded | — | Observed 0xFF 0xFF 0xFF |

**Verified decode examples:**

| State | Raw bytes | Voltage | Current |
|---|---|---|---|
| Inverting (~13.25V, ~7A) | `01 09 01 8C 7D FF FF FF` | 265/20 = **13.25 V** | (32140−32000)/20 = **7.0 A** |
| Charging (13.75V) | `01 13 01 00 7D FF FF FF` | 275/20 = **13.75 V** | (32000−32000)/20 = **0.0 A** |
| Charging (14.1V, bulk) | `01 1A 01 00 7D FF FF FF` | 282/20 = **14.1 V** | (32000−32000)/20 = **0.0 A** |

**Notes:**
- Current drops to 0 A during charging — FFC7 is the authoritative source for charge current.
- A calibration gain and offset can be applied to the current value to correct for unit-specific measurement error.

---

### FFC7 – DC Charger Output

Describes current being delivered to the battery when shore power or generator is connected. Only broadcast when the charger is active.

| Byte(s) | Field | Scaling | Notes |
|---|---|---|---|
| 0 | Status | — | Observed 0x01 |
| 1–2 | DC Voltage | `raw / 20.0` V | Same DC bus voltage as FEE8 d[1–2] |
| 3–4 | Charge Current | `(raw - 32000) / 20.0` A | Published as absolute value; positive = charging |
| 5 | Charge Current (byte copy) | `raw / 2.0` A | Redundant; always equals d[3–4] decode at 0.5 A resolution |
| 6–7 | Not decoded | — | Observed 0x02 0xFF |

**Verified decode examples:**

| State | Raw bytes | DC Volts | Charge Amps |
|---|---|---|---|
| Charging | `01 13 01 30 84 B8 02 FF` | 13.75 V | (33840−32000)/20 = **92.0 A** |
| Higher load | `01 13 01 E4 84 CA 02 FF` | 13.75 V | (34020−32000)/20 = **101.0 A** |
| Bulk charge | `01 1A 01 94 84 C2 02 FF` | 14.1 V | (33940−32000)/20 = **97.0 A** |

**Notes:**
- Charge currents of 92–101 A at 13.75–14.1 V are plausible for the XC 2000 Pro (2000 W rated; max theoretical ~143 A at 14 V).
- d[5] byte copy: for capture with 101 A, d[5]=0xCA=202, 202/2=101.0 A — consistently matches d[3–4].
- DC voltage in d[1–2] is redundant with FEE8 but can serve as a direct source for charger watt calculation.

---

### FFCA – AC Input

Describes the shore power / generator input to the inverter/charger.

| Byte(s) | Field | Scaling | Notes |
|---|---|---|---|
| 0 | Status | — | Observed 0x01 |
| 1–2 | AC Input Voltage | `raw / 20.0` V | RMS; reads 0 when no AC source connected |
| 3–4 | AC Input Current | `(raw - 32000) / 20.0` A | RMS draw from AC source |
| 5–6 | AC Frequency | `raw / 128.0` Hz | 0 when no AC present; reads 0 when inverting |
| 7 | Not decoded | — | Observed 0xFF |

**Verified decode examples:**

| State | Raw bytes | AC Volts | AC Amps | Frequency |
|---|---|---|---|---|
| Inverting (no AC) | `01 00 00 00 7D 00 00 FF` | 0.0 V | 0.0 A | 0.0 Hz |
| Charging | `01 88 08 22 7E 00 1E FF` | 109.2 V | 14.5 A | **60.0 Hz** |
| Charging (higher load) | `01 A0 08 22 7E 00 1E FF` | 110.4 V | 14.5 A | **60.0 Hz** |
| Utility (high accuracy) | `01 C6 08 28 7E F3 1D FF` | 112.3 V | 14.8 A | **59.9 Hz** |

**Notes:**
- AC input voltage in these captures ranged 109–112 V. This is within normal utility variation under load.
- Frequency resolution is excellent: `0x1DF3` = 7667 → 7667/128 = **59.9 Hz** — sub-Hz precision confirmed.
- Do not rely on frequency byte alone to detect AC presence; it can read 0xFF (not available) in some firmware states.

---

### FFD7 – AC Output

Describes the AC output delivered to loads when the unit is inverting.

| Byte(s) | Field | Scaling | Notes |
|---|---|---|---|
| 0 | Status | — | Observed 0x41 |
| 1–2 | AC Output Voltage | `raw / 20.0` V | RMS |
| 3–4 | AC Output Current | `(raw - 32000) / 20.0` A | RMS; load current |
| 5–6 | AC Frequency | `raw / 128.0` Hz | Output frequency |
| 7 | Not decoded | — | Observed 0xFF |

**Verified decode examples:**

| State | Raw bytes | AC Volts | AC Amps | Frequency |
|---|---|---|---|---|
| Inverting | `41 5E 09 16 7D 00 1E FF` | 119.9 V | 1.1 A | **60.0 Hz** |
| Charging (pass-through) | `41 88 08 14 7D 00 1E FF` | 109.2 V | 1.0 A | **60.0 Hz** |
| AC + load | `41 A0 08 15 7D 00 1E FF` | 110.4 V | 1.05 A | **60.0 Hz** |
| Utility (high accuracy) | `41 C6 08 17 7D F3 1D FF` | 112.3 V | 1.15 A | **59.9 Hz** |

**Notes:**
- Bytes 5–6 frequency decode matches FFCA exactly in all captures — the unit synchronises output frequency to the AC source when charging.
- Output voltage tracks input voltage in pass-through/charging mode.

---

### FFD5 – Unknown control/state fields (decode in progress)

FFD5 is broadcast by the unit but **uses a completely different byte layout from FFD7**. Applying the FFD7 voltage/current formula produces 1212 V / −1595 A — verified impossible against raw captures.

**Observed structure across all captures:**

| Byte | Value | Status |
|---|---|---|
| 0 | 0x41 | Fixed — same status byte as FFD7 d[0] |
| 1 | 0xC0 | **Always fixed** — status/selector, not a measurement |
| 2 | varies 0x58–0x63 | Field A — independently varying |
| 3 | 0x00 | **Always fixed** — padding |
| 4 | varies 0x46–0x63 | Field B — independently varying |
| 5 | 0x00 | **Always fixed** — padding |
| 6–7 | 0xFF 0xFF | Not available |

The two meaningful fields are **d[2] (Field A)** and **d[4] (Field B)** — single bytes.

**Static capture values (four operating snapshots):**

| Capture | State | d[2] | d[4] |
|---|---|---|---|
| 1 | Inverting | 94 | 99 |
| 2 | Charging begins | 88 | 70 |
| 3 | Charging harder | 94 | 72 |
| 4 | Utility charging | 99 | 87 |

**Live data observation — temperature hypothesis ruled out:**

When logged live during pass-through mode (AC input present, charger active), **Field B (d[4]) oscillates in large, regular square-wave transitions with a ~20–30 second period**. Field A (d[2]) is more stable but also changes faster than any thermal sensor would. Temperature sensors have thermal time constants of minutes — these fields do not.

**Revised hypothesis: charger control or state fields**

The square-wave behavior of Field B is consistent with:
- **Charger on/off cycling** — many inverter/chargers pulse charge current to take unloaded battery voltage measurements; Field B toggles between "charging" and "measuring" states
- **Duty cycle percentage** — PWM duty cycle of the charger's switching stage
- **Internal charge state counter** — an index into a charge profile state machine

Field A's slower, smaller variations across static captures may represent a setpoint or averaged value that shifts with operating mode (inverting vs. charging vs. utility).

**Status:** Decode unknown. Both fields are logged as raw integer values in firmware. Correlation with FFC7 charge current and charger on/off events during a controlled test session would help identify Field B. Correlation of Field A with AC load or DC voltage across a wider operating range would help identify it.

---

### FFD4 – Operating Mode Indicator

Only broadcast when AC input is present. Absent in inverting-only mode.

| Byte(s) | Field | Notes |
|---|---|---|
| 0 | Status | Observed 0x01 |
| 1 | Mode | 0x02 = AC present / charging mode |
| 2 | Unknown | Observed 0xF0 |
| 3–7 | Not available | 0xFF |

**Captured:**
```
Inverting only:          (FFD4 not broadcast)
Charging / AC present:   01 02 F0 FF FF FF FF FF
```

This PGN can be used to detect AC input presence independently of FFCA voltage.

---

### FFFC – Battery State of Charge (probable)

Broadcast in all operating modes. d[4] correlates strongly with battery SOC.

| Byte(s) | Field | Notes |
|---|---|---|
| 0 | Status | Observed 0x01 |
| 1 | Unknown | Constant 0x64 = 100 across all captures |
| 2–3 | Unknown | Constant 0x20 0x27 |
| 4 | Battery SOC (probable) | Integer percent; 0x64=100%, 0x5F=95% |
| 5–7 | Not available | 0xFF |

**Observed:**

| State | d[4] | Probable SOC |
|---|---|---|
| Inverting from full charge | 0x64 = 100 | 100% |
| Charging (all other captures) | 0x5F = 95 | 95% |

**Note:** The interpretation of d[4] as SOC is inferred from correlation with operating state. Not confirmed against Xantrex protocol documentation.

---

### FFC8 – Static Status Frame

Broadcast in all modes, content identical across all captures. Likely a device capability or configuration advertisement.

```
All captures:  01 FC FF FF FF FF 00 FF
```

d[1]=0xFC (binary 11111100) may represent capability flags. Not decoded.

---

### FEBD – Not decoded

Present in captures 1, 2, and 4. d[1–2] values (10080–10656 as little-endian uint16) do not correspond to any plausible voltage at known scaling factors. May represent an energy counter, thermal measurement, or a parameter requiring the Xantrex SPN specification to interpret.

---

### FF8F – Not decoded

Minimal content: d[0]=0x41, d[1]=0x00, all other bytes 0xFF. Present in captures 1, 2, and 4. Purpose unknown.

---

## Firmware decode (C++ / ESPHome lambda)

The core decode runs inside an ESPHome `on_frame` lambda. Key patterns:

```cpp
// PGN extraction from 29-bit arbitration ID
const uint8_t  pf  = (can_id >> 16) & 0xFF;
const uint8_t  ps  = (can_id >>  8) & 0xFF;
const uint32_t pgn = (pf < 0xF0)
                     ? ((uint32_t)pf << 8)
                     : (((uint32_t)pf << 8) | ps);

// Little-endian uint16 from two consecutive bytes
auto u16_le = [](uint8_t b0, uint8_t b1) -> uint16_t {
    return (uint16_t)b1 << 8 | b0;
};

// Voltage (DC or AC RMS)
float volts = u16_le(d[1], d[2]) / 20.0f;

// Current
float amps  = (u16_le(d[3], d[4]) - 32000) / 20.0f;

// Frequency (bytes 5–6, FFCA and FFD7 only)
float hz    = u16_le(d[5], d[6]) / 128.0f;
```

`0xFFFF` (not-available) check before scaling:

```cpp
auto is_na16 = [](uint16_t v){ return v == 0xFFFF; };
if (!is_na16(vraw)) { /* decode */ }
```

---

## Observed PGN summary

| PGN | Description | Decoded | Broadcast when |
|---|---|---|---|
| FEE8 | DC Inverter Input | Yes — V, A | Always |
| FFC7 | DC Charger Output | Yes — V, A | AC input present |
| FFCA | AC Input | Yes — V, A, Hz | Always (V=0 when no AC) |
| FFD7 | AC Output | Yes — V, A, Hz | Always |
| FFD5 | Unknown control/state | Partial — raw fields logged, decode TBD | Always |
| FFD4 | Operating Mode | Partial — mode byte | AC input present only |
| FFFC | Battery SOC (probable) | Partial — d[4] = SOC% | Always |
| FEBD | Unknown | No | Always (inverting + utility) |
| FFC8 | Static status | No | Always |
| FF8F | Unknown | No | Always (inverting + utility) |

---

## Sniffing setup used

- Hardware: ESP32-S3 + SN65HVD230 transceiver
- Software: ESPHome with `on_frame` catch-all handler logging raw arbitration IDs and payloads
- Method: Monitored under several operating conditions — idle (no AC), inverting (DC load), charging (shore power connected), and charge + invert simultaneously
- All frames were 8 bytes (DLC = 8), extended IDs only, at 250 kbps
- Source address 0x42 observed on all frames from the Xantrex unit
