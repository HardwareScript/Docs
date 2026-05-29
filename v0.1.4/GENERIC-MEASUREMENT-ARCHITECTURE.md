# Generic Measurement Architecture (v0.1.4)

## The Problem We Solved

The compiler was suffering from "Lexer Bloat" - hardcoding every possible unit (GΩ, µF, %, ppm, mAh, dBm) directly into the lexer. This created an unmaintainable whack-a-mole situation where every new datasheet unit required compiler changes.

## The Solution: Generic Measurement Pattern

We implemented the industry-standard approach used by C++, Rust, and CSS:

### 1. Lexer Stays Dumb
The lexer no longer knows what physics is. It has ONE regex pattern:
```
NUMBER + UNIT_STRING (NO SPACE ALLOWED)
```

**Critical Rule**: Units must be attached directly to numbers with NO space between them.
- ✅ Correct: `100ppm`, `8960kg/m³`, `1.68e-8Ω·m`
- ❌ Incorrect: `100 ppm`, `8960 kg/m³`, `1.68e-8 Ω·m`

This follows the CSS (`10px` not `10 px`) and Rust (`10u32` not `10 u32`) convention.

**Why No Spaces?**
- Prevents ambiguity with keywords (`500 by 500` vs `500by`)
- Keeps lexer simple and deterministic
- Avoids stack overflow in regex compilation
- Matches industry standards (CSS, Rust, Swift)

### 2. Unit Classification at Parse Time
The parser classifies units into two categories:

**Known Physics Units** (needed for geometry/routing):
- Distance: mm, cm, µm (for voxel placement)
- Voltage: V, mV, kV (for safety clearances)
- Current: A, mA, µA (for trace width calculations)
- Resistance: Ω, kΩ, MΩ, GΩ (for voltage drop)
- Temperature: C (for thermal limits)
- Material properties: kg/m³, W/mK, Ω·m, A/mm²

**Custom Units** (metadata only):
- Tolerance: %, ppm
- Battery: mAh
- Signal: dBm
- Any future units

### 3. The Escape Hatch: `Unit::Custom(String)`

```rust
pub enum Unit {
    Distance(DistanceUnit),
    Voltage(VoltageUnit),
    Current(CurrentUnit),
    // ... other physics units
    
    /// Custom/unknown units - stored as strings
    /// The compiler doesn't need to understand these
    Custom(String),
}
```

## Three Architectural Fixes Implemented

### Fix 1: Keywords as Property Keys
Property blocks now accept reserved keywords as keys:
```hw
electrical:
    category: "Passive"  # 'category' is a keyword but valid as a key
    type: "Resistor"     # 'type' is a keyword but valid as a key
```

### Fix 2: Block Boundary Detection
Block parsers now detect when they've hit the next block:
```rust
// Check if we've hit the start of the next block
if self.check(&Token::Render) || self.check(&Token::Layout) {
    break;  // Let top-level parser handle it
}
```

### Fix 3: Generic Measurement Parser
Single regex handles all measurements:
```rust
#[regex(r"-?\d+\.?\d*([eE][+-]?\d+)?[a-zA-ZµΩ°%·/²³]+", 
        priority = 10, 
        callback = parse_generic_measurement)]
Measurement(Measurement),
```

## What This Achieves

### Compiler Benefits
- **Tiny Core**: Only 4 unit types needed (Distance, Voltage, Current, Temperature)
- **Future-Proof**: New units don't require compiler changes
- **No Crashes**: Unknown units become `Custom(String)` instead of errors

### User Benefits
- **Tolerance works**: `tolerance: 1%` parses correctly
- **Datasheet units work**: `100ppm`, `2000mAh`, `10dBm` all parse
- **Scientific notation**: `1.68e-8` works correctly

### Example: Before vs After

**Before** (would crash):
```hw
electrical:
    tolerance: 1%        # ERROR: Unknown token '%'
    temp_coeff: 100ppm   # ERROR: Unknown unit 'ppm'
```

**After** (works perfectly):
```hw
electrical:
    tolerance: 1%        # Measurement { value: 1.0, unit: Custom("%") }
    temp_coeff: 100ppm   # Measurement { value: 100.0, unit: Custom("ppm") }
```

**Material Properties** (no spaces):
```hw
properties:
    density: 8960kg/m³              # NOT: 8960 kg/m³
    resistivity: 1.68e-8Ω·m         # NOT: 1.68e-8 Ω·m
    thermal_conductivity: 401W/mK   # NOT: 401 W/mK
    max_current_density: 35A/mm²    # NOT: 35 A/mm²
```

## The Boundary: What Stays in Compiler vs Library

### Must Stay in Compiler (Rust) - The "No-Go" Area

Units needed for physical calculations (hardcoded in `hwc-engine`):
- **Distance (mm)** - voxel placement, geometry calculations
- **Voltage (V)** - safety clearances, dielectric breakdown
- **Current (A)** - trace width calculations, ampacity
- **Temperature (C)** - thermal limits, material properties

**The Unbreakable Rule**: Third-party HPM libraries **CANNOT** redefine, alter, or override these 4 core units. They are baked into the Rust binary.

```hw
# This will cause a fatal error
define unit "Millimeter":  # ❌ Error[C15]: Cannot redefine core physics unit
    symbol: "mm"
```

### Can Move to Library (.hw files) - The Open Ecosystem

Everything else is **metadata** that can be defined by community packages:
- Capacitance (F, µF, nF, pF)
- Inductance (H, µH, mH)
- Frequency (Hz, kHz, MHz, GHz)
- Tolerance (%, ppm)
- Battery (mAh)
- Signal (dBm)
- Domain-specific units (Roentgens, Horsepower, etc.)

### Community-Defined Units

Because the Lexer is "dumb" (just sees `NUMBER + UNIT_STRING`), the AST doesn't crash on unknown units—it stores them as `Unit::Custom(String)`. This enables community packages to extend the unit system:

```hw
# Inside @rf-engineering/microwave-units/lib.hw
define unit "Decibel-milliwatt":
    symbol: "dBm"
    dimension: "logarithmic_power"
    description: "Used for RF signal strength"

define unit "Gigahertz":
    symbol: "GHz"
    base_si: "Hz"
    multiplier: 1e9
```

**User's code**:
```hw
# Explicitly import the community standard
import "@rf-engineering/microwave-units" as RF_Units

define component "Custom_Antenna" (freq: Measurement, power: Measurement):
    electrical:
        operating_frequency: freq  # Compiler now understands GHz
        max_power: power           # Compiler now understands dBm

add Custom_Antenna (freq: 2.4GHz, power: 10dBm) named WiFi_Antenna
```

**How the compiler handles this**:

**Pass 1 (Symbol Registration)**: Loads `@rf-engineering/microwave-units`, registers `define unit "dBm"` into Symbol Table

**Pass 2 (Evaluation)**: Sees `10dBm`, checks Symbol Table, finds community definition, validates successfully

**Without import**: `Error[C22]: Unknown unit 'dBm'. Did you forget to import @rf-engineering/microwave-units?`

### The "Promotion" Pathway

If a community package becomes universally adopted (e.g., 90% of users install `@community/automotive-units`), the HardwareScript organization can absorb it into the official `@std` library in a future version.

This mirrors how:
- **JavaScript**: Community built `moment.js` → JS committee created `Temporal` API
- **Rust**: Community built `serde` → Remains community-maintained but universally used

### Why This Prevents Maintainer Burnout

**Without this architecture**: Thousands of PRs requesting "Please add dBm!", "Please add Roentgens!", "Please add Horsepower!"

**With this architecture**: 
- Core team maintains 4 physics units + compiler
- Community builds domain-specific units (RF, nuclear, automotive)
- Users explicitly import what they need
- Language scales into industries not yet imagined

## Future Evolution: Standard Library

The next step is to extract non-essential units to a `.hw` standard library file:

```hw
# ~/.hw/stdlib/units.hw
define unit "Capacitance":
    base: "F"
    aliases:
        "pF" = 1e-12
        "nF" = 1e-9
        "µF" = 1e-6
        "mF" = 1e-3
```

This follows the F# and Nim approach: keep the compiler tiny, move domain knowledge to standard library code.

## Test Results

All tests pass:
- ✅ Component parsing (resistors, capacitors with %, ppm)
- ✅ Material parsing (scientific notation 1.68e-8)
- ✅ Profile parsing
- ✅ Coordinate parsing
- ✅ Integration tests

## Files Modified

1. `hwc/crates/hwc-parser/src/lexer/units.rs` - Added `Unit::Custom(String)`
2. `hwc/crates/hwc-parser/src/lexer/token.rs` - Generic measurement regex
3. `hwc/crates/hwc-parser/src/lexer/parsers.rs` - Generic parser function
4. `hwc/crates/hwc-parser/src/ast/common.rs` - AST `Unit::Custom(String)`
5. `hwc/crates/hwc-parser/src/parser/definitions.rs` - Keywords as keys, block boundaries
6. `hwc/crates/hwc-parser/src/lexer/tests.rs` - Updated test expectations

## Conclusion

We've successfully implemented the "Generic Measurement Pattern" used by world-class compilers. The lexer is now dumb (just reads text), the parser is smart (classifies units), and the compiler stays tiny (only knows essential physics). 

**Key Design Decision**: Following CSS and Rust conventions, units must be attached to numbers with NO space. This keeps the lexer simple, deterministic, and prevents ambiguity with language keywords.

**Examples**:
- ✅ `10mm`, `4.7kΩ`, `100nF`, `8960kg/m³`, `1400cm2/Vs`
- ❌ `10 mm`, `4.7 kΩ`, `100 nF`, `8960 kg/m³`, `1400 cm2/Vs`

This is the foundation for a truly extensible, future-proof unit system.

**See Also**: `STANDARD-LIBRARY-ARCHITECTURE.md` for the user-facing standard library system that builds on this generic measurement foundation.
