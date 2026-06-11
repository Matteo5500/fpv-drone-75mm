# Gyroscope Filter Analysis — Betaflight Source Code

## Why Filters Are Necessary

Raw gyroscope data contains noise from multiple sources:
- Motor vibrations (100–400Hz mechanical frequencies)
- ESC switching noise (>400Hz electrical harmonics)
- Frame resonance (amplifies specific frequencies)

Without filtering, the D term in the PID controller amplifies
this noise, causing motors to oscillate at high frequency —
leading to excessive heat and ESC failure.

**Filter chain objective:** remove noise while preserving
the actual flight signal (0–80Hz typical) with minimal latency.

---

## Filter Chain — Processing Order

```
Raw Gyro Data
      │
      ▼
[Gyro LPF1]          ← first low-pass filter (gyro_lpf1_static_hz)
      │
      ▼
[Gyro LPF2]          ← second low-pass filter (gyro_lpf2_static_hz)
      │
      ▼
[Dynamic Notch]      ← auto-detected frequency notches
      │
      ▼
[RPM Filter]         ← motor-frequency notches (requires Bidir DSHOT)
      │
      ▼
Filtered Gyro → PID Controller
      │
      ▼
[D-term LPF1]        ← filters D term output
      │
      ▼
[D-term LPF2]        ← second D term filter
      │
      ▼
Final Motor Commands
```

---

## 1. Gyro Low-Pass Filter (LPF)

Passes frequencies **below** the cutoff, attenuates frequencies above.

**Trade-off:** lower cutoff = more noise removed, more phase delay.
Higher cutoff = less delay, more noise passes through.

```
set gyro_lpf1_type = PT1
set gyro_lpf1_static_hz = 250

set gyro_lpf2_type = PT1
set gyro_lpf2_static_hz = 500
```

### Filter Types
| Type | Order | Attenuation | Phase Delay | Use Case |
|------|-------|-------------|-------------|----------|
| PT1 | 1st | -20dB/decade | Low | Default, good balance |
| Biquad | 2nd | -40dB/decade | Medium | More filtering needed |
| PT2 | 2nd | -40dB/decade | Medium | Smoother than Biquad |
| PT3 | 3rd | -60dB/decade | High | Maximum filtering |

### Source Code — PT1 Filter
**File:** `src/main/common/filter.c`

```c
typedef struct pt1Filter_s {
    float state;  // current filtered value
    float k;      // coefficient: dT / (RC + dT)
} pt1Filter_t;

float pt1FilterApply(pt1Filter_t *filter, float input)
{
    // Weighted average: (1-k) × previous + k × new_input
    filter->state = filter->state + filter->k * (input - filter->state);
    return filter->state;
}
```

`k` determines the filter aggressiveness:
- k close to 0 → heavily weighted toward past → smooth, high delay
- k close to 1 → heavily weighted toward new input → responsive, low delay

---

## 2. RPM Filter

The most effective filter. Requires **Bidirectional DSHOT** active.

**Principle:** each motor generates vibrations at a predictable
frequency based on its RPM. With eRPM telemetry from Bidir DSHOT,
Betaflight calculates exact motor frequencies and places
precise notch filters on them.

```
eRPM from ESC → calculate motor frequency → place notch filter
                                           → update every loop (8kHz)
```

### Frequency Calculation
```c
// Motor frequency (fundamental)
float motorFrequency = (eRPM / motorPoles) / 60.0f;

// Harmonics (integer multiples)
// 1st harmonic: motorFrequency × 1
// 2nd harmonic: motorFrequency × 2
// 3rd harmonic: motorFrequency × 3
```

With 4 motors × 3 harmonics = **12 dynamic notch filters**
updated in real-time at 8kHz.

```
set rpm_filter_harmonics = 3
set rpm_filter_min_hz = 100
```

### Example — Motor 1 at 50,000 RPM (12-pole motor)
```
eRPM = 50,000 × 6 = 300,000 eRPM
Motor frequency = 300,000 / 6 / 60 = 833 Hz
Notch filters placed at: 833Hz, 1666Hz, 2499Hz
```

---

## 3. Dynamic Notch Filter

Automatically analyzes the gyro signal spectrum using a
simplified FFT algorithm, identifies dominant noise peaks,
and places notch filters on them — without RPM data.

```
set dyn_notch_count = 4      # number of dynamic notches
set dyn_notch_q = 250        # Q factor (higher = narrower notch)
set dyn_notch_min_hz = 100   # minimum frequency to analyze
set dyn_notch_max_hz = 600   # maximum frequency to analyze
```

**Q factor:** controls notch width.
- High Q (narrow notch): precise, less impact on nearby frequencies
- Low Q (wide notch): less precise, affects broader frequency range

---

## 4. D-Term Low-Pass Filter

The D term amplifies high-frequency content. A dedicated
LPF is applied after D term calculation:

```
set dterm_lpf1_type = PT1
set dterm_lpf1_static_hz = 100

set dterm_lpf2_type = PT1
set dterm_lpf2_static_hz = 200
```

Lower cutoff than gyro LPF because D term already amplified
the noise — more aggressive filtering is needed.

---

## Filter Tuning Philosophy

**Start conservative (more filtering, lower cutoff):**
- Protects motors and ESCs
- Drone feels slightly sluggish

**Reduce filtering progressively:**
- Raise cutoff frequencies
- Monitor motor temperature after each flight
- Stop when motors get warm (>60°C after 2min hover)

**With RPM Filter enabled:**
- Can reduce static LPF (raise cutoff) significantly
- RPM Filter handles motor-frequency noise precisely
- Static LPF only needs to handle remaining broadband noise

---

## Betaflight Filter Pipeline — Source Reference

**File:** `src/main/sensors/gyro.c`

```c
void gyroUpdate(void)
{
    // 1. Read raw SPI data
    gyro.gyroADCRaw[axis] = readRawGyroSPI();

    // 2. Apply Gyro LPF1
    float filtered = pt1FilterApply(&gyroLpf1[axis],
                                    gyro.gyroADCRaw[axis]);
    // 3. Apply Gyro LPF2
    filtered = pt1FilterApply(&gyroLpf2[axis], filtered);

    // 4. Apply Dynamic Notch
    filtered = dynamicNotchFilterApply(&dynNotch[axis], filtered);

    // 5. Apply RPM Filter (if Bidir DSHOT active)
    if (isRpmFilterEnabled()) {
        filtered = rpmFilterApply(axis, filtered);
    }

    // 6. Pass filtered signal to PID
    gyro.gyroADCf[axis] = filtered;
}
```