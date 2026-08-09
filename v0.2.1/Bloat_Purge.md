# HardwareScript Architecture: Bloat Purge Specification (v0.2.1+)

**Document Type:** Canonical Architectural Refactoring & Language Cleanup Guide  
**Version:** v0.2.1+  
**Status:** Approved for Core Implementation  
**Date:** August 2026  
**Focus:** Eradication of Legacy Voxel Artifacts, Token Pollution, Ghost Keywords, User-Space Anti-Patterns, and PDK Profile Heuristic Dumping  

---

## Executive Summary

HardwareScript is guided by an uncompromising philosophy: **when a feature or subsystem creates technical debt or syntax redundancy, it must be removed without sympathy.**

During early compiler stages (v0.1.2–v0.2.0), several temporary hacks, ghost keywords, and workaround patterns were introduced. As HardwareScript evolved into a $i64$ picometer continuous vector engine with Salsa-driven query memoization and Comptime Anchor Arithmetic, these early hacks mutated into **user-facing bloat and dual-authority bugs**.

This specification executes a workspace-wide **Bloat Purge** across six core categories:
1. **Space Block Purge:** Deletion of `origin:`, deletion of `resolution:`, and removal of $Z$-depth from `dimensions:`.
2. **Ghost Block Purge:** Deletion of the unused `absolute:` block wrapper.
3. **Lexer Token Purge:** Removal of 22+ hardcoded preposition/directional tokens from `Token` enum in favor of context-aware identifier parsing.
4. **User-Space Anti-Pattern Purge:** Elimination of $1\text{nm}$ `Air` dummy pours for SPICE virtual terminals, elimination of physical resistor body splitting, and replacement of 35-line pour assemblies with 1-line PCell instantiations.
5. **Material Definition Purge:** Eradication of quoted symbols (`symbol: "Poly"`) and absurd mandatory fields (`process: deposited` on `Air`).
6. **Solder Mask Hardcoding Purge:** Deletion of hardcoded solder mask fields from `StackupManager` and migration to pure data-driven stackup layers for universal PCB/ASIC/Flex/RF support.

**Note:** Routing cost weights (`via_penalty`, `direction_penalty`, `crosstalk_penalty`, `reference_void_penalty`) are **NOT bloat**. They are declarative policy parameters protected by the Zero-Magic Compiler Mandate. They must remain fully exposed in PDK `.hw` files to allow foundry-specific tuning and Connection Interface Routing (CIR) intent differentiation.

---

## Category 1: The Space Block Purge

```
                            THE SPACE BLOCK PURGE
                            
  BEFORE (v0.1.2 – v0.2.0):                 AFTER (v0.2.1+ Purged):
  space Simple_Space:                        space Simple_Space:
    dimensions: 40.0um by 20.0um by 1200nm     dimensions: 40.0um by 20.0um
    resolution: 10nm                           profile: Silicon_180nm
    origin: bl by b
    profile: Silicon_180nm
```

### 1.1 Deletion of `origin:`
* **Reason for Purge:** `origin: tl` created semantic inversions in Comptime Anchor Arithmetic (`Pad_A.top + 10um` moved *downwards* in top-left space, but *upwards* in bottom-left space), corrupted affine matrix compositions (`FixedTransform2D`), and broke cross-package module imports in `hpm`.
* **Action:** `origin:` and its associated syntax tokens are purged. All HardwareScript spaces operate on a single, canonical, immutable 3D Cartesian coordinate space ($X \ge 0, Y \ge 0, Z \ge 0$ in picometers, Bottom-Left / Z-Up).

### 1.2 Deletion of `resolution:`
* **Reason for Purge:** `resolution:` was a leftover relic of the old Voxel Grid (`grid: X by Y by Z`). Specifying `resolution: 10nm` inside `space` created a dual-authority conflict with PDK profiles (`profile:`) that declare `track_pitch` and `manufacturing_grid`.
* **Action:** `resolution:` is deleted from `space`. Internal engine computations run continuously at $1\text{ pm}$ ($10^{-12}\text{ m}$), and manufacturing track snapping is governed strictly by `profile:`.

### 1.3 Transformation of `dimensions:` ($Z$-Height Removal)
* **Reason for Purge:** Writing `dimensions: 40.0um by 20.0um by 1200nm` forced users to manually type a $Z$-height ($1200\text{nm}$) that conflicted with the actual sum of layer thicknesses in `profile.stackup` ($200\text{nm} + 300\text{nm} + 840\text{nm} = 1340\text{nm}$).
* **Action:** $Z$-depth is removed from `dimensions:`. $X$ and $Y$ remain as the physical die/board boundary footprint. The $Z$-height is derived automatically as the exact sum of stackup layers in `profile.stackup`.

---

## Category 2: The Ghost Block Purge (`absolute:`)

```
                            THE GHOST BLOCK PURGE
                            
  BEFORE (v0.2.0 Spec):                      AFTER (v0.2.1+ Purged):
  absolute:                                  add plane(Aluminum) named M1_Pad:
      add plane(Aluminum) named M1_Pad:          on layer: metal1
          on layer: metal1                       at: [x: 5um, y: 5um]
          at: [x: 5um, y: 5um]
```

### 2.1 Deletion of `absolute:`
* **Reason for Purge:** `absolute:` was written into documentation as a theoretical "escape hatch" but was **never used once in real code or test suites**. Specifying `at: [x: 5um, y: 5um]` already tells the AST that an explicit coordinate placement is requested. Adding an `absolute:` wrapper block adds an extra level of indentation and token overhead for zero functional gain.
* **Action:** `absolute:` keyword and parser block wrapper are completely removed.

---

## Category 3: The Lexer & Keyword Token Purge

To reduce DFA state machine size in `crates/hwc-parser/src/tokens.rs` and eliminate lexer collisions, hardcoded tokens for prepositions, directional shorthand, and loop state are pruned from Logos.

```
                  22+ LOGOS TOKENS PRUNED FROM TOKENS.RS
                  
  • RightOf (#[token("right_of")])     ──► Parsed as Token::Identifier
  • LeftOf  (#[token("left_of")])      ──► Parsed as Token::Identifier
  • Above   (#[token("above")])        ──► Parsed as Token::Identifier
  • Below   (#[token("below")])        ──► Parsed as Token::Identifier
  • TopLeft (#[token("tl")])           ──► Parsed as Token::Identifier
  • BottomLeft (#[token("bl")])        ──► Parsed as Token::Identifier
  • Last    (#[token("last")])         ──► DELETED (Replaced by loop index `i`)
  • Absolute (#[token("absolute")])    ──► DELETED (Redundant block wrapper)
```

### 3.1 Context-Aware Identifier Parsing
Identifiers like `above`, `below`, `right_of`, and `tl` can now be freely used as pin, signal, or net names without triggering lexer syntax errors. `hwc-parser` inspects `Token::Identifier(s)` in context inside `align:` or `region` blocks.

### 3.2 Deletion of `last`
The `last` keyword (`after: last.right + 1mm`) is deleted. Loop-based spatial positioning is expressed cleanly using standard compile-time loop index math (`base_x + i * pitch`).

---

## Category 4: The User-Space Anti-Pattern Purge

### 4.1 Elimination of $1\text{nm}$ `Air` Pours for Virtual SPICE Terminals
* **Anti-Pattern:**
  ```hardware
  # ❌ PURGED: Creating a 1nm Air box just to bind a substrate SPICE node!
  add pour(Air) named R1_Bulk on layer: polyres:
      device: R1.BULK
      net: GND
      dimensions: 1nm by 1nm
      at: [x: 9.5um, y: 1.0um]
  ```
* **Purge Fix:** Non-physical/virtual SPICE terminals (`BULK`, `SUBSTRATE`) are bound directly in the device net map without creating fake 3D geometry objects:
  ```hardware
  # ✅ CLEAN: Direct terminal net binding
  nets: { A: In1, B: Out1, BULK: GND }
  ```

### 4.2 Elimination of Physical Resistor Body Splitting
* **Anti-Pattern:** Slicing a single $10\mu\text{m} \times 10\mu\text{m}$ polysilicon resistor into two separate pours (`R1_Body_A` on `In1` and `R1_Body_B` on `Out1`) down the middle because the netlist extractor couldn't handle terminal taps on a single pour.
* **Purge Fix:** A resistor body is **ONE continuous physical pour** (`add pour(Polysilicon) named R1_Body`). Terminals $A$ and $B$ are defined by where the contacts/vias touch the single body.

### 4.3 Replacement of 35-Line Pour Assemblies with 1-Line PCells
* **Anti-Pattern:** Writing 35 lines of manual `add pour` and `add contact` statements for EVERY resistor instance ($R1, R2, R3$), resulting in 250-line test files for 3 simple components.
* **Purge Fix:** Instantiate parameterized PCell devices defined in the PDK in **1 clean line**:
  ```hardware
  # ✅ CLEAN: 1 Line per Resistor
  add Resistor_SKY130(W: 10.0um, L: 10.0um) named R1 at: [x: 7.0um, y: 3.0um]:
      nets: { A: In1, B: Out1, BULK: GND }
  ```

---

## Category 5: The Material Definition Purge

```
                      MATERIAL DEFINITION PURGE
                      
  BEFORE (v0.1.6):                          AFTER (v0.2.1+ Purged):
  export material Polysilicon:              export material Polysilicon:
      category: conductor                       category: conductor
      symbol: "Poly"                            symbol: Poly
      process: deposited                        process: deposited
      properties:                               properties:
          opacity: 1.0                              color: "#FF6060"
          resistivity: 7e-5ohm_m                    resistivity: 7e-5ohm_m
          
  export material Air:                      export material Air:
      category: insulator                       category: insulator
      symbol: "Air"                             symbol: Air
      process: deposited  # ❌ Absurd!           properties:
                                                    relative_permittivity: 1.0
```

### 5.1 Enforce Bare Identifiers
Quotes on material symbols (`symbol: "Poly"`) are purged in favor of bare identifiers (`symbol: Poly`).

### 5.2 Optional `process:` Field & Default Properties
* `process:` is made optional with sensible defaults (`deposited` for conductors/semiconductors, `ambient`/`none` for gases). This eliminates the absurdity of declaring `process: deposited` on Air!
* Default properties (e.g. `opacity: 1.0` for conductors) are implicit and do not need to be declared on every material.

---

## Category 6: The Solder Mask Hardcoding Purge

```
                  THE SOLDER MASK HARDCODING TRAP
                  
  HARDCODED RUST LOGIC IN StackupManager:
    - Automatically injects 20µm top/bottom "Green PCB Solder Mask"
    
         ┌──────────────────────┬──────────────────────┬───────────────────┐
         ▼                      ▼                      ▼
  1. BREAKS ASIC/IC      2. BREAKS FLEX PCBs     3. BREAKS RF/MICROWAVE
  ASICs use Nitride/     Flex PCBs use          RF PCBs use NO mask
  Oxide Passivation      Polyimide Coverlay     (degrades loss tangent)
  (500nm)                (Adhesive film)
```

### 7.1 Why Hardcoding Solder Mask in Rust is Toxic

By hardcoding `solder_mask_thickness_nm` inside `StackupManager` as a special Rust field, HardwareScript v0.1.7 broke four major hardware domains:

1. **Breaks ASIC / Semiconductor Design:** Silicon chips do not have "green PCB solder mask." They have Nitride/Oxide Passivation layers (300nm - 500nm) with pad openings for wire bonding or flip-chip bumps.

2. **Breaks Flexible PCBs:** Flex PCBs do not use liquid photoimageable (LPI) solder mask. They use Polyimide Coverlay (a polyimide sheet bonded with acrylic adhesive).

3. **Breaks High-Frequency RF PCBs:** At 10 GHz+, solder mask has a terrible loss tangent (tan δ ≈ 0.02) that degrades signals. RF engineers intentionally build boards with NO solder mask over microwave traces.

4. **Creates "Phantom Layers" in Memory:** `StackupManager` secretly creates two memory layers (`top_mask` and `bottom_mask`) that never existed in the user's `profile.stackup` file! This creates invisible layers that confuse DRC checkers and exporters.

### 7.2 How Physical Stackups Actually Work (No Magic)

In physical reality, a board or chip stackup is just an ordered sandwich of material layers from bottom to top:

```
PHYSICAL UNIFIED STACKUP (Bottom to Top)

Layer 4 (Top):    top_mask     │ Material: LPI_Green / Passivation │ 20µm  │ routable: false
Layer 3:          top_copper   │ Material: Copper / Aluminum       │ 35µm  │ routable: true
Layer 2:          dielectric   │ Material: FR4 / Silicon_Dioxide   │ 1.5mm │ routable: false
Layer 1:          bot_copper   │ Material: Copper / Aluminum       │ 35µm  │ routable: true
Layer 0 (Bottom): bot_mask     │ Material: LPI_Green / Passivation │ 20µm  │ routable: false
```

Copper traces sit on **Routable Layers** (`routable: true`). Protective coatings sit on **Non-Routable Layers** (`routable: false`).

**There is zero need for Rust code to special-case solder mask!**

### 7.3 The Pure Data-Driven Solution (v0.2.1+)

Solder mask, coverlay, and passivation become standard, explicit layers in `profile.stackup`:

#### Standard 4-Layer PCB with Solder Mask:
```hardware
profile PCB_Standard:
    technology: "PCB"
    stackup:
        bot_mask: [material: LPI_Green,       thickness: 20um,  routable: false]
        bot_cu:   [material: Copper,          thickness: 35um,  routable: true]
        core:     [material: FR4,             thickness: 1.5mm, routable: false]
        top_cu:   [material: Copper,          thickness: 35um,  routable: true]
        top_mask: [material: LPI_Green,       thickness: 20um,  routable: false]
```

#### TSMC 180nm ASIC with Passivation (NO PCB Solder Mask!):
```hardware
profile Silicon_180nm:
    technology: "ASIC"
    stackup:
        substrate:   [material: Silicon_P,       thickness: 300um, routable: false]
        polyres:     [material: Polysilicon,     thickness: 200nm, routable: true]
        d1:          [material: Silicon_Dioxide, thickness: 300nm, routable: false]
        metal1:      [material: Aluminum,        thickness: 840nm, routable: true]
        passivation: [material: Silicon_Nitride, thickness: 500nm, routable: false]
```

### 7.4 What Happens to StackupManager in Rust?

`StackupManager` loses all special solder mask fields. It becomes a pure 100% data-driven parser that reads `stackup.layers` from index 0 to N-1:

```rust
// ✅ PURGED: No hardcoded solder mask fields!
pub struct StackupManager {
    /// Maps semantic layer name (e.g. "bot_mask", "metal1") -> Z-start (nm)
    layer_start_z_nm: HashMap<String, i64>,
    layer_thickness_nm: HashMap<String, i64>,
    ordered_layers: Vec<String>,
    layer_materials: HashMap<String, String>,
    conductive_layers: HashSet<String>,
    
    // ❌ DELETED: pub solder_mask_thickness_nm: i64
}

impl StackupManager {
    pub fn new(
        stackup_opt: Option<&LayerStackup>,
        symbol_table: &SymbolTable,
        eval_context: &EvaluationContext,
    ) -> Result<Self, IrError> {
        // 1. Iterate layers in stackup_opt in exact order (0 to N-1)
        // 2. Set Z_start for Layer 0 = 0nm (The Absolute Floor!)
        // 3. Set Z_start for Layer i = Z_start(i-1) + thickness(i-1)
        
        // ZERO SPECIAL CASES. ZERO MAGIC.
    }
    
    /// Board thickness is simply the sum of ALL layers in stackup!
    pub fn board_thickness_nm(&self) -> i64 {
        self.layer_thickness_nm.values().sum()
    }
    
    /// Surface Z for top/bottom mounting
    pub fn board_surface_z(&self, side: MountingSide) -> i64 {
        match side {
            MountingSide::Top => self.board_thickness_nm(),
            MountingSide::Bottom => 0, // ✅ Absolute floor at Z=0
            MountingSide::Embedded => self.board_thickness_nm() / 2,
        }
    }
}
```

### 7.5 How Openings / Cutouts Are Generated

When a component pad or via is placed on an outer conductive layer (e.g. `top_cu`), the compiler's subtractive mask pass looks at adjacent protective layers (`top_mask` or `passivation`) and automatically cuts an opening over the pad based on the pad's `solder_mask_expansion` property or `is_tented: false` flag.

### 7.6 Summary of Solder Mask Purge Gains

| Aspect               | Old v0.1.7 Hardcoded Solder Mask                             | New v0.2.1+ Data-Driven Stackup               |
| :------------------- | :----------------------------------------------------------- | :-------------------------------------------- |
| **Rust Code**        | `solder_mask_thickness_nm` special cases in `StackupManager` | **DELETED.** 100% generic layer iteration.    |
| **ASIC Support**     | ❌ Injected fake 20µm green solder mask onto silicon dies     | ✅ Native `Silicon_Nitride` passivation layer. |
| **Flex PCB Support** | ❌ Couldn't model polyimide coverlay                          | ✅ Native `Polyimide` coverlay layer.          |
| **RF Support**       | ❌ Forced solder mask on microwave traces                     | ✅ Omit mask layer in profile for bare copper. |
| **$Z$-Axis Floor**   | ❌ Negative $Z$ coordinates ($-20\mu\text{m}$)                | ✅ $Z = 0$ at Layer 0 (bottom of stackup).     |

**Conclusion:** Deleting hardcoded solder mask fields from `StackupManager` completes the Bloat Purge and makes HardwareScript fully universal across PCB, ASIC, Flex, and RF domains.

---

## Section 8: Before & After Code Transformation

To demonstrate the power of the Bloat Purge, consider the complete transformation of `variable_resistor_test.hw` from **250 lines of bloated boilerplate** down to **~35 lines of pure declarative hardware truth**.

### ❌ BEFORE (v0.1.6 Bloated Test File - 250 Lines)

```hardware
# variable_resistor_test.hw (BEFORE PURGE)
import * from @std/primitives/units
import * from "./resistor_pdk"

module GeometryVariableResistor:
    pins: [input In1, input In2, input In3, output Out1, output Out2, output Out3]
    route In1 to Out1
    route In2 to Out2
    route In3 to Out3

space Geometry_Variable_Resistor_Space implements GeometryVariableResistor:
    dimensions: 40.0um by 20.0um by 1200nm  # ❌ Redundant Z-height
    resolution: 10nm                        # ❌ Obsolete voxel relic
    origin: bl by b                         # ❌ Obsolete origin token
    profile: Resistor_3D

    nets:
        In1:  { classification: signal, potential: 1.8V, current: 1.0uA }
        Out1: { classification: signal, potential: 0.0V, current: 1.0uA }
        In2:  { classification: signal, potential: 1.8V, current: 1.0uA }
        Out2: { classification: signal, potential: 0.0V, current: 1.0uA }
        In3:  { classification: signal, potential: 1.8V, current: 1.0uA }
        Out3: { classification: signal, potential: 0.0V, current: 1.0uA }
        GND:  { classification: ground, potential: 0.0V, current: 0.0uA }

    # RESISTOR 1 (Requires 35 lines of individual pour/contact/via/air-hack placements!)
    add pour(Polysilicon) named R1_Body_A on layer: polyres:
        device: R1.A
        dimensions: 5.0um by 10.0um
        at: [x: 7.0um, y: 3.0um]
        net: In1

    add pour(Polysilicon) named R1_Body_B on layer: polyres:
        device: R1.B
        dimensions: 5.0um by 10.0um
        at: [x: R1_Body_A.right, y: 3.0um]
        net: Out1
    
    add pour(Air) named R1_Bulk on layer: polyres:  # ❌ 1nm Air Pour Hack!
        device: R1.BULK
        net: GND
        dimensions: 1nm by 1nm
        at: [x: 9.5um, y: 1.0um]

    add pour(Aluminum) named R1_Contact_Left on layer: metal1:
        net: In1
        dimensions: 800nm by 800nm
        at: [x: R1_Body_A.center_x, y: R1_Body_A.center_y]

    add pour(Aluminum) named R1_Contact_Right on layer: metal1:
        net: Out1
        dimensions: 800nm by 800nm
        at: [x: R1_Body_B.center_x, y: R1_Body_B.center_y]

    add contact(Tungsten) named R1_Via_Left spanning layer: polyres to metal1:
        net: In1
        diameter: 300nm
        at: [x: R1_Contact_Left.center_x, y: R1_Contact_Left.center_y]

    add contact(Tungsten) named R1_Via_Right spanning layer: polyres to metal1:
        net: Out1
        diameter: 300nm
        at: [x: R1_Contact_Right.center_x, y: R1_Contact_Right.center_y]

    # ... (Repeated 2 more times for R2 and R3 = 250 lines total!) ...
```

---

### ✅ AFTER (v0.2.1+ Purged & Streamlined - ~35 Lines)

```hardware
# variable_resistor_test.hw (AFTER BLOAT PURGE)
import * from @std/primitives/units
import * from "./resistor_pdk"

module GeometryVariableResistor:
    pins: [input In1, input In2, input In3, output Out1, output Out2, output Out3]
    route In1 to Out1
    route In2 to Out2
    route In3 to Out3

space Geometry_Variable_Resistor_Space implements GeometryVariableResistor:
    dimensions: 40.0um by 20.0um    # ✅ Pure X by Y space footprint
    profile: Resistor_3D             # ✅ Stackup defines Z-height and fab rules

    nets:
        In1:  { classification: signal, potential: 1.8V, current: 1.0uA }
        Out1: { classification: signal, potential: 0.0V, current: 1.0uA }
        In2:  { classification: signal, potential: 1.8V, current: 1.0uA }
        Out2: { classification: signal, potential: 0.0V, current: 1.0uA }
        In3:  { classification: signal, potential: 1.8V, current: 1.0uA }
        Out3: { classification: signal, potential: 0.0V, current: 1.0uA }
        GND:  { classification: ground, potential: 0.0V, current: 0.0uA }

    # ========================================================================
    # RESISTOR INSTANTIATIONS (1 Line Each using PDK PCells)
    # ========================================================================
    
    # R1: Square Geometry (10μm × 10μm → L/W = 1 → ~1074 Ω)
    add Resistor_SKY130(W: 10.0um, L: 10.0um) named R1 at: [x: 7.0um, y: 3.0um]:
        nets: { A: In1, B: Out1, BULK: GND }

    # R2: Medium Thin Geometry (10μm × 1μm → L/W = 10 → ~4224 Ω)
    add Resistor_SKY130(W: 1.0um, L: 10.0um) named R2 at: [x: 7.0um, y: 9.0um]:
        nets: { A: In2, B: Out2, BULK: GND }

    # R3: Long Thin Geometry (20μm × 1μm → L/W = 20 → ~7724 Ω)
    add Resistor_SKY130(W: 1.0um, L: 20.0um) named R3 at: [x: 10.0um, y: 15.0um]:
        nets: { A: In3, B: Out3, BULK: GND }
```

---

## Section 9: Summary of Purge Gains

| Area | Before Purge | After Purge | Architectural Gain |
| :--- | :--- | :--- | :--- |
| **Code Length** | 250 lines for 3 resistors | **~35 lines** | **7× reduction** in source code verbosity |
| **Space Block** | `dimensions`, `resolution`, `origin` | `dimensions: X by Y`, `profile:` | Single source of truth; zero dual-authority bugs |
| **Ghost Keyword** | `absolute:` wrapper block | Direct placement with `at:` | Zero useless block wrappers |
| **SPICE Terminals** | $1\text{nm}$ dummy `Air` pour hacks | `nets: { BULK: GND }` | Zero fake geometry in database or GLB models |
| **Resistor Geometry**| Sliced into two separate pours | Single continuous PCell body | 100% accurate physical DRC and Clipper2 unioning |
| **PDK Profile** | 15 internal pathfinding weights | Physical stackup & rules ONLY | Clean API separation between engine and PDK |
| **Material Definitions**| Quoted symbols (`"Poly"`), `Air: process: deposited` | Bare identifiers (`Poly`), optional process | 100% grammar compliance with no logical contradictions |
| **Solder Mask** | Hardcoded 20µm in `StackupManager` Rust code | Explicit stackup layer in `.hw` PDK | Universal support: PCB/ASIC/Flex/RF domains |

**Document Status:** Approved for Core Implementation  
**Version:** v0.2.1+ (Canonical Reference)  
**HardwareScript Compiler Team — August 2026**