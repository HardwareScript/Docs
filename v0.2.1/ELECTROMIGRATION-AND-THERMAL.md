# Electromigration and Thermal Rise Validation

**Version**: v0.2.1+  
**Critical Architectural Boundary**: Static Budget Validation vs. Dynamic Operating Point Simulation

---

## ⚠️ THE GOLDEN BOUNDARY: What `current` Really Means

### Semantic Clarity

In HardwareScript `.hw` files, the `current` field is **NOT** a simulated operating point. It is a **DESIGN BUDGET**:

```hw
nets:
    In: { classification: signal, potential: 1.8V, current: 1.0uA }
```

**What this declares:**
- ✅ "This trace must be fabricated to safely carry **up to** 1.0μA without electromigration failure"
- ✅ A **capability constraint** for trace width sizing during routing
- ✅ A **static DRC budget** to validate geometry vs. material limits

**What this does NOT declare:**
- ❌ "The actual operating current when the circuit is powered on"
- ❌ A simulated DC operating point (requires SPICE matrix solver)
- ❌ A measured branch current (requires `.op` or `.tran` simulation)

### The Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. HARDWARESCRIPT COMPILER (hwc)                                        │
│    • Role: Physical Synthesis, Geometric Extraction & Budget DRC        │
│    • Uses current: for trace width sizing & static ampacity checks      │
│    • Output: Extracted SPICE netlist with W, L, R, C parasitics         │
│    • Validation: "Can this wire geometry support its declared budget?"  │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ Emits circuit.sp, dc.sp, tran.sp
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. SIMULATION ENGINE (ngspice.wasm / Xyce / LTSpice)                    │
│    • Role: Matrix Numerical Physics (Newton-Raphson, KCL, KVL)          │
│    • Solves: Real operating currents from V = IR, diode equations, etc. │
│    • Output: Node voltages & branch currents (.raw simulation data)     │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ Back-annotates I_simulated
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. ELECTRICAL SIGN-OFF (hwc test / hsm)                                 │
│    • Dynamic EM/Thermal Check: Compares I_simulated vs. Wire Capability │
│    • Budget Assertion: Validates I_simulated ≤ I_declared_budget        │
│    • Triggers P21-D/P22-D violations if budget exceeded                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Compiler Validation (This Document)

The compiler performs **STATIC** validation at build time:

1. **Static Ampacity Check (P21)**: Does the declared `current: 1.0uA` budget exceed the wire's physical capability based on geometry and `max_current_density`?
2. **Static Thermal Budget (P22)**: If the trace carried its **declared budget** current, would self-heating exceed `max_temp_rise`?

**The compiler does NOT:**
- ❌ Calculate actual operating currents (requires solving I = V/R for the full circuit)
- ❌ Solve non-linear equations (diode I-V curves, MOSFET operating points)
- ❌ Run SPICE simulations internally

**Why This Boundary?**
- Keeps compiler lightweight (<100ms build times)
- Avoids bloating Rust codebase with SPICE matrix solvers
- Maintains zero-magic philosophy (user declares budgets explicitly)
- Matches commercial EDA methodology (Calibre, PVS, Virtuoso)

---

## Overview

The compiler validates two distinct current-related failure modes:

- **P21: Electromigration (EM)** - Metal atom migration under high current density
- **P22: Thermal Rise** - Localized heating from I²R power dissipation

These are separate physical phenomena with different failure mechanisms, time scales, and mitigation strategies.

**⚠️ CRITICAL**: These checks validate **declared budgets** vs. **geometric capability**, NOT simulated operating points vs. physical limits. Dynamic validation happens post-simulation in `hwc test` or `hsm`.

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

### Static Compiler Check (P21)

The compiler validates **declared budget vs. wire capability**:

```
J_budget = I_declared / (Width × Thickness)
```

**Violation:** `J_budget > Material.max_current_density`

**Example:**
```hw
nets:
    In: { current: 2.0mA }  # User declares 2mA budget

route In to Out:
    width: 300nm
    layer: metal1  # Aluminum, thickness 360nm
```

**Compiler Check:**
```
Wire cross-section: 300nm × 360nm = 0.108 μm²
Budget current density: 2.0mA / 0.108μm² = 18,518 A/mm²
Material limit (Aluminum): 1,000 A/mm²

❌ STATIC EM VIOLATION (P21)
Declared budget of 2.0mA exceeds wire capability.
Wire geometry (300nm × 360nm) can only support 108.0μA.
Widen trace to at least 1.85μm or reduce budget constraint.
```

**This check does NOT use simulated currents** - it validates the user's declared budget against physical geometry.

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

### Static Compiler Check (P22)

The compiler validates **declared budget vs. thermal capacity**:

```
R = ρ × (L / A)                 [Trace resistance from geometry]
P = I_budget² × R               [Power if trace carries its budget]
ΔT = P / (k × Surface_Area)     [Temperature rise, 1D diffusion model]
```

**Violation:** `ΔT > Profile.max_temp_rise`

**Example:**
```hw
nets:
    In: { current: 1.8mA }  # User declares 1.8mA budget

route In to Out:
    width: 300nm
    layer: metal1  # 8μm long aluminum trace
```

**Compiler Check:**
```
R = 2.82e-8 × (8e-6 / 1.08e-13) = 2.09 Ω
P = (1.8e-3)² × 2.09 = 6.77 μW
ΔT = 6.77e-6 / (237 × 4.8e-12) = 5,950°C  (!)

❌ STATIC THERMAL VIOLATION (P22)
If trace carries declared budget of 1.8mA, self-heating is 5,950°C.
Exceeds thermal budget of 20°C by 297×.
Widen trace, shorten route, or reduce budget constraint.
```

**This check uses the declared budget current**, not actual operating current from simulation.

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

## Example: Static vs. Dynamic Validation

### Test Case: Resistor Circuit

```hw
# simple_resistor_test.hw

space Simple_Resistor:
    profile: Test_Profile  # max_temp_rise: 20C
    
    nets:
        In:  { potential: 1.8V, current: 1.0uA }  # BUDGET: 1.0μA
        Out: { potential: 0.0V, current: 1.0uA }
    
    # 4μm polysilicon resistor → R ≈ 1kΩ
    device R1 Resistor: { ... }
    
    # Aluminum routing trace: 300nm × 360nm
    route In to R1.A:
        width: 300nm
        layer: metal1
```

### Compiler Static Validation (BUILD TIME)

**P21 (Electromigration):**
```
Declared budget: 1.0μA
Wire cross-section: 300nm × 360nm = 0.108 μm²
Budget density: 1.0μA / 0.108μm² = 9.26 A/mm²
Material limit (Aluminum): 1,000 A/mm²

✅ STATIC CHECK PASSED: 9.26 < 1,000
```

**P22 (Thermal Rise):**
```
Budget current: 1.0μA
Trace resistance: ~0.65 Ω
Budget power: (1.0μA)² × 0.65Ω = 6.5×10⁻¹³ W
ΔT ≈ 0.0°C

✅ STATIC CHECK PASSED: 0.0°C < 20°C
```

**Result:** Compiler produces `build/` directory with clean SPICE netlist.

---

### Reality Check: What SPICE Simulation Reveals

When `hwc test` or `hsm` runs the DC operating point:

```spice
* Actual circuit stimulus
V_In nIn_entry 0 DC 1.800

* Real operating current
I_actual = (1.8V - 0V) / 1000Ω = 1.8 mA
```

**Actual vs. Declared:**
- **Declared budget**: 1.0 μA
- **Simulated operating current**: 1.8 mA
- **Ratio**: 1,800× over budget!

**Actual Current Density:**
```
J_real = 1.8mA / 0.108μm² = 16,666 A/mm²
Material limit: 1,000 A/mm²

❌ DYNAMIC EM VIOLATION (P21-D)
Simulated current 1.8mA exceeds wire capability by 16.7×
```

**Actual Thermal Rise:**
```
P_real = (1.8mA)² × 0.65Ω = 2.1 μW
ΔT_real ≈ 1,800°C (!)

❌ DYNAMIC THERMAL VIOLATION (P22-D)
Simulated power dissipation causes 1,800°C temperature rise
```

---

### The False-Positive Bug (PRE-v0.2.2)

**Before this architectural clarification:**
- Compiler used `current: 1.0uA` as if it were the actual operating current
- Reported `✅ PASSED` for a circuit that would fail catastrophically in silicon
- No mechanism to validate simulated currents vs. wire capability

**After v0.2.2+ (This Document):**
- Compiler validates **declared budgets** vs. **geometry** (static DRC)
- `hwc test` / `hsm` validates **simulated currents** vs. **wire capability** (dynamic sign-off)
- Clear error messages distinguish static (P21/P22) from dynamic (P21-D/P22-D) violations

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

---

## Summary Table: Static vs. Dynamic Validation

| Aspect | Compiler Static DRC (P21/P22) | Post-Sim Sign-Off (P21-D/P22-D) |
|--------|------------------------------|----------------------------------|
| **When** | Build time (`hwc build`) | Test time (`hwc test`, `hsm`) |
| **Input** | Declared `current` budget | SPICE-simulated branch currents |
| **Check** | Budget ≤ Wire Capability | I_simulated ≤ Wire Capability |
| **Purpose** | Validate geometry can support declared intent | Validate actual operating point is safe |
| **Failure** | ❌ Trace too thin for budget | ❌ Circuit operates beyond safe limits |
| **Fix** | Widen trace or reduce budget | Change circuit design (add resistor, reduce voltage) |

---

## Roadmap: Dynamic Sign-Off (v0.3.0+)

**Status:** Static validation (P21/P22) is implemented. Dynamic validation (P21-D/P22-D) is planned.

**Implementation Plan:**

1. **SPICE Simulation Integration**
   - `hwc test` runs `ngspice.wasm` or `Xyce` on `dc.sp` testbench
   - Extracts branch currents from `.raw` output files
   
2. **Back-Annotation Pipeline**
   - Parse simulation results: `I(V_In) = 1.8mA`
   - Match branch currents to net names
   - Store in `build/simulation_results.json`

3. **Dynamic Validation Pass**
   ```rust
   fn validate_dynamic_em(
       simulated_currents: &HashMap<Net, f64>,
       analytic_routes: &[AnalyticTrace],
       material_registry: &MaterialRegistry,
   ) -> Vec<DrcViolation> {
       // Compare I_simulated vs. wire capability
   }
   ```

4. **HUD Integration (hsm)**
   - Ghost View displays dynamic violations as red highlights
   - Tooltip shows: "⚠️ Net 'In': 1.8mA simulated > 108μA capability (16.7× over)"

**Design Principle:** The compiler remains a pure geometric extractor. SPICE handles physics. Sign-off tool validates safety.

---

## References

- **Black's Equation**: J.R. Black, "Electromigration—A brief survey and some recent results," IEEE Trans. Electron Devices (1969)
- **IPC-2221**: Generic Standard on Printed Board Design (trace width for current capacity)
- **JEDEC JESD51**: Thermal measurement standards
- **Commercial EDA Equivalents**: Mentor Calibre nmDRC, Synopsys IC Validator, Cadence Voltus
