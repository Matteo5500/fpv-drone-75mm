# Component Selection — 75mm 1S Tiny Whoop

## Selection Methodology

Components are selected using constraint propagation:
frame size → motor size → battery voltage → FC compatibility → firmware target → protocols.

No component is chosen independently — each decision constrains the next.

---

## Bill of Materials (BOM)

### Flight Controller
- **Model:** BetaFPV Meteor75 Pro FC (AIO)
- **MCU:** STM32F411CEU6 @ 100MHz
- **IMU:** ICM-42688-P via SPI
- **ESC:** 4x 5A integrated
- **RX:** ExpressLRS 2.4GHz integrated
- **VTX:** 25-200mW 5.8GHz integrated
- **Betaflight Target:** `BETAFPVF411`
- **Connector:** BT2.0
- **Datasheet:** [BetaFPV Meteor75 Pro](https://betafpv.com/products/meteor75-pro-whoop-quadcopter)

### Motors (x4)
- **Model:** BetaFPV 0802SE 22000KV
- **Stator:** 08mm × 02mm
- **KV:** 22000 (RPM per Volt)
- **Max current:** 4.8A
- **Operating voltage:** 1S (3.7V nominal)
- **Shaft:** 1.0mm
- **Calculated max RPM:** 22000 × 4.35V = ~95,700 RPM

### Propellers (x4)
- **Model:** Gemfan 40mm 4-blade
- **Shaft hole:** 1.0mm
- **Configuration:** 2x CW + 2x CCW (counter-rotating pairs)

### Battery
- **Model:** BetaFPV BT2.0 300mAh 1S 30C HV
- **Nominal voltage:** 3.7V | Max charge: 4.35V
- **Max discharge current:** 300mAh × 30C = 9A
- **Connector:** BT2.0

---

## Weight Budget

| Component      | Weight |
|----------------|--------|
| FC AIO         | 4.2g   |
| 4x Motors      | 6.4g   |
| 4x Props       | 1.2g   |
| Frame 75mm     | ~5.0g  |
| Battery        | 7.0g   |
| **Total**      | **~23.8g** |

Target range for 75mm 1S: 20–30g ✅

---

## Key Design Decisions

1. **AIO board chosen over separate FC+ESC:** reduces weight, wiring complexity,
   and failure points. Critical for a 75mm build.

2. **22000KV motors on 1S:** low voltage requires high KV to achieve sufficient RPM.
   Formula: RPM = KV × V → 22000 × 3.7 = ~81,400 RPM nominal.

3. **BT2.0 over PH2.0 connector:** lower internal resistance = less voltage sag
   under load = more consistent power delivery.

4. **Integrated ELRS RX:** eliminates external receiver, saves ~0.5g and one
   UART port. ELRS chosen for open-source stack consistency.