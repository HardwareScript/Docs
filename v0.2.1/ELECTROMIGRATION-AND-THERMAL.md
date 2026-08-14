# Electromigration and Thermal Rise Validation

## Overview

The compiler validates two distinct current-related failure modes:

- **P21: Electromigration (EM)** - Metal atom migration under high current density
- **P22: Thermal Rise** - Localized heating from I²R power dissipation

These are separate physical phenomena with different failure mechanisms, time scales, and mitigation strategies.

---

## P21: Electromigration (EM)

### Physical Phenomenon
High-velocity electrons physically collide with metal atoms (electron wind), pushing them downstream. Over time:
- **Voids** form where atoms leave → Open circuits
- **Hillocks** grow where atoms accumulate → Short circuits

### Governing Equation
**Black's Equation:**
```
MTTF = A / J^n × exp(Ea / kT)
```
Mean Time To Failure decreases exponentially with current density (J) and temperature (T).

### Check Formula
```
J = I_peak / (Width × Thickness)
```
**Violation:** `J > Material.max_current_density`

### Typical Limits
| Material | Max Current Density |
|----------|---------------------|
| Aluminum | 1.0 mA/μm² (1000 A/mm²) |
| Copper   | 2.0 mA/μm² (2000 A/mm²) |
| Polysilicon | 0.1 mA/μm² (100 A/mm²) |

### Material Declaration
Define `max_current_density` in your materials file:

```hw
material Aluminum_Met1:
    category: conductor
    resistivity: 2.82e-8 Ohm*m
    thermal_conductivity: 237 W/(m*K)
    max_current_density: 1000 A/mm²   # ← EM threshold
```

### Solutions
1. Increase trace width (reduces J)
2. Reduce operating current
3. Use material with higher EM threshold (Cu > Al > Poly)
4. Reduce operating temperature

---

## P22: Thermal Rise

### Physical Phenomenon
RMS current through resistive traces generates Joule heat. When heat cannot dissipate fast enough, local temperature rises, causing:
- Dielectric delamination
- Dopant drift in semiconductors
- Thermal runaway in high-resistance materials
- Solder joint degradation

### Governing Equations
```
R = ρ × (L / A)                 [Trace resistance]
P = I_RMS² × R                  [Joule heating power]
ΔT = P / (k × Surface_Area)     [Temperature rise, 1D diffusion model]
```

**Violation:** `ΔT > Profile.max_temp_rise`

### Profile Declaration
**Required field** - no defaults. Declare thermal budget explicitly:

```hw
profile My_Profile:
    technology: "ASIC"
    
    thermal:
        ambient_temp: 25C
        max_operating_temp: 125C
        max_temp_rise: 20C          # ← REQUIRED for P22 validation
```

### Typical Thermal Budgets
| Application | Max ΔT |
|-------------|--------|
| Consumer electronics | 20°C (JEDEC standard) |
| Automotive | 15°C (conservative) |
| High-reliability | 10°C (aerospace/medical) |

### Solutions
1. Reduce current (lower power)
2. Increase trace width (lower resistance)
3. Shorten trace length (lower resistance)
4. Improve thermal path to substrate
5. Use material with lower resistivity

---

## Key Differences

| Aspect | P21: Electromigration | P22: Thermal Rise |
|--------|----------------------|-------------------|
| **Root Cause** | Electron momentum transfer | I²R Joule heating |
| **Formula** | J = I / A | ΔT = (I²R) / (k×Area) |
| **Threshold Source** | Material property | Profile declaration |
| **Failure Mode** | Voids & hillocks (months/years) | Immediate overheating |
| **Current Type** | Peak DC current | RMS current |
| **Material Dependence** | max_current_density | resistivity + thermal_conductivity |

---

## Example: Test Case

```hw
# test_em_violation.hw - Triggers BOTH violations

space EM_Test implements MyModule:
    dimensions: 10.0um by 5.0um
    profile: Test_Profile
    
    nets:
        In: { current: 80uA }  # High current
    
    # Thin aluminum trace: 150nm × 360nm × 8μm
    route In_Pad to Out_Pad:
        width: 150nm    # Too thin!
        layer: metal1
```

### Expected Violations

**P21 (Electromigration):**
```
J = 80μA / (150nm × 360nm) = 1481 A/mm²
Limit = 1000 A/mm² for Aluminum
Violation: 1481 > 1000 ✗
```

**P22 (Thermal Rise):**
```
R = 2.82e-8 × (8e-6 / 5.4e-14) = 3.92 Ω
P = (80e-6)² × 3.92 = 25 nW
ΔT = 25e-9 / (237 × 2.4e-12) = 47°C
Violation: 47°C > 20°C ✗
```

---

## Debug Output

```
[DRC EM DEBUG] Checking net 'In': material_id=11, max_density=1000.00 A/mm²
[DRC EM DEBUG] • EM violation for In: 1481.48 A/mm² > 1000.00 A/mm²

[DRC THERMAL DEBUG] Checking net 'In': ρ=2.82e-8 Ω·m, k=237 W/(m·K)
[DRC THERMAL DEBUG] • Net 'In': R=3.92Ω, P=25nW, ΔT=47.0°C (limit: 20.0°C)
[DRC THERMAL DEBUG] • Thermal violation for In: ΔT=47.0°C > 20.0°C

❌ DRC VIOLATIONS DETECTED:
• Electromigration violation for In: 1481.48 A/mm² > 1000.00 A/mm²
• Thermal rise violation for In: ΔT=47.0°C > 20.0°C
```

---

## References

- **Black's Equation**: J.R. Black, "Electromigration—A brief survey and some recent results," IEEE Trans. Electron Devices (1969)
- **IPC-2221**: Generic Standard on Printed Board Design (trace width for current capacity)
- **JEDEC JESD51**: Thermal measurement standards
