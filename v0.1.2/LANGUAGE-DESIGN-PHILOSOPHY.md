# Hardware Script Language Design Philosophy

**Version**: 0.1.2  
**Status**: Authoritative Reference  
**Last Updated**: March 17, 2026

---

## Table of Contents

1. [Natural Language Inspiration](#natural-language-inspiration)
2. [Why Ruby-Inspired Syntax](#why-ruby-inspired-syntax)
3. [The Tri-Fold Case Sensitivity Model](#the-tri-fold-case-sensitivity-model)
4. [Engineering Standards Compliance](#engineering-standards-compliance)
5. [Comment and Documentation System](#comment-and-documentation-system)
6. [Design Principles](#design-principles)

---

## Natural Language Inspiration

### The Adoption Challenge

Hardware description languages have historically been difficult to learn and adopt. Traditional HDLs like VHDL and Verilog use syntax that is:
- Verbose and cryptic
- Far removed from natural language
- Intimidating to newcomers
- Difficult to read without deep expertise

**Our Goal**: Create a language that reads like English, making hardware design accessible to software engineers, students, and domain experts without extensive HDL experience.

### Languages Considered

We evaluated three languages known for their readability:

#### 1. Python-Inspired Syntax

**Pros**:
- Indentation-based (clean, minimal punctuation)
- Widely known and loved
- Excellent for beginners
- Strong ecosystem

**Cons**:
- Too permissive (multiple ways to do things)
- Lacks the elegance for DSL design
- Verbose method chaining
- Not optimized for declarative hardware descriptions

**Example**:
```python
# Python-style (rejected)
space = Space("Board")
space.set_dimensions(50, 50, 4, unit="mm")
space.add_component(Battery(voltage=12), name="Power", position=[1,10,10])
```

#### 2. SQL-Inspired Syntax

**Pros**:
- Declarative and clear
- Familiar to many developers
- Excellent for expressing constraints
- Natural for querying and filtering

**Cons**:
- Too verbose for hardware descriptions
- Awkward for hierarchical structures
- Not designed for spatial relationships
- Limited expressiveness for complex logic

**Example**:
```sql
-- SQL-style (rejected)
CREATE SPACE Board
  WITH DIMENSIONS 50mm BY 50mm BY 4mm
  AND GRID 500 BY 500 BY 4;

INSERT INTO Board.components
  VALUES (Battery, 'Power', [1,10,10], 12V);
```

#### 3. Ruby-Inspired Syntax ✅ **CHOSEN**

**Pros**:
- Reads like natural English
- Minimal punctuation (clean and airy)
- Excellent for DSLs (Domain-Specific Languages)
- Flexible yet structured
- Beautiful, expressive syntax
- Optimized for human readability

**Cons**:
- Less widely known than Python
- Can be "too magical" if not disciplined

**Example**:
```hw
# Ruby-style (CHOSEN)
define space "Board":
    dimensions: 50mm by 50mm by 4mm
    grid: 500 by 500 by 4
    
    add Battery (12V) named Power at [1,10,10]
```

---

## Why Ruby-Inspired Syntax

### 1. Reads Like Natural English

Ruby was designed with the philosophy that code should read like prose. This aligns perfectly with our goal of making hardware design accessible.

**Hardware Script**:
```hw
add Battery (12V) named Power at [1,10,10]
route Power.Plus to LED.Anode
```

**Reads as**: "Add a 12-volt battery named Power at position [1,10,10]. Route Power's Plus pin to LED's Anode."

### 2. Minimal Punctuation

Ruby minimizes visual noise by making parentheses, semicolons, and braces optional in many contexts.

**Hardware Script** (Ruby-inspired):
```hw
add Resistor (4.7kΩ) named PullUp at [1,20,15] rotated 90
```

**vs. C-style** (rejected):
```c
add_component(Resistor(4.7kOhm), "PullUp", [1,20,15], rotation=90);
```

### 3. Excellent for DSLs

Ruby is famous for DSLs like:
- **RSpec** (testing): `describe "User" do it "should login" do`
- **Rails routes**: `resources :users do member do get :profile end end`
- **Rake**: `task :deploy => :environment do`

Hardware Script follows this tradition, creating a DSL optimized for hardware description.

### 4. Indentation-Based Blocks

Like Python, we use indentation for blocks, but with Ruby's elegance:

```hw
define space "Board":
    dimensions: 50mm by 50mm by 4mm
    
    add Battery named Power at [1,10,10]
    
    route Power.Plus to LED.Anode:
        path:
            - [1,10,10]
            - [1,50,50]
```

Clean, hierarchical, and easy to scan.

### 5. Prepositions as Keywords

Ruby popularized using English prepositions as part of the syntax:

**Ruby**:
```ruby
5.days.from.now
user.posts.where(published: true)
```

**Hardware Script**:
```hw
add Battery at [1,10,10]
route Power.Plus to LED.Anode
dimensions: 50mm by 50mm by 4mm
```

This makes the code self-documenting.

### 6. Symbol Elegance

Ruby's use of symbols (`:symbol`) inspired our use of colons for block definitions:

```hw
define space "Board":
    dimensions: 50mm by 50mm by 4mm
    grid: 500 by 500 by 4
```

The colon signals "here comes a definition" in a visually clean way.

---

## The Tri-Fold Case Sensitivity Model

### Philosophy: Three Domains, Three Rules

Hardware Script's case sensitivity follows an elegant mental model that makes the language instantly learnable and impossible to forget. Every identifier in the language falls into exactly one of three domains, each with its own simple rule.

This is not arbitrary—it reflects the fundamental nature of what you're expressing.

---

### Domain 1: The Software Domain (Strictly Lowercase)

**Rule**: If it is built into the Hardware Script language, it is lowercase. No exceptions, no Shift key.

**Rationale**: Language keywords and built-in constructs are tools provided by the compiler. They should be instantly recognizable, visually consistent, and effortless to type. Lowercase signals "this is part of the language itself."

**Examples**:

**Keywords**:
```hw
define          # Define a new construct
space           # Declare a space
component       # Declare a component
add             # Add an instance
route           # Create a connection
import          # Import from another file
expose          # Expose a pin
```

**Prepositions**:
```hw
at              # Position specifier
to              # Connection target
by              # Dimension separator
rotated         # Rotation modifier
spanning        # Multi-layer specification
```

**Properties & Axes**:
```hw
dimensions:     # Space dimensions
grid:           # Voxel grid resolution
path:           # Routing path
x:              # X-axis (named coordinate)
y:              # Y-axis (named coordinate)
z:              # Z-axis (named coordinate)
```

**Settings & Origins**:
```hw
tl              # Top-left origin
bl              # Bottom-left origin
tr              # Top-right origin
br              # Bottom-right origin
```

**Complete Example**:
```hw
define space "Board":
    dimensions: 50mm by 50mm by 4mm
    grid: 500 by 500 by 4
    
    add Battery named Power at [x:10, y:10, z:1]
    
    route Power.Plus to LED.Anode:
        path:
            - [10, 10, 1]
            - [20, 20, 1]
```

**Why This Works**: When you see lowercase, you know immediately: "This is a language feature. The compiler understands this."

---

### Domain 2: The Physics Domain (Strictly SI Standard Case)

**Rule**: If it represents a law of physics or measurement, it follows strict international scientific notation (SI standards).

**Rationale**: Physics doesn't care about programming conventions. The scientific community has spent centuries standardizing notation. We respect that. When you see `mV` or `kΩ`, you're reading the same notation used in datasheets, textbooks, and laboratories worldwide.

**Examples**:

**Distance**:
```hw
mm              # Millimeters
cm              # Centimeters
m               # Meters (if needed)
```

**Voltage**:
```hw
V               # Volts
mV              # Millivolts
kV              # Kilovolts
```

**Current**:
```hw
A               # Amperes
mA              # Milliamperes
µA              # Microamperes
```

**Resistance**:
```hw
Ω               # Ohms (or 'Ohm' as ASCII alias)
kΩ              # Kilohms (or 'kOhm')
MΩ              # Megohms (or 'MOhm')
```

**Frequency**:
```hw
Hz              # Hertz
kHz             # Kilohertz
MHz             # Megahertz
GHz             # Gigahertz
```

**Complete Example**:
```hw
add Battery (12V) named Power at [10, 10, 1]
add Resistor (4.7kΩ) named PullUp at [15, 10, 1]
add Capacitor (100µF) named Decoupling at [20, 10, 1]
add Oscillator (16MHz) named Clock at [25, 10, 1]
```

**Why This Works**: When you see mixed case with specific patterns (`kΩ`, `MHz`), you know immediately: "This is a physical quantity. It follows SI standards."

**Critical Note**: We do NOT accept SPICE notation (`4.7k`) or IEC 60062 notation (`4K7`). Only explicit SI units with proper case are permitted. This eliminates ambiguity and ensures global readability.

---

### Domain 3: The User Domain (Free Casing, Strictly Matched)

**Rule**: If a human or manufacturer named it, you type it exactly as they named it.

**Rationale**: Respect the source. Component manufacturers, library authors, and hardware designers choose specific casing for their identifiers. We preserve that choice. This ensures compatibility with datasheets, part libraries, and existing designs.

**Examples**:

**Component Types** (from manufacturers):
```hw
eFuse           # Manufacturer's casing
ESP32_WROOM     # Manufacturer's casing
NE555           # Manufacturer's casing
STM32F407       # Manufacturer's casing
MyCustomBoard   # Your casing choice
```

**Instance Names** (your choice):
```hw
MainPower       # You chose PascalCase
status_led      # You chose snake_case
pullup_1        # You chose lowercase with underscore
USB_Port        # You chose SCREAMING_SNAKE for acronym
```

**Complete Example**:
```hw
define component "ESP32_DevKit":
    dimensions: 25mm by 50mm by 10mm
    
    expose GPIO2 as DataPin
    expose GPIO4 as ClockPin

define space "IoT_Hub":
    dimensions: 100mm by 100mm by 4mm
    grid: 1000 by 1000 by 4
    
    add ESP32_DevKit named MainController at [50, 50, 1]
    add eFuse named ProtectionCircuit at [30, 50, 1]
    add NE555 named timer_chip at [70, 50, 1]
    
    route MainController.DataPin to timer_chip.Trigger
```

**Why This Works**: When you see mixed casing that doesn't follow SI patterns, you know immediately: "This is a user-defined or manufacturer-defined name. I need to match it exactly."

**Matching Rule**: Hardware Script is case-sensitive for user identifiers. `MainPower` and `mainpower` are different identifiers. This prevents subtle bugs and respects naming intent.

---

### Summary Table

| Domain | Rule | Examples | Recognition Pattern |
|--------|------|----------|---------------------|
| **Software** | Strictly lowercase | `define`, `add`, `route`, `at`, `to`, `dimensions:` | All lowercase = language feature |
| **Physics** | Strictly SI case | `mm`, `V`, `mA`, `kΩ`, `MHz` | SI patterns = physical quantity |
| **User** | Free (strictly matched) | `ESP32_WROOM`, `MainPower`, `status_led` | Everything else = user/manufacturer name |

---

### Teaching the Model

This tri-fold model is designed to be taught in under 60 seconds:

1. **Language keywords? Lowercase.** No thinking required.
2. **Physics units? SI standard.** Same as your datasheet.
3. **Everything else? Match the source.** Respect the original naming.

Once learned, it becomes second nature. You never have to wonder "should this be capitalized?" The domain tells you instantly.

---

### Error Messages

When case sensitivity is violated, Hardware Script provides clear, domain-aware error messages:

**Example 1: Language keyword with wrong case**:
```
❌ Syntax Error: Unknown identifier 'Define'
  ╭─[main.hw:1:1]
1 │ Define space "Board":
  · ───┬──
  ·    ╰── Did you mean 'define'? (language keywords are lowercase)
  ╰────

Help: Hardware Script language keywords are always lowercase.
      Change 'Define' to 'define'.
```

**Example 2: SI unit with wrong case**:
```
❌ Syntax Error: Unknown unit 'mv'
  ╭─[main.hw:5:15]
5 │ add Battery (12mv) named Power at [10,10,1]
  ·                 ─┬
  ·                  ╰── Did you mean 'mV'? (SI units follow standard case)
  ╰────

Help: SI units must use standard scientific notation.
      'mV' = millivolts (lowercase 'm', uppercase 'V')
      See: LANGUAGE-DESIGN-PHILOSOPHY.md#the-physics-domain
```

**Example 3: User identifier mismatch**:
```
❌ Reference Error: Unknown component 'mainpower'
  ╭─[main.hw:8:10]
8 │ route mainpower.Plus to LED.Anode
  ·       ─────┬────
  ·            ╰── Component not found
  ╰────

Help: Did you mean 'MainPower'? (defined at line 5)
      User identifiers are case-sensitive.
```

---

### Design Rationale

**Why Three Domains?**
- Reflects the fundamental nature of hardware design
- Each domain has different sources of truth (compiler, physics, humans)
- Makes the rules memorable and logical

**Why Strict Enforcement?**
- Eliminates ambiguity
- Prevents subtle bugs from case mismatches
- Ensures code is readable across teams and time
- Respects established conventions (SI standards, manufacturer naming)

**Why Not "Case-Insensitive"?**
- Would break SI notation (`mV` ≠ `MV`)
- Would break manufacturer names (`ESP32` ≠ `esp32`)
- Would make error messages less helpful
- Would violate the principle of respecting source naming

**Why Not "All Lowercase"?**
- Would break SI standards (required for datasheets)
- Would break manufacturer compatibility
- Would make physics notation unreadable

---

## Engineering Standards Compliance

### The Standards Challenge

Hardware engineering has decades of established standards for component values, units, and notation. We must respect these standards while maintaining readability.

### Standards Evaluated

#### 1. IEC 60062:2016 (Resistor/Capacitor Marking)

**Standard**: Uses letter codes for multipliers (e.g., `4K7` = 4.7kΩ, `100n` = 100nF)

**Decision**: **REJECTED** for Hardware Script

**Reasoning**:
- Cryptic to non-experts (`4K7` is not intuitive)
- Ambiguous (is `K` kilo or Kelvin?)
- Not self-documenting
- Optimized for physical space on components, not code readability

**Our Alternative**:
```hw
# Hardware Script (clear and explicit)
add Resistor (4.7kΩ) named R1 at [1,10,10]
add Capacitor (100nF) named C1 at [1,15,15]

# NOT: add Resistor (4K7) named R1 at [1,10,10]  ❌
```

#### 2. SPICE Notation (Circuit Simulation)

**Standard**: Implied units (e.g., `4.7k` = 4.7kΩ, `100n` = 100nF)

**Decision**: **REJECTED** for Hardware Script

**Reasoning**:
- Context-dependent (unit depends on component type)
- Not explicit (what does `100n` mean without context?)
- Requires domain knowledge
- Error-prone for beginners

**Our Alternative**:
```hw
# Hardware Script (explicit units)
add Resistor (4.7kΩ) named R1 at [1,10,10]
add Capacitor (100nF) named C1 at [1,15,15]

# NOT: add Resistor (4.7k) named R1 at [1,10,10]  ❌
```

#### 3. SI Units (International System of Units)

**Standard**: Standard prefixes (k, M, G, m, µ, n, p) with base units

**Decision**: **ADOPTED** with strict enforcement

**Reasoning**:
- Universally recognized
- Unambiguous
- Self-documenting
- Internationally standardized

**Our Implementation**:
```hw
# Distance
50mm, 5cm

# Voltage
12V, 500mV

# Current
2A, 100mA, 50µA

# Resistance
4.7kΩ, 1MΩ

# Capacitance
100µF, 10nF, 1pF

# Frequency
60Hz, 2.4GHz
```

### Strict Unit System

**Philosophy**: Exactly **two allowed formats** per unit:
1. **Unicode symbol**: `Ω`, `µF`, `°`
2. **Keyboard-friendly alias**: `Ohm`, `uF`, `deg`

**Rationale**:
- Eliminates ambiguity
- Supports both Unicode-capable and ASCII-only environments
- Enforces consistency across all codebases
- Enables beautiful error messages

**Examples**:
```hw
# Both valid (symbol and alias)
add Resistor (4.7kΩ) named R1 at [1,10,10]
add Resistor (4.7kOhm) named R1 at [1,10,10]

# Both valid (symbol and alias)
add Capacitor (100µF) named C1 at [1,15,15]
add Capacitor (100uF) named C1 at [1,15,15]

# INVALID (SPICE notation)
add Resistor (4.7k) named R1 at [1,10,10]  ❌

# INVALID (IEC 60062)
add Resistor (4K7) named R1 at [1,10,10]  ❌
```

### Error Messages for Standards Violations

When users attempt to use non-standard notation, Hardware Script provides helpful error messages:

```
❌ Syntax Error: Unrecognized unit format '4K7'
  ╭─[main.hw:5:15]
5 │ add Resistor (4K7) named R1 at [1,10,10]
  ·               ─┬─
  ·                ╰── Invalid unit formatting
  ╰────

Help: Hardware Script uses strict SI notation to ensure readability.
      Suggestion: Change '4K7' to '4.7kOhm' or '4.7kΩ'.
      
      IEC 60062 notation (4K7) is not permitted.
      Use explicit SI units for clarity.
      
      Docs: See UNIT-SYSTEM-AND-ERROR-HANDLING.md
```

---

## Comment and Documentation System

### Philosophy: Dual-Purpose Comments

Hardware Script provides a comprehensive comment system that serves two distinct purposes:
1. **Code comments** for developers (ignored by compiler)
2. **Documentation comments** for automated documentation generation (extracted and attached to AST nodes)

This dual-purpose approach ensures that documentation lives alongside code, reducing drift and improving maintainability.

### Single-Line Comments

**Code Comments** (ignored by compiler):
```hw
# This is a regular comment
# It helps developers understand the code
add Battery (12V) named Power at [1,10,10]
```

**Documentation Comments** (extracted for docs):
```hw
## This documentation describes the Battery component
## It will be extracted and attached to the AST node
add Battery (12V) named Power at [1,10,10]
```

### Multi-Line Comment Blocks

For longer comments or temporarily disabling code, Hardware Script supports block comments with paired delimiters.

**Multi-Line Code Comments**:
```hw
#[
This is a multi-line comment block.
Everything between #[ and ]# is ignored by the compiler.
Perfect for temporarily disabling code sections or
adding detailed explanations without line-by-line # symbols.
]#

add Battery (12V) named Power at [1,10,10]
```

**Multi-Line Documentation Blocks**:
```hw
##[
Advanced Motor Driver Module

This module provides a complete motor control solution with:
- 12V LiPo battery power
- Overcurrent protection via 4.7kΩ pull-up
- Decoupling capacitor for noise reduction
- Fault detection exposed as ErrorSignal

Usage:
  import MotorDriver from @myproject/drivers
  add MotorDriver named Motor at [1,100,100]
]##
define space "Smart_Motor_Driver":
    dimensions: 50mm by 50mm by 4mm
    grid: 500 by 500 by 4
```

### Syntax Rules

**Block Comment Delimiters**:
- Start: `#[` (must have whitespace after)
- End: `]#` (must have whitespace before)
- Content: Everything between delimiters is ignored

**Documentation Block Delimiters**:
- Start: `##[` (must have whitespace after)
- End: `]##` (must have whitespace before)
- Content: Extracted and attached to next AST node

**Whitespace Requirement**: The mandatory whitespace after opening and before closing delimiters ensures:
- Clear visual separation between delimiters and content
- No ambiguity with potential future syntax
- Consistent, readable code style
- Simpler lexer implementation

**Valid Examples**:
```hw
#[ This has proper spacing ]#

#[
Multi-line comment
with proper spacing
]#

##[ Documentation with spacing ]##

##[
Multi-line documentation
with proper spacing
]##
```

**Invalid Examples** (no whitespace):
```hw
#[No space after opening]#      ❌
#[ No space before closing]#    ❌
##[No space after opening]##    ❌
##[ No space before closing]##  ❌
```

### Use Cases

**1. Temporarily Disable Code**:
```hw
define space "Test":
    dimensions: 50mm by 50mm by 4mm
    grid: 500 by 500 by 4
    
    add Battery named Power at [1,5,5]
    
    #[
    # Temporarily disabled for testing
    add LED named Light at [1,8,8]
    route Power.Plus to Light.Anode:
        path:
            - [1,5,5]
            - [1,8,8]
    ]#
    
    expose Power.Plus as VCC
```

**2. Section Documentation**:
```hw
define space "Complex_Board":
    dimensions: 100mm by 100mm by 4mm
    grid: 1000 by 1000 by 4
    
    ##[
    Power Supply Section
    Provides regulated 5V and 3.3V rails
    ]##
    add Battery (12V) named MainPower at [1,50,50]
    add Regulator_5V named Reg5V at [1,100,50]
    
    ##[
    Communication Section
    UART and I2C interfaces
    ]##
    add UART named Serial at [1,50,200]
    add I2C named Bus at [1,100,200]
```

**3. Rich Component Documentation**:
```hw
##[
Battery Component

Provides main power for the circuit.

Specifications:
- Voltage: 12V nominal
- Chemistry: LiPo
- Capacity: 2200mAh
- Discharge rate: 20C

Safety:
- Always use with protection circuit
- Monitor cell voltage during discharge
]##
add Battery (12V) named Power at [1,10,10]
```

### Design Rationale

**Why Block Comments?**
- Reduces visual noise for multi-line comments
- Makes it easier to temporarily disable code sections
- Familiar pattern from other languages (Rust's `/* */`, C's `/* */`)
- Improves developer experience without breaking backward compatibility

**Why Separate Documentation Blocks?**
- Clear distinction between code comments and documentation
- Enables automated documentation generation
- Encourages developers to write comprehensive docs
- Documentation lives with the code (reduces drift)

**Why Whitespace Requirement?**
- Prevents ambiguity with potential future syntax
- Ensures consistent, readable code style
- Simplifies lexer implementation
- Forces developers to write clean, well-formatted comments

### Backward Compatibility

All existing single-line comments (`#` and `##`) continue to work exactly as before. Block comments are a purely additive feature with no breaking changes.

---

## Design Principles

### 1. Readability First

Code is read far more often than it is written. Every syntax decision prioritizes human readability over compiler convenience.

**Example**:
```hw
# Clear and readable
add Battery (12V) named Power at [1,10,10]

# NOT: add(Battery, 12V, "Power", [1,10,10])  ❌
```

### 2. Explicitness Over Brevity

We prefer explicit, self-documenting code over terse, cryptic notation.

**Example**:
```hw
# Explicit (good)
dimensions: 50mm by 50mm by 4mm

# Terse (rejected)
dim: (50,50,4)mm
```

### 3. One Obvious Way

Unlike Python's "there should be one obvious way to do it," we enforce it strictly. There is exactly one correct way to write each construct.

**Example**:
```hw
# The one correct way
add Battery (12V) named Power at [1,10,10]

# All other variations are syntax errors
```

### 4. Fail Fast with Helpful Messages

When users make mistakes, provide immediate, actionable feedback with suggestions.

**Example**:
```
❌ Expected 'space' or 'component' after 'define'
  ╭─[main.hw:1:8]
1 │ define Board "MyBoard":
  ·        ─┬───
  ·         ╰── Unexpected identifier
  ╰────

Help: Did you mean 'define space "MyBoard":'?
```

### 5. Consistency Across All Levels

The same patterns apply everywhere in the language:
- Indentation for blocks
- Colons for definitions
- Prepositions for relationships
- Strict unit notation

### 6. Future-Proof

The syntax is designed to scale from simple PCBs to complex ASICs without breaking changes.

---

## Summary

**Language Inspiration**: Ruby (for readability and DSL elegance)  
**Standards Compliance**: SI Units (strict enforcement)  
**Standards Rejected**: IEC 60062, SPICE notation (too cryptic)  
**Philosophy**: Explicit, readable, consistent, beginner-friendly

Hardware Script proves that hardware design can be as accessible and beautiful as modern software development.

---

## References

- **Ruby Programming Language**: https://www.ruby-lang.org/
- **IEC 60062:2016**: Marking codes for resistors and capacitors
- **SI Units**: International System of Units (BIPM)
- **SPICE**: Simulation Program with Integrated Circuit Emphasis
- **Hardware Script Documentation**: See `hwc/crates/hwc-parser/doc/v0.1.2/`


---

## Coordinate System Design

### Previous Approach: [Z, X, Y]

Early versions used `[Z, X, Y]` ordering (layer-first), reflecting hardware-centric thinking where designers often consider "which layer, then where on that layer."

**Example**:
```hw
add LED at [2, 10, 15]  # Z=2 (layer), X=10, Y=15
```

**Problems**:
- Breaks alphabetical convention
- Counterintuitive for newcomers
- Forces hardware-specific thinking on all users
- Not self-documenting

### New Approach: Alphabetical Default + Named Coordinates

**Default (Positional)**: `[X, Y, Z]` - Alphabetical order
```hw
add LED at [10, 15, 2]  # X=10, Y=15, Z=2
```

**Named (Declarative)**: Explicit labels, any order
```hw
add LED at [z:2, x:10, y:15]  # Layer-first when it matters
add LED at [x:10, z:2, y:15]  # Same result
add LED at [y:15, x:10, z:2]  # Same result
```

### Rationale

**1. Intuitive Default**
- Alphabetical order is universal
- Matches math/physics conventions (X, Y horizontal plane, Z vertical)
- Natural for beginners

**2. Flexibility for Experts**
- Named syntax enables layer-first thinking: `[z:2, x:10, y:15]`
- Self-documenting when order matters
- Choose what makes sense for your use case

**3. Best of Both Worlds**
- Simple cases: Use `[10, 15, 2]` (X, Y, Z)
- Complex cases: Use `[z:2, x:10, y:15]` when layer-first thinking helps
- Mixed: `[x:10, y:15, 1]` (explicit X/Y, positional Z)

### Implementation Impact

This is a breaking change, but pre-1.0 is the time to get fundamentals right. The alphabetical default is more intuitive, and named coordinates provide flexibility without sacrificing clarity.

