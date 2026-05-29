# Hardware Script v0.1.2 - Material Database Architecture

**Document Type**: Data Architecture Strategy  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## The Fundamental Question

Should we use YAML files or a database for materials? Should PCB materials be separate from silicon materials?

The answer reveals a deeper architectural truth about how Hardware Script must scale.

---

## Current State: YAML is Perfect for Now

### Why YAML Works (Phase 1: MVP)

Right now, we are building the logic of the engine—how the compiler reads a constraint, and how the 3D voxel water flows.

**Benefits of YAML**:

1. **Speed**: Parsing YAML into a Python dictionary, C++ Struct, or Rust Hashmap takes microseconds
2. **Git-Friendly**: Text files are easily tracked in GitHub. SQL databases are opaque binary blobs
3. **Human-Readable**: When debugging why traces are catching fire in simulation, you can just open the YAML and instantly see `copper.max_current_density_a_mm2: 35`
4. **No Dependencies**: No database server, no connection strings, no migrations

**Action**: Keep `standard-materials.yaml` exactly as it is for now to prove the engine works.

---

## The Macro vs. Nano Problem

### The Question

Should we define standard materials for PCB different from standard materials for components (like silicon chips)?

### The Answer: Yes and No

You must separate the **Physics** from the **Manufacturing**.

#### The Physics (The Material Library)

**Copper is copper.**

Whether it is:
- A giant 2mm thick busbar on a power supply
- A 5-nanometer trace inside an Apple M3 chip

It is still Cu. It still has:
- Melting point: 1085°C
- Resistivity: 1.68e-8 Ω·m
- Thermal conductivity: 401 W/mK

**Therefore**: You should have **one unified material library**.

The laws of physics don't change based on scale.

#### The Manufacturing (The "PDK")

What **does** change between a PCB and a microchip is the **factory**.

In professional silicon design, foundries (like TSMC or Intel) provide something called a **PDK (Process Design Kit)**.

**You should eventually separate your standard files into two concepts**:

1. **materials.yaml**: The raw laws of physics
   - Density
   - Resistivity
   - Thermal Conductivity
   - Melting Point
   - Dielectric Strength

2. **manufacturing-rules.yaml** (The PDK): The rules of the factory
   - Minimum trace width
   - Available materials
   - Layer stackup options
   - Cost per square millimeter

**Example PCB Factory Rule**:
```yaml
name: "JLCPCB Standard PCB"
minimum_trace_width: 0.1mm
available_materials:
  - FR4
  - Copper
  - Gold (surface finish)
```

**Example Microchip Factory Rule**:
```yaml
name: "TSMC 3nm Process"
minimum_trace_width: 5nm
available_materials:
  - Silicon
  - Silicon Dioxide
  - Hafnium
  - Aluminum
```

**The Power of This Separation**:

By separating the material physics from the factory rules, your engine can route:
- A 5-foot-long motherboard
- A 5-nanometer transistor

Using the **exact same underlying C++/Rust code**. It just loads a different ruleset.

---

## The Future Architecture: The 3-Tiered Material System

When your engine gets so good that people are building crazy custom stuff, a single YAML file will become too messy.

Here is the industry-standard way to handle custom user data in EDA software.

### Level 1: The Core Standard Library (Immutable)

**File**: `standard-materials.yaml` (ships with the software)

**Contents**:
- The undeniable laws of physics for standard elements
- Copper, Silicon, Gold, FR4, Air, etc.

**Rules**:
- Users **cannot** edit this file directly
- This is the source of truth for fundamental physics

**Example**:
```yaml
conductors:
  copper:
    name: Copper
    symbol: Cu
    resistivity_ohm_m: 1.68e-08
    melting_point_c: 1085
    thermal_conductivity_w_mk: 401
    density_kg_m3: 8960.0
    materials_project_id: mp-1056079
```

### Level 2: The Factory/PDK Library (Provided by manufacturers)

**File**: `jlcpcb-materials.yaml` or `tsmc-3nm-materials.yaml`

**Contents**:
- Specific composite materials
- Specific grades of FR4
- Factory-specific alloys

**Downloaded from**: Manufacturer websites or package registry

**Example**:
```yaml
# JLCPCB-specific materials
insulators:
  fr4_jlcpcb_standard:
    name: FR4 (JLCPCB Standard Grade)
    base_material: fr4  # References Level 1
    dielectric_constant: 4.6  # Slightly different from generic
    color_options: [green, blue, red, black]
    cost_per_sqm: 2.50
```

### Level 3: The User/Project Library (Fully Custom)

**File**: `my-custom-materials.yaml` (lives in the user's project folder)

**Contents**:
- Experimental materials
- Custom alloys
- Proprietary compounds

**Use Case**: If a user is building a crazy new battery and needs to invent a material called "Lithium-Unicorn-Alloy", they put it here.

**Example**:
```yaml
# My experimental battery project
conductors:
  lithium_unicorn_alloy:
    name: Lithium-Unicorn Alloy (Experimental)
    resistivity_ohm_m: 2.1e-08
    melting_point_c: 180
    thermal_conductivity_w_mk: 85
    density_kg_m3: 534
    notes: "Experimental material for low-temp superconductor research"
```

### The Overwrite Rule

**Priority Order** (highest to lowest):
1. User/Project Library (Level 3)
2. Factory/PDK Library (Level 2)
3. Core Standard Library (Level 1)

**Example**:

If the user defines "Copper" in their project file, the engine uses **their version** of Copper for that project, overriding the Standard Library.

This allows:
- Testing hypothetical materials
- Simulating degraded components
- Research and experimentation

---

## Nano-Scale Physics: The Resistivity Curve

### The Problem at Small Scales

At the macro scale (PCB traces), Copper's resistivity is fixed: `1.68e-8 Ω·m`

But when a wire gets smaller than the **mean free path** of an electron (around 40 nanometers), electrons start bouncing off the walls of the wire, and resistivity goes up.

**This is called the "Size Effect" or "Surface Scattering".**

### Current Implementation (v0.1)

```yaml
copper:
  resistivity_ohm_m: 1.68e-08  # Flat number
```

This works fine for PCBs (traces are 100+ micrometers wide).

### Future Implementation (v2.0+ for Silicon)

Eventually, your YAML won't just have a flat number. It will need a **curve** or a **formula**:

```yaml
copper:
  resistivity_base_ohm_m: 1.68e-08
  resistivity_curve: "size_effect_copper.csv"
  # or
  resistivity_formula: "rho_base * (1 + (lambda / width))"
  lambda_nm: 40  # Mean free path
```

**The CSV file**:
```csv
width_nm,resistivity_ohm_m
100,1.68e-08
50,1.85e-08
40,2.10e-08
30,2.50e-08
20,3.20e-08
10,5.00e-08
5,8.50e-08
```

**Why This Matters**:

When routing a 5nm chip, the compiler needs to know that a 5nm copper trace has **5× higher resistance** than bulk copper. This affects:
- Voltage drop calculations
- Power dissipation
- Signal integrity

---

## Implementation Timeline

### Phase 1: Now (v0.1 - v0.2)

**Status**: ✅ Done

- Single `standard-materials.yaml` file
- Flat resistivity values
- Basic material properties

**Action**: Keep using this. Build the core engine.

### Phase 2: Next Month (v0.3)

**Status**: 🔄 In Progress

- Implement the compiler logic that reads YAML
- Create "Clearance Hitboxes" (for voltage)
- Create "Trace Widths" (for current)
- Integrate into 3D voxel engine

### Phase 3: Next Quarter (v0.4 - v0.5)

**Status**: 📋 Planned

- Implement the 3-Tiered system
- Allow users to inject custom YAML files
- Add factory/PDK library support
- Create material override mechanism

### Phase 4: Next Year (v1.0+)

**Status**: 🔮 Future

- Add resistivity curves for nano-scale
- Implement size-effect calculations
- Support for silicon foundry PDKs
- Advanced material modeling

---

## File Structure

### Current (v0.1)

```
Hardware-Script/
├── live-test/
│   └── standard-materials.yaml
└── hwc/
    └── data/
        └── standard-materials.yaml
```

### Future (v1.0)

```
Hardware-Script/
├── materials/
│   ├── core/
│   │   └── standard-materials.yaml      # Level 1: Core
│   ├── factories/
│   │   ├── jlcpcb-standard.yaml         # Level 2: Factory
│   │   ├── oshpark-standard.yaml
│   │   └── tsmc-3nm.yaml
│   └── curves/
│       ├── copper-size-effect.csv
│       └── silicon-temperature.csv
└── my-project/
    └── custom-materials.yaml             # Level 3: User
```

---

## Key Takeaways

1. **YAML is perfect for now** - Don't over-engineer a database system yet

2. **Physics is universal** - One material library for all scales (PCB to silicon)

3. **Manufacturing is specific** - Separate factory rules from material properties

4. **3-Tiered system** - Core → Factory → User (with override priority)

5. **Nano-scale needs curves** - Flat resistivity values won't work for <40nm features

6. **Build the engine first** - Data structures can always be optimized later

---

## Summary: Your Next Steps

### Right Now
- ✅ Do not change a thing
- ✅ Your `standard-materials.yaml` is brilliant
- ✅ Use it to build Phase 1

### Next Month
- 🔄 Write the compiler logic that reads this YAML
- 🔄 Create "Clearance Hitboxes" (for voltage)
- 🔄 Create "Trace Widths" (for current)
- 🔄 Integrate into 3D voxel engine

### Next Year
- 📋 Implement the 3-Tiered system
- 📋 Allow custom YAML injection
- 📋 Add resistivity curves for nano-scale

**Build the engine first. The data structures can always be optimized later.**

---

**Document Status**: Data Architecture Strategy  
**Last Updated**: March 2026  
**Next Document**: Physics to Constraints Translation

