# Compatibility Matrix — 75mm 1S Tiny Whoop

This document verifies electrical, protocol, and physical compatibility
between all components in the BOM.

---

## Communication Map
                ┌─────────────────────────────┐
                │     BetaFPV Meteor75 Pro     │
                │          FC AIO              │
                │       STM32F411CEU6          │
                │                             │
      SPI       │  ┌──────────────────────┐  │
┌──────────────→ │  │  ICM-42688-P (IMU)   │  │
│                │  │  Gyro + Accelerometer │  │
│                │  └──────────────────────┘  │
│                │                             │
│     DSHOT300   │  ┌──────────────────────┐  │      DSHOT300
│   ┌──────────→ │  │   ESC (4x 5A)        │  │ ──────────────→ Motor 1 (CW)
│   │            │  │   integrated         │  │ ──────────────→ Motor 2 (CCW)
│   │            │  └──────────────────────┘  │ ──────────────→ Motor 3 (CW)
│   │            │                             │ ──────────────→ Motor 4 (CCW)
│   │  UART/CRSF │  ┌──────────────────────┐  │
│   │  ┌───────→ │  │  ELRS RX 2.4GHz      │  │ ············→ Radio TX (air)
│   │  │         │  │  integrated          │  │
│   │  │         │  └──────────────────────┘  │
│   │  │         │                             │
│   │  │  UART   │  ┌──────────────────────┐  │
│   │  │  ┌────→ │  │  VTX 5.8GHz          │  │ ──────────→ Analog video out
│   │  │  │      │  │  integrated          │  │
│   │  │  │      │  └──────────────────────┘  │
│   │  │  │      └─────────────────────────────┘
│   │  │  │                  │
│   │  │  │                  │ 1S Power (3.2–4.35V)
│   │  │  │                  ↓
│   │  │  │      ┌─────────────────────┐
│   │  │  │      │  BT2.0 300mAh 1S   │
│   │  │  │      │  LiPo Battery       │
│   │  │  │      └─────────────────────┘

---

## Electrical Compatibility

| Interface | Component A | Component B | Voltage A | Voltage B | ✓/✗ |
|-----------|-------------|-------------|-----------|-----------|-----|
| Power input | Battery 1S | FC AIO | 3.2–4.35V | 3.2–4.35V | ✅ |
| Motor power | ESC (on FC) | 0802SE motors | 1S | 1S rated | ✅ |
| IMU power | FC (3.3V rail) | ICM-42688-P | 3.3V | 1.71–3.6V | ✅ |
| RX power | FC (3.3V rail) | ELRS RX | 3.3V | 3.3V | ✅ |
| VTX power | FC (VBAT rail) | VTX 5.8GHz | VBAT | 3.2–4.35V | ✅ |

---

## Protocol Compatibility

| Connection | Protocol | Component A | Component B | Speed | ✓/✗ |
|------------|----------|-------------|-------------|-------|-----|
| FC → IMU | SPI | STM32F411 | ICM-42688-P | up to 24MHz | ✅ |
| FC → Motors | DSHOT300 | ESC (integrated) | 0802SE (any brushless) | 300 kbit/s | ✅ |
| FC ↔ RX | UART / CRSF | STM32F411 UART | ELRS RX | 420000 baud | ✅ |
| FC → VTX | UART / MSP | STM32F411 UART | VTX SA2.1 | 115200 baud | ✅ |

---

## Physical Compatibility

| Interface | Component A | Component B | Standard | ✓/✗ |
|-----------|-------------|-------------|----------|-----|
| Battery connector | FC AIO | LiPo 300mAh | BT2.0 | ✅ |
| Motor shaft | 0802SE motor | Gemfan 40mm prop | 1.0mm shaft | ✅ |
| Motor mount | FC AIO board | 0802SE | M2 screws, 9mm pattern | ✅ |
| Frame mount | FC AIO board | 75mm frame | M2 screws, 26.5×26.5mm | ✅ |

---

## UART Allocation Map

The STM32F411 on this AIO has 3 usable UARTs. Each peripheral that
communicates serially needs a dedicated UART port.

| UART | Assigned To | Protocol | Baud Rate |
|------|-------------|----------|-----------|
| UART1 | ExpressLRS RX | CRSF | 420000 |
| UART2 | VTX (SmartAudio) | MSP/SA | 115200 |
| UART3 | Available / Blackbox | — | — |

> ⚠️ UART ports are a finite resource on embedded systems.
> Each peripheral requires exclusive UART assignment.
> Conflicts in UART allocation are a common source of
> configuration bugs in Betaflight setups.

---

## Compatibility Summary

| Check | Result |
|-------|--------|
| All components operate on 1S voltage | ✅ |
| No protocol conflicts | ✅ |
| No connector mismatches | ✅ |
| UART ports sufficient for all peripherals | ✅ |
| Motor KV appropriate for 1S voltage | ✅ |
| Propeller shaft matches motor shaft | ✅ |
| FC mounting pattern matches frame | ✅ |

**All compatibility checks passed. Architecture is valid for Phase 2 assembly.**