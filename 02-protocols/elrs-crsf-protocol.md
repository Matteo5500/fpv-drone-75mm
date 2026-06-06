# ExpressLRS & CRSF Protocol — Radio Control Link

## Why ExpressLRS

Previous RC radio systems (FrSky, Spektrum) were proprietary,
high-latency, and limited in range. ExpressLRS (ELRS) was created
in 2020 as an open-source alternative.

| System | Latency | Range | Open Source |
|--------|---------|-------|-------------|
| FrSky D8 | ~22ms | ~1-2km | ✗ |
| FrSky D16 | ~6ms | ~2-3km | ✗ |
| ExpressLRS 2.4GHz | ~1ms | ~5-10km | ✅ |
| ExpressLRS 900MHz | ~4ms | ~30km+ | ✅ |

**Selected: ELRS 2.4GHz** — integrated in FC AIO.
Optimal for indoor Tiny Whoop where latency matters more than range.

---

## System Architecture

```
[TX Module / Radio Controller]
        │
        │  2.4GHz LoRa radio signal
        │  CSS (Chirp Spread Spectrum) modulation
        ▼
[RX Module — integrated in FC AIO]
        │
        │  CRSF protocol
        │  UART1 @ 420000 baud
        ▼
[STM32F411 — Betaflight]
```

### LoRa Modulation
ELRS uses **CSS (Chirp Spread Spectrum)**: the signal continuously
sweeps across frequencies in a predictable pattern.
- Interference resistant (a disturbance hits only a fraction of the signal)
- Obstacle penetrating
- Energy efficient (long range at low transmission power)

---

## Packet Rate — The Latency Parameter

Packet rate = how many command packets are sent per second.

| Packet Rate | Interval | Latency | Range | Use Case |
|-------------|----------|---------|-------|----------|
| 50Hz | 20ms | ~20ms | Maximum | Long range FPV |
| 150Hz | 6.7ms | ~6.7ms | High | Sport flying |
| 250Hz | 4ms | ~4ms | Medium | Racing |
| 500Hz | 2ms | ~2ms | Reduced | **Tiny Whoop** |
| 1000Hz | 1ms | ~1ms | Minimum | Indoor racing |

**Trade-off:** higher packet rate = lower latency + reduced range.
For indoor Tiny Whoop: **500Hz or 1000Hz recommended**.

---

## CRSF Protocol — Frame Structure

CRSF (Crossfire Serial Protocol) is the application-layer protocol
carried over UART between the RX module and the Flight Controller.

```
┌──────┬──────────┬──────────┬─────────────────┬─────┐
│ SYNC │  LENGTH  │   TYPE   │    PAYLOAD      │ CRC │
│ 0xC8 │   1 B    │   1 B    │   variable      │ 1 B │
└──────┴──────────┴──────────┴─────────────────┴─────┘
```

| Field | Size | Description |
|-------|------|-------------|
| SYNC | 1 byte | Always 0xC8 — marks frame start |
| LENGTH | 1 byte | Payload + TYPE + CRC length |
| TYPE | 1 byte | Packet type (RC channels, telemetry...) |
| PAYLOAD | variable | Actual data |
| CRC | 1 byte | CRC8 integrity check |

### RC Channels Payload
16 channels × 11 bits each = 176 bits of channel data.
Channel range: 172–1811 (center: 992).
Mapped internally by Betaflight to 1000–2000µs range.

| Channel | Function |
|---------|----------|
| CH1 | Aileron (Roll) |
| CH2 | Elevator (Pitch) |
| CH3 | Throttle |
| CH4 | Rudder (Yaw) |
| CH5 | Arm switch |
| CH6+ | Flight modes, Beeper, Turtle mode... |

### UART Configuration
CRSF requires exactly **420000 baud** on a dedicated UART.
This non-standard baud rate must be explicitly set in Betaflight CLI:
```
serial 0 64 115200 57600 0 420000
```

---

## Bidirectional Telemetry

CRSF supports a reverse telemetry channel:

```
TX Controller  ──── RC commands ────→  Drone RX
TX Controller  ←─── telemetry data ──  Drone RX
```

Telemetry data available:
- Battery voltage (real-time on TX display)
- Link Quality % (LQ) — packet reception rate
- RSSI — received signal strength
- GPS speed and altitude (if GPS present)

**Most useful for Tiny Whoop:** LQ% and battery voltage —
know when to land before damaging the LiPo.

---

## Betaflight Configuration

In Betaflight, ELRS/CRSF is configured under:
- **Ports tab:** UART1 → Serial RX enabled
- **Configuration tab:** Receiver Mode → Serial, Provider → CRSF
- **CLI:** `set serialrx_provider = CRSF`

---

## ELRS vs Legacy Systems

| Property | PWM/PPM | SBUS | CRSF/ELRS |
|----------|---------|------|-----------|
| Channels | 1-8 | 16 | 16 |
| Resolution | 10-bit | 11-bit | 11-bit |
| Latency | ~20ms | ~9ms | ~1ms |
| Connection | Analog | UART 100000 baud | UART 420000 baud |
| Telemetry | None | None | Bidirectional |
| Error detection | None | Framing only | CRC8 |