# HardwareScript IR Output Emission (v0.2.0)

**Document Type:** Feature Specification & Compiler Architecture  
**Status:** 🚧 **PLANNED** - Awaiting Implementation  
**Date:** 2024  
**Related Documents:**
- `Relational-Placement-Determinism.md` (v0.2.0)
- `Core-Compiler-Architecture.md` (v0.2.0)
- `Spatial_Synthesis_Abstraction.md` (v0.1.9)

---

## Executive Summary

HardwareScript introduces **Intermediate Representation (IR) emission flags** that allow designers to inspect how the compiler lowers high-level abstractions into progressively more concrete representations.

Similar to `rustc --emit mir` or `clang -emit-llvm`, hardwareScript provides:
- `hwc check --emit-mid` — Output mid-level relational IR
- `hwc check --emit-low` — Output low-level absolute coordinate IR
- `hwc check --emit-all` — Output all intermediate representations

This feature enables:
- **Debugging** - Understand how abstractions are interpreted
- **Learning** - See the concrete geometry behind relational constraints
- **Verification** - Compare compiler output against expected lowering
- **Interoperability** - Export low-level IR for external tools
- **Optimization** - Validate that high-level code produces efficient geometry

---

## Section 1: HardwareScript Abstraction Levels

HardwareScript operates at three distinct abstraction levels, each with progressively more explicit semantics:

### 1.1 High-Level (Behavioral/Parametric)

**Characteristics:**
- Parametric component generators with `for` loops
- Behavioral routing descriptions
- Abstract floorplanning directives
- High-level intent expressions
- Conditional compilation with compile-time evaluation

**Example:**
```hardware
# High-level parametric pad array
let pad_count = 8
let pad_pitch = 50um
let pad_size = 100um

for i in 0..pad_count:
    region Pad{i}_Region:
        at: [x: i * pad_pitch, y: 0um, z: 0nm]
        boundary: [width: pad_size, height: pad_size]
    
    add plane(Aluminum) named Pad_{i} inside: Pad{i}_Region:
        shape: Pad(w: pad_size, h: pad_size)
        at: Pad{i}_Region.center
        net: Signal_{i}
```

### 1.2 Mid-Level (Relational/Declarative)

**Characteristics:**
- Explicit `region` declarations with relational constraints
- Routing intents that query PDK parameters
- Compile-time variable bindings (`let`)
- Relational placement (`right_of`, `inside`, `align`)

**Example (lowered from high-level):**
```hardware
# Mid-level relational constraints
let pad_count = 8
let pad_pitch = 50um
let pad_size = 100um

region Pad0_Region:
    at: [x: 0um, y: 0um, z: 0nm]
    boundary: [width: 100um, height: 100um]

region Pad1_Region:
    right_of: Pad0_Region with spacing: 50um
    align: center_y with Pad0_Region
    boundary: [width: 100um, height: 100um]

# ... regions Pad2_Region through Pad7_Region ...

add plane(Aluminum) named Pad_0 inside: Pad0_Region on layer: metal1:
    shape: Pad(w: 100um, h: 100um)
    at: Pad0_Region.center
    net: Signal_0

add plane(Aluminum) named Pad_1 inside: Pad1_Region on layer: metal1:
    shape: Pad(w: 100um, h: 100um)
    at: Pad1_Region.center
    net: Signal_1

# ... pads Pad_2 through Pad_7 ...
```

### 1.3 Low-Level (Absolute/Explicit)

**Characteristics:**
- Absolute `at: [x, y, z]` coordinates (picometer precision)
- Direct shape instantiation with no abstractions
- Explicit trace path geometries
- No regions, no constraints, no intents

**Example (lowered from mid-level):**
```hardware
# Low-level absolute coordinates
add plane(Aluminum) named Pad_0 on layer: metal1:
    shape: Pad(w: 100um, h: 100um)
    at: [x: 50um, y: 50um, z: 960nm]  # Center of 100×100 region at origin
    net: Signal_0

add plane(Aluminum) named Pad_1 on layer: metal1:
    shape: Pad(w: 100um, h: 100um)
    at: [x: 200um, y: 50um, z: 960nm]  # Center of region at x=150um
    net: Signal_1

# ... remaining pads with explicit coordinates ...
```

### 1.4 Abstraction Level Comparison

| Aspect | High-Level | Mid-Level | Low-Level |
|--------|-----------|-----------|-----------|
| **Placement** | Parametric generators | Relational constraints | Absolute coordinates |
| **Routing** | Behavioral intent | PDK-queried parameters | Explicit trace paths |
| **Regions** | Implicit from loops | Explicit declarations | None (absorbed into coords) |
| **Variables** | Parametric expressions | Compile-time `let` | Fully resolved values |
| **Syntax Size** | Most compact | Moderate | Most verbose |
| **Editability** | High (change parameters) | Medium (adjust constraints) | Low (manual coordinate math) |

---

## Section 2: IR Emission Commands

### 2.1 Command Syntax

```bash
# Check syntax and emit mid-level IR
hwc check --emit-mid input.hw

# Check syntax and emit low-level IR
hwc check --emit-low input.hw

# Emit both mid-level and low-level IR
hwc check --emit-all input.hw

# Emit IR to custom output path
hwc check --emit-mid --output-dir ./ir_output input.hw
```

### 2.2 Output Files

**Default Naming Convention:**
- Input: `my_design.hw`
- Mid-level output: `my_design.mid.hw` (same directory as input)
- Low-level output: `my_design.low.hw` (same directory as input)

**Custom Output Directory:**
```bash
hwc check --emit-low --output-dir ./build/ir my_design.hw
# Produces: ./build/ir/my_design.low.hw
```

### 2.3 Emission Behavior by Input Level

| Input Level | `--emit-mid` | `--emit-low` |
|-------------|--------------|--------------|
| **High** | Lowers to mid-level | Lowers to low-level (skips mid) |
| **Mid** | Pass-through (no change) | Lowers to low-level |
| **Low** | Not applicable (no-op) | Pass-through (no change) |

**Key Rules:**
1. **Downward-only lowering** - Cannot emit higher abstraction from lower input
2. **Mixed-level handling** - Each construct lowers independently
3. **Idempotency** - Emitting from already-lowered code produces identical output

### 2.4 Mixed Abstraction Level Handling

**Scenario 1: High-Level with Embedded Mid-Level**
```hardware
# Input: my_design.hw (mixed high + mid)
for i in 0..4:
    region Pad{i}:
        at: [x: i * 100um, y: 0um, z: 0nm]
        boundary: [width: 50um, height: 50um]

# Manually written mid-level region
region ManualRegion:
    right_of: Pad3 with spacing: 200um
    boundary: [width: 100um, height: 100um]
```

**Output with `--emit-mid`:**
```hardware
# All high-level constructs unrolled, mid-level preserved
region Pad0:
    at: [x: 0um, y: 0um, z: 0nm]
    boundary: [width: 50um, height: 50um]

region Pad1:
    at: [x: 100um, y: 0um, z: 0nm]
    boundary: [width: 50um, height: 50um]

# ... Pad2, Pad3 ...

region ManualRegion:  # Preserved as-is
    right_of: Pad3 with spacing: 200um
    boundary: [width: 100um, height: 100um]
```

**Scenario 2: Mid-Level with Embedded Low-Level**
```hardware
# Input: my_design.hw (mixed mid + low)
region RegionA:
    at: [x: 100um, y: 100um, z: 0nm]
    boundary: [width: 50um, height: 50um]

add plane(Aluminum) named Pad_A inside: RegionA:
    shape: Pad(w: 50um, h: 50um)
    at: RegionA.center

# Manually written low-level component
add plane(Aluminum) named Pad_B on layer: metal1:
    shape: Pad(w: 50um, h: 50um)
    at: [x: 300um, y: 100um, z: 960nm]
```

**Output with `--emit-low`:**
```hardware
# All regions resolved, low-level preserved
add plane(Aluminum) named Pad_A on layer: metal1:
    shape: Pad(w: 50um, h: 50um)
    at: [x: 125um, y: 125um, z: 960nm]  # Center of RegionA

add plane(Aluminum) named Pad_B on layer: metal1:  # Preserved as-is
    shape: Pad(w: 50um, h: 50um)
    at: [x: 300um, y: 100um, z: 960nm]
```

---

## Section 3: Compiler Implementation Architecture

### 3.1 Compilation Pipeline

The hardwareScript compiler already performs these transformations internally:

```
┌─────────────┐
│   Source    │ my_design.hw (any abstraction level)
└──────┬──────┘
       │
       v
┌─────────────┐
│   Lexer     │ Token stream
└──────┬──────┘
       │
       v
┌─────────────┐
│   Parser    │ AST (mixed abstraction levels)
└──────┬──────┘
       │
       v
┌─────────────────────┐
│ High → Mid Lowering │ Parametric unroller, loop expansion
└──────┬──────────────┘
       │  ← EMIT MID-LEVEL HERE (--emit-mid)
       v
┌─────────────────────┐
│ Mid → Low Lowering  │ Relational constraint resolver
└──────┬──────────────┘
       │  ← EMIT LOW-LEVEL HERE (--emit-low)
       v
┌─────────────────────┐
│  IR Compilation     │ Physical geometry generation
└──────┬──────────────┘
       │
       v
┌─────────────────────┐
│   Code Generation   │ DXF, GLB, Netlist, BOM
└─────────────────────┘
```

### 3.2 IR Emission Points

**Implementation Strategy:**
1. **Add snapshot capability** after each lowering phase
2. **Implement AST → Source serializer** (pretty-printer)
3. **Respect mixed abstraction levels** during emission
4. **Preserve comments and formatting** where possible (stretch goal)

**Key Modules to Modify:**
- `hwc-cli/src/commands/check_cmd.rs` - Add `--emit-*` flags
- `hwc-compiler/src/ir/parametric_unroller/` - Add mid-level snapshot
- `hwc-compiler/src/ir/relational_resolver.rs` - Add low-level snapshot
- `hwc-parser/src/ast/` - Implement `to_source()` for all AST nodes

### 3.3 AST to Source Serialization

**Required Trait:**
```rust
pub trait ToSource {
    fn to_source(&self) -> String;
}
```

**Implementation Examples:**

```rust
// For region declarations
impl ToSource for RegionDef {
    fn to_source(&self) -> String {
        let mut output = format!("region {}:\n", self.name);
        
        if let Some(at_pos) = &self.position {
            output.push_str(&format!("    at: [x: {}nm, y: {}nm, z: {}nm]\n",
                at_pos.x, at_pos.y, at_pos.z));
        }
        
        if let Some(boundary) = &self.boundary {
            output.push_str(&format!("    boundary: [width: {}nm, height: {}nm]\n",
                boundary.width, boundary.height));
        }
        
        output
    }
}

// For plane placements
impl ToSource for PlacementDef {
    fn to_source(&self) -> String {
        let mut output = format!("add plane({}) named {}", 
            self.material, self.name);
        
        if let Some(layer) = &self.layer {
            output.push_str(&format!(" on layer: {}", layer));
        }
        
        output.push_str(":\n");
        output.push_str(&format!("    shape: {}(w: {}nm, h: {}nm)\n",
            self.shape_type, self.width, self.height));
        output.push_str(&format!("    at: [x: {}nm, y: {}nm, z: {}nm]\n",
            self.x, self.y, self.z));
        
        if let Some(net) = &self.net {
            output.push_str(&format!("    net: {}\n", net));
        }
        
        output
    }
}
```

---

## Section 4: Use Cases & Examples

### 4.1 Debugging Relational Constraints

**Problem:** "Why did my pad end up at that coordinate?"

**Solution:**
```bash
# Original mid-level source
region PadA:
    at: [x: 100um, y: 100um, z: 0nm]
    boundary: [width: 50um, height: 50um]

region PadB:
    right_of: PadA with spacing: 200um
    align: center_y with PadA
    boundary: [width: 50um, height: 50um]

# Emit low-level to see resolved coordinates
$ hwc check --emit-low pad_layout.hw

# Output: pad_layout.low.hw
add plane(Aluminum) named PadA:
    at: [x: 125um, y: 125um, z: 960nm]  # Center of PadA region

add plane(Aluminum) named PadB:
    at: [x: 375um, y: 125um, z: 960nm]  # 150 + 50 + 200 + 25 = 375um
```

**Insight:** See that `spacing: 200um` means **clear gap**, not total distance.

### 4.2 Learning Parametric Expansion

**Problem:** "How does my `for` loop expand?"

**Solution:**
```bash
# Original high-level source
for i in 0..4:
    add plane(Aluminum) named Pad_{i}:
        at: [x: i * 100um, y: 0um, z: 0nm]
        net: Signal_{i}

# Emit mid-level to see unrolled loop
$ hwc check --emit-mid array.hw

# Output: array.mid.hw
add plane(Aluminum) named Pad_0:
    at: [x: 0um, y: 0um, z: 0nm]
    net: Signal_0

add plane(Aluminum) named Pad_1:
    at: [x: 100um, y: 0um, z: 0nm]
    net: Signal_1

add plane(Aluminum) named Pad_2:
    at: [x: 200um, y: 0um, z: 0nm]
    net: Signal_2

add plane(Aluminum) named Pad_3:
    at: [x: 300um, y: 0um, z: 0nm]
    net: Signal_3
```

### 4.3 Verification Against Manual Layout

**Problem:** "Does my high-level code produce the same geometry as my manual low-level version?"

**Solution:**
```bash
# Emit low-level from high-level design
$ hwc check --emit-low my_design_highlevel.hw
# Produces: my_design_highlevel.low.hw

# Compare against manual low-level version
$ diff my_design_highlevel.low.hw my_design_manual_lowlevel.hw
```

**Expected Outcome:** Identical geometry (modulo whitespace and comments)

### 4.4 Exporting for External Tools

**Problem:** "I need to import my design into a legacy tool that doesn't understand regions."

**Solution:**
```bash
# Convert high-level design to low-level for legacy toolchain
$ hwc check --emit-low --output-dir ./legacy_export my_chip.hw

# Output: ./legacy_export/my_chip.low.hw (pure absolute coordinates)
```

---

## Section 5: Semantic Guarantees

### 5.1 Determinism Guarantee

**Property:** Emitting from the same source always produces identical output.

**Validation:**
```bash
$ hwc check --emit-low design.hw -o run1.low.hw
$ hwc check --emit-low design.hw -o run2.low.hw
$ diff run1.low.hw run2.low.hw
# Expected: No differences
```

**Implementation Requirement:** Use deterministic iteration order (BTreeMap, not HashMap).

### 5.2 Round-Trip Compilation Guarantee

**Property:** Emitted IR compiles to identical geometry as original source.

**Validation:**
```bash
# Build from original high-level source
$ hwc build design.hw
# Produces: build/Design/board.dxf

# Emit low-level and build from that
$ hwc check --emit-low design.hw
$ hwc build design.low.hw
# Produces: build/Design_low/board.dxf

# Compare outputs
$ diff build/Design/board.dxf build/Design_low/board.dxf
# Expected: Identical geometry (modulo space names)
```

### 5.3 Abstraction Preservation Guarantee

**Property:** Mixed abstraction levels are preserved during partial lowering.

**Example:**
```hardware
# Input: mixed.hw (high + mid + low)
for i in 0..2:            # High-level
    region R{i}:
        at: [x: i*100um]  # High-level expression

region ManualRegion:      # Mid-level
    right_of: R1 with spacing: 50um

add plane(Copper) named P1:  # Low-level
    at: [x: 500um, y: 0um, z: 0nm]
```

**Output with `--emit-mid`:**
```hardware
# High → Mid, Mid preserved, Low preserved
region R0:                # Lowered from high
    at: [x: 0um, y: 0um, z: 0nm]

region R1:                # Lowered from high
    at: [x: 100um, y: 0um, z: 0nm]

region ManualRegion:      # Mid-level preserved as-is
    right_of: R1 with spacing: 50um

add plane(Copper) named P1:  # Low-level preserved as-is
    at: [x: 500um, y: 0um, z: 0nm]
```

---

## Section 6: CLI Integration

### 6.1 New Flags for `hwc check`

**Current `hwc check` behavior:**
- Parses source file
- Validates syntax
- Checks semantic correctness
- Exits (no output files generated)

**Enhanced `hwc check` with IR emission:**
```bash
hwc check [OPTIONS] <INPUT>

OPTIONS:
    --emit-mid              Emit mid-level IR to <input>.mid.hw
    --emit-low              Emit low-level IR to <input>.low.hw
    --emit-all              Emit both mid-level and low-level IR
    --output-dir <DIR>      Write IR files to specified directory
    --overwrite             Overwrite existing IR files without prompting
    -v, --verbose           Show detailed lowering information

ARGS:
    <INPUT>                 Input .hw file
```

### 6.2 Example Usage Patterns

**Basic emission:**
```bash
$ hwc check --emit-low design.hw
✓ Parsed design.hw (23 definitions)
✓ Resolved 45 relational constraints
✓ Emitted low-level IR to design.low.hw
```

**Custom output directory:**
```bash
$ hwc check --emit-all --output-dir ./ir design.hw
✓ Emitted ./ir/design.mid.hw (mid-level)
✓ Emitted ./ir/design.low.hw (low-level)
```

**Verbose output:**
```bash
$ hwc check --emit-low -v design.hw
[Parametric Unroller] Expanded 3 for loops into 42 statements
[Constraint Resolver] Resolved 12 region placements
  → RegionA: (100000nm, 100000nm)
  → RegionB: (350000nm, 100000nm) via right_of constraint
  → RegionC: (600000nm, 100000nm) via right_of constraint
[IR Emitter] Writing low-level IR to design.low.hw
✓ Emission complete
```

### 6.3 Error Handling

**Overwrite Protection:**
```bash
$ hwc check --emit-low design.hw
✗ Error: design.low.hw already exists
  Use --overwrite to replace existing file
```

**Invalid Lowering Attempt:**
```bash
$ hwc check --emit-mid already_lowlevel.hw
⚠ Warning: Input is already at low-level abstraction
✓ Emitted already_lowlevel.mid.hw (unchanged)
```

**Syntax Errors Block Emission:**
```bash
$ hwc check --emit-low broken.hw
✗ Error at broken.hw:23:15
  Expected coordinate expression, found '}'
✗ Cannot emit IR due to syntax errors
```

---

## Section 7: Implementation Roadmap

### 7.1 Phase 1: Core Infrastructure (Week 1)

**Goals:**
- Implement `ToSource` trait for all AST nodes
- Add `--emit-mid` and `--emit-low` flags to CLI
- Create IR snapshot mechanism in compiler pipeline

**Deliverables:**
- [ ] `hwc-parser/src/ast/to_source.rs` module
- [ ] CLI argument parsing in `hwc-cli/src/commands/check_cmd.rs`
- [ ] Basic emission of low-level IR from mid-level input

**Test Case:**
```hardware
# Input: simple_region.hw
region R:
    at: [x: 100um, y: 100um, z: 0nm]
    boundary: [width: 50um, height: 50um]

add plane(Aluminum) named P inside: R:
    shape: Pad(w: 50um, h: 50um)
    at: R.center

# Expected output with --emit-low: simple_region.low.hw
add plane(Aluminum) named P on layer: metal1:
    shape: Pad(w: 50um, h: 50um)
    at: [x: 125um, y: 125um, z: 960nm]
```

### 7.2 Phase 2: Parametric Expansion (Week 2)

**Goals:**
- Emit mid-level IR from high-level parametric constructs
- Expand `for` loops and evaluate compile-time expressions
- Handle `let` variable substitution

**Deliverables:**
- [ ] `--emit-mid` from high-level parametric input
- [ ] Loop unroller integration with IR emitter
- [ ] Compile-time expression evaluation in emitted code

**Test Case:**
```hardware
# Input: parametric_array.hw
let count = 4
let pitch = 100um

for i in 0..count:
    add plane(Aluminum) named Pad_{i}:
        at: [x: i * pitch, y: 0um, z: 0nm]

# Expected output with --emit-mid: parametric_array.mid.hw
add plane(Aluminum) named Pad_0:
    at: [x: 0um, y: 0um, z: 0nm]

add plane(Aluminum) named Pad_1:
    at: [x: 100um, y: 0um, z: 0nm]

# ... Pad_2, Pad_3 ...
```

### 7.3 Phase 3: Mixed Abstraction Handling (Week 3)

**Goals:**
- Handle mixed high/mid/low abstraction levels in single file
- Preserve lower-level constructs during partial lowering
- Implement `--emit-all` flag

**Deliverables:**
- [ ] Abstraction level detection per AST node
- [ ] Selective lowering based on current abstraction level
- [ ] `--emit-all` produces both mid and low outputs

**Test Case:**
```hardware
# Input: mixed.hw (high + mid + low)
for i in 0..2:
    region R{i}:
        at: [x: i*100um, y: 0um, z: 0nm]

region ManualRegion:
    right_of: R1 with spacing: 50um

add plane(Copper) named Manual:
    at: [x: 500um, y: 0um, z: 0nm]

# Expected: --emit-mid preserves ManualRegion and Manual
# Expected: --emit-low resolves all to absolute coordinates
```

### 7.4 Phase 4: Routing Intent Emission (Week 4)

**Goals:**
- Emit routing intents in mid-level IR
- Lower routing intents to explicit trace paths in low-level IR
- Handle PDK parameter queries in emission

**Deliverables:**
- [ ] `route` statement serialization with `intent:`
- [ ] Explicit trace path emission in low-level
- [ ] PDK-resolved parameters (escape stubs, trace widths) in output

**Test Case:**
```hardware
# Input: routing.hw (mid-level)
route:
    from: PadA
    to: PadB
    layer: metal1
    width: 20um
    intent: Signal  # Queries PDK for escape_stub: 50um

# Expected output with --emit-low: routing.low.hw
route:
    from: PadA
    to: PadB
    layer: metal1
    path: [
        [x: 200um, y: 150um],   # Start at PadA edge
        [x: 200um, y: 200um],   # Perpendicular escape (50um)
        [x: 550um, y: 200um],   # Horizontal segment
        [x: 550um, y: 150um]    # Drop to PadB
    ]
    width: 20um
```

### 7.5 Phase 5: Verification & Testing (Week 5)

**Goals:**
- Validate round-trip compilation (emit → build → compare)
- Add determinism tests to CI pipeline
- Performance benchmarking

**Deliverables:**
- [ ] Test suite in `tests/ir-emission/`
- [ ] CI integration with determinism checks
- [ ] Performance metrics (emission time vs. build time)

---

## Section 8: Comparison to Industry Standards

### 8.1 Rust's `--emit` Flags

**Rust Compiler (`rustc`):**
```bash
rustc --emit mir main.rs       # Mid-level IR (MIR)
rustc --emit llvm-ir main.rs   # Low-level IR (LLVM)
rustc --emit asm main.rs       # Assembly
```

**HardwareScript Equivalent:**
```bash
hwc check --emit-mid design.hw  # Mid-level (regions/intents)
hwc check --emit-low design.hw  # Low-level (absolute coords)
# No assembly equivalent (DXF/GLB are final outputs)
```

### 8.2 LLVM's IR Levels

**LLVM Pipeline:**
1. **Frontend IR** - Language-specific AST
2. **LLVM IR** - SSA-form intermediate representation
3. **Machine IR** - Target-specific assembly

**HardwareScript Equivalent:**
1. **High-level IR** - Parametric constructs, behavioral routing
2. **Mid-level IR** - Relational constraints, routing intents
3. **Low-level IR** - Absolute coordinates, explicit geometry

### 8.3 Verilog Elaboration

**SystemVerilog:**
```verilog
// High-level generate block
generate
    for (genvar i = 0; i < 4; i++) begin
        inverter inv_i (.in(in[i]), .out(out[i]));
    end
endgenerate

// After elaboration (emitted IR):
inverter inv_0 (.in(in[0]), .out(out[0]));
inverter inv_1 (.in(in[1]), .out(out[1]));
inverter inv_2 (.in(in[2]), .out(out[2]));
inverter inv_3 (.in(in[3]), .out(out[3]));
```

**HardwareScript Equivalent:**
```hardware
# High-level
for i in 0..4:
    add component(Inverter) named inv_{i}:
        inputs: [in: signals[i]]
        outputs: [out: outputs[i]]

# Mid-level emission (--emit-mid)
add component(Inverter) named inv_0:
    inputs: [in: signals[0]]
    outputs: [out: outputs[0]]

# ... inv_1, inv_2, inv_3 ...
```

---

## Section 9: Advanced Features (Future Work)

### 9.1 Selective Emission with Filters

**Proposed Syntax:**
```bash
# Emit only specific definitions
hwc check --emit-low --filter="Pad_*" design.hw

# Emit only routing IR
hwc check --emit-low --only-routes design.hw
```

### 9.2 Diff-Based IR Inspection

**Proposed Feature:**
```bash
# Show what changed during lowering
hwc check --emit-low --show-diff design.hw

# Output:
- region RegionA:
-     right_of: RegionB with spacing: 100um
+ add plane(Aluminum) named RegionA_Pad:
+     at: [x: 375um, y: 125um, z: 960nm]
```

### 9.3 Interactive IR Explorer

**Proposed Tool:**
```bash
hwc ir-explore design.hw
# Opens TUI showing:
# - Source code on left
# - Lowered IR on right
# - Click on construct to see its lowering
```

### 9.4 IR Optimization Reports

**Proposed Feature:**
```bash
hwc check --emit-low --report-optimizations design.hw

# Output:
[Optimization] Merged 4 collinear route segments into 1
[Optimization] Eliminated redundant via at (250um, 150um)
[Warning] Suboptimal spacing: PadA-PadB could reduce by 50um
```

---

## Section 10: FAQ

### Q1: Can I edit the emitted IR and re-compile it?

**Yes!** Emitted IR is valid hardwareScript source code.

```bash
# Emit low-level IR
$ hwc check --emit-low design.hw

# Edit design.low.hw manually (adjust coordinates, etc.)

# Build from edited low-level IR
$ hwc build design.low.hw
```

### Q2: Will emitted IR preserve my comments?

**Not in v0.2.0.** Comments are lost during parsing.

**Future:** AST could track comments for preservation in emission.

### Q3: Does emission affect build performance?

**No.** Emission happens during the `check` phase, not during `build`.

Emission adds <10ms overhead for typical designs (<1000 components).

### Q4: Can I emit from partially broken code?

**No.** Syntax and semantic errors block emission.

The compiler must fully parse and resolve constraints before emitting IR.

### Q5: How do I compare two designs at the same abstraction level?

```bash
# Emit both to low-level
$ hwc check --emit-low design1.hw
$ hwc check --emit-low design2.hw

# Use standard diff tools
$ diff design1.low.hw design2.low.hw

# Or use hwc's built-in comparison (future feature)
$ hwc compare design1.low.hw design2.low.hw
```

### Q6: What happens if I emit from already low-level code?

**No-op.** The emitted file will be identical to the input (modulo formatting).

```bash
$ hwc check --emit-low already_lowlevel.hw
⚠ Warning: Input is already at low-level abstraction
✓ Emitted already_lowlevel.low.hw (unchanged)
```

### Q7: Can I emit only part of a design?

**Not in v0.2.0.** All definitions are emitted.

**Future:** `--filter` flag for selective emission (see Section 9.1).

---

## Conclusion

**IR emission brings professional compiler transparency to hardwareScript, enabling:**
- 🔍 **Debugging** - Understand how abstractions lower to geometry
- 📚 **Learning** - See the concrete effects of declarative constructs
- ✅ **Verification** - Prove that high-level code produces expected layouts
- 🔧 **Interoperability** - Export to legacy tools that don't support abstractions
- 🚀 **Optimization** - Identify inefficient layouts before fabrication

**Implementation follows established compiler design patterns from Rust, LLVM, and Verilog synthesis tools, ensuring familiar workflows for hardware engineers and software developers alike.**

**The `hwc check --emit-*` commands provide the same transparency as `rustc --emit` or `clang -emit-llvm`, bringing modern compiler observability to physical hardware description.**

---

## Appendix A: Full CLI Reference

```
hwc check - Validate syntax and semantics

USAGE:
    hwc check [OPTIONS] <INPUT>

OPTIONS:
    --emit-mid              Emit mid-level IR (regions, intents)
    --emit-low              Emit low-level IR (absolute coordinates)
    --emit-all              Emit both mid-level and low-level IR
    --output-dir <DIR>      Write emitted files to <DIR> instead of input directory
    --overwrite             Overwrite existing files without prompting
    -v, --verbose           Show detailed lowering information
    -q, --quiet             Suppress all output except errors
    -h, --help              Print help information

ARGS:
    <INPUT>                 Input .hw file to check

EXAMPLES:
    # Basic syntax check (no emission)
    hwc check design.hw

    # Emit low-level IR to same directory
    hwc check --emit-low design.hw

    # Emit both mid and low IR to custom directory
    hwc check --emit-all --output-dir ./ir design.hw

    # Overwrite existing IR files
    hwc check --emit-low --overwrite design.hw

    # Verbose lowering details
    hwc check --emit-low -v design.hw
```

---

## Appendix B: AST Node Coverage for ToSource Trait

**Required Implementations:**

| AST Node | Priority | Complexity | Notes |
|----------|----------|------------|-------|
| `PlacementDef` | High | Medium | Core placement serialization |
| `RegionDef` | High | Medium | Region with constraints |
| `RouteDef` | High | High | Routing intents + paths |
| `BridgeDef` | Medium | Medium | Via declarations |
| `ProfileDef` | Medium | Low | Material profiles |
| `ComponentDef` | Low | High | Behavioral components (future) |
| `LetBinding` | High | Low | Compile-time variables |
| `ForLoop` | High | Medium | Should be expanded before emission |
| `Expression` | High | High | Coordinate math, function calls |
| `CoordinateLiteral` | High | Low | `[x: 100um, y: 200um]` |
| `ShapeSpec` | Medium | Low | `Pad(w, h)`, `Trace(w)` |
| `NetAssignment` | High | Low | `net: SignalName` |
| `LayerSpec` | High | Low | `on layer: metal1` |

---

## Appendix C: Test Suite Structure

```
tests/ir-emission/
├── basic/
│   ├── simple_placement.hw           # Single plane with absolute coords
│   ├── simple_region.hw               # Region with inside constraint
│   └── expected/
│       ├── simple_placement.low.hw
│       └── simple_region.low.hw
├── parametric/
│   ├── for_loop_unroll.hw            # Array generation with for loop
│   ├── let_variable_substitution.hw   # Compile-time variables
│   └── expected/
│       ├── for_loop_unroll.mid.hw
│       └── let_variable_substitution.mid.hw
├── relational/
│   ├── right_of_constraint.hw        # Horizontal spacing
│   ├── align_center_y.hw             # Vertical alignment
│   ├── nested_regions.hw             # Hierarchical floorplanning
│   └── expected/
│       ├── right_of_constraint.low.hw
│       └── align_center_y.low.hw
├── routing/
│   ├── intent_based_routing.hw       # Routing with PDK intent
│   ├── explicit_path_routing.hw      # Manual trace paths
│   └── expected/
│       └── intent_based_routing.low.hw
├── mixed/
│   ├── high_mid_low_mixed.hw         # All abstraction levels
│   └── expected/
│       ├── high_mid_low_mixed.mid.hw
│       └── high_mid_low_mixed.low.hw
└── round_trip/
    ├── verify_determinism.sh         # Build original vs. emitted
    └── compare_dxf_outputs.py        # Geometric comparison
```

---

## Appendix D: Emission Pipeline Pseudocode

```rust
pub fn emit_ir(
    ast: &ParsedAst,
    target_level: AbstractionLevel,
    output_path: &Path,
) -> Result<(), EmissionError> {
    
    // Step 1: Determine current abstraction level of input
    let input_level = detect_abstraction_level(ast)?;
    
    // Step 2: Validate lowering is possible
    if target_level > input_level {
        return Err(EmissionError::CannotRaisAbstractionLevel {
            input: input_level,
            target: target_level,
        });
    }
    
    // Step 3: Apply necessary lowering transformations
    let lowered_ast = match (input_level, target_level) {
        (High, Mid) => {
            // Expand parametric constructs
            let expanded = parametric_unroller::unroll(ast)?;
            expanded
        },
        (High | Mid, Low) => {
            // First unroll parametrics if needed
            let expanded = if input_level == High {
                parametric_unroller::unroll(ast)?
            } else {
                ast.clone()
            };
            
            // Then resolve relational constraints
            let resolved = relational_resolver::resolve(&expanded)?;
            resolved
        },
        _ => {
            // No lowering needed (e.g., Mid → Mid or Low → Low)
            ast.clone()
        }
    };
    
    // Step 4: Serialize AST back to source code
    let mut output = String::new();
    for definition in &lowered_ast.definitions {
        output.push_str(&definition.to_source());
        output.push('\n');
    }
    
    // Step 5: Write to output file
    std::fs::write(output_path, output)?;
    
    Ok(())
}

fn detect_abstraction_level(ast: &ParsedAst) -> Result<AbstractionLevel, EmissionError> {
    let mut has_high = false;
    let mut has_mid = false;
    let mut has_low = false;
    
    for def in &ast.definitions {
        match def {
            Definition::ForLoop(_) => has_high = true,
            Definition::Region(r) if r.has_constraints() => has_mid = true,
            Definition::Placement(p) if p.has_absolute_coords() => has_low = true,
            _ => {}
        }
    }
    
    // Return highest abstraction level present
    if has_high { Ok(AbstractionLevel::High) }
    else if has_mid { Ok(AbstractionLevel::Mid) }
    else { Ok(AbstractionLevel::Low) }
}
```

---

**End of Document**

