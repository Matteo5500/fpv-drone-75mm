# DSHOT Protocol — Digital Motor Control

## Why DSHOT Exists

Before DSHOT, flight controllers used **PWM (Pulse Width Modulation)**
to command ESCs — an analog protocol from the 1970s RC hobby world.

### PWM Problems
- Required manual calibration per ESC (analog timing drift)
- Susceptible to electromagnetic interference from brushless motors
- Unidirectional — FC had no feedback from ESC

### DSHOT Solution
Fully digital protocol. Each command is a 16-bit packet with
checksum verification. Immune to calibration drift and EMI.

---

## Packet Structure

```
Bit:  15  14  13  12  11  10  9   8   7   6   5   4   3   2   1   0
      [          THROTTLE (11 bits)          ] [T] [    CRC (4 bits)   ]
```

| Field | Bits | Range | Description |
|-------|------|-------|-------------|
| Throttle | 15–5 | 0–2047 | Motor power command |
| Telemetry | 4 | 0–1 | Request ESC telemetry response |
| CRC | 3–0 | 0–15 | Packet integrity check |

**CRC calculation:** `(packet ^ (packet >> 4) ^ (packet >> 8)) & 0x0F`

---

## Bit Encoding

DSHOT encodes bits using pulse duration (not voltage level):

```
Bit "1": HIGH for ~75% of bit period  ████████░░░
Bit "0": HIGH for ~37% of bit period  ████░░░░░░░
```

This makes DSHOT resistant to electromagnetic interference —
a critical property in a drone where 4 brushless motors
generate significant EMI.

---

## Protocol Variants

| Variant | Speed | Bit Period | Use Case |
|---------|-------|------------|----------|
| DSHOT150 | 150 kbit/s | 6.67µs | Legacy, slow ESCs |
| DSHOT300 | 300 kbit/s | 3.33µs | **Our target** — 1S whoops |
| DSHOT600 | 600 kbit/s | 1.67µs | High-performance builds |
| DSHOT1200 | 1200 kbit/s | 0.83µs | Experimental |

**Selected: DSHOT300** — optimal balance of speed and reliability on 1S.

---

## Bidirectional DSHOT (Bidir)

Standard DSHOT is unidirectional (FC → ESC only).
Bidirectional DSHOT adds a reverse telemetry channel:

```
FC ──── DSHOT300 ──────→ ESC   (throttle command)
FC ←─── eRPM telemetry ── ESC   (electrical RPM response)
```

**eRPM** (electrical RPM) = mechanical RPM × (motor poles / 2)  
For 0802SE with 12 poles: mechanical RPM = eRPM / 6

### Why Bidir DSHOT matters
Enables **RPM Filtering** in Betaflight — the flight controller
can calculate exact motor noise frequencies and filter them
precisely, instead of using broad spectrum filters that
also remove useful flight data.

---

## Throttle Value Map

| Value | Meaning |
|-------|---------|
| 0 | Motor disarmed |
| 1–47 | Reserved (special commands) |
| 48 | Minimum throttle (motor spinning) |
| 2047 | Maximum throttle |

Special commands (values 1–47) include: spin direction,
3D mode, ESC telemetry enable, LED control.

---

## Betaflight Source Reference

**File:** `src/main/drivers/dshot.c`  
**Key function:** `prepareDshotPacket()`

```c
uint16_t prepareDshotPacket(dshotProtocolControl_t *pcb)
{
    uint16_t packet = (pcb->value << 1) | (pcb->requestTelemetry ? 1 : 0);
    unsigned crc = (packet ^ (packet >> 4) ^ (packet >> 8)) & 0x0F;
    packet = (packet << 4) | crc;
    return packet;
}
```

The CRC uses XOR of three 4-bit nibbles — computationally
cheap, critical on an embedded MCU where every CPU cycle counts.

---

## Comparison: PWM vs DSHOT

| Property | PWM (analog) | DSHOT (digital) |
|----------|-------------|-----------------|
| Calibration | Required per ESC | Not required |
| EMI resistance | Low | High |
| Precision | ~1000 steps | 2048 steps |
| Direction | Unidirectional | Bidirectional (Bidir) |
| Feedback | None | eRPM telemetry |
| Error detection | None | CRC checksum |