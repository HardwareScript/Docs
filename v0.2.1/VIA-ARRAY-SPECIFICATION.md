# Via Arrays & Contact Matrices Specification

**Document Type:** Implementation Blueprint (Test-Driven Development)  
**Target Version:** v0.2.2 (Opt-In Feature)  
**Status:** ⏸️ **APPROVED - NOT YET NEEDED**  
**Implementation Trigger:** Wide resistors (W ≥ 5µm) OR MOSFET transistors  
**Philosophy:** Test-first, no bloat. Add only when testable.  
**Date:** 2026-08-11

---

## Executive Summary

Via arrays are **NOT needed yet** for current milestone components (1µm resistors, simple diodes, voltage dividers). This specification defines the exact moment via arrays become testable and mandatory, plus the complete implementation blueprint.

**DO NOT IMPLEMENT** until you reach one of these milestones:
- **Milestone A:** Wide precision resistors or MIM capacitors (W ≥ 5µm)
- **Milestone B:** MOSFET transistors (NMOS/PMOS/CMOS inverters)

**When that moment arrives:** Open this document, implement the specification in ~2 hours, and immediately test with the provided test cases.

---

## Part 1: The Exact Trigger Conditions

### When Via Arrays Become Mandatory

```
┌─────────────────────────────────────────┐   ┌─────────────────────────────────────────┐
│ MILESTONE A: Wide Precision Resistors   │   │ MILESTONE B: MOSFET Transistors         │
│ or MIM Capacitors (W ≥ 5µm)            │ OR│ (NMOS / PMOS / CMOS Inverter)         │
├─────────────────────────────────────────┤   ├─────────────────────────────────────────┤
│ • A 10µm wide resistor head MUST have   │   │ • Source and Drain regions are wide    │
│   15+ contacts across the width         │   │   silicon fingers (W = 5µm)             │
│ • Prevents contact resistance and       │   │ • A single 170nm via fails DRC and     │
│   current crowding (P21 DRC rule)       │   │   looks physically broken in 3D         │
│ • Single via = 362Ω bottleneck          │   │ • Foundries require 5-10 contacts      │
└─────────────────────────────────────────┘   └─────────────────────────────────────────┘
```

### Why Not Needed Yet

**Current Components (v0.2.1):**
- 1µm × 1µm resistors → Single 170nm via is sufficient
- Simple diodes → Single contact per terminal
- Small capacitors → Single via handles low current

**Single via resistance:** 362Ω (acceptable for µA currents)  
**Contact area:** 0.023 µm² (sufficient for small geometry)

### Trigger Milestone A: Wide Precision Resistors (W ≥ 5µm)

**Problem Scenario:**
```
Wide 10µm Resistor Head with Single Via:

     10.0µm wide contact pad
  ┌─────────────────────────────┐
  │         Aluminum            │
  │                             │
  │           ⚫                 │  ← Single 170nm via (WRONG!)
  │         (via)               │     Creates 362Ω bottleneck
  │                             │     Looks ridiculous in 3D
  └─────────────────────────────┘
```

**Solution with Via Array:**
```
Wide 10µm Resistor Head with 2×12 Via Grid:

     10.0µm wide contact pad
  ┌─────────────────────────────┐
  │ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ │  ← 24 vias in parallel
  │ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ ⚫ │     R_eff = 362Ω / 24 = 15Ω
  └─────────────────────────────┘     DRC clean, professional
```

**Immediate Test:** Create `wide_resistor_array_test.hw`, run `hwc build`, see 24 vias in GLB/DXF/SPICE.

### Trigger Milestone B: MOSFET Transistors

**Problem Scenario:**
```
NMOS Transistor Drain (5µm wide) with Single Via:

     5.0µm wide drain region
  ┌─────────────────────────┐
  │   Silicon_N (drain)     │
  │                         │
  │          ⚫              │  ← Single via (WRONG!)
  │        (via)            │     Foundry DRC violation
  │                         │     Poor current distribution
  └─────────────────────────┘
```

**Solution with Via Array:**
```
NMOS Transistor Drain with 2×5 Via Column:

     5.0µm wide drain region
  ┌─────────────────────────┐
  │ ⚫ ⚫ ⚫ ⚫ ⚫              │  ← 10 vias distribute current
  │ ⚫ ⚫ ⚫ ⚫ ⚫              │     Meets foundry requirements
  └─────────────────────────┘     Clean 3D rendering
```

---

## Part 2: Syntax Specification

### Form 1: Explicit Matrix Grid (`matrix:`)

Used when the designer wants **exact control** over row/column layout and pitch.

```hardware
# 2 rows × 5 columns of Tungsten vias at 300nm pitch
add contact(Tungsten) named VDD_Power_Vias spanning layer: li1 to metal1:
    matrix: [rows: 2, cols: 5, pitch_x: 300nm, pitch_y: 300nm]
    at: [x: 10.0um, y: 5.0um]
    net: VDD
```

**Parameters:**
- `rows`: Number of via rows (positive integer)
- `cols`: Number of via columns (positive integer)
- `pitch_x`: Horizontal spacing between via centers
- `pitch_y`: Vertical spacing between via centers
- `pitch`: Shorthand for `pitch_x: X, pitch_y: X` (same in both directions)

**Grid Layout:**
```
     pitch_x
    <------->
  ⚫ ⚫ ⚫ ⚫ ⚫   ^
                |  pitch_y
  ⚫ ⚫ ⚫ ⚫ ⚫   v

  Center point at: [x: 10.0um, y: 5.0um]
```

### Form 2: PDK Boundary Auto-Fill (`fill:`)

Used when the compiler should **automatically calculate** the maximum number of vias that fit within a boundary using PDK DRC rules.

```hardware
# Fills the 10.0um × 2.0um pad boundary automatically
add contact(Tungsten) named Resistor_Head_Vias spanning layer: li1 to metal1:
    fill: Contact_A_LI.boundary
    spacing: 400nm              # Optional override (defaults to PDK via.min_spacing)
    net: In
```

**Parameters:**
- `fill`: Target boundary entity name (e.g., `Contact_A_LI.boundary`)
- `spacing`: Optional DRC spacing override (defaults to `profile.via.min_spacing`)

**Auto-Fill Algorithm:**
1. Query target entity bounding box
2. Get PDK DRC rules: `via.min_spacing`, `via.min_annular_ring`
3. Calculate usable area (bbox - 2×annular_ring)
4. Calculate maximum rows/cols that fit
5. Unroll to explicit matrix with calculated dimensions

---

## Part 3: AST & Parser Implementation

### 3.1 AST Data Structures

**File:** `hwc-parser/src/ast/contact.rs`

```rust
use serde::{Deserialize, Serialize};
use compact_str::CompactString;

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ViaArraySpec {
    /// Explicit matrix grid (rows × cols with fixed pitches)
    ExplicitMatrix {
        rows: u32,
        cols: u32,
        pitch_x_pm: i64,  // Picometers
        pitch_y_pm: i64,  // Picometers
    },
    
    /// Automatic boundary fill using PDK DRC rules
    BoundaryFill {
        target_boundary_name: CompactString,
        spacing_override_pm: Option<i64>,  // Override PDK min_spacing if present
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContactPlacement {
    pub name: CompactString,
    pub material_name: CompactString,
    pub start_layer: CompactString,
    pub end_layer: CompactString,
    pub diameter_pm: i64,
    pub position: Option<Coordinate>,
    pub net_name: Option<CompactString>,
    
    /// NEW (v0.2.2): Optional Via Array / Matrix specification
    /// If None, compiler treats as a standard 1×1 single via
    pub array_spec: Option<ViaArraySpec>,
}
```

### 3.2 Parser Implementation

**File:** `hwc-parser/src/parser/definitions/contact.rs`

```rust
// Inside parse_contact_placement() function

let mut array_spec: Option<ViaArraySpec> = None;

// Parse optional matrix: [...] block
if self.check_identifier("matrix") {
    self.advance();
    self.expect(&Token::Colon)?;
    self.expect(&Token::OpenBracket)?;
    
    let mut rows = 1u32;
    let mut cols = 1u32;
    let mut pitch_x_pm = 300_000i64;  // 300nm default
    let mut pitch_y_pm = 300_000i64;
    
    while !self.check(&Token::CloseBracket) {
        let key = self.expect_identifier()?;
        self.expect(&Token::Colon)?;
        
        match key.as_str() {
            "rows" => rows = self.parse_integer()? as u32,
            "cols" => cols = self.parse_integer()? as u32,
            "pitch" => {
                let p = self.parse_measurement_to_pm()?;
                pitch_x_pm = p;
                pitch_y_pm = p;
            },
            "pitch_x" => pitch_x_pm = self.parse_measurement_to_pm()?,
            "pitch_y" => pitch_y_pm = self.parse_measurement_to_pm()?,
            _ => return Err(ParseError::UnknownMatrixProperty(key)),
        }
        
        if self.check(&Token::Comma) { 
            self.advance(); 
        }
    }
    
    self.expect(&Token::CloseBracket)?;
    
    array_spec = Some(ViaArraySpec::ExplicitMatrix {
        rows,
        cols,
        pitch_x_pm,
        pitch_y_pm,
    });
}

// Parse optional fill: <boundary> block
else if self.check_identifier("fill") {
    self.advance();
    self.expect(&Token::Colon)?;
    
    let target_boundary_name = self.parse_expression_string()?;
    let mut spacing_override_pm = None;
    
    if self.check_identifier("spacing") {
        self.advance();
        self.expect(&Token::Colon)?;
        spacing_override_pm = Some(self.parse_measurement_to_pm()?);
    }
    
    array_spec = Some(ViaArraySpec::BoundaryFill {
        target_boundary_name: CompactString::from(target_boundary_name),
        spacing_override_pm,
    });
}

// ... continue with rest of contact parsing
```

### 3.3 Validation Rules

**Compile-time checks:**
1. `rows` and `cols` must be positive integers (> 0)
2. `pitch_x` and `pitch_y` must be ≥ `via.min_spacing` from PDK profile
3. `fill:` target entity must exist in symbol table
4. `matrix:` and `fill:` are mutually exclusive (cannot use both)

---

## Part 4: Compiler Unrolling Implementation

### 4.1 Unrolling Architecture

The via array unroller operates during the **semantic lowering pass** in `space_builder.rs`. It converts a single `ViaArraySpec` into N × M discrete, picometer-precise `ContactPlacement` entries in the `EntityGraph`.

**Critical Design Principle:** Unrolling happens **early** in the compilation pipeline, so downstream subsystems (DRC, routing, meshing, SPICE extraction) see N × M individual vias and require **zero changes**.

```
Input:  1 ContactPlacement with array_spec
         ↓
Unroller: space_builder.rs
         ↓
Output: N × M ContactPlacement entries (array_spec = None)
         ↓
Downstream subsystems see discrete vias (no changes needed!)
```

### 4.2 Unrolling Algorithm

**File:** `hwc-compiler/src/ir/space_builder.rs`

```rust
pub fn unroll_via_placement(
    contact: &ContactPlacement,
    symbol_table: &SymbolTable,
    profile: &ProfileDef,
) -> Result<Vec<ContactPlacement>, CompileError> {
    let mut unrolled_vias = Vec::new();
    
    match &contact.array_spec {
        // Standard 1×1 single via - pass through unchanged
        None => {
            unrolled_vias.push(contact.clone());
        }
        
        // Explicit matrix: unroll to rows × cols grid
        Some(ViaArraySpec::ExplicitMatrix { 
            rows, 
            cols, 
            pitch_x_pm, 
            pitch_y_pm 
        }) => {
            let center_pos = contact.position.as_ref()
                .ok_or(CompileError::MissingPosition(contact.name.clone()))?;
            
            // Calculate starting top-left offset to keep array centered
            let total_width_pm = (*cols as i64 - 1) * pitch_x_pm;
            let total_height_pm = (*rows as i64 - 1) * pitch_y_pm;
            let start_x_pm = center_pos.x_pm - (total_width_pm / 2);
            let start_y_pm = center_pos.y_pm - (total_height_pm / 2);
            
            // Generate rows × cols discrete vias
            for r in 0..*rows {
                for c in 0..*cols {
                    let mut single_via = contact.clone();
                    single_via.name = format!("{}_{}_{}", contact.name, r, c).into();
                    single_via.array_spec = None;  // Unrolled to single via
                    single_via.position = Some(Coordinate {
                        x_pm: start_x_pm + (c as i64 * pitch_x_pm),
                        y_pm: start_y_pm + (r as i64 * pitch_y_pm),
                        z_pm: center_pos.z_pm,
                    });
                    unrolled_vias.push(single_via);
                }
            }
        }
        
        // Boundary fill: calculate optimal grid, delegate to explicit matrix
        Some(ViaArraySpec::BoundaryFill { 
            target_boundary_name, 
            spacing_override_pm 
        }) => {
            // Get target entity bounding box
            let target_bbox = symbol_table.get_entity_bbox(target_boundary_name)?;
            
            // Get DRC parameters from PDK profile
            let min_spacing_pm = spacing_override_pm
                .unwrap_or(profile.via.min_spacing_pm);
            let via_dia_pm = contact.diameter_pm;
            let enclosure_pm = profile.via.min_annular_ring_pm;
            
            // Calculate usable inner area respecting DRC enclosure
            let usable_width_pm = (target_bbox.max_x_pm - target_bbox.min_x_pm) 
                - (2 * enclosure_pm);
            let usable_height_pm = (target_bbox.max_y_pm - target_bbox.min_y_pm) 
                - (2 * enclosure_pm);
            
            // Calculate max rows and cols that fit without DRC violation
            let pitch_pm = via_dia_pm + min_spacing_pm;
            let cols = ((usable_width_pm + min_spacing_pm) / pitch_pm).max(1) as u32;
            let rows = ((usable_height_pm + min_spacing_pm) / pitch_pm).max(1) as u32;
            
            // Delegate to explicit matrix unrolling
            let explicit = ViaArraySpec::ExplicitMatrix {
                rows,
                cols,
                pitch_x_pm: pitch_pm,
                pitch_y_pm: pitch_pm,
            };
            
            let mut temp_contact = contact.clone();
            temp_contact.array_spec = Some(explicit);
            temp_contact.position = Some(Coordinate {
                x_pm: (target_bbox.min_x_pm + target_bbox.max_x_pm) / 2,
                y_pm: (target_bbox.min_y_pm + target_bbox.max_y_pm) / 2,
                z_pm: target_bbox.min_z_pm,
            });
            
            return unroll_via_placement(&temp_contact, symbol_table, profile);
        }
    }
    
    Ok(unrolled_vias)
}
```

### 4.3 Integration Point

**File:** `hwc-compiler/src/ir/space_builder.rs` (existing function)

```rust
// Inside build_space() or process_contact_placement()

for contact in &ast_space.contacts {
    let unrolled = unroll_via_placement(contact, &symbol_table, &profile)?;
    
    for via in unrolled {
        entity_graph.add_contact(via);
    }
}
```


---

## Part 5: Downstream Subsystem Integration

### 5.1 Zero Changes Required

Because unrolling happens **early** in `space_builder.rs`, downstream subsystems see N × M discrete vias and require **zero modifications**.

```
┌─────────────────────────────────────────────────────────────┐
│ 1. UNROLLER (space_builder.rs)                              │
│    Unrolls 1 ViaArraySpec into N × M discrete vias          │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼ N × M Vias in EntityGraph
┌─────────────────────────────────────────────────────────────┐
│ 2. SPATIAL INDEX & SIMD DRC (hwc-physics)                   │
│    • Registers N × M via bboxes in rstar R-tree             │
│    • Checks via-to-via min_spacing (400nm) automatically    │
│    • No code changes - sees individual vias                 │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. COPPER WELDER (hwc-export/src/substrate.rs)              │
│    • Clipper2 Non-Zero Winding merge                        │
│    • All via landing pads on metal1 merge into single pool  │
│    • No code changes - processes individual via boundaries  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. SPICE PARASITIC EXTRACTOR (hwc-export/src/netlist.rs)    │
│    • Extracts N parallel resistors and inductances          │
│    • R_eff = R_via / N,  L_eff = L_via / N                  │
│    • No code changes - sees individual vias on same net     │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Automatic Behavior

**DRC Engine (hwc-physics):**
- Registers each via as separate bounding box in R-tree spatial index
- Via-to-via clearance checks happen automatically
- If pitch < `via.min_spacing`, DRC error is triggered

**3D Mesh Export (hwc-export/src/scene_graph):**
- Each via becomes a cylinder mesh in GLB output
- 2×12 grid renders as 24 individual cylinders
- No special array handling needed

**DXF Export (hwc-export/src/dxf.rs):**
- Each via exports as individual CIRCLE or POLYLINE entity
- 24 vias = 24 DXF entities on via layer
- No special array handling needed

**SPICE Extraction (hwc-export/src/netlist/parasitics.rs):**
- Each via contributes resistance: R_via = 362Ω
- N vias in parallel on same net: R_eff = R_via / N
- Parallel resistance calculation already exists (no changes)

---

## Part 6: Complete Test Case

### 6.1 Wide Resistor Test

**File:** `hwc/tests/Wide-Resistor-Basics/wide_resistor_array_test.hw`

```hardware
# wide_resistor_array_test.hw - SKY130 Wide Resistor with 2×12 Via Array
import * from @std/primitives/units
import * from "./resistor_pdk"

device Resistor:
    terminals: [A, B, BULK]
    materials:
        A: Polysilicon
        B: Polysilicon
        BULK: Air

module WideResistorSystem:
    pins: [input In, output Out]
    route In to Out

space Wide_Resistor_Space implements WideResistorSystem:
    dimensions: 30.0um by 15.0um by 1.0um
    resolution: 10nm
    profile: Resistor_3D
    
    nets:
        In:  { classification: signal, potential: 1.8V, current: 10mA }
        Out: { classification: signal, potential: 0.0V, current: 10mA }
        GND: { classification: ground, potential: 0.0V, current: 0.0uA }
    
    device_nets R1:
        BULK: GND
    
    # ========================================================================
    # WIDE RESISTOR BODY (10.0μm wide × 4.0μm long)
    # ========================================================================
    
    add pour(Polysilicon) named Resistor_Body on layer: polyres:
        device: R1.A, R1.B
        net: In
        dimensions: 4.0um by 10.0um
        at: [x: 15.0um, y: 7.5um]
    
    # ========================================================================
    # WIDE CONTACT HEAD A (10.0μm wide × 1.0μm tall)
    # ========================================================================
    
    add pour(Titanium_Silicide) named Contact_A_LI on layer: li1:
        device: R1.A
        net: In
        dimensions: 1.0um by 10.0um
        at: [x: Resistor_Body.left + 500nm, y: Resistor_Body.center_y]
    
    # ========================================================================
    # WIDE CONTACT HEAD B (10.0μm wide × 1.0μm tall)
    # ========================================================================
    
    add pour(Titanium_Silicide) named Contact_B_LI on layer: li1:
        device: R1.B
        net: Out
        dimensions: 1.0um by 10.0um
        at: [x: Resistor_Body.right - 500nm, y: Resistor_Body.center_y]
    
    # ========================================================================
    # VIA ARRAYS (AUTO-FILL): 2×12 grid of 170nm vias
    # ========================================================================
    
    # Terminal A via array: fills 10.0μm × 1.0μm contact head
    add contact(Tungsten) named Via_Array_A spanning layer: polyres to li1:
        fill: Contact_A_LI.boundary
        spacing: 400nm
        net: In
    
    # Terminal B via array: fills 10.0μm × 1.0μm contact head
    add contact(Tungsten) named Via_Array_B spanning layer: polyres to li1:
        fill: Contact_B_LI.boundary
        spacing: 400nm
        net: Out
    
    # ========================================================================
    # EXTERNAL PADS
    # ========================================================================
    
    add pour(Aluminum) named In_Pad on layer: metal1:
        net: In
        dimensions: 2.0um by 2.0um
        at: [x: 3.0um, y: 7.5um]
    
    add pour(Aluminum) named Out_Pad on layer: metal1:
        net: Out
        dimensions: 2.0um by 2.0um
        at: [x: 27.0um, y: 7.5um]
    
    # ========================================================================
    # ROUTING
    # ========================================================================
    
    route In_Pad to Contact_A_LI:
        net: In
        width: 500nm
        layer: li1
    
    route Contact_B_LI to Out_Pad:
        net: Out
        width: 500nm
        layer: li1
```

### 6.2 Expected Build Output

**Command:**
```bash
hwc build wide_resistor_array_test.hw
```

**Expected Console Output:**
```
✓ Parsing complete (12 ms)
✓ Semantic analysis complete (8 ms)
✓ Via array unrolling: Contact_A_LI → 2×12 grid (24 vias)
✓ Via array unrolling: Contact_B_LI → 2×12 grid (24 vias)
✓ DRC checks passed (24 ms) - 48 vias registered
✓ Geometry meshing complete (35 ms)
✓ SPICE extraction complete (18 ms)
✓ Build successful

Outputs:
  - build/Wide_Resistor_Space/space.glb (3D mesh)
  - build/Wide_Resistor_Space/layout.dxf (2D layout)
  - build/Wide_Resistor_Space/spice/circuit.sp (SPICE netlist)
```

### 6.3 Validation Checks

**1. 3D GLB Viewport (hsm):**
```
Expected: Two clean 2×12 grids of 24 Tungsten via pillars
          spanning the 10.0μm contact heads

Visual Check:
  ✓ 24 cylinders visible at Terminal A
  ✓ 24 cylinders visible at Terminal B
  ✓ Vias evenly distributed across 10.0μm width
  ✓ No visual gaps or crowding
```

**2. 2D DXF Viewport:**
```
Expected: 48 individual 170nm via boundary circles

Layer: Tungsten
  ✓ 24 CIRCLE entities at Terminal A (X: 13.5μm region)
  ✓ 24 CIRCLE entities at Terminal B (X: 16.5μm region)
  ✓ Regular grid spacing (400nm pitch)
```

**3. SPICE Netlist (circuit.sp):**
```spice
* Via Resistance Extraction (Terminal A)
RVia_Array_A_0_0 Contact_A_LI In 362.0
RVia_Array_A_0_1 Contact_A_LI In 362.0
... (22 more parallel resistors)

* Effective Parallel Resistance: R_eff = 362Ω / 24 = 15.08Ω

Expected:
  ✓ 24 parallel resistors for Terminal A
  ✓ 24 parallel resistors for Terminal B
  ✓ Total via resistance < 20Ω (eliminates contact bottleneck)
```

**4. Build Time:**
```
Expected: < 200ms total build time
  - Via unrolling should add < 5ms overhead
  - DRC processing 48 vias should add < 10ms
```

---

## Part 7: MOSFET Test Case

### 7.1 NMOS Transistor with Via Arrays

**File:** `hwc/tests/MOSFET-Basics/nmos_via_array_test.hw`

```hardware
# nmos_via_array_test.hw - SKY130 NMOS with Source/Drain Via Arrays
import * from @std/primitives/units
import * from "./sky130_pdk"

device NMOS:
    terminals: [gate, source, drain, bulk]
    materials:
        gate: Polysilicon
        source: Silicon_N
        drain: Silicon_N
        bulk: Silicon_P

module NMOS_Inverter:
    pins: [input Gate, output Drain, power VDD, ground VSS]

space NMOS_Space implements NMOS_Inverter:
    dimensions: 20.0um by 15.0um by 1.0um
    resolution: 10nm
    profile: SKY130_ASIC
    
    nets:
        Gate:  { classification: signal }
        Drain: { classification: signal }
        VDD:   { classification: power, potential: 1.8V }
        VSS:   { classification: ground, potential: 0.0V }
    
    device_nets M1:
        bulk: VSS
    
    # ========================================================================
    # TRANSISTOR GEOMETRY (W = 5.0μm, L = 180nm)
    # ========================================================================
    
    # Gate (Polysilicon strip crossing active region)
    add pour(Polysilicon) named M1_Gate on layer: poly:
        device: M1.gate
        net: Gate
        dimensions: 180nm by 5.0um
        at: [x: 10.0um, y: 7.5um]
    
    # Source region (Silicon_N, left side of gate)
    add pour(Silicon_N) named M1_Source on layer: nwell:
        device: M1.source
        net: VSS
        dimensions: 2.0um by 5.0um
        at: [x: 8.0um, y: 7.5um]
    
    # Drain region (Silicon_N, right side of gate)
    add pour(Silicon_N) named M1_Drain on layer: nwell:
        device: M1.drain
        net: Drain
        dimensions: 2.0um by 5.0um
        at: [x: 12.0um, y: 7.5um]
    
    # ========================================================================
    # CONTACT ARRAYS: 2×5 via columns on Source and Drain
    # ========================================================================
    
    # Source via array (2 rows × 5 columns along 5.0μm height)
    add contact(Tungsten) named Source_Vias spanning layer: nwell to metal1:
        matrix: [rows: 2, cols: 5, pitch_x: 300nm, pitch_y: 1.0um]
        at: [x: 8.0um, y: 7.5um]
        net: VSS
    
    # Drain via array (2 rows × 5 columns along 5.0μm height)
    add contact(Tungsten) named Drain_Vias spanning layer: nwell to metal1:
        matrix: [rows: 2, cols: 5, pitch_x: 300nm, pitch_y: 1.0um]
        at: [x: 12.0um, y: 7.5um]
        net: Drain
    
    # Gate contact (single via at top)
    add contact(Tungsten) named Gate_Via spanning layer: poly to metal1:
        at: [x: 10.0um, y: 10.0um]
        net: Gate
```

### 7.2 Expected MOSFET Build Output

**Console:**
```
✓ Via array unrolling: Source_Vias → 2×5 grid (10 vias)
✓ Via array unrolling: Drain_Vias → 2×5 grid (10 vias)
✓ MOSFET device extraction: M1 (NMOS, W=5.0um, L=180nm)
✓ DRC checks passed (21 vias registered)
```

**3D Visualization:**
```
Expected: Clean columnar via arrays running along the 5.0μm transistor finger

Side View:
  Gate (Poly)
      ║
  ────╬──── Active Region
      ║
  ⚫⚫⚫⚫⚫  Drain vias (right side, 2×5 grid)
  ⚫⚫⚫⚫⚫

  ⚫⚫⚫⚫⚫  Source vias (left side, 2×5 grid)
  ⚫⚫⚫⚫⚫
```

---

## Part 8: Implementation Checklist

### When You Reach Milestone A or B

**Phase 1: AST & Parser (30 minutes)**
- [ ] Add `ViaArraySpec` enum to `hwc-parser/src/ast/contact.rs`
- [ ] Add `array_spec: Option<ViaArraySpec>` to `ContactPlacement`
- [ ] Implement `matrix:` parser in `hwc-parser/src/parser/definitions/contact.rs`
- [ ] Implement `fill:` parser
- [ ] Add validation (positive rows/cols, mutually exclusive matrix/fill)
- [ ] Test: Parse `wide_resistor_array_test.hw` → AST should contain `ExplicitMatrix`

**Phase 2: Compiler Unrolling (45 minutes)**
- [ ] Implement `unroll_via_placement()` in `hwc-compiler/src/ir/space_builder.rs`
- [ ] Add explicit matrix unrolling (rows × cols grid calculation)
- [ ] Add boundary fill unrolling (bbox query + DRC calculation)
- [ ] Integrate into `build_space()` or `process_contact_placement()`
- [ ] Test: Build `wide_resistor_array_test.hw` → EntityGraph should contain 48 vias

**Phase 3: Integration Testing (45 minutes)**
- [ ] Test: DRC engine registers 48 vias without errors
- [ ] Test: GLB export shows 24 via cylinders at each terminal
- [ ] Test: DXF export contains 48 CIRCLE entities on via layer
- [ ] Test: SPICE netlist contains 48 parallel resistors
- [ ] Test: Calculate effective resistance (should be ~15Ω)
- [ ] Test: Build time < 200ms

**Phase 4: Documentation (15 minutes)**
- [ ] Mark this specification as ✅ IMPLEMENTED in header
- [ ] Update `Core-Architecture-Proposal.md` Phase 4 status
- [ ] Add via array examples to `DEVICE-DEFINITIONS.md`
- [ ] Update changelog with v0.2.2 via array support

**Total Estimated Implementation Time:** ~2 hours

---

## Part 9: Design Principles

### 9.1 Test-First Philosophy

**Do NOT implement** via arrays until:
1. You have a concrete test case that needs them
2. You can immediately verify correct behavior visually (3D/2D) and electrically (SPICE)
3. The test case represents a real milestone component (wide resistor or MOSFET)

**Why?**
- Untestable code creates technical debt
- Features without use cases add bloat
- Test-driven development prevents architectural mistakes

### 9.2 Early Unrolling Strategy

**Critical Decision:** Unroll via arrays during semantic lowering (space_builder.rs), NOT during export.

**Benefits:**
- ✅ Downstream subsystems unchanged (DRC, routing, meshing, SPICE)
- ✅ Symbol table sees discrete vias (simplifies bbox queries)
- ✅ Error messages reference individual vias (better debugging)
- ✅ Parallel processing opportunities (SIMD DRC on N vias)

**Alternative (rejected):** Unroll during DXF/GLB export
- ❌ DRC sees 1 entity, exports 48 → inconsistent state
- ❌ SPICE extraction needs special array handling
- ❌ Symbol table bbox queries incorrect

### 9.3 Zero Magic, Explicit Control

**User declares:**
- Exact row/column counts (`matrix:`)
- Exact pitch values
- OR target boundary for auto-fill (`fill:`)

**Compiler calculates:**
- Via positions (picometer-precise)
- Optimal grid for `fill:` mode using PDK DRC rules

**No hidden behavior:**
- No auto-detection of "wide" pads
- No silent insertion of vias
- User explicitly requests array → compiler unrolls

### 9.4 PDK-Driven Auto-Fill

**Boundary fill algorithm:**
1. Query target entity bounding box (exact dimensions)
2. Read PDK DRC rules: `via.min_spacing`, `via.min_annular_ring`
3. Calculate usable area (bbox - 2×annular_ring for DRC compliance)
4. Calculate maximum rows/cols that fit: `(usable_width + spacing) / (diameter + spacing)`
5. Delegate to explicit matrix unrolling

**Why this works:**
- Respects foundry DRC rules automatically
- Maximizes via count without violations
- User doesn't manually calculate grid dimensions
- Portable across PDKs (different spacing rules)

---

## Part 10: Future Extensions (v0.3.0+)

### 10.1 GDSII AREF Support

**Current Implementation (v0.2.2):** Via arrays export as N × M discrete entities

**Future Optimization:** GDSII Array Reference (AREF)

```gdsii
AREF                  # Array Reference
SNAME "Via_170nm"     # Reference cell
COLROW 12 2           # 12 columns, 2 rows
XY x0 y0 x1 y1 x2 y2  # Origin, column vector, row vector
```

**Benefits:**
- File size reduction (1 AREF vs. 48 entities)
- Faster GDSII load times in KLayout/Magic
- Industry-standard array representation

**Implementation Trigger:** When GDSII export is added (Section 1 of Core-Architecture-Proposal.md)

### 10.2 Current Density Validation

**Proposed Feature:** Automatic via count calculation based on net current

```hardware
nets:
    VDD: { classification: power, current: 100mA }  # High current power net

# Compiler calculates required via count:
# Current density limit: 1mA per via (foundry spec)
# Required vias: 100mA / 1mA = 100 vias minimum

add contact(Tungsten) named Power_Vias spanning layer: li1 to metal1:
    fill: VDD_Pad.boundary
    spacing: 400nm
    net: VDD
    # Compiler warns if calculated grid < 100 vias
```

### 10.3 Staggered Grid Patterns

**Proposed Syntax:**
```hardware
add contact(Tungsten) named Vias spanning layer: li1 to metal1:
    matrix: [rows: 2, cols: 10, pitch: 400nm]
    pattern: staggered  # Offset alternating rows by pitch/2
    net: VDD
```

**Use Case:** Higher via density without violating min_spacing

```
Regular:     Staggered:
⚫ ⚫ ⚫ ⚫      ⚫ ⚫ ⚫ ⚫
⚫ ⚫ ⚫ ⚫       ⚫ ⚫ ⚫ ⚫
```

---

## Conclusion

Via arrays are **not needed yet** for v0.2.1 milestones. This specification provides the complete blueprint for implementation when you reach:

1. **Wide precision resistors** (W ≥ 5µm) → Need multi-via contact heads
2. **MOSFET transistors** → Need via columns on source/drain regions

**At that moment:**
1. Open this document
2. Implement Phases 1-4 (~2 hours)
3. Run `wide_resistor_array_test.hw`
4. Verify 24 vias render in 3D, export to DXF, extract to SPICE
5. Mark Section 5 of Core-Architecture-Proposal.md as complete

**Test-first philosophy preserved.** No bloat. No untestable code. Implement only when you can immediately verify correctness.

---

**Document Status:** ✅ Ready for Implementation (when triggered)  
**Last Updated:** 2026-08-11  
**Author:** HardwareScript Architecture Team
