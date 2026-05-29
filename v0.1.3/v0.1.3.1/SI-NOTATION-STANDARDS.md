# Official SI Notation Standards

Based on authoritative sources: BIPM SI Brochure 9th Edition, NIST Guide to SI, and ISO 80000

## Key Typography Rules

### 1. Multiplication of Units
**Official Rule**: Units formed by multiplication use either:
- **Center dot (·)** - PREFERRED by NIST
- Space (non-breaking)

**Source**: [NIST Guide to SI, Chapter 6.1.5](https://www.nist.gov/physical-measurement-laboratory/nist-guide-si-chapter-6)

> "Symbols for units formed from other units by multiplication are indicated by means of either a half-high (that is, centered) dot or a space. However, this Guide, as does Ref. [6], prefers the half-high dot because it is less likely to lead to confusion."

**Examples**:
- N·m (newton meter) - PREFERRED
- N m (also acceptable)

### 2. Division of Units
**Official Rule**: Units formed by division use:
- Solidus (/) 
- Horizontal line
- Negative exponents

**Important**: Only ONE solidus per line unless parentheses are used.

**Examples**:
- m/s or m·s⁻¹
- m·kg/(s³·A) or m·kg·s⁻³·A⁻¹
- **NOT**: m/s/s (use m/s² or m·s⁻²)

### 3. Superscripts
**Official Rule**: Use proper Unicode superscripts for exponents.

**Examples**:
- m² (square meter)
- m³ (cubic meter)
- A/m² (amperes per square meter)
- **NOT**: m2, m3, A/m2

### 4. Temperature (Degree Celsius)
**Official Rule**: The degree symbol (°) is REQUIRED and attached directly to C with NO space.

**Source**: [NIST Writing with SI Units](https://physics.nist.gov/cuu/Units/rules.html)

> "There is one exception: in 'degree Celsius' (unit symbol °C) the unit 'degree' is lowercase but the modifier 'Celsius' is capitalized."

**Correct**: °C
**Incorrect**: C, ° C (with space)

### 5. Angle Degrees
**Official Rule**: Degree symbol (°) is placed immediately after the number with NO space.

**Example**: 45° (not 45 °)

## Official Units for Our System

### Distance
- **mm** (millimeter)
- **cm** (centimeter)  
- **µm** (micrometer)
- **m** (meter)

### Electrical - Basic
- **V** (volt), **mV** (millivolt), **kV** (kilovolt)
- **A** (ampere), **mA** (milliampere), **µA** (microampere)
- **Ω** (ohm), **kΩ** (kiloohm), **MΩ** (megaohm)
- **F** (farad), **pF** (picofarad), **nF** (nanofarad), **µF** (microfarad)
- **H** (henry), **µH** (microhenry), **mH** (millihenry)
- **Hz** (hertz), **kHz** (kilohertz), **MHz** (megahertz), **GHz** (gigahertz)

### Electrical - Derived (with multiplication/division)

#### Resistivity
**Official**: **Ω·m** (ohm meter)

**Sources**: 
- Wikipedia: "measured in W·m⁻¹·K⁻¹" (thermal conductivity, same pattern)
- Multiple sources confirm: Ω·m with center dot

**Alternative notations**:
- Ω·m (PREFERRED - center dot)
- Ω m (acceptable - space)
- **NOT**: Ωm (no separator)

#### Current Density
**Official**: **A/m²** or **A·m⁻²**

**Source**: SI base units - amperes per square meter

**Alternative notations**:
- A/m² (common)
- A·m⁻² (negative exponent)
- A/mm² (practical for PCB work)

#### Thermal Conductivity
**Official**: **W/(m·K)** or **W·m⁻¹·K⁻¹**

**Sources**: 
- Wikipedia: "measured in W·m⁻¹·K⁻¹"
- Multiple educational sources: W/mK or W/(m·K)

**Alternative notations**:
- W/(m·K) (PREFERRED - clear grouping)
- W·m⁻¹·K⁻¹ (negative exponents)
- W/mK (acceptable but can be ambiguous)

### Physical Properties

#### Density
**Official**: **kg/m³** or **kg·m⁻³**

**Examples**:
- kg/m³ (common)
- g/cm³ (practical)

#### Temperature
**Official**: **°C** (degree Celsius) or **K** (kelvin)

**Note**: °C requires the degree symbol; K does not.

## Summary of Changes Needed

### Current Issues in Codebase

1. **Resistivity**: Currently may use "Ωm" → Should be **Ω·m**
2. **Thermal Conductivity**: Currently may use "W/mK" → Should be **W/(m·K)** or **W·m⁻¹·K⁻¹**
3. **Current Density**: Verify using **A/m²** (with superscript)
4. **Temperature**: Verify using **°C** (with degree symbol, no space)
5. **All compound units**: Prefer center dot (·) over space

### Files to Update

1. **hwc/crates/hwc-parser/src/lexer/token.rs**
   - Regex patterns for unit recognition
   - Add support for · (center dot) in compound units
   - Ensure superscript support (², ³, ⁻¹, etc.)

2. **hwc/crates/hwc-parser/src/lexer/units.rs**
   - Display implementations for units
   - Format compound units with center dot
   - Use proper superscripts

3. **Documentation files**
   - Update all examples to use correct notation
   - Ensure consistency across all docs

## Implementation Strategy

### Phase 1: Parser Support
- Add center dot (·) to unit token regex
- Add Unicode superscripts (⁰¹²³⁴⁵⁶⁷⁸⁹⁻) to regex
- Update unit parsing to handle both input styles

### Phase 2: Display Formatting
- Update Display traits to output canonical form
- Use center dot for multiplication
- Use proper superscripts for exponents

### Phase 3: Documentation
- Update all examples in docs
- Add style guide for unit notation
- Update material database examples

## References

1. [NIST Guide to SI, Chapter 6](https://www.nist.gov/physical-measurement-laboratory/nist-guide-si-chapter-6) - Official US implementation
2. [BIPM SI Brochure 9th Edition](https://www.bipm.org/si-brochure-9) - International standard
3. [ISO 80000](https://www.wikiwand.com/en/articles/ISO_80000) - Quantities and units standard
4. [NIST Writing with SI Units](https://physics.nist.gov/cuu/Units/rules.html) - Style guide

---

**Decision**: We will use the NIST-preferred notation with center dot (·) for compound units and proper Unicode superscripts for exponents. This is the most unambiguous and internationally recognized format.
