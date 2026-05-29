# Language Specification (v0.1.6)

**Base Documentation**: [v0.1.5 LANGUAGE-SPEC.md](../v0.1.5/LANGUAGE-SPEC.md)  
**Status**: Major syntax overhaul - Universal Grammar  
**Version**: 0.1.6

---

## What's New in v0.1.6

This version represents a **complete syntax unification**. Nearly every aspect of the language has been refined for consistency and clarity.

**Major Changes**:
1. Drop `define` keyword - hardware types are now first-class keywords
2. Drop quotes on identifiers - bare identifiers for all type names
3. Universal list syntax - `[]` works everywhere
4. Universal assignment - `=` for all values and math
5. Flexible metadata/profile blocks - accept arbitrary key-value pairs
6. Simplified coordinate origin syntax
7. Property keys are identifiers, not reserved keywords

**For all other syntax (measurements, coordinates, routing, etc.)**, see [v0.1.5 LANGUAGE-SPEC.md](../v0.1.5/LANGUAGE-SPEC.md).

---

## Universal Grammar Rules

Hardware Script v0.1.6 follows three universal grammar rules that apply consistently across the entire language.

### Rule 1: Type-as-Keyword (No More `define`)

Hardware types are promoted to first-class language keywords. No more `define`. No more quotes around type names.

**Keywords**: `component`, `space`, `material`, `profile`, `module`, `enum`, `struct`

```hw
# ✅ v0.1.6 - Clean and consistent
material Copper:
    category: conductor

component Resistor_0805:
    pins: [A, B]

space Motherboard:
    dimensions: 100mm by 100mm by 2mm

module Counter8:
    pins: [input Clk, output Count[8]]

enum State:
    values: [Idle, Active, Done]

struct Instruction:
    opcode[4]
    rs1[5]
    rd[5]
```

```hw
# ❌ v0.1.5 - Verbose and inconsistent
define material "Copper":
    category: conductor

define component "Resistor_0805":
    pins:
        A, B

define space "Motherboard":
    dimensions: 100mm by 100mm by 2mm
```

### Rule 2: The Boundary Law (`:` for Facts, `=` for Actions)

This is the most important rule in Hardware Script. It separates the **Declarative World** (atoms at rest) from the **Behavioral World** (electrons in motion).

**The Colon (`:`)** has two jobs:

1. **Opening Blocks** (Universal) - Any line that expects indented children ends with `:`
2. **Static Facts** (Declarative) - Property assignments that define what something IS

```hw
# Job 1: Opening blocks (everywhere)
component Resistor:              # Opens component block
    electrical:                  # Opens electrical block
        resistance: 10kΩ         # Job 2: Static fact

logic:                           # Opens logic block
    if count = 0:                # Opens if block (note: comparison uses =, but : opens block)
        Output = 1               # Action uses =
    else:                        # Opens else block
        Output = 0               # Action uses =

# Job 2: Static facts (declarative world)
pins: [A, B, C]
electrical:
    resistance: 10kΩ
    tolerance: 5%
    power_rating: 0.125W
metadata:
    manufacturer: "Yageo"
unit Microfarad:
    symbol: "µF"
    multiplier: 1e-6

# Component instantiation (static facts about the instance)
add Resistor_0805 (val: 10kΩ, tol: 5%) named R1 at [x: 10mm, y: 10mm]
```

**The Equals (`=`)** is for the Behavioral World - Dynamic Actions

Use `=` ONLY inside `logic:`, `test:`, or `pattern:` blocks. If it describes a calculation, state change, or dynamic relationship, use equals.

**Critical: Single `=` for BOTH assignment AND comparison.** Context determines meaning:

```hw
# Behavioral: These are actions and calculations
logic:
    # Assignment (standalone statement)
    let Sum = A + B
    count.next = count + 1
    
    # Comparison (after if/match, or in expression)
    if count = 0:                # Comparison
        Output = 1               # Assignment
    
    let is_ready = (status = 1)  # Comparison in expression
    
    Output = match State:        # Assignment
        Idle: 0                  # Match arms are expressions
        Active: 1
```

**The One-Sentence Rule**: "Use `:` to define what it IS or to open a block; use `=` to define what it DOES."

**Why This Matters**: This boundary prevents the "Verilog Trap" where `=` (blocking) and `<=` (non-blocking) cause race conditions. In Hardware Script, the punctuation itself tells you whether you're describing static structure or dynamic behavior.

---

### The Golden Truth Table

| Context | Operator | Usage Example |
|---------|----------|---------------|
| **Block Header** | `:` | `component Name:`, `if cond:`, `else:`, `logic:` |
| **Static Fact** | `:` | `tolerance: 5%`, `val: 10kΩ`, `origin: tl by t` |
| **Dynamic Action** | `=` | `let sum = a + b`, `count.next = count + 1` |
| **Equality Check** | `=` | `if count = 0:`, `let is_ready = (status = 1)` |
| **Collections** | `[]` | `pins: [A, B]`, `let bus = [bits, sign]` |
| **Hierarchy** | Indent | Tab or 4 spaces |

### Rule 3: Universal Lists (`[]` Everywhere)

Any collection of items can use bracket syntax. This is now the preferred and canonical form.

```hw
# Pins
pins: [VCC, GND, Data0, Data1, Clock]

# Array pins
pins: [Data[32], Addr[16]]

# Enum values
enum TrafficLight:
    values: [Red, Yellow, Green]

# Material constraints
allowed_dielectrics: [FR4, Rogers4350B, Air]

# Match expressions
Output = match Selector:
    cases: [
        0: ValueA,
        1: ValueB,
        2: ValueC
    ]
```

**Backward Compatibility**: Comma-separated and newline-separated lists still work, but `[]` is now the canonical form.

---

## Import Paths (Bare Identifiers)

Following the "No Quotes for Types/Names" rule, import paths use bare identifiers when possible. Quotes are ONLY needed if the path contains spaces.

```hw
# ✅ Bare identifier paths (preferred)
import logic/adders as Adders
import @std/components as Parts
import ../common/utils as Utils

# ✅ Quotes only for paths with spaces
import "Custom Path/Board.hw" as CustomBoard
import "My Components/resistors.hw" as MyResistors

# ❌ Unnecessary quotes (avoid)
import "logic/adders" as Adders
```

**The Rule**: If the path is a valid identifier (no spaces, no special characters), use it bare. If it has spaces or special characters, use quotes.

---

## Declaration Syntax Changes

### Materials

```hw
# ✅ v0.1.6
material Copper:
    category: conductor
    properties:
        resistivity: 1.68e-8Ω·m
        density: 8960kg/m³
        thermal_conductivity: 401W/(m·K)

material FR4:
    category: insulator
    properties:
        dielectric_constant: 4.5
        loss_tangent: 0.02
        thermal_conductivity: 0.3W/(m·K)
```

```hw
# ❌ v0.1.5
define material "Copper":
    category: conductor
    properties:
        resistivity: 1.68e-8Ω·m
        density: 8960kg/m³
```

### Components

```hw
# ✅ v0.1.6
component Resistor_0805:
    pins: [A, B]
    
    electrical:
        resistance = 10kΩ
        tolerance = 5%
        power_rating = 0.125W
    
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.6mm)
        pin_positions:
            A at [0mm, 0.625mm]
            B at [2.0mm, 0.625mm]
    
    metadata:
        manufacturer = "Yageo"
        package = "0805"
        datasheet = "https://example.com/resistor.pdf"

# Parametric components
component Resistor_0805 (resistance: Measurement, tolerance: Measurement):
    pins: [A, B]
    
    electrical:
        resistance = resistance
        tolerance = tolerance
        power_rating = 0.125W
```

```hw
# ❌ v0.1.5
define component "Resistor_0805":
    pins:
        A, B
    
    electrical:
        resistance: 10kΩ
        tolerance: 5%
```

### Spaces

```hw
# ✅ v0.1.6
space Motherboard:
    dimensions: 100mm by 100mm by 2mm
    grid: 10000 by 10000 by 4
    origin: tl by t
    
    # Add components
    add Resistor_0805 (resistance: 10kΩ, tolerance: 5%) named R1 at [x: 10mm, y: 10mm, z: 1]
    add Capacitor_0805 (capacitance: 100nF) named C1 at [x: 20mm, y: 10mm, z: 1]
    
    # Route connections
    route R1.B to C1.A
    route C1.B to GND
```

```hw
# ❌ v0.1.5
define space "Motherboard":
    dimensions: 100mm by 100mm by 2mm
    grid: 10000 by 10000 by 4
    origin: tl by t
    
    add Resistor_0805 named R1 at [x: 10mm, y: 10mm, z: 1]
```

### Modules

**Context-Aware Pin Directions** (v0.1.6 Architectural Feature):

Pin directions (`input`, `output`, `power`, `ground`, `inout`) are NOT global keywords. They are property-level identifiers that the module parser recognizes in context. This prevents keyword pollution and allows these words to be used freely elsewhere (e.g., as unit names in stdlib).

```hw
# ✅ v0.1.6 - Property-style role declarations (recommended)
module Inverter_Logic:
    # Context-aware: 'input' is recognized as a property name here
    input: VIN
    output: VOUT
    power: VDD
    ground: GND
    
    # Structural netlist
    add NMOS named M1
    route M1.drain to VOUT
    route M1.gate to VIN
    route M1.source to GND
    route M1.bulk to GND
    
    add PMOS named M2
    route M2.drain to VOUT
    route M2.gate to VIN
    route M2.source to VDD
    route M2.bulk to VDD

# ✅ v0.1.6 - Legacy bracket notation (also supported)
module Counter8:
    pins: [input Clk, input Rst, output Count[8]]
    
    logic:
        let count = reg(clock: Clk, reset: Rst, init: 0)
        count.next = count + 1
        Count = count

# ✅ v0.1.6 - Structural instantiation with comptime
module RippleCarryAdder8:
    pins: [input A[8], input B[8], input Cin, output Sum[8], output Cout]
    
    # Structural instantiation
    for i in 0..7:
        add FullAdder named Adder[i]
        
        route A[i] to Adder[i].A
        route B[i] to Adder[i].B
        route Adder[i].Sum to Sum[i]
        
        if i = 0:
            route Cin to Adder[0].Cin
        else:
            route Adder[i-1].Cout to Adder[i].Cin
    
    route Adder[7].Cout to Cout
```

**Why Context-Aware Parsing?**
- **Zero Keyword Pollution**: `input`, `output`, etc. can be used as identifiers in other contexts
- **Standard Library Compatibility**: No conflicts with units.hw or other stdlib files
- **Property-Based Semantics**: Follows the Boundary Law - roles are properties, not inline keywords
- **Parser Intelligence**: The module parser understands context - recognizes `input:` as a pin role declaration

```hw
# ❌ v0.1.5
define module "Counter8":
    pins:
        input Clk
        input Rst
        output Count[8]
    
    logic:
        let count = reg(clock: Clk, reset: Rst, init: 0)
        count.next = count + 1
        Count = count
```

### Enums and Structs

```hw
# ✅ v0.1.6 - Clean and minimal
enum CpuState:
    values: [Fetch, Decode, Execute, WriteBack]

enum TrafficLight:
    values: [Red, Yellow, Green]

struct Instruction:
    opcode[4]
    rs1[5]
    rs2[5]
    rd[5]

struct MemoryRequest:
    address[32]
    data[64]
    write_enable[1]
```

**Key Changes**:
- Enums use `values:` with bracket notation
- Structs are bare bit-width tables (no `fields:` keyword)
- Every line in a struct is `name[width]`

```hw
# ❌ v0.1.5
define enum CpuState:
    Fetch
    Decode
    Execute
    WriteBack

define struct Instruction:
    opcode[4]
    rs1[5]
    rs2[5]
    rd[5]
```

---

## Assignment Operator Unification

In v0.1.6, the `=` operator is used consistently for all value assignments, whether in property blocks or logic blocks.

### Property Blocks (Declarative World - Use `:`)

```hw
# ✅ v0.1.6 - Declarative properties use colons
electrical:
    resistance: 10kΩ
    tolerance: 5%
    power_rating: 0.125W
    voltage_rating: 50V

mechanical:
    length: 2.0mm
    width: 1.25mm
    height: 0.6mm

thermal:
    thermal_resistance: 100°C/W
    max_temp: 125°C

metadata:
    manufacturer: "Yageo"
    part_number: "RC0805FR-0710KL"
```

**Why colons?** These are static facts about the component. They don't change or calculate—they simply ARE.

### Logic Blocks (Behavioral World - Use `=`)

```hw
# ✅ v0.1.6 - Behavioral logic uses equals
logic:
    # Wire assignments (instant calculations)
    let Sum = A + B
    let Carry = (A & B) | (B & Cin) | (A & Cin)
    
    # Register updates (state changes)
    let count = reg(clock: Clk, reset: Rst, init: 0)
    count.next = count + 1
    
    # Comparisons (use single =, not ==)
    let is_zero = (count = 0)
    let is_max = (count = 255)
    
    # Output assignments (dynamic routing)
    Output = Sum
    CarryOut = Carry
```

**Why equals?** These are actions and calculations. They describe what the circuit DOES when electrons flow, not what it IS when powered off.

**Single `=` for Everything**: Hardware Script uses `=` for both assignment and comparison. The compiler knows the difference from context:
- Standalone: `count = 1` is assignment
- After `if`: `if count = 0` is comparison
- In parentheses: `let x = (a = b)` is comparison

### Where Both Worlds Meet

```hw
# DECLARATIVE WORLD (Use Colons)
component Resistor (val: Measurement):
    pins: [Pin1, Pin2]
    
    electrical:
        resistance: val
        max_voltage: 150V

# BEHAVIORAL WORLD (Use Equals)
    logic:
        let voltage = Pin1.v - Pin2.v
        let current = voltage / resistance
        
        # Comparison uses single =
        let is_overload = (current > 0.1A)
```

---

## Logic Operators (v0.1.6)

Hardware Script uses a minimal set of operators that map directly to hardware primitives.

### Bitwise/Logical Operators

| Operator | Word Form | Symbol | Description | Example |
|----------|-----------|--------|-------------|---------|
| AND | `and` | `&` | Bitwise AND | `a & b` or `a and b` |
| OR | `or` | `|` | Bitwise OR | `a | b` or `a or b` |
| NOT | `not` | `!` | Bitwise NOT | `!a` or `not a` |
| XOR | `xor` | (none) | Bitwise XOR | `a xor b` |

**Key Design Decisions**:

1. **Single symbols only**: No `&&` vs `&` confusion. One operator works for 1-bit or 64-bit values.
2. **XOR is word-only**: XOR is a specialty operator (parity, CRC, crypto). Keeping it as a word makes it stand out.
3. **No operator overloading**: Each symbol has exactly one meaning.

```hw
logic:
    # Bitwise operations
    let result = (a & b) | (c xor d)
    let inverted = !enable
    
    # Works on any bit width
    let bus_and = data[8] & mask[8]
    let parity = d0 xor d1 xor d2 xor d3
```

### Comparison Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equal to | `if count = 0` |
| `!=` | Not equal to | `if state != Idle` |
| `<` | Less than | `if temp < 100` |
| `>` | Greater than | `if voltage > 5V` |
| `<=` | Less than or equal | `if i <= 7` |
| `>=` | Greater than or equal | `if i >= 0` |

**Critical: Single `=` for comparison**. Hardware Script eliminates the `==` operator. The compiler knows from context whether `=` is assignment or comparison:

```hw
logic:
    # Assignment context (standalone)
    count = 5
    
    # Comparison context (after if)
    if count = 5:
        Output = 1
    
    # Comparison context (in expression)
    let is_ready = (status = 0x1)
```

**Why no `==`?** In C/Rust, `==` exists to prevent accidental assignment in conditions (`if (x = 5)`). In Hardware Script, assignments are statements, not expressions. You cannot accidentally assign in a condition. The double equals is unnecessary weight.

---

## List Syntax Unification

All lists in Hardware Script v0.1.6 support bracket notation `[]`. This is now the canonical and preferred form.

### Pin Lists

```hw
# ✅ v0.1.6 - Bracket notation (preferred)
pins: [VCC, GND, Data0, Data1, Clock, Reset]

# ✅ v0.1.6 - Array pins
pins: [Data[32], Addr[16], Control[8]]

# ✅ v0.1.6 - Mixed
pins: [VCC, GND, Data[8], Clock]

# ✅ Still supported (backward compatibility)
pins:
    VCC
    GND
    Data0
    Data1

# ✅ Still supported (backward compatibility)
pins: VCC, GND, Data0, Data1
```

### Enum Values

```hw
# ✅ v0.1.6 - Bracket notation (preferred)
enum State:
    values: [Idle, Running, Paused, Stopped]

# ✅ Still supported (backward compatibility)
enum State:
    values:
        Idle
        Running
        Paused
        Stopped
```

### Material Constraints

```hw
# ✅ v0.1.6 - Bracket notation (preferred)
profile StandardPCB:
    allowed_materials:
        conductors: [Copper, Silver, Gold]
        dielectrics: [FR4, Rogers4350B, Air]
        soldermask: [LPI_Green, LPI_Blue]

# ✅ Still supported (backward compatibility)
profile StandardPCB:
    allowed_materials:
        conductors:
            - Copper
            - Silver
        dielectrics:
            - FR4
            - Rogers4350B
```

---

## Flexible Metadata and Profile Blocks

In v0.1.6, `metadata` and `profile` blocks accept arbitrary key-value pairs. The compiler no longer crashes on unknown keys.

### Metadata Blocks

```hw
# ✅ v0.1.6 - Any key-value pair is valid
component CustomChip:
    pins: [VCC, GND, Data[16]]
    
    metadata:
        # Standard fields
        manufacturer = "ACME Corp"
        part_number = "AC-1234"
        datasheet = "https://example.com/datasheet.pdf"
        
        # Custom fields (no compiler error!)
        internal_code = "PROJ-2024-001"
        revision = "Rev C"
        notes = "Requires external pull-up on Data[0]"
        certification = "RoHS compliant"
        supplier = "Digi-Key"
        lead_time = "6 weeks"
```

**Key Change**: The compiler treats `metadata` as a flexible dictionary. Any `key = "value"` pair is accepted. This future-proofs BOM generation and allows teams to add custom tracking fields.

### Profile Blocks

```hw
# ✅ v0.1.6 - Custom profile fields
profile CustomManufacturing:
    allowed_materials:
        conductors: [Copper]
        dielectrics: [FR4]
    
    constraints:
        min_trace_width = 0.1mm
        min_via_diameter = 0.3mm
    
    # Custom fields for specific manufacturers
    manufacturer_code = "FAB-001"
    process_node = "Standard PCB"
    quality_grade = "IPC Class 2"
    special_instructions = "Gold plating on edge connectors"
```

---

## Coordinate Origin Syntax

The coordinate origin syntax maintains the elegant `by` keyword for visual alignment with `dimensions` and `grid`.

```hw
# ✅ v0.1.6 - Unified spatial syntax
space Board:
    dimensions: 100mm by 100mm by 2mm    # X by Y by Z
    grid: 1000 by 1000 by 2              # X by Y by Z
    origin: tl by t                       # XY by Z
```

**Why `by` stays**: The `by` keyword creates perfect visual rhythm across all spatial specifications. It's terse, professional, and industry-standard shorthand.

**Valid Origin Values**:
- Horizontal: `tl` (top-left), `tr` (top-right), `bl` (bottom-left), `br` (bottom-right), `c` (center)
- Vertical: `t` (top-down), `b` (bottom-up)

---

## Property Keys Are Identifiers

In v0.1.6, property keys are parsed as regular identifiers, not reserved keywords. This eliminates the "soft keyword" hack from v0.1.5.

```hw
# ✅ v0.1.6 - No special treatment needed
electrical:
    tolerance = 5%
    trace = 0.2mm
    via = 0.3mm
    clearance = 0.5mm

# These are just identifiers, not keywords
# The lexer doesn't need special cases
```

**Implementation Note**: The lexer no longer treats `tolerance`, `trace`, `via`, `clearance`, etc. as reserved keywords. They are tokenized as `IDENTIFIER` tokens, simplifying the lexer significantly.

---

## Component Instantiation (Keyword Arguments Only)

All component instantiations must use explicit keyword arguments. No more "magic" positional arguments.

**Critical**: Parameters use colons (`:`) because they define static facts about the instance. You're not changing the resistance at runtime; you're declaring what it IS.

```hw
# ✅ v0.1.6 - Explicit keyword arguments with colons
add Resistor_0805 (resistance: 10kΩ, tolerance: 5%) named R1 at [x: 10mm, y: 10mm, z: 1]
add Capacitor_0805 (capacitance: 100nF, voltage: 50V) named C1 at [x: 20mm, y: 10mm, z: 1]
add CustomALU (bits: 8, signed: true) named ALU1 at [x: 30mm, y: 10mm, z: 1]

# ✅ v0.1.6 - No parameters (empty parens optional)
add LED_0805 named LED1 at [x: 40mm, y: 10mm, z: 1]
add LED_0805() named LED2 at [x: 50mm, y: 10mm, z: 1]
```

```hw
# ❌ v0.1.5 - Magic positional arguments (removed)
add Battery (5V) named Power
add Resistor (10kΩ) named R1
```

**Rationale**: 
1. Explicit keyword arguments make code self-documenting
2. Colons maintain the declarative boundary (these are static facts, not runtime actions)
3. The resistance of R1 is a permanent property of that atom

---

## Complete Example: Before and After

### v0.1.5 (Old Syntax)

```hw
define material "Copper":
    category: conductor
    properties:
        resistivity: 1.68e-8Ω·m

define component "Resistor_0805":
    pins:
        A, B
    
    electrical:
        resistance: 10kΩ
        tolerance: 5%
    
    metadata:
        manufacturer: "Yageo"

define space "Motherboard":
    dimensions: 100mm by 100mm by 2mm
    grid: 10000 by 10000 by 4
    origin: tl by t
    
    add Resistor_0805 named R1 at [x: 10mm, y: 10mm, z: 1]
    add Resistor_0805 named R2 at [x: 20mm, y: 10mm, z: 1]
    
    route R1.B to R2.A

define module "Counter8":
    pins:
        input Clk
        input Rst
        output Count[8]
    
    logic:
        let count = reg(clock: Clk, reset: Rst, init: 0)
        count.next = count + 1
        Count = count
```

### v0.1.6 (New Syntax)

```hw
material Copper:
    category: conductor
    properties:
        resistivity = 1.68e-8Ω·m

component Resistor_0805:
    pins: [A, B]
    
    electrical:
        resistance = 10kΩ
        tolerance = 5%
    
    metadata:
        manufacturer = "Yageo"

space Motherboard:
    dimensions: 100mm by 100mm by 2mm
    grid: 10000 by 10000 by 4
    origin: tl by t
    
    add Resistor_0805 named R1 at [x: 10mm, y: 10mm, z: 1]
    add Resistor_0805 named R2 at [x: 20mm, y: 10mm, z: 1]
    
    route R1.B to R2.A

module Counter8:
    pins: [input Clk, input Rst, output Count[8]]
    
    logic:
        let count = reg(clock: Clk, reset: Rst, init: 0)
        count.next = count + 1
        Count = count
```

---

## Spatial Variables and Relative Positioning (v0.1.6)

v0.1.6 introduces **Spatial Variables**, enabling structured assembly where components are placed relative to each other or to the substrate bounds.

### Anchor Points
Every component and the space substrate provide anchor points that can be referenced in coordinates:

*   **Substrate Anchors**: `substrate.min_z`, `substrate.max_z`
*   **Component Anchors**:
    *   X-axis: `last.left`, `last.right`
    *   Y-axis: `last.top`, `last.bottom` (In horizontal view)
    *   Z-axis: `last.top`, `last.bottom` (In vertical stacking)

The `last` keyword refers to the previously added component in the space.

### Spatial Arithmetic
Coordinates now support full mathematical expressions using anchors and measurements:

```hw
# Place relative to substrate surface
add Chip named C1 at [x: 10mm, y: 10mm, z: substrate.max_z]

# Stack relative to previous component with a 1mm gap
add Chip named C2 at [x: 10mm, y: 10mm, z: last.top + 1mm]

# Place relative to previous component's right edge
add Chip named C3 at [x: last.right + 2mm, y: 10mm, z: substrate.max_z]

# Complex expressions (e.g. centering in substrate)
add Chip named C4 at [x: 50mm, y: 50mm, z: (substrate.max_z + substrate.min_z) / 2]
```

---

## Migration Guide

All v0.1.5 code can be automatically migrated using a syntax transformer. The changes are mechanical and deterministic.

### Automated Transformations

1. Remove `define` keyword
2. Remove quotes from type names
3. Origin syntax stays unchanged (`tl by t`)
4. Replace `Reg` with `reg` in logic blocks
5. Replace `==` with `=` in logic blocks
6. Remove `fields:` from struct definitions
7. Optionally convert comma/newline lists to bracket notation

### Manual Review Needed

- Custom metadata fields (verify they're still needed)
- Positional component arguments (convert to keyword arguments)

---

## Summary of Changes

| Feature | v0.1.5 | v0.1.6 |
|---------|--------|--------|
| Declaration | `define component "Name":` | `component Name:` |
| Identifiers | `"Copper"`, `"Resistor"` | `Copper`, `Resistor` |
| Property Assignment | `resistance: 10kΩ` | `resistance: 10kΩ` (unchanged) |
| Logic Assignment | `count.next = count + 1` | `count.next = count + 1` (unchanged) |
| Comparison | `if count == 0` | `if count = 0` (single `=`) |
| Lists | Mixed (`,`, newlines, `-`) | Unified `[]` (preferred) |
| Metadata | Rigid struct | Flexible dictionary |
| Origin | `origin: tl by t` | `origin: tl by t` (unchanged) |
| Keywords | Soft keyword hacks | Property keys are identifiers |
| Parameters | Positional allowed | Keyword arguments only |
| Struct | `fields:` keyword | Bare bit-width table |
| Register | `Reg(...)` | `reg(...)` (lowercase) |
| XOR | `^` | `xor` (word-only) |

---

## References

- [MANIFESTO.md](./MANIFESTO.md) - The philosophical foundation
- [SYNTAX-UNIFICATION-PHILOSOPHY.md](./SYNTAX-UNIFICATION-PHILOSOPHY.md) - Detailed rationale
- [v0.1.5 LANGUAGE-SPEC.md](../v0.1.5/LANGUAGE-SPEC.md) - Previous version
