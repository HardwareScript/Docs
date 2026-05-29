# Language Specification (v0.1.5)

**Base Documentation**: [v0.1.4 LANGUAGE-SPEC.md](../v0.1.4/LANGUAGE-SPEC.md)  
**Status**: Incremental update - Module syntax  
**Version**: 0.1.5

---

## ⚠️ CRITICAL: Hardware Script Syntax (NOT Software!)

**Hardware Script follows Python/Ruby-style syntax, NOT C/Rust/JavaScript!**

### The Three Syntax Rules:
1. **NO braces `{}`** - Use colons `:` and indentation
2. **NO arrows `=>`** - Use colons `:` for all mappings  
3. **NO semicolons `;`** - Newlines are statement separators

### Common Mistakes to Avoid:
```hw
# ❌ WRONG (Software Heritage trap):
match state {
    0 => A
    1 => B
}

# ✅ CORRECT (Hardware Script):
match state:
    0: A
    1: B
    else: C
```

**See "Software Heritage" trap in [SOFTWARE-HERITAGE.md](SOFTWARE-HERITAGE.md)**

---

## What's New in v0.1.5

This document covers ONLY the new `define module` syntax added in v0.1.5. For all other syntax (materials, profiles, components, spaces, etc.), see [v0.1.4 LANGUAGE-SPEC.md](../v0.1.4/LANGUAGE-SPEC.md).

**New Syntax**:
1. `define module` blocks with `logic:` for behavioral synthesis
2. Control flow in modules (`for`, `if`/`else`)
3. Module instantiation using `add` keyword (same as components)
4. Logic synthesis primitives (`let`, `let mut`, `match`, `Reg()`)
5. Automatic width inference for logic expressions
6. **Dual Coordinate System** - Physical measurements in component placement

---

## Coordinate System (v0.1.5 - Physical-Intent)

**Core Principle**: Atoms don't stretch. Physical mass is absolute. Position can be relative.

### The Three Rules

**Rule 1: Physical Mass is Absolute**  
Component sizes, space dimensions, trace widths, via holes MUST use absolute SI units (`m`, `cm`, `mm`, `µm`, `nm`, `pm`).

**Rule 2: Layers are Logical Indices**  
Z-axis represents logical layer indices (1=Top, 2=Inner1, etc.), not physical measurements. Use integers only.

**Rule 3: Placement Can Be Relative**  
X/Y coordinates support BOTH absolute measurements AND percentages for scale-invariant positioning.

### Valid Syntax Examples

#### Absolute Positioning (Traditional)
```hw
define space "FixedBoard":
    dimensions: 100mm by 100mm by 1.6mm
    grid: 10000 by 10000 by 2
    
    # Fixed position - works but brittle if board size changes
    add MCU_ESP32 named Brain at [x: 50mm, y: 50mm, z: 1]
    add Resistor_0805 named R1 at [x: 10mm, y: 15mm, z: 1]
```

#### Relative Positioning (Scale-Invariant)
```hw
define space "FlexibleBoard":
    dimensions: 100mm by 100mm by 1.6mm
    grid: 10000 by 10000 by 2
    
    # Centers on ANY board size (50% of 100mm = 50mm)
    add MCU_ESP32 named Brain at [x: 50%, y: 50%, z: 1]
    
    # 10% from left edge, 90% from top (10mm, 90mm on 100mm board)
    add Antenna named WiFi at [x: 10%, y: 90%, z: 1]
```

**Scale-Invariant Benefit**: If you change `dimensions: 100mm by 100mm` to `dimensions: 80mm by 80mm`, the Brain stays centered at 40mm, 40mm automatically!

#### Mixed Positioning (Absolute + Relative)
```hw
define space "SmartWatch":
    dimensions: 40mm by 40mm by 1.6mm
    grid: 4000 by 4000 by 4
    
    # Center horizontally, but 5mm from top edge
    add MCU_ESP32 named Brain at [x: 50%, y: 5mm, z: 1]
    
    # 10mm from left, centered vertically
    add Battery named Power at [x: 10mm, y: 50%, z: 1]
    
    # 25% from left (10mm), 75% from top (30mm)
    add Sensor named Accel at [x: 25%, y: 75%, z: 1]
```

#### Math Expressions with Percentages
```hw
define space "LED_Bar":
    dimensions: 100mm by 10mm by 1.6mm
    grid: 10000 by 1000 by 2
    
    # Distribute 8 LEDs evenly across the board
    # Each LED at 12.5mm intervals, centered vertically
    for i in 0..7:
        add LED_0805 named Light[i] at [x: 12.5mm + (i * 12.5mm), y: 50%, z: 1]
    
    # Alternative: Use percentages for X too
    # for i in 0..7:
    #     add LED_0805 named Light[i] at [x: 12.5% + (i * 12.5%), y: 50%, z: 1]
```

#### Complex Layout Example
```hw
define space "QuadCore_Processor":
    dimensions: 200mm by 200mm by 1.6mm
    grid: 20000 by 20000 by 4
    
    # Place 4 CPU cores in a grid pattern using percentages
    add ProcessorCore named Core0 at [x: 25%, y: 25%, z: 1]  # Top-left
    add ProcessorCore named Core1 at [x: 75%, y: 25%, z: 1]  # Top-right
    add ProcessorCore named Core2 at [x: 25%, y: 75%, z: 1]  # Bottom-left
    add ProcessorCore named Core3 at [x: 75%, y: 75%, z: 1]  # Bottom-right
    
    # Center the memory controller
    add MemoryController named MemCtrl at [x: 50%, y: 50%, z: 1]
    
    # Place power regulators along the edges
    add VoltageRegulator named VReg1 at [x: 10mm, y: 50%, z: 1]   # Left edge
    add VoltageRegulator named VReg2 at [x: 190mm, y: 50%, z: 1]  # Right edge
    add VoltageRegulator named VReg3 at [x: 50%, y: 10mm, z: 1]   # Top edge
    add VoltageRegulator named VReg4 at [x: 50%, y: 190mm, z: 1]  # Bottom edge
```

#### Percentage Calculation Examples

On a `100mm by 100mm` board:
- `x: 50%` → `50mm` (center)
- `x: 25%` → `25mm` (quarter from left)
- `x: 10%` → `10mm` (near left edge)
- `y: 90%` → `90mm` (near bottom edge)

On a `40mm by 40mm` board (same code):
- `x: 50%` → `20mm` (center)
- `x: 25%` → `10mm` (quarter from left)
- `x: 10%` → `4mm` (near left edge)
- `y: 90%` → `36mm` (near bottom edge)

**The Power**: Write once, works on any board size!

### Deprecated Syntax

```hw
# ❌ Raw grid indices (deprecated in v0.1.5)
add Resistor named R1 at [50, 50, 1]

# ❌ Z-axis measurements (invalid - Z must be layer index)
add Resistor named R1 at [x: 10mm, y: 10mm, z: 1.6mm]

# ❌ Positional syntax without names (deprecated)
add Resistor named R1 at [10mm, 10mm, 1]
```

**Why deprecated**: Raw grid indices were brittle and required manual calculation. The new system uses physical measurements and percentages for clarity and scale-invariance.

---

## How Percentages Work (Implementation Details)

### Compile-Time Conversion

When the compiler encounters a percentage coordinate, it converts it to nanometers based on the space dimensions:

```rust
// For a 100mm by 100mm board:
// x: 50% → 50% of 100mm → 50mm → 50,000,000 nanometers

let x_nm = (percentage / 100.0) * space_width_nm;
let y_nm = (percentage / 100.0) * space_height_nm;
```

### Mixed Expression Evaluation

The compiler evaluates mixed expressions at compile time:

```hw
# Example: x: 50% + 5mm on a 100mm board
# Step 1: Evaluate 50% → 50mm → 50,000,000nm
# Step 2: Evaluate 5mm → 5,000,000nm
# Step 3: Add them → 55,000,000nm (55mm)
add Component at [x: 50% + 5mm, y: 10mm, z: 1]
```

### Why This is Better Than CSS

CSS has `em`, `rem`, `vh`, `vw` because web layouts are fluid and relative to viewport/font size.

Hardware Script doesn't need these because:

1. **Physical reality is absolute**: A chip is always the same size
2. **Math is better than magic units**: `12.5mm + (i * 12.5mm)` is clearer than `justify-content: space-between`
3. **Percentages are enough**: `50%` for centering, combined with absolute offsets, covers all use cases
4. **Compile-time expressions**: For-loops and arithmetic replace CSS Flexbox/Grid

### The "Flexbox for Hardware" Pattern

```hw
# CSS equivalent: display: flex; justify-content: center; align-items: center;
add Component at [x: 50%, y: 50%, z: 1]

# CSS equivalent: display: flex; justify-content: space-between;
for i in 0..3:
    add Component[i] at [x: 25% + (i * 25%), y: 50%, z: 1]
    # Results: 25%, 50%, 75% (evenly distributed)
```

---

---

## Component Layout Clarifications (v0.1.5)

### Pin Positions: Strictly 2D Positional Offsets

Pin positions inside `layout:` blocks use **2D positional coordinates only**:

```hw
define component "Resistor_0805":
    pins:
        Pin1
        Pin2
    
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.6mm)
        pin_positions:
            Pin1 at [0mm, 0.625mm]      # ✅ Correct: 2D positional [X, Y]
            Pin2 at [2.0mm, 0.625mm]    # ✅ Correct
```

**Array Pin Support** (NEW in v0.1.5):

Array pins can now be referenced in `pin_positions:` and `pad_shapes:` blocks:

```hw
define component "BGA256":
    pins:
        Ball[256]
    
    layout:
        shape: Rectangle(17mm, 17mm, 1.6mm)
        pin_positions:
            Ball[0] at [0mm, 0mm]       # ✅ Array syntax works
            Ball[1] at [1mm, 0mm]       # ✅ Array syntax works
            Ball[255] at [15mm, 15mm]   # ✅ Array syntax works
        pad_shapes:
            Ball[0]: Circle(0.4mm)      # ✅ Array syntax in pad_shapes
            Ball[255]: Circle(0.4mm)    # ✅ Array syntax in pad_shapes
```

**Invalid syntax** (will cause parser error):
```hw
pin_positions:
    Pin1 at [x: 0mm, y: 0.625mm, z: 0mm]  # ❌ Wrong: named 3D coordinates
    Pin1 at [0mm, 0.625mm, 0mm]            # ❌ Wrong: 3D positional
```

**Rationale**: 
- Pins are offsets from the component's anchor point, not absolute space coordinates
- SMD components are flat - Z is always 0 (pins are on the surface)
- Keeping syntax clean: `[0mm, 0.625mm]` is clearer than `[x: 0mm, y: 0.625mm, z: 0mm]`
- Named coordinates are reserved for component placement in spaces

### Metadata Block: String-Only Rule

The `metadata:` block strictly enforces string values for all fields. This is intentional architectural design.

**Purpose**: 
- Generate Bill of Materials (BOM) CSV files
- Provide datasheet links
- Give human-readable descriptions
- Export to spreadsheets and documentation

**Rule**: Everything in `metadata:` MUST be a string.

**Valid syntax**:
```hw
metadata:
    manufacturer: "TDK"
    layer_count: "500"           # ✅ Correct: String
    special_features: "Low ESL"
    notes: "High capacitance MLCC with 500 internal layers"
```

**Invalid syntax**:
```hw
metadata:
    layer_count: 500             # ❌ Wrong: Raw number (Error S14)
    is_polarized: true           # ❌ Wrong: Boolean
    voltage: 25V                 # ❌ Wrong: Measurement
```

**Why strings only**:
- Clean, predictable BOM generation
- No type ambiguity in documentation
- The compiler doesn't have to guess how to format numbers when exporting to CSV/spreadsheets
- If you want to say "500", write `"500"` or `"500 layers"`

**For physics calculations**: Use the `electrical:` block with measurements:
```hw
electrical:
    capacitance: 100µF           # ✅ Correct: Measurement type
    voltage_rating: 25V          # ✅ Correct: Measurement type
    esr_100khz: 0.003Ω          # ✅ Correct: Measurement type
```

**The Separation of Concerns**:
- `metadata:` = Strings only (human-readable, BOM export)
- `electrical:` = Measurements and numbers (physics calculations)

---

## Number Literals (v0.1.5 - Scientific Notation Support)

Hardware Script supports multiple number formats for flexibility and precision.

### Integer Literals

**Decimal**: `42`, `1000`, `0`, `+42`, `-1000`  
**Hexadecimal**: `0xDEADBEEF`, `0xFF`, `0x00`  
**Binary**: `0b11111111`, `0b1010`, `0b0`  
**Octal**: `0o777`, `0o644`, `0o0`

**Positive/Negative Prefixes**: All integer formats support explicit `+` or `-` prefixes.

**Example**:
```hw
electrical:
    register_value: 0xDEADBEEF  # Hex for hardware registers
    bitmask: 0b11111111         # Binary for bit patterns
    permissions: 0o755          # Octal for Unix-style permissions
    positive_value: +100        # Explicit positive
    negative_value: -50         # Explicit negative
```

### Float Literals

**Standard Notation**: `3.14159`, `0.001`, `123.456`, `+3.14159`, `-0.001`  
**Scientific Notation**: `1.23e10`, `2.5e-6`, `1e-15`, `+1.23e10`, `-2.5e-6`

**Positive/Negative Prefixes**: All float formats support explicit `+` or `-` prefixes.

**Example**:
```hw
electrical:
    capacitance: 1.23456789e-15F  # 1.23456789 femtofarads
    resistance: 1.5e6Ω            # 1.5 megaohms
    q_factor: 1.23456789e1        # 12.3456789 (unitless)
    positive_voltage: +5.0V       # Explicit positive
    negative_voltage: -12.0V      # Explicit negative
```

### Unitless Scientific Notation (NEW in v0.1.5)

**Problem Solved**: Before v0.1.5, unitless scientific notation caused lexer errors because the Float token had lower priority than the Measurement token.

**Now Works**:
```hw
electrical:
    q_factor: 1.23456789e1        # Q-factor is unitless
    gain: 2.5e3                   # Voltage gain (unitless)
    ratio: 1.5e-2                 # Percentage as decimal
    test_positive: 1.5e10
    test_negative: 2.5e-6
```

**Technical Details**: Float token priority increased from 2 to 12 (higher than Measurement token's priority of 10), ensuring scientific notation is recognized before attempting to parse as a measurement.

### Measurements (with Units)

**Syntax**: `Number Unit` (e.g., `100mm`, `1.5kΩ`, `10µF`, `+10V`, `-40°C`)

**Positive/Negative Prefixes**: All measurements support explicit `+` or `-` prefixes.

**Supported Units**:
- Length: `m`, `cm`, `mm`, `µm`, `nm`, `pm`
- Resistance: `Ω`, `kΩ`, `MΩ`
- Capacitance: `F`, `µF`, `nF`, `pF`, `fF`
- Inductance: `H`, `mH`, `µH`, `nH`
- Voltage: `V`, `mV`, `kV`
- Current: `A`, `mA`, `µA`
- Frequency: `Hz`, `kHz`, `MHz`, `GHz`
- Temperature: `°C`, `K`
- Percentage: `%`

**Example**:
```hw
electrical:
    capacitance: 100µF
    voltage_rating: 25V
    esr_100khz: 0.003Ω
    tolerance_positive: +80%      # Explicit positive tolerance
    tolerance_negative: -20%      # Explicit negative tolerance
    temp_coefficient: -0.05%/°C
    operating_temp_min: -40°C     # Negative temperature
    operating_temp_max: +85°C     # Positive temperature
```

### Extreme Values (Stress Tested)

Hardware Script handles extreme values correctly:

**Tiny Values**:
```hw
capacitance: 0.000001pF          # 1 attofarad
capacitance: 1.23456789e-15F     # 1.23456789 femtofarads
```

**Large Values**:
```hw
capacitance: 3000F               # 3000 farads (supercapacitor)
resistance: 1e10Ω                # 10 gigaohms
```

**Negative Values**:
```hw
temperature: -273.15°C           # Absolute zero
voltage: -55V                    # Negative supply
temp_coefficient: -0.05%/°C      # Negative temperature coefficient
```

---

## Keywords as Identifiers (v0.1.5 - Soft Keyword Support)

Hardware Script implements "soft keywords" - keywords that can be used as identifiers in specific contexts.

### The Problem (Before v0.1.5)

Keywords like `tolerance`, `trace`, `via`, `clearance` couldn't be used as parameter names or variable references:

```hw
# ❌ BEFORE (Error S14: Unexpected the 'tolerance' keyword)
define component "Resistor" (resistance: Measurement, tolerance: Measurement):
    electrical:
        resistance: resistance
        tolerance: tolerance  # Error: keyword as value
```

### The Solution (v0.1.5)

The parser now accepts keywords in contexts where identifiers are expected:

**As Parameter Names**:
```hw
define component "Resistor" (resistance: Measurement, tolerance: Measurement):
    electrical:
        resistance: resistance
        tolerance: tolerance  # ✅ Works!
```

**As Property Names**:
```hw
electrical:
    tolerance: 5%       # ✅ 'tolerance' is a keyword, but works as property name
    trace: 0.2mm        # ✅ 'trace' is a keyword, but works as property name
    via: 0.3mm          # ✅ 'via' is a keyword, but works as property name
    clearance: 0.5mm    # ✅ 'clearance' is a keyword, but works as property name
```

**Note**: The `±` symbol is not valid Hardware Script syntax. Use separate properties for positive/negative tolerances:
```hw
electrical:
    tolerance_positive: 5%
    tolerance_negative: 5%
```

**As Variable References**:
```hw
define component "Test" (tolerance: Measurement):
    electrical:
        tolerance: tolerance  # ✅ Parameter reference works
```

### Supported Soft Keywords

All Hardware Script keywords can be used as identifiers in appropriate contexts:
- `tolerance`, `category`, `material`, `trace`, `via`, `clearance`
- `electrical`, `mechanical`, `thermal`, `optical`
- `pins`, `layout`, `metadata`, `routing`
- And all other language keywords

### Technical Implementation

**Lexer**: Always outputs keywords (stays "dumb")  
**Parser**: Accepts keywords in identifier contexts (stays "smart")

This is the same approach used by Rust (`async`, `await`), C# (`var`, `dynamic`), and TypeScript (`type`, `interface`).

---

## Unicode Support and Identifier Restrictions (v0.1.5 - Manufacturing Reality)

**CRITICAL**: Hardware Script has strict rules about where Unicode is allowed. This is NOT a limitation - it's an architectural requirement for manufacturing export.

### The Three Unicode Domains

**1. Strings (Unicode ALLOWED)**  
Strings can contain ANY Unicode characters - they're for human documentation only:

```hw
metadata:
    manufacturer: "Test Corp 测试公司 テスト会社"
    notes: "Greek: αβγδε, Math: ∑∫∂∇, Emoji: 📡📶⚡, RTL: العربية"
    description: "All punctuation: !@#$%^&*()_+-=[]{}|\\;':\",./<>?`~"
```

**2. Measurement Units (Unicode ALLOWED)**  
SI units use Unicode symbols as defined by international standards:

```hw
electrical:
    resistance: 4.7kΩ      # ✅ Ω (Omega) is a standard SI symbol
    capacitance: 100µF     # ✅ µ (micro) is a standard SI prefix
    temperature: 85°C      # ✅ ° (degree) is a standard SI symbol
```

**3. Identifiers (Unicode FORBIDDEN)**  
Component names, pin names, net names, variable names MUST be ASCII-only:

```hw
# ❌ WRONG - Unicode in identifiers
pins:
    Signal_α     # Error S11: Invalid character 'α'
    Signal_β     # Error S11: Invalid character 'β'
    Ground_∞     # Error S11: Invalid character '∞'

# ✅ CORRECT - ASCII identifiers only
pins:
    Signal_Alpha
    Signal_Beta
    Ground_Inf
```

### Why Identifiers Must Be ASCII

**Manufacturing Export Reality**: Hardware Script compiles to industry-standard manufacturing formats:

1. **Gerber files** (PCB fabrication) - ASCII-only format from the 1970s
2. **GDSII files** (Silicon layout) - ASCII-only format from the 1980s
3. **SPICE netlists** (Circuit simulation) - ASCII-only format
4. **Excellon drill files** (PCB drilling) - ASCII-only format

If you name a pin `Signal_α`, the compiler would eventually crash when the Gerber Emitter tries to write a non-ASCII character into a `.gtl` text file, potentially ruining a manufacturing run.

**The Rule**: If a name will appear in a manufacturing file (component names, pin names, net names), it MUST be ASCII.

### Valid Identifier Syntax

**Pattern**: `[a-zA-Z_][a-zA-Z0-9_]*`

**Valid Examples**:
```hw
pins: VCC, GND, Signal_A, Data_Bus_0, RF_In, Power_3V3
```

**Invalid Examples**:
```hw
pins: Signal_α, Pin_Ω, Net_∞, Data_µ  # All rejected by lexer
```

### The Tri-Fold Case Sensitivity Rule

Hardware Script has THREE domains with different case rules:

1. **Software Domain (Strictly Lowercase)**: Keywords (`define`, `space`, `add`, `route`) must be lowercase
2. **Physics Domain (SI Standard)**: Units must match SI casing (`mV`, `kΩ`, `MHz`)
3. **User Domain (ASCII, Case-Sensitive)**: Identifiers (`ESP32`, `MainPower`) are exact-match ASCII

### Why This Is Correct Architecture

The agent tried to use Greek letters in pin names and called the rejection a "compiler gap." This is WRONG.

The compiler correctly rejected `Signal_α` because:
- It protects users from manufacturing export failures
- It enforces industry-standard compatibility
- It prevents $10,000 manufacturing runs from failing due to encoding issues

**Verdict**: If the lexer rejects Unicode in identifiers, it's doing its job perfectly. This is NOT a bug - it's a feature that saves you from catastrophic failures at Layer 5 (Manufacturing Export).

---

## Logic Synthesis (v0.1.5 - "Custom Rust" Paradigm)

**What it is**: Native behavioral synthesis that compiles high-level logic expressions into physical hardware components

**Why it matters**: Eliminates dependency on external EDA tools (Yosys, Vivado) by implementing synthesis directly in the compiler

### The `logic:` Block

Modules can contain a `logic:` block that defines behavioral hardware using Rust-like syntax:

```hw
define module "Counter8":
    pins:
        input Clk
        input Rst
        output Count[8]
    
    logic:
        let count = Reg(clock: Clk, reset: Rst, init: 0)
        count.next = count + 1
        Count = count
```

**Key Point**: The `logic:` block uses "Custom Rust" syntax - it looks like Rust but compiles to hardware primitives.

### Bare-Metal Primitives

**Wires**: `let A = B + C`
- Declares a wire that carries the result of an expression
- Width is automatically inferred from operands
- Immutable (cannot be reassigned)

**Multiplexers**: `let mut result = 0`
- Declares a mutable wire (can be reassigned in different branches)
- Compiles to a multiplexer with inputs from all assignments
- Final value depends on control flow

**Registers**: `let state = Reg(clock: Clk, reset: Rst, init: 0)`
- Declares a D flip-flop with clock, reset, and initial value
- Creates both `state` (current value) and `state.next` (next value)
- Breaks combinational loops

### Hardware Control Flow

**If/Else** (2-to-1 MUX):
```hw
logic:
    let mut result = 0
    if condition {
        result = A
    } else {
        result = B
    }
    Output = result
```

Compiles to: `Mux2to1(sel: condition, in0: B, in1: A, out: result)`

**Match** (N-to-1 MUX):
```hw
logic:
    Result = match Selector:
        0: 0x00
        1: 0x01
        2: 0x02
        3: 0x03
        else: 0xFF
```

Compiles to: `MuxNto1(sel: Selector, inputs: [0x00, 0x01, 0x02, 0x03, 0xFF], out: Result)`

**CRITICAL SYNTAX RULES:**
- NO braces `{}`: Hardware Script uses colons `:` and indentation (Python/Ruby style)
- NO arrows `=>`: Use colons `:` for all mappings
- NO semicolons `;`: Newlines are statement separators
- Use `else:` for default case (NOT `_` or `default`)

### Automatic Width Inference

The compiler infers bit-widths bottom-up from operands:

```hw
logic:
    let A[8] = DataIn      # A is 8 bits (explicit)
    let B[8] = DataIn2     # B is 8 bits (explicit)
    let Sum = A + B        # Sum is 9 bits (inferred: 8+8 needs 9 bits)
    let Truncated[8] = Sum # Truncated is 8 bits (explicit truncation)
```

**Warning L10**: Implicit truncation (assigning 9-bit value to 8-bit wire) generates a warning, not an error.

### Operator Overloading

Math operators compile to Standard Library components:

| Operator | Component | Example |
|----------|-----------|---------|
| `+` | RippleCarryAdder | `let Sum = A + B` |
| `-` | Subtractor | `let Diff = A - B` |
| `<<` | BarrelShifter | `let Shifted = A << 2` |
| `==` | Comparator | `let Equal = A == B` |
| `&` | BitwiseAnd | `let Result = A & B` |
| `\|` | BitwiseOr | `let Result = A \| B` |

### Enums and Structs

**Enums** (for state machines):
```hw
define enum CpuState:
    Fetch
    Decode
    Execute
    WriteBack

define module "SimpleCPU":
    pins:
        input Clk, Rst
        output State[2]
    
    logic:
        let state = Reg(clock: Clk, reset: Rst, init: CpuState.Fetch)
        
        match state:
            CpuState.Fetch: state.next = CpuState.Decode
            CpuState.Decode: state.next = CpuState.Execute
            CpuState.Execute: state.next = CpuState.WriteBack
            CpuState.WriteBack: state.next = CpuState.Fetch
        
        State = state as [2]
```

**Structs** (for wire bundling):
```hw
define struct Instruction:
    opcode[4]
    rs1[5]
    rs2[5]
    rd[5]

define module "Decoder":
    pins:
        input RawInstr[32]
        output Opcode[4]
    
    logic:
        let instr = RawInstr as Instruction
        Opcode = instr.opcode
```

### Complete Example: 8-bit Counter with Enable

```hw
define module "Counter8":
    pins:
        input Clk
        input Rst
        input Enable
        output Count[8]
        output Overflow
    
    logic:
        let count = Reg(clock: Clk, reset: Rst, init: 0)
        let mut next_count = count
        
        if Enable {
            next_count = count + 1
        }
        
        count.next = next_count
        Count = count
        Overflow = (count == 255) & Enable
```

**Generated Components**:
- 1× DFlipFlop8 (for `count` register)
- 1× RippleCarryAdder8 (for `count + 1`)
- 1× Mux2to1_8bit (for `if Enable`)
- 1× Comparator8 (for `count == 255`)
- 1× BitwiseAnd (for `& Enable`)

---

## Module Definition

**Purpose**: Define reusable hardware patterns with parameterized routing

**Syntax**:
```hw
define module "ModuleName" {
    pins {
        input PinName[ArraySize]
        output PinName[ArraySize]
        // ... more pins
    }
    
    // Routing statements with control flow
    for variable in start..end {
        route Source -> Destination
        
        if condition {
            // conditional routing
        } else {
            // alternative routing
        }
    }
}
```

### Pin Declarations

**Syntax**: `direction PinName` or `direction PinName[Size]`

**Directions**: `input`, `output`, `bidirectional`

**Examples**:
```hw
pins {
    input A[8]           // 8-bit input bus
    input B[8]           // 8-bit input bus
    input Cin            // Single input
    output Sum[8]        // 8-bit output bus
    output Cout          // Single output
    bidirectional Data[16]  // 16-bit bidirectional bus
}
```

**Key Point**: Module pins are scoped to the module. They do NOT appear in the global Symbol Table.

---

## Control Flow

### For Loops

**Syntax**: `for variable in start..end { statements }`

**Semantics**: Ruby-style inclusive ranges (both endpoints included)

**Examples**:
```hw
// Iterate 0, 1, 2, 3
for i in 0..3 {
    route A[i] -> B[i]
}

// Iterate 1, 2, 3, 4, 5
for j in 1..5 {
    route Bus[j] -> Component[j].Pin
}

// Nested loops
for i in 0..3 {
    for j in 0..3 {
        route Matrix[i][j] -> Output[i*4 + j]
    }
}
```

**Evaluation**: Loops are unrolled at compile time (comptime evaluation)

### Conditionals

**Syntax**: `if condition { statements } else { statements }`

**Operators**: `==`, `!=`, `<`, `>`

**Examples**:
```hw
// First element special case
for i in 0..7 {
    if i == 0 {
        route Cin -> Adder[0].Cin
    } else {
        route Adder[i-1].Cout -> Adder[i].Cin
    }
}

// Range checks
for i in 0..15 {
    if i < 8 {
        route LowBus[i] -> Output[i]
    } else {
        route HighBus[i-8] -> Output[i]
    }
}
```

**Evaluation**: Conditions are evaluated at compile time using loop variable values

---

## Array Indexing

**Supported Expressions**:
- Literals: `A[0]`, `B[7]`
- Variables: `A[i]`, `B[j]`
- Arithmetic: `A[i+1]`, `B[i-1]`, `C[i*2]`

**Examples**:
```hw
// Direct indexing
route A[0] -> B[0]

// Variable indexing
for i in 0..7 {
    route A[i] -> B[i]
}

// Arithmetic indexing
for i in 0..6 {
    route A[i] -> B[i+1]      // Shift right
    route C[i+1] -> D[i]      // Shift left
}

// Carry chain
for i in 0..7 {
    if i == 0 {
        route GND -> Adder[0].Cin
    } else {
        route Adder[i-1].Cout -> Adder[i].Cin
    }
}
```

**Limitations**:
- Only `+` and `-` operators supported
- No multiplication or division in indices (use explicit arithmetic)
- All indices must resolve to compile-time constants

---

## Pin References

**Syntax**: `PinName` or `Component[Index].PinName`

**Examples**:
```hw
// Module pin reference
route A[i] -> Sum[i]

// Component pin reference
route A[i] -> Adder[i].A
route Adder[i].Sum -> Sum[i]

// Nested component reference
route Bus[i] -> Chip[i].Data[j]
```

**Key Point**: Component references like `Adder[i]` assume components are placed in the space. Module flattening does NOT generate component placements.

---

## Module Instantiation

**Syntax**: `add ModuleName named InstanceName at [coordinates]`

**Location**: Inside `define space` blocks

**Key Point**: Modules are instantiated using the same `add` keyword as atomic components, maintaining perfect syntax consistency.

**Example**:
```hw
define space "ALU" {
    dimensions: 100mm by 100mm by 2mm
    grid: 10µm
    
    // Instantiate module (same syntax as components)
    add RippleCarryAdder64 named MainAdder at [x:10, y:10, z:1]
}
```

**Evaluation**:
1. Parser encounters `add ModuleName named InstanceName`
2. Compiler checks if ModuleName is a module (not a component)
3. Module flattener unrolls loops and evaluates conditionals
4. Flattened components and routes are prefixed with InstanceName
5. Module expands into physical components and nets in the IR

---

## Complete Example: 8-bit Ripple Carry Adder

```hw
// Define the module
define module "8-bit Ripple Carry Adder" {
    pins {
        input A[8]
        input B[8]
        input Cin
        output Sum[8]
        output Cout
    }
    
    // Connect inputs to adders
    for i in 0..7 {
        route A[i] -> Adder[i].A
        route B[i] -> Adder[i].B
        route Adder[i].Sum -> Sum[i]
    }
    
    // Carry chain
    for i in 0..7 {
        if i == 0 {
            route Cin -> Adder[0].Cin
        } else {
            route Adder[i-1].Cout -> Adder[i].Cin
        }
    }
    
    // Final carry out
    route Adder[7].Cout -> Cout
}

// Use the module in a space
define space "Calculator" {
    dimensions: 50mm by 50mm by 2mm
    grid: 10µm
    
    // Instantiate the module (generates components and routes)
    add RippleCarryAdder8 named MainAdder at [x:10, y:10, z:1]
    
    // Optional: Define physical layout for module components
    layout MainAdder:
        for i in 0..7:
            Adder[i] at [x: 10 + (i*5), y: 10, z: 1]
}
```

**Generated Routes** (33 total):
- 8 routes: `A[0..7] -> Adder[0..7].A`
- 8 routes: `B[0..7] -> Adder[0..7].B`
- 8 routes: `Adder[0..7].Sum -> Sum[0..7]`
- 8 routes: `Adder[0..6].Cout -> Adder[1..7].Cin`
- 1 route: `Cin -> Adder[0].Cin`
- 1 route: `Adder[7].Cout -> Cout`

---

## Grammar

```ebnf
ModuleDefinition ::= "define" "module" StringLiteral "{" PinsBlock ModuleStatement* "}"

PinsBlock ::= "pins" "{" PinDeclaration* "}"

PinDeclaration ::= Direction Identifier ("[" Number "]")? Newline

Direction ::= "input" | "output" | "bidirectional"

ModuleStatement ::= RouteStatement
                  | ForLoop
                  | IfConditional

ForLoop ::= "for" Identifier "in" Number ".." Number "{" ModuleStatement* "}"

IfConditional ::= "if" Condition "{" ModuleStatement* "}" ("else" "{" ModuleStatement* "}")?

Condition ::= ArrayIndex ComparisonOp ArrayIndex

ComparisonOp ::= "==" | "!=" | "<" | ">"

ArrayIndex ::= Number
             | Identifier
             | Identifier "+" Number
             | Identifier "-" Number

PinReference ::= Identifier ("[" ArrayIndex "]")?
               | Identifier ("[" ArrayIndex "]")? "." Identifier ("[" ArrayIndex "]")?
```

---

## Semantic Rules

1. **Module Scope**: Module pins are scoped to the module and do NOT pollute the global namespace
2. **Comptime Evaluation**: All control flow is evaluated at compile time (no runtime loops)
3. **Inclusive Ranges**: `0..7` means [0,1,2,3,4,5,6,7] (both endpoints included)
4. **Array Bounds**: Compiler does NOT check array bounds during flattening (deferred to routing phase)
5. **Component Existence**: Module assumes components exist in space (no validation during flattening)
6. **No Recursion**: Modules cannot instantiate other modules (v0.1.5 limitation)

---

## Error Handling

**Compile-Time Errors**:
- Undefined module name in `use module`
- Invalid loop range (start > end)
- Undefined loop variable in condition
- Malformed array index expression

**Deferred Errors** (caught in routing phase):
- Array index out of bounds
- Undefined component reference
- Undefined pin reference

---

## References

**Implementation**:
- [v0.1.5 COMPILER-INTERNALS.md](./COMPILER-INTERNALS.md) - Module flattening and comptime evaluation
- [v0.1.4 LANGUAGE-SPEC.md](../v0.1.4/LANGUAGE-SPEC.md) - Base language syntax

**Code**:
- `hwc/crates/hwc-parser/src/ast/module.rs` - Module AST
- `hwc/crates/hwc-parser/src/parser/definitions/module.rs` - Module parser
- `hwc/crates/hwc-compiler/src/module_flattener.rs` - Comptime evaluation
