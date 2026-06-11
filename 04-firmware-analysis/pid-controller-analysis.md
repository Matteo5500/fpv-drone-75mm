# PID Controller Analysis — Betaflight Source Code

## What is a PID Controller

A PID (Proportional-Integral-Derivative) controller is a feedback
control algorithm. A drone is inherently unstable — without active
control it falls. The PID controller runs 8000 times per second,
continuously measuring the error between the desired state and the
actual state, and correcting the motors accordingly.

```
Desired state (setpoint) ──→ [ ERROR ] ──→ [ PID ] ──→ Motors
        ↑                        ↑
        │                        │
        └──────── Gyroscope measurement (feedback) ────────┘
```

---

## The Three Terms

### P — Proportional
```
P_output = Kp × error
error = setpoint - gyro_measurement
```
Reacts to the current error. Larger error = larger correction.
Problem: tends to oscillate around the setpoint alone.

### I — Integral
```
I_output = Ki × Σ(error × dt)
```
Accumulates error over time. Eliminates persistent steady-state errors
(e.g., slightly asymmetric weight, one weaker motor).
Problem: integrator windup if the drone is physically blocked.
Solution: Betaflight clamps I with `itermLimit` (anti-windup).

### D — Derivative
```
D_output = Kd × Δerror / dt
```
Predicts future behavior based on the rate of change of error.
Acts as a damper — brakes rapid changes before they become large.
Problem: amplifies high-frequency noise from motor vibrations.
Solution: D term is always filtered before use.

---

## Source Code — pid.c

**File:** `src/main/flight/pid.c`

### Data Structure
```c
typedef struct pidAxisData_s {
    float P;      // proportional contribution
    float I;      // integral accumulator
    float D;      // derivative contribution
    float F;      // feedforward contribution
    float Sum;    // P + I + D + F = final output
} pidAxisData_t;

pidAxisData_t pidData[XYZ_AXIS_COUNT]; // [ROLL, PITCH, YAW]
```

### Main PID Loop (called at 8kHz)
```c
void pidController(const pidProfile_t *pidProfile, timeUs_t currentTimeUs)
{
    const float dT = pidProfile->pid_process_denom
                     / (float)gyro.targetLooptime;

    for (int axis = FD_ROLL; axis <= FD_YAW; axis++) {

        // Error calculation
        float currentPidSetpoint = getSetpointRate(axis);
        float gyroRate = gyro.gyroADCf[axis];
        float errorRate = currentPidSetpoint - gyroRate;

        // P term
        pidData[axis].P = pidCoefficient[axis].Kp * errorRate;

        // I term (with anti-windup clamping)
        pidData[axis].I = constrainf(
            pidData[axis].I + pidCoefficient[axis].Ki * errorRate * dT,
            -itermLimit, itermLimit
        );

        // D term — "D on measurement" technique
        float delta = -(gyroRate - previousGyroRateFiltered[axis]) / dT;
        pidData[axis].D = pidCoefficient[axis].Kd * delta;

        // Final sum
        pidData[axis].Sum = pidData[axis].P
                          + pidData[axis].I
                          + pidData[axis].D;
    }
}
```

---

## Key Implementation Details

### "D on Measurement" vs "D on Error"
Standard PID derives the **error**. Betaflight derives the
**gyro measurement** instead:

```c
// Classic (derivative kick on setpoint changes):
float delta = (errorRate - previousError) / dT;

// Betaflight (no derivative kick):
float delta = -(gyroRate - previousGyroRate) / dT;
```

When the pilot moves the stick, the setpoint changes instantly.
Deriving the error would produce a massive D spike (derivative kick).
Deriving the measurement (which changes gradually due to drone
inertia) eliminates this spike completely.

### Anti-Windup
```c
pidData[axis].I = constrainf(accumulated_I, -itermLimit, itermLimit);
```
If the drone is physically blocked (e.g., held in hand while armed),
I would accumulate indefinitely. Clamping prevents the windup that
would cause violent overcorrection when the drone is released.

---

## Mixer Matrix — PID to Motor Commands

**File:** `src/main/flight/mixer.c`

The mixer converts 4 inputs (Throttle, Roll, Pitch, Yaw) into
4 motor commands:

```c
// X-frame quadcopter mixer
motor[0] = throttle + roll + pitch - yaw;  // M1 front-right (CW)
motor[1] = throttle - roll + pitch + yaw;  // M2 front-left  (CCW)
motor[2] = throttle + roll - pitch + yaw;  // M3 rear-right  (CCW)
motor[3] = throttle - roll - pitch - yaw;  // M4 rear-left   (CW)
```

Sign convention derives from motor position and rotation direction.
Each motor receives a weighted combination of all control axes.

---

## PID Tuning Parameters

In Betaflight CLI these are set per axis:
```
set p_roll  = 45    # Kp for Roll axis
set i_roll  = 80    # Ki for Roll axis
set d_roll  = 35    # Kd for Roll axis
set p_pitch = 47    # Kp for Pitch axis
set i_pitch = 84    # Ki for Pitch axis
set d_pitch = 38    # Kd for Pitch axis
set p_yaw   = 45    # Kp for Yaw axis
set i_yaw   = 80    # Ki for Yaw axis
```

These values are starting points for a 75mm 1S Tiny Whoop.
Physical tuning (Phase 2) will require adjustment based on
actual flight behavior.