# HEMI-HEMMINKI Engine Technology Documentation

*Author: Juho Artturi Hemminki*

## 1. System Description & Core Concept: Physics of Dynamic Geometric Morphing

The dynamic geometric morphing system utilizes a ceramic silicon carbide (\(\text{C/SiC}\)) movable sleeve integrated into the combustion chamber architecture. This mechanism alters the effective compression ratio and chamber geometry in real time based on engine load and operational parameters.

* **Eco/Cruising State (12.5:1 Compression Ratio):** At partial load and low rpm ranges, the \(\text{C/SiC}\) sleeve retracts to minimize clearance volume, yielding a high geometric compression ratio (12.5:1) for optimal thermal efficiency and lean-burn fuel economy.
* **HEMI Power State (7.5:1 Compression Ratio):** Under heavy acceleration or high boost conditions, the hydraulic actuation shifts the sleeve mechanically. This expands the squish and quench area into a pseudo-hemispherical profile, dropping the effective compression ratio to 7.5:1 to tolerate high specific output and elevated peak cylinder pressures without detrimental auto-ignition.

---

## 2. Governing Thermodynamic & Kinematic Equations

The dynamic behavior of the fluid mass within the chamber and the mechanical boundary condition transitions are governed by the following mathematical formulations:

### 2.1 Polytropic Compression and Expansion

The idealized instantaneous pressure \(P(\theta)\) as a function of crank angle \(\theta\) and variable volume \(V(\theta, x)\)—where \(x\) represents the instantaneous vertical sleeve displacement—is given by:

\[P(\theta) = P_{IVC} \left( \frac{V_{IVC}}{V(\theta, x)} \right)^n\]

Where:
* \(P_{IVC}\) = Pressure at Intake Valve Closing
* \(V_{IVC}\) = Volume at Intake Valve Closing
* \(n\) = Polytropic exponent (\(1.30 \le n \le 1.37\) depending on localized gas temperature and composition)

The total instantaneous cylinder volume \(V(\theta, x)\) is computed via:

\[V(\theta, x) = V_c(x) + \frac{\pi D^2}{4} \cdot f(\theta)\]

Where \(V_c(x)\) is the variable clearance volume modulated by the sleeve:

\[V_c(x) = V_{c,\min} + \frac{\pi D^2}{4} \cdot x\]

### 2.2 Ideal Otto Cycle Thermal Efficiency Modification

The theoretical thermal efficiency \(\eta_{th}(x)\) shifts as a function of the sleeve position \(x\) altering the dynamic compression ratio \(r_c(x)\):

\[\eta_{th}(x) = 1 - \frac{1}{\left[r_c(x)\right]^{\gamma - 1}}\]

Where:
* \(\gamma\) = Ratio of specific heats (\(c_p / c_v \approx 1.4\) for ambient air, dropping to \(\approx 1.3\) during high-temperature combustion)
* \(r_c(x)\) = Dynamic compression ratio ranging bounded by \(7.5 \le r_c(x) \le 12.5\)

---

## 3. Control Logic and C++23 Core (`ecu.cpp`)

The electronic control unit executes a predictive look-ahead algorithm written in C++23, compensating for a 3.5 ms hydraulic actuation latency by applying up to a \(+2.2\text{ mm}\) phase shift prior to the ignition event. Cylinder-specific PID controllers maintain precise sleeve positioning.

```cpp
#ifndef HEMI_ACTUATION_CORE_HPP
#define HEMI_ACTUATION_CORE_HPP

#include <cmath>
#include <algorithm>
#include <numbers>

namespace HemiControl {

struct PIDCoefficients {
    double kp = 350.0;
    double ki = 45.0;
    double kd = 12.5;
};

class SleeveActuator {
private:
    PIDCoefficients pid_;
    double integral_error_ = 0.0;
    double prev_error_ = 0.0;
    const double hydraulic_latency_ms_ = 3.5;
    const double max_travel_mm_ = 2.2;

public:
    explicit SleeveActuator(PIDCoefficients coeffs) : pid_(coeffs) {}

    double calculateLookAheadShift(double rpm, double load_derivative) {
        // Predictive look-ahead phase shift calculation based on RPM and load gradient
        double time_factor = hydraulic_latency_ms_ * 0.001; // convert to seconds
        double predicted_delta = load_derivative * time_factor * (rpm / 60.0);
        return std::clamp(predicted_delta, -max_travel_mm_, max_travel_mm_);
    }

    double updatePID(double setpoint, double measurement, double dt) {
        double error = setpoint - measurement;
        integral_error_ += error * dt;
        
        // Anti-Windup Clamp on integral term
        integral_error_ = std::clamp(integral_error_, -10.0, 10.0);
        
        double derivative = (error - prev_error_) / (dt > 0.0 ? dt : 1e-3);
        prev_error_ = error;

        return (pid_.kp * error) + (pid_.ki * integral_error_) + (pid_.kd * derivative);
    }
};

} // namespace HemiControl

#endif // HEMI_ACTUATION_CORE_HPP
```

---

## 4. Safety Mechanisms and Fault States (Fail-Safe)

* **Emergency Override (\(<250\,\mu\text{s}\)):** In the event of knock detection, oil pressure loss, or critical over-boost, the system triggers a rapid fail-safe routine within 250 microseconds. This dumps manifold pressure via the wastegate, retards ignition timing to \(-5.0^\circ\), and retracts the \(\text{C/SiC}\) sleeves to baseline position.
* **Limp-Home Mode:** Upon sensory failure (e.g., position feedback error on a specific cylinder), the control unit locks the hydraulic rail pressure, disables dynamic morphing, and maps the engine to a safe, fixed high-compression or low-compression default profile depending on thermal load.

---

## 5. Acoustic Dynamics and Morphing Soundscape

The acoustic signature shifts dynamically due to the shifting geometry of the combustion chamber and varying effective volume:
* **Low-Frequency Cavity Resonance:** When operating in the 7.5:1 HEMI configuration, the larger open chamber volume and altered flame propagation paths amplify primary engine firing pulses at lower frequencies, producing a distinct, heavy, "thumping" exhaust and mechanical beat.
* **Turbocharger Acoustics:** The rapid transition of mass flow and enthalpy changes across the turbine housing—coupled with the adjustable geometry—modulates the blade-passing frequencies, generating a sharp, high-decibel turbo spool and surge-whistle characteristic under transient throttle shifts.

---

**Author: Juho Artturi Hemminki**
