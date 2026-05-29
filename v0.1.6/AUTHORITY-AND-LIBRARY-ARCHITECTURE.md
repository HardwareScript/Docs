# Authority and Library Architecture (v0.1.6)

**Base Documentation**: [v0.1.5 COMPILER-INTERNALS.md](../v0.1.5/COMPILER-INTERNALS.md)  
**Status**: Architectural Authoritative Reference  
**Version**: 0.1.6  
**Focus**: Compiler Authority System, Standard Library, and Foundry Validation

---

## Overview

This document defines the boundary between the Compiler Core, the Implicit Prelude, and the Manual Foundry. It establishes how Hardware Script handles physics validation and the "Stack of Truths" override system.

**Key Insight**: The compiler is "dumb" about physics but "smart" about authority. It knows where to look for definitions and how to merge them, but it doesn't hardcode material properties or component specifications.

---

## 1. Compiler CLI: The HWS Command Prefix

We are expanding the `hwc` command suite to support library authors who need to validate components and materials in isolation without building a full 3D board.

### `hwc check`

**Level**: Syntax Pass (L1/L2)  
**Function**: Validates that the grammar is correct and the code is readable. It does not look for physical properties.  
**Usage**: Quick developer feedback during coding.

```bash
hwc check main.hw
```

**Output**:
```
✓ Syntax valid
✓ All identifiers resolved
✓ Type checking passed
```

### `hwc check --foundry`

**Level**: Semantic & Physics Pass (L4)  
**Function**: The "Library Author" Pass. It validates that components, materials, or patterns satisfy the Minimum Physical Viability (MPV) contract.  
**Target**: Validates things that are not "Spaces" (e.g., a `.hw` file containing only a material or a single NAND gate).  
**Benefit**: Enables developers to test if their HPM library is "Foundry-Ready" before publishing.

```bash
hwc check --foundry materials/copper.hw
```

**Output**:
```
✓ Syntax valid
✓ Semantic analysis passed
✓ Physical viability check:
  - resistivity: ✓ present
  - thermal_conductivity: ✓ present
  - density: ✓ present
  - melting_point: ✓ present
  - max_current_density: ✓ present
✓ Foundry-ready
```

**Impact**: Library authors can validate their work before publishing to HPM. Users can trust that imported materials are physically complete.

---

## 2. The Two-Tiered Standard Library

Hardware Script ships with a Standard Library that is physically bundled in the binary but logically divided by Loading Priority.

### Tier 1: The Implicit Prelude (`stdlib/primitives/`)

**The "Soul" of the compiler.** These files are loaded automatically at startup.

**Content**:
- `units.hw`: Definitions for all SI measurements (µF, nH, MHz)
- `math.hw`: Physical constants (π, ε₀, c)

**Behavior**: The compiler "knows" these natively. You do not need an import statement to use `Percent` or `Nanofarad`.

**Why Only Two Files?**

The primitives folder contains only **Data that the compiler needs to understand the physical world**, not Logic it already knows.

- **Logic operators** (`and`, `or`, `xor`, `not`) are native to the language grammar, hardcoded in Rust
- **Arithmetic operators** (`+`, `-`, `*`, `/`) are native to the language grammar
- **Units** need to be defined because the compiler must know that `µF` means `× 10⁻⁶`
- **Constants** need to be defined because typing `3.14159265` is error-prone

**Why Keep Them Separate?**

We keep `units.hw` and `math.hw` specialized because they serve two different masters in the "God-Tier" Engine:

1. **`units.hw`** serves the **Lexer** (Grammar of Measurement)
2. **`math.hw`** serves the **Parser** (Names of Values)

This separation ensures clean compiler architecture and optimal performance.

#### `units.hw` - Master of the Lexer

**Purpose**: Defines the Grammar of Measurement.

**Syntax**: Block-based unit definitions.

**Authority**: Tells the Lexer what characters are allowed after a number.

**Long-term Stability**: Very stable. You rarely add units once the SI system is defined.

```hw
# SI Base Units
unit Meter:
    symbol: "m"
    dimension: length
    multiplier: 1

unit Kilogram:
    symbol: "kg"
    dimension: mass
    multiplier: 1

unit Second:
    symbol: "s"
    dimension: time
    multiplier: 1

unit Ampere:
    symbol: "A"
    dimension: current
    multiplier: 1

unit Kelvin:
    symbol: "K"
    dimension: temperature
    multiplier: 1

# Derived Units (Length)
unit Millimeter:
    symbol: "mm"
    dimension: length
    multiplier: 1e-3

unit Micrometer:
    symbol: "µm"
    dimension: length
    multiplier: 1e-6

unit Nanometer:
    symbol: "nm"
    dimension: length
    multiplier: 1e-9

# Derived Units (Capacitance)
unit Farad:
    symbol: "F"
    dimension: capacitance
    multiplier: 1

unit Microfarad:
    symbol: "µF"
    dimension: capacitance
    multiplier: 1e-6

unit Nanofarad:
    symbol: "nF"
    dimension: capacitance
    multiplier: 1e-9

unit Picofarad:
    symbol: "pF"
    dimension: capacitance
    multiplier: 1e-12

# Derived Units (Frequency)
unit Hertz:
    symbol: "Hz"
    dimension: frequency
    multiplier: 1

unit Kilohertz:
    symbol: "kHz"
    dimension: frequency
    multiplier: 1e3

unit Megahertz:
    symbol: "MHz"
    dimension: frequency
    multiplier: 1e6

unit Gigahertz:
    symbol: "GHz"
    dimension: frequency
    multiplier: 1e9

# Dimensionless
unit Percent:
    symbol: "%"
    dimension: dimensionless
    multiplier: 0.01
```

**How the Lexer Uses This**: When the lexer sees `10µF`, it:
1. Recognizes `10` as a number
2. Looks up `µF` in the unit table
3. Validates that `µF` is a valid unit symbol
4. Passes both to the parser as a `Measurement` token

**Impact**: The lexer can validate unit syntax at tokenization time, providing immediate feedback on typos like `10uF` (missing µ) or `10XF` (invalid unit).

#### `math.hw` - Master of the Parser

**Purpose**: Defines the Names of Values.

**Syntax**: Line-based constant declarations using the `const` keyword.

**Authority**: Tells the Parser how to replace a word with a value during compilation.

**Long-term Extensibility**: Users might add hundreds of these for specific physical coefficients.

```hw
# Mathematical Constants
const PI: 3.14159265358979323846
const E: 2.71828182845904523536

# Physical Constants
const SPEED_OF_LIGHT: 299792458  # m/s
const VACUUM_PERMITTIVITY: 8.854187817e-12  # F/m (ε₀)
const VACUUM_PERMEABILITY: 1.25663706212e-6  # H/m (µ₀)
const BOLTZMANN_CONSTANT: 1.380649e-23  # J/K
```

**How the Parser Uses This**: When the parser sees `PI * radius`, it:
1. Recognizes `PI` as an identifier
2. Looks up `PI` in the constant table
3. Replaces `PI` with `3.14159265358979323846` during AST construction
4. Performs the multiplication at compile time if `radius` is also constant

**Impact**: Constants are resolved at parse time, enabling compile-time constant folding and eliminating runtime lookups.

**Why These Constants?**

- **PI, E**: Universal mathematical constants
- **SPEED_OF_LIGHT**: Needed for high-speed signal timing calculations
- **VACUUM_PERMITTIVITY**: Essential for calculating capacitance
- **VACUUM_PERMEABILITY**: Essential for calculating inductance
- **BOLTZMANN_CONSTANT**: Needed for thermal noise calculations

### Tier 2: The Manual Foundry (`stdlib/materials|components|profiles/`)

**The "Physical World" selection.** These files are dormant until called.

**Content**: Industry-standard parts (0805 Resistors, FR4, Copper, BGA packages).

**Behavior**: Must be explicitly imported (e.g., `import @std/materials`).

**Requirement**: If a user chooses NOT to import these and declares a material manually, the compiler enforces the Minimum Physical Viability (MPV).

**The Contract**: A material declaration MUST provide `resistivity`, `thermal_conductivity`, `density`, `melting_point`, and `max_current_density` if it is to be used in a route.

**Example**:
```hw
# User imports standard copper
import @std/materials/Copper

# Or declares their own (must satisfy MPV)
material CustomCopper:
    properties:
        resistivity: 1.68e-8  # Ω·m
        thermal_conductivity: 401  # W/(m·K)
        density: 8960  # kg/m³
        melting_point: 1358  # K
        max_current_density: 3e9  # A/m²
```

**Impact**: 
- Beginners get a "batteries included" experience with standard materials
- Advanced users can define custom materials with full control
- Library authors can publish validated material libraries to HPM

---

## 3. Symbol Resolution: The "Triple Duty Symbol" Problem

In most programming languages, `%` only does one thing (Modulo). In Hardware Script, we need it to serve three contexts:

1. **Measurement**: `tolerance: 1%` (A ratio)
2. **Coordinate**: `at [x: 50%]` (A fraction of space)
3. **Mathematics**: `count % 2` (The modulo operator)

If we don't handle this correctly, the system will collide and become ambiguous. Here is the Architecture of Symbol Resolution to ensure `%` never causes a conflict.

### 3.1. Resolving the "Coordinate vs. Measurement" Conflict

**Question**: Will `%` in units interfere with defining space coordinates?

**Solution**: Contextual Scaling

In Hardware Script, `10%` is mathematically just the number `0.10`.

**In an Electrical Block**:
```hw
component Resistor:
    electrical:
        tolerance: 1%  # Compiler evaluates: 1 * 0.01 = 0.01 (stored as ratio)
```

**In a Layout Block**:
```hw
space PCB:
    layout:
        at [x: 50%]  # Compiler evaluates: 50 * 0.01 = 0.5
                     # Layout Evaluator multiplies: 0.5 * Space.width
```

**The Result**: There is no conflict. The unit definition for `%` provides the math, and the block type (Layout vs Electrical) provides the context-specific logic.

### 3.2. Resolving the "Math (Modulo) vs. Unit" Conflict

**The Danger**: If a user writes `x = y % z`, does the compiler think they are trying to apply a "Percent" unit to `y`?

**The God-Tier Decision**: Protect the Unit

To keep Hardware Script simple and avoid "Symbol Collision," we make the following rule for the v0.1.6 Grammar:

**The `%` symbol is strictly a Unit Suffix. It belongs to the Lexer.**

**The Modulo operation uses the `mod` keyword.**

```hw
# ❌ Confusing (Is this 10 percent or Modulo 10?)
let result = count % 10

# ✅ Clean (Explicit and no conflict)
let result = count mod 10

# OR (function-style)
let result = mod(count, 10)
```

**Why This is Better**:

- It ensures that when the Lexer sees `%`, it always knows it is a unit
- Hardware designers rarely use Modulo compared to software engineers
- Hardware designers use Percent (`%`) in almost every single component (tolerance, accuracy, yield)

**Conclusion**: We give the `%` symbol to the Units, and we use `mod` for the Math.

### 3.3. Lexer "Greedy Consumption" Rule

The Lexer must be **Greedy** to prevent symbol collisions:

**Rule**: If the lexer sees a number followed by `%` (e.g., `5%`), it must consume them together as a single `Measurement` token.

**Implementation**:
```rust
// In the Lexer
fn lex_number(&mut self) -> Token {
    let number = self.consume_digits();
    
    // Greedy unit consumption
    if let Some(unit_symbol) = self.try_consume_unit_symbol() {
        return Token::Measurement {
            value: number,
            unit: unit_symbol,
        };
    }
    
    Token::Number(number)
}
```

**Impact**: The parser never sees `5` and `%` as two separate tokens. This prevents accidental interpretation of `%` as a mathematical operator.

### 3.4. Symbol Collision Resolution Table

| Symbol | Unit Usage | Math Usage | Conflict Resolution |
|--------|-----------|------------|---------------------|
| `%` | `1%` (Percent) | **Forbidden** | Use `mod` keyword for modulo |
| `·` | `Ω·m` (Composite unit) | `*` (Multiply) | Lexer consumes `·` only inside unit strings |
| `/` | `W/mK` (Composite unit) | `/` (Divide) | Context: Unit `/` has no spaces. Math `/` should have spaces. |
| `-` | `1e-6` (Scientific notation) | `-` (Subtract/Negate) | Lexer consumes `-` as part of number if preceded by `e` |

### 3.5. Grammar Rule: Reserved Symbols

**Explicit Language Rule**:

> The symbol `%` is reserved for units and coordinates. Mathematical modulo operations must use the `mod` keyword.

**Error Message**:
```
Error: '%' cannot be used as a binary operator
  --> logic.hw:15:20
   |
15 |     let result = count % 10
   |                        ^ Use 'mod' keyword for modulo operation
   |
   = help: The '%' symbol is reserved for percentage units (e.g., 5%)
   = note: Change to: let result = count mod 10
```

---

## 4. The "Stack of Truths" (Authority System)

When the compiler resolves a name (a unit, a material, or a variable), it searches the **Stack of Truths** in descending order. The first match wins.

### The Search Order (Highest to Lowest)

1. **Local Authority**: Definitions and variables declared inside the current `.hw` file
2. **HPM Authority**: Definitions imported from external libraries (last import takes priority)
3. **Prelude Authority**: The auto-loaded primitives from `stdlib/primitives/`
4. **Core Authority**: Hardcoded engine bootstraps (`mm`, `V`, `A`)

### Property-Level Shadowing (Variable Overrides)

Hardware Script supports **Deep Overrides**. A user can import a standard block and replace a single property without rewriting the entire block.

**Example Logic**: If a user imports `@std/Copper` but writes a local override:

```hw
# Import standard copper
import @std/materials/Copper

# Local declaration higher on the stack
material Copper:
    properties:
        density: 9000kg/m³  # Local truth wins over Prelude truth
```

**Compiler Behavior**: The compiler merges these. It takes the Base Copper and replaces only the `density` value. Everything else (`resistivity`, etc.) remains as defined in the library.

**Impact**: Minimal code duplication. Users can tweak one property without copy-pasting entire definitions.

### Authority Resolution Example

```hw
# User file
import @std/materials/FR4

space PCB:
    material: FR4  # Which FR4?
```

**Resolution Process**:

1. **Local Layer**: Check if `FR4` is defined in this file → Not found
2. **HPM Layer**: Check if `FR4` was imported → Found in `@std/materials/FR4`
3. **Prelude Layer**: (Skipped, already found)
4. **Core Layer**: (Skipped, already found)

**Result**: Uses the imported FR4 definition.

---

## 5. Logical Interaction Flow

### Bootstrap Phase

1. `hwc` starts
2. Parses `stdlib/primitives/units.hw` into Global Symbol Table (Prelude Layer)
3. Parses `stdlib/primitives/math.hw` into Global Symbol Table (Prelude Layer)

### User Parse Phase

1. `hwc` parses the user's `.hw` file
2. If `import` is found, it loads the requested library into the HPM Layer
3. If a local unit or material is found, it adds it to the Local Layer

### Resolution Phase

When the compiler sees `100µF`:

1. Looks for `µF` in the **Local Layer** → Not found
2. Looks in the **HPM Layer** → Found if user imported a high-precision unit library
3. Looks in the **Prelude Layer** → Found in standard `units.hw`

### Validation Phase

If `check --foundry` is active, the compiler ensures that the final "Resolved" object contains all keys required by the physics engine.

**Minimum Physical Viability (MPV) for Materials**:
- `resistivity` (Ω·m)
- `thermal_conductivity` (W/(m·K))
- `density` (kg/m³)
- `melting_point` (K)
- `max_current_density` (A/m²)

**Example Error**:
```
Error: Material 'CustomCopper' missing required property 'thermal_conductivity'
  --> materials.hw:15:1
   |
15 | material CustomCopper:
   | ^^^^^^^^^^^^^^^^^^^^^^ Missing MPV property
   |
   = help: Materials used in routing must define thermal_conductivity for thermal analysis
   = note: Add: thermal_conductivity: <value> W/(m·K)
```

---

## 6. Implementation Checklist

### Compiler Core Changes

| Feature | Action Required in Rust Core | File Location |
|---------|------------------------------|---------------|
| **Foundry Flag** | Add `Foundry` variant to CLI Argument Parser | `hwc/src/main.rs` |
| **VFS Prelude** | Implement auto-loading for `stdlib/primitives/` on initialization | `hwc/crates/hwc-compiler/src/prelude.rs` |
| **Explicit Manuals** | Ensure `materials/` and `components/` are excluded from auto-load | `hwc/crates/hwc-compiler/src/stdlib.rs` |
| **MPV Enforcement** | Add logic to Pass 3 (Validation) to check for required physical keys | `hwc/crates/hwc-validator/src/mpv.rs` |
| **Authority Stack** | Modify Symbol Table to store symbols in layers (Local > HPM > Prelude) | `hwc/crates/hwc-resolver/src/symbol_table.rs` |
| **Key Overrides** | Enable "Deep Merge" for properties when a name collision occurs on the stack | `hwc/crates/hwc-resolver/src/merge.rs` |

### Symbol Table Structure

```rust
pub struct SymbolTable {
    // Layered symbol storage
    core: HashMap<String, Symbol>,      // Hardcoded bootstraps
    prelude: HashMap<String, Symbol>,   // Auto-loaded primitives
    hpm: Vec<HashMap<String, Symbol>>,  // Imported libraries (stack)
    local: HashMap<String, Symbol>,     // Current file definitions
}

impl SymbolTable {
    pub fn resolve(&self, name: &str) -> Option<&Symbol> {
        // Search order: Local > HPM > Prelude > Core
        self.local.get(name)
            .or_else(|| self.hpm.iter().rev().find_map(|layer| layer.get(name)))
            .or_else(|| self.prelude.get(name))
            .or_else(|| self.core.get(name))
    }
    
    pub fn merge_properties(&mut self, base: &Symbol, override_props: &Properties) -> Symbol {
        // Deep merge: take base, replace only specified properties
        let mut merged = base.clone();
        for (key, value) in override_props {
            merged.properties.insert(key.clone(), value.clone());
        }
        merged
    }
}
```

### MPV Validator

```rust
pub struct MPVValidator {
    required_material_properties: Vec<&'static str>,
}

impl MPVValidator {
    pub fn new() -> Self {
        Self {
            required_material_properties: vec![
                "resistivity",
                "thermal_conductivity",
                "density",
                "melting_point",
                "max_current_density",
            ],
        }
    }
    
    pub fn validate_material(&self, material: &Material) -> Result<(), ValidationError> {
        for required_prop in &self.required_material_properties {
            if !material.properties.contains_key(*required_prop) {
                return Err(ValidationError::MissingMPVProperty {
                    material_name: material.name.clone(),
                    property: required_prop.to_string(),
                });
            }
        }
        Ok(())
    }
}
```

---

## 7. The "God-Tier" Hierarchy

| Layer | Responsibility | Authority | Example |
|-------|---------------|-----------|---------|
| **Core Grammar** | `and`, `or`, `not`, `xor`, `+`, `-`, `*`, `/` | Native (Hardcoded in Rust) | `A and B` |
| **Primitives** | `units.hw`, `math.hw` | Implicit (Autoloaded Data) | `10µF`, `PI` |
| **Foundry** | Materials, Components, Patterns | Explicit (Imported when needed) | `import @std/materials/Copper` |

**Why This is the "Perfect Middle Ground"**:

- **Zero Redundancy**: Logic is logic. It's built into the language's DNA. You don't need to "define" it.
- **Infinite Extension**: If we ever find a new SI unit (like Ronnafarads), we just add one line to `units.hw`. We don't have to touch the Rust code.
- **Clean Code**: The user just writes `Output = A xor B`. No imports, no setup, no bloat.

---

## 8. Testing Strategy

### Authority Resolution Tests

```rust
#[test]
fn test_local_overrides_hpm() {
    let mut table = SymbolTable::new();
    
    // Add HPM import
    table.import_hpm("@std/materials/Copper", copper_definition());
    
    // Add local override
    table.define_local("Copper", local_copper_override());
    
    // Local should win
    let resolved = table.resolve("Copper").unwrap();
    assert_eq!(resolved.layer, SymbolLayer::Local);
}

#[test]
fn test_property_deep_merge() {
    let base = Material {
        name: "Copper",
        properties: hashmap! {
            "resistivity" => "1.68e-8",
            "density" => "8960",
        },
    };
    
    let override_props = hashmap! {
        "density" => "9000",  // Override only this
    };
    
    let merged = SymbolTable::merge_properties(&base, &override_props);
    
    assert_eq!(merged.properties["resistivity"], "1.68e-8");  // Unchanged
    assert_eq!(merged.properties["density"], "9000");  // Overridden
}
```

### MPV Validation Tests

```rust
#[test]
fn test_mpv_complete_material() {
    let material = Material {
        name: "Copper",
        properties: hashmap! {
            "resistivity" => "1.68e-8",
            "thermal_conductivity" => "401",
            "density" => "8960",
            "melting_point" => "1358",
            "max_current_density" => "3e9",
        },
    };
    
    let validator = MPVValidator::new();
    assert!(validator.validate_material(&material).is_ok());
}

#[test]
fn test_mpv_incomplete_material() {
    let material = Material {
        name: "CustomCopper",
        properties: hashmap! {
            "resistivity" => "1.68e-8",
            "density" => "8960",
            // Missing: thermal_conductivity, melting_point, max_current_density
        },
    };
    
    let validator = MPVValidator::new();
    let result = validator.validate_material(&material);
    
    assert!(result.is_err());
    assert!(matches!(result.unwrap_err(), ValidationError::MissingMPVProperty { .. }));
}
```

### Foundry Flag Tests

```rust
#[test]
fn test_check_without_foundry() {
    let input = r#"
material Copper:
    properties:
        resistivity: 1.68e-8
"#;
    
    // Should pass syntax check even though MPV incomplete
    let result = compile_with_flags(input, &["check"]);
    assert!(result.is_ok());
}

#[test]
fn test_check_with_foundry() {
    let input = r#"
material Copper:
    properties:
        resistivity: 1.68e-8
"#;
    
    // Should fail foundry check due to missing MPV properties
    let result = compile_with_flags(input, &["check", "--foundry"]);
    assert!(result.is_err());
}
```

---

## 9. Migration from v0.1.5

No breaking changes. The authority system is a new feature that enhances the existing compiler without changing syntax.

**New Capabilities**:
- Library authors can validate materials in isolation
- Users can override imported definitions
- Deep property merging reduces code duplication

**Backward Compatibility**:
- All v0.1.5 code continues to work
- No syntax changes required
- Existing materials are automatically validated against MPV when used in routing

---

## Summary

The v0.1.6 Authority and Library Architecture establishes a clean separation of concerns:

1. **Compiler Core**: Knows grammar and logic operators natively
2. **Implicit Prelude**: Provides units and constants automatically
3. **Manual Foundry**: Offers industry-standard materials and components on-demand
4. **Authority Stack**: Resolves names with clear precedence (Local > HPM > Prelude > Core)
5. **MPV Enforcement**: Ensures physical completeness for foundry-ready libraries

**The compiler remains a lean, silicon-native binary that empowers the community to build and override their own physical "Truths" through the HPM, while providing a foolproof, high-performance experience for beginners through the Implicit Prelude.**

