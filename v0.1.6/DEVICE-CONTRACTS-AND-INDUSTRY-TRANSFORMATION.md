# Device Contracts: The Interface for Atoms

**Document Type**: Strategic Industry Analysis  
**Status**: Foundational Architecture  
**Date**: April 2026  
**Purpose**: Understand Hardware Script's transformative impact on the EDA industry

---

## Executive Summary

To understand the significance of Hardware Script (HWS), we have to look at what happens when a **fragmented, GUI-based industry** suddenly gets hit by a **Unified, Text-Based Compiler**.

This document explores:
1. **Device Contracts**: The missing bridge between logical intent and physical reality
2. **Historical Mirror**: How HWS parallels the transformation that happened in software
3. **Industry Impact**: Why this is a "Superweapon" against legacy EDA tools

---

## Part 1: The Device Contract - The "Interface" for Atoms

### The Fundamental Problem

**Current State**: The compiler "sees" a block of Polysilicon and a block of Silicon_N, but it doesn't "know" it's looking at an NMOS transistor unless you hard-code that knowledge into the engine.

**The Gap**: There's no formal contract between:
- What the designer **intends** (logical behavior: `add NMOS named M1`)
- What the designer **builds** (physical geometry: pours of Silicon_N, Polysilicon, etc.)

### The Solution: Device Contracts as Interfaces

Think of Device Contracts like:
- **Header Files in C**: Define the interface before implementation
- **Interfaces in Java/TypeScript**: Define the contract that implementations must satisfy
- **Protocols in Swift**: Define behavior requirements
- **Traits in Rust**: Define shared behavior

**But for hardware, not software.**

### The Device Contract Definition

```hw
# stdlib/foundry/contracts.hw

device NMOS:
    # The Interface: What terminals must exist
    terminals: [gate, source, drain, bulk]
    
    # The Contract: What materials are physically required
    physics_check:
        gate: must_be [Polysilicon, Aluminum]
        source: must_be [Silicon_N]
        drain: must_be [Silicon_N]
        bulk: must_be [Silicon_P]
    
    # The Geometry Rules: How to identify this device in the voxel grid
    extraction_rules:
        # Gate is the control terminal (poly or metal crossing active region)
        gate: 
            material in [Polysilicon, Aluminum]
            crosses active_region
            orientation: perpendicular
        
        # Source/Drain are N-doped regions on either side of gate
        source: 
            material = Silicon_N
            adjacent_to gate
            side: left or right
        
        drain: 
            material = Silicon_N
            adjacent_to gate
            opposite_side_of source
        
        # Bulk is P-doped substrate contact (prevents latch-up)
        bulk: 
            material = Silicon_P
            overlaps active_region
            electrically_connected_to source or drain
    
    # The Physics: How this device behaves electrically
    spice_model:
        type: nmos
        parameters: [W, L, AS, AD, PS, PD]
        model_card: "NMOS_TSMC180"

device PMOS:
    terminals: [gate, source, drain, bulk]
    
    physics_check:
        gate: must_be [Polysilicon, Aluminum]
        source: must_be [Silicon_P]
        drain: must_be [Silicon_P]
        bulk: must_be [Silicon_N]  # N-well for PMOS
    
    extraction_rules:
        gate: 
            material in [Polysilicon, Aluminum]
            crosses active_region
        
        source: 
            material = Silicon_P
            adjacent_to gate
        
        drain: 
            material = Silicon_P
            adjacent_to gate
            opposite_side_of source
        
        bulk: 
            material = Silicon_N
            overlaps active_region
    
    spice_model:
        type: pmos
        parameters: [W, L, AS, AD, PS, PD]
        model_card: "PMOS_TSMC180"

device Resistor:
    terminals: [A, B]
    
    physics_check:
        body: must_be [NiCr, Polysilicon, DiffusionResistor]
    
    extraction_rules:
        A: metal_contact at start_of body
        B: metal_contact at end_of body
    
    spice_model:
        type: resistor
        parameters: [R, W, L]
        calculation: R = ρ × (L / (W × t))

device Capacitor:
    terminals: [A, B]
    
    physics_check:
        plate1: must_be [Copper, Aluminum]
        dielectric: must_be [SiO2, HfO2, TantalumOxide]
        plate2: must_be [Copper, Aluminum]
    
    extraction_rules:
        A: top_plate
        B: bottom_plate
    
    spice_model:
        type: capacitor
        parameters: [C, A]
        calculation: C = ε₀ × εᵣ × (A / d)

# Future-proof: Emerging devices
device GrapheneTransistor:
    terminals: [gate, source, drain]
    
    physics_check:
        gate: must_be [Graphene]
        source: must_be [Graphene]
        drain: must_be [Graphene]
        dielectric: must_be [hBN, SiO2]
    
    extraction_rules:
        gate: 
            material = Graphene
            layer: top
        
        source: 
            material = Graphene
            layer: bottom
            side: left
        
        drain: 
            material = Graphene
            layer: bottom
            side: right
    
    spice_model:
        type: gfet
        parameters: [W, L, Vgs, Vds]
        model_card: "GRAPHENE_FET_V1"

device MemristorDevice:
    terminals: [top, bottom]
    
    physics_check:
        active_layer: must_be [TiO2, HfO2, TaOx]
        electrode_top: must_be [Platinum, TiN]
        electrode_bottom: must_be [Platinum, TiN]
    
    extraction_rules:
        top: electrode_top
        bottom: electrode_bottom
    
    spice_model:
        type: memristor
        parameters: [Ron, Roff, D, uv, p]
        model_card: "MEMRISTOR_VTEAM"
```

### Why This Is a Superweapon

**The Power of Abstraction**:

1. **Compiler Independence**: If tomorrow a researcher invents a "Graphene Transistor," you don't update your compiler. You just write a new Device Contract for Graphene.

2. **Foundry Portability**: Different foundries can publish their own device contracts:
   - `@tsmc/5nm/contracts.hw` - TSMC 5nm devices
   - `@samsung/7nm/contracts.hw` - Samsung 7nm devices
   - `@intel/18a/contracts.hw` - Intel 18A devices

3. **Validation Automation**: The compiler uses the contract to "grade" the user's 3D Voxel drawing:
   ```
   ✅ NMOS M1: All terminals found
   ✅ Gate material: Polysilicon (valid)
   ✅ Source material: Silicon_N (valid)
   ✅ Drain material: Silicon_N (valid)
   ✅ Bulk material: Silicon_P (valid)
   ✅ Bulk connected to GND (valid)
   ```

4. **Research Enablement**: Researchers can experiment with novel devices without waiting for EDA vendors to support them.

### The Compiler Implementation

```rust
pub struct DeviceContract {
    pub device_type: String,
    pub terminals: Vec<String>,
    pub physics_check: HashMap<String, MaterialConstraint>,
    pub extraction_rules: Vec<ExtractionRule>,
    pub spice_model: SpiceModelTemplate,
}

pub enum MaterialConstraint {
    MustBe(Vec<MaterialId>),
    MustNotBe(Vec<MaterialId>),
    MustHaveProperty { property: String, value: f64 },
}

pub struct ExtractionRule {
    pub terminal_name: String,
    pub material_filter: MaterialFilter,
    pub geometric_constraint: GeometricConstraint,
    pub connectivity_constraint: Option<ConnectivityConstraint>,
}

impl DeviceExtractor {
    pub fn extract_with_contract(
        &self,
        grid: &VoxelGrid,
        contract: &DeviceContract,
    ) -> Result<Vec<ExtractedDevice>, ExtractionError> {
        let mut devices = Vec::new();
        
        // Step 1: Find candidate regions that match material requirements
        let candidates = self.find_candidate_regions(grid, &contract.physics_check);
        
        for region in candidates {
            // Step 2: Apply extraction rules to identify terminals
            let mut terminals = HashMap::new();
            
            for rule in &contract.extraction_rules {
                if let Some(terminal_region) = self.apply_rule(rule, region, grid) {
                    terminals.insert(rule.terminal_name.clone(), terminal_region);
                }
            }
            
            // Step 3: Verify all required terminals found
            if terminals.len() == contract.terminals.len() {
                // Step 4: Validate physics constraints
                if self.validate_physics_check(region, &contract.physics_check, grid)? {
                    devices.push(ExtractedDevice {
                        device_type: contract.device_type.clone(),
                        terminals,
                        geometry: region.clone(),
                        spice_model: contract.spice_model.clone(),
                    });
                }
            } else {
                return Err(ExtractionError::IncompleteDevice {
                    device_type: contract.device_type.clone(),
                    found_terminals: terminals.keys().cloned().collect(),
                    required_terminals: contract.terminals.clone(),
                    location: region.center(),
                });
            }
        }
        
        Ok(devices)
    }
    
    fn validate_physics_check(
        &self,
        region: &Region3D,
        physics_check: &HashMap<String, MaterialConstraint>,
        grid: &VoxelGrid,
    ) -> Result<bool, ExtractionError> {
        for (terminal, constraint) in physics_check {
            let material = grid.get_material_in_region(region, terminal)?;
            
            match constraint {
                MaterialConstraint::MustBe(allowed_materials) => {
                    if !allowed_materials.contains(&material) {
                        return Err(ExtractionError::PhysicsViolation {
                            terminal: terminal.clone(),
                            expected: allowed_materials.clone(),
                            found: material,
                        });
                    }
                }
                MaterialConstraint::MustNotBe(forbidden_materials) => {
                    if forbidden_materials.contains(&material) {
                        return Err(ExtractionError::PhysicsViolation {
                            terminal: terminal.clone(),
                            forbidden: forbidden_materials.clone(),
                            found: material,
                        });
                    }
                }
                MaterialConstraint::MustHaveProperty { property, value } => {
                    let actual_value = material.get_property(property)?;
                    if (actual_value - value).abs() > 0.01 {
                        return Err(ExtractionError::PropertyMismatch {
                            terminal: terminal.clone(),
                            property: property.clone(),
                            expected: *value,
                            found: actual_value,
                        });
                    }
                }
            }
        }
        
        Ok(true)
    }
}
```

### Error Messages with Contracts

**Before Device Contracts**:
```
Error: Device extraction failed
  Location: [x: 2.5mm, y: 3.1mm, z: 1]
  
  (No additional context - user must manually debug)
```

**After Device Contracts**:
```
❌ Physics Error: NMOS device contract violation

  ┌─ inverter.hw:45:5
  │
45│     add pour(Silicon_P) named M1_Source on z:6:
  │     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Invalid material for NMOS source
  │
  = note: NMOS source terminal must be N-doped silicon
  = found: Silicon_P (P-doped)
  = expected: Silicon_N (N-doped)
  = help: Change material to Silicon_N or use PMOS instead
  = contract: @std/foundry/contracts.hw::NMOS
```

---

## Part 2: The Historical Mirror - Software's Transformation

### The Parallel: GUI Tools → Text-Based Compilers

Hardware Script's impact on EDA mirrors what happened in software development 30-40 years ago.

#### The Software Industry Transformation

**Before (1970s-1980s)**:
- **GUI-Based IDEs**: Visual Basic, Delphi, HyperCard
- **Drag-and-Drop Programming**: Connect components visually
- **Proprietary Formats**: Binary project files, vendor lock-in
- **Limited Collaboration**: Hard to version control, hard to merge
- **Slow Iteration**: Compile-test-debug cycle measured in minutes

**After (1990s-2000s)**:
- **Text-Based Languages**: C, C++, Java, Python
- **Plain Text Source**: Human-readable, diff-able, merge-able
- **Open Standards**: Standard file formats, cross-platform tools
- **Version Control**: Git, SVN - track every change
- **Fast Iteration**: Edit-compile-run cycle measured in seconds
- **AI-Friendly**: LLMs can read and write code

**The Result**: Software development velocity increased 100×

#### The EDA Industry Today (2024)

**Current State** (Stuck in the 1980s):
- **GUI-Based Tools**: Cadence Virtuoso, Altium Designer, KiCad
- **Schematic Capture**: Draw circuits visually with mouse
- **Proprietary Formats**: Binary files (.brd, .dsn, .odb++)
- **Limited Collaboration**: Hard to diff, impossible to merge
- **Slow Iteration**: Design-verify-fabricate cycle measured in months
- **AI-Hostile**: LLMs cannot read binary schematic files

**The Problem**: EDA tools are where software development was in 1985.

### The Transformation Hardware Script Enables

**Hardware Script (2026+)**:
- **Text-Based Language**: Human-readable `.hw` files
- **Declarative Syntax**: Describe intent, compiler handles implementation
- **Open Standards**: Plain text, universal export formats (GDSII, Gerber, SPICE)
- **Version Control**: Git-native, every change tracked
- **Fast Iteration**: Edit-compile-export cycle measured in seconds
- **AI-Native**: LLMs can read and write Hardware Script

**The Result**: Hardware development velocity will increase 100×

### The Analogy Table

| Aspect | Software (1985) | Software (2025) | EDA (2024) | Hardware Script (2026+) |
|--------|-----------------|-----------------|------------|-------------------------|
| **Primary Interface** | GUI (Visual Basic) | Text (VS Code) | GUI (Cadence) | Text (Hardware Script) |
| **File Format** | Binary (.frm, .vbp) | Plain Text (.c, .py) | Binary (.brd, .dsn) | Plain Text (.hw) |
| **Version Control** | Difficult | Git-native | Difficult | Git-native |
| **Collaboration** | Email files | Pull requests | Email files | Pull requests |
| **AI Integration** | Impossible | Native (Copilot) | Impossible | Native (LLM-friendly) |
| **Iteration Speed** | Minutes | Seconds | Months | Seconds |
| **Portability** | Vendor lock-in | Cross-platform | Vendor lock-in | Universal exports |
| **Learning Curve** | Steep (GUI-specific) | Moderate (language) | Steep (GUI-specific) | Moderate (language) |

### The Tipping Point

**In software**, the tipping point was when:
- **Linus Torvalds** built Linux using text-based tools and Git
- **Open source** exploded because code was shareable and merge-able
- **GitHub** made collaboration trivial
- **Stack Overflow** made knowledge searchable
- **AI Copilot** made coding 10× faster

**In hardware**, the tipping point will be when:
- **Hardware Script** makes designs text-based and Git-native
- **HPM (Hardware Package Manager)** makes IP blocks shareable
- **GitHub for Hardware** makes collaboration trivial
- **Hardware Stack Overflow** makes knowledge searchable
- **AI Hardware Copilot** makes design 10× faster

**We are at the inflection point.**

---

## Part 3: The Industry Impact - Why This Is a Superweapon

### The Fragmented EDA Landscape

**Current State**: The EDA industry is fragmented across dozens of incompatible tools:

**For PCB Design**:
- Altium Designer ($7,000/year)
- Cadence OrCAD ($5,000/year)
- Mentor Graphics PADS ($4,000/year)
- KiCad (Free, but limited)
- Eagle (Acquired by Autodesk, now Fusion 360)

**For IC Design**:
- Cadence Virtuoso ($50,000+/year)
- Synopsys Custom Compiler ($40,000+/year)
- Mentor Graphics Pyxis ($30,000+/year)

**For Simulation**:
- SPICE (Dozens of variants: HSPICE, LTspice, ngspice)
- Verilog simulators (ModelSim, VCS, Icarus)
- MATLAB/Simulink ($2,000+/year)

**The Problem**: Each tool has its own:
- File format (incompatible)
- Scripting language (Tcl, Python, SKILL, Perl)
- Workflow (different for each vendor)
- License model (expensive, node-locked)

**The Result**: Engineers spend more time fighting tools than designing hardware.

### The Hardware Script Unification

**One Language, Universal Output**:

```hw
# Single .hw file
space MyDesign:
    # ... design in Hardware Script ...
```

**Compiler Outputs**:
- `mydesign.gds` → GDSII for any silicon foundry (TSMC, Samsung, Intel)
- `mydesign.gerber` → Gerber for any PCB fab (JLCPCB, PCBWay, OSH Park)
- `mydesign.spice` → SPICE for any simulator (HSPICE, LTspice, ngspice)
- `mydesign.glb` → 3D visualization for any viewer (Blender, Three.js)
- `mydesign.bom` → Bill of Materials for any assembler

**The Impact**: Hardware Script becomes the "universal translator" for hardware design.

### The Cost Disruption

**Traditional EDA Stack** (per engineer per year):
- Cadence Virtuoso: $50,000
- Altium Designer: $7,000
- MATLAB: $2,000
- **Total**: $59,000/year

**Hardware Script Stack**:
- Compiler: Free (open source)
- Standard Library: Free (community-maintained)
- HPM Packages: Free or paid (marketplace)
- **Total**: $0 - $1,000/year

**The Disruption**: 50-100× cost reduction

### The Velocity Multiplier

**Traditional Workflow** (CMOS Inverter):
1. Open Cadence Virtuoso (2 minutes - license checkout)
2. Draw schematic with mouse (10 minutes)
3. Create symbol (5 minutes)
4. Draw layout with mouse (60 minutes)
5. Run DRC (5 minutes)
6. Run LVS (5 minutes)
7. Fix errors, repeat (30 minutes)
8. Export GDSII (2 minutes)
9. **Total**: ~2 hours

**Hardware Script Workflow** (CMOS Inverter):
1. Write 15 lines of Hardware Script (5 minutes)
2. Run `hwc build inverter.hw` (1 second)
3. Compiler outputs GDSII, SPICE, 3D model
4. **Total**: ~5 minutes

**The Multiplier**: 24× faster iteration

### The AI Amplification

**Traditional Tools** (AI-Hostile):
- Binary file formats → LLMs cannot read
- GUI-based → LLMs cannot interact
- Proprietary → LLMs cannot learn from examples

**Hardware Script** (AI-Native):
- Plain text → LLMs can read and write
- Declarative syntax → LLMs understand intent
- Open source → LLMs trained on millions of designs

**The Amplification**: AI can now design hardware

**Example AI Workflow**:
```
Human: "Design a CMOS inverter optimized for low power"

AI: *generates Hardware Script*

module Inverter_LowPower:
    input: VIN
    output: VOUT
    power: VDD
    ground: GND

space Inverter_LowPower_Layout implements Inverter_LowPower:
    # AI-optimized for minimum leakage
    add NMOS named M1:
        W: 360nm  # Minimum width for low leakage
        L: 180nm  # Standard length
    
    add PMOS named M2:
        W: 720nm  # 2× NMOS for balanced drive
        L: 180nm
    
    # ... rest of layout ...

Compiler: ✅ Design complete
          Power: 0.8µW @ 1.8V (excellent)
          Area: 0.45µm² (compact)
          Delay: 45ps (acceptable)
```

**The Result**: Hardware design becomes as fast as software development.

---

## Part 4: The Device Contract Ecosystem

### Foundry-Published Contracts

**The Vision**: Foundries publish official device contracts for their process nodes.

**Example: TSMC 5nm**:
```hw
# @tsmc/5nm/contracts.hw

device NMOS_5nm:
    terminals: [gate, source, drain, bulk]
    
    physics_check:
        gate: must_be [TiN, TaN]  # Metal gate
        source: must_be [Silicon_N_FinFET]
        drain: must_be [Silicon_N_FinFET]
        bulk: must_be [Silicon_P_Substrate]
        gate_dielectric: must_be [HfO2]  # High-k dielectric
    
    design_rules:
        min_gate_length: 5nm
        min_gate_width: 20nm
        min_spacing: 10nm
    
    extraction_rules:
        # FinFET-specific geometry
        gate: 
            material = TiN
            wraps_around fin
            height: 50nm
        
        source: 
            material = Silicon_N_FinFET
            fin_structure: true
            adjacent_to gate
        
        drain: 
            material = Silicon_N_FinFET
            fin_structure: true
            opposite_side_of source
    
    spice_model:
        type: nmos_finfet
        model_card: "NMOS_TSMC5"
        parameters: [NFIN, L, W, AS, AD, PS, PD]
```

**Usage**:
```hw
import NMOS_5nm from @tsmc/5nm/contracts

space MyChip:
    profile: TSMC_5nm
    
    add NMOS_5nm named M1 at [x: 1um, y: 1um]
    # Compiler validates against TSMC 5nm contract
```

### Research-Published Contracts

**The Vision**: Researchers publish contracts for novel devices.

**Example: Graphene FET**:
```hw
# @research/graphene/contracts.hw

device GrapheneFET:
    terminals: [gate, source, drain]
    
    physics_check:
        channel: must_be [Graphene]
        gate: must_be [Graphene]
        dielectric: must_be [hBN, SiO2]
    
    extraction_rules:
        gate: 
            material = Graphene
            layer: top
            separated_by dielectric
        
        source: 
            material = Graphene
            layer: bottom
            contact: left_edge
        
        drain: 
            material = Graphene
            layer: bottom
            contact: right_edge
    
    spice_model:
        type: gfet
        model_card: "GRAPHENE_FET_MIT_2025"
        parameters: [W, L, Vgs, Vds, mobility]
```

**Impact**: Researchers can share novel devices without waiting for EDA vendors.

### Community-Published Contracts

**The Vision**: The community publishes contracts for common patterns.

**Example: Standard Cells**:
```hw
# @community/standard_cells/contracts.hw

device NAND_Gate:
    terminals: [A, B, OUT, VDD, GND]
    
    composition:
        # NAND is composed of 4 transistors
        add PMOS named M1  # Pull-up network
        add PMOS named M2  # Pull-up network
        add NMOS named M3  # Pull-down network
        add NMOS named M4  # Pull-down network
    
    connectivity:
        # PMOS in parallel
        route M1.source to VDD
        route M2.source to VDD
        route M1.drain to OUT
        route M2.drain to OUT
        
        # NMOS in series
        route M3.drain to OUT
        route M3.source to INTERNAL
        route M4.drain to INTERNAL
        route M4.source to GND
        
        # Gates
        route M1.gate to M3.gate to A
        route M2.gate to M4.gate to B
    
    validation:
        truth_table:
            [0, 0]: 1
            [0, 1]: 1
            [1, 0]: 1
            [1, 1]: 0
```

**Impact**: Standard cell libraries become shareable and verifiable.

---

## Part 5: The Competitive Moat

### Why Legacy EDA Cannot Compete

**The Innovator's Dilemma**:

1. **Legacy Codebase**: 30+ years of C++ code, millions of lines
2. **Binary Formats**: Entire ecosystem built on proprietary formats
3. **GUI Dependency**: Workflows assume mouse-driven interaction
4. **License Model**: Revenue depends on expensive per-seat licenses
5. **Customer Lock-In**: Customers have decades of designs in proprietary formats

**The Problem**: Legacy EDA vendors cannot pivot to text-based without:
- Rewriting entire codebase (5-10 years, $100M+)
- Abandoning proprietary formats (loses lock-in)
- Cannibalizing license revenue (loses profit)
- Retraining entire customer base (loses customers)

**The Result**: Legacy vendors are trapped. They cannot compete.

### Hardware Script's Advantages

**Clean-Room Design**:
- No legacy code to maintain
- No proprietary formats to support
- No GUI baggage to carry
- No license model to protect

**Modern Architecture**:
- Rust (memory-safe, fast, concurrent)
- Voxel grid (universal 3D representation)
- Text-based (Git-native, AI-friendly)
- Open standards (GDSII, Gerber, SPICE)

**Network Effects**:
- Open source → Community contributions
- HPM → Shared IP blocks
- Device contracts → Foundry partnerships
- AI-native → LLM integration

**The Moat**: Once Hardware Script reaches critical mass, it becomes the de facto standard.

### The Adoption Curve

**Phase 1: Early Adopters** (2026-2027)
- Hobbyists and makers
- University researchers
- Startups without legacy designs
- Open source hardware projects

**Phase 2: Professional Adoption** (2027-2028)
- Small design houses
- Consultants and contractors
- Companies with new projects
- AI-driven design automation

**Phase 3: Enterprise Migration** (2028-2030)
- Large semiconductor companies
- Automotive and aerospace
- Consumer electronics
- Government and defense

**Phase 4: Industry Standard** (2030+)
- Legacy tools deprecated
- Hardware Script taught in universities
- Foundries require Hardware Script for tape-out
- AI designs 90% of new chips

**The Timeline**: 5-10 years to industry dominance

---

## Part 6: Conclusion - The Interface for Atoms

### The Fundamental Insight

**Device Contracts are to hardware what Interfaces are to software.**

They provide:
1. **Abstraction**: Separate intent from implementation
2. **Validation**: Compiler can verify correctness
3. **Extensibility**: New devices without compiler changes
4. **Portability**: Same design, different foundries
5. **Collaboration**: Shared understanding of device behavior

### The Transformation

**Before Hardware Script**:
- Hardware design is GUI-driven, slow, expensive
- Each tool has its own format, workflow, license
- Collaboration is difficult, version control is impossible
- AI cannot help because files are binary
- Innovation is slow because tools are rigid

**After Hardware Script**:
- Hardware design is text-driven, fast, cheap
- One language, universal outputs, open source
- Collaboration is trivial, version control is native
- AI can design hardware because files are text
- Innovation is fast because language is extensible

### The Vision

**Hardware Script will do for hardware what C did for software.**

- **C** made software portable across platforms
- **Hardware Script** makes hardware portable across foundries

- **C** enabled the open source movement
- **Hardware Script** enables the open hardware movement

- **C** made software development accessible
- **Hardware Script** makes hardware development accessible

- **C** powered the software revolution
- **Hardware Script** will power the hardware revolution

### The Call to Action

**For Foundries**: Publish device contracts for your process nodes. Enable your customers to design faster.

**For Researchers**: Publish device contracts for novel devices. Share your innovations with the world.

**For Engineers**: Learn Hardware Script. Be part of the transformation.

**For Companies**: Adopt Hardware Script. Gain 100× velocity advantage over competitors.

**For Investors**: Fund Hardware Script ecosystem. This is the next platform shift.

---

**Document Status**: Strategic Industry Analysis  
**Purpose**: Understand Device Contracts and Hardware Script's transformative impact  
**Last Updated**: April 2026  
**Part of**: Hardware Script v0.1.6 Documentation Suite

**Key Insights**:
- Device Contracts are the "Interface for Atoms"
- Hardware Script mirrors software's GUI → Text transformation
- Legacy EDA cannot compete (Innovator's Dilemma)
- 100× velocity increase, 50× cost reduction
- AI-native design becomes possible
- 5-10 year timeline to industry dominance

