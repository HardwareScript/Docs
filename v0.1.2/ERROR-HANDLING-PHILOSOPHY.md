# Hardware Script v0.1.2 - Error Handling Philosophy

**Document Type**: Developer Experience Guide  
**Status**: Revised Architecture (March 2026)  
**Last Updated**: March 18, 2026

---

## Document History

**Original Plan** (Early March 2026): Use verbose error codes like `hwc::parse::E0001`, `hwc::physics::E0042` following Rust's pattern.

**Implementation Reality** (Mid-March 2026): During actual implementation, we realized these codes are:
- Too long to speak aloud ("hwc-parse-E-zero-zero-zero-one")
- Not memorable (users can't recall them)
- Not searchable (too verbose for quick lookups)
- Machine-oriented, not human-oriented

**The Insight** (March 18, 2026): DSTV error codes (E16, E48) became the universal language for technical communication. When something goes wrong, people don't read the full error text - they just say "I'm getting E16" and everyone knows how to help.

**New System**: 3-character error codes (1 Letter + 2 Digits) that become the spoken vocabulary of Hardware Script.

---

## The Critical Architectural Decision

**Should the compiler embed automated LLM calls to fix errors?**

**Answer**: NO. Absolutely not.

---

## Why Embedding LLMs in the Compiler Is Wrong

### The Fundamental Problem

Embedding an automated LLM call inside the compiler is a **terrible architectural mistake**.

**Compilers must be**:
- ✅ Fast
- ✅ Offline
- ✅ Deterministic
- ✅ Pure (no side effects)

**If `hws build` requires**:
- ❌ An internet connection
- ❌ An OpenAI API key
- ❌ Network latency
- ❌ API rate limits

**Just to resolve a routing error**, you have broken the **UNIX philosophy**.

### The UNIX Philosophy

```
Write programs that do one thing and do it well.
Write programs to work together.
Write programs to handle text streams, because that is a universal interface.
```

**Hardware Script compiler should**:
- Compile .hw files
- Validate physics
- Generate outputs
- Report errors

**Hardware Script compiler should NOT**:
- Call external APIs
- Require internet
- Depend on third-party services
- Make non-deterministic decisions

---

## The Right Approach: Short, Memorable Error Codes

### The DSTV Insight

In Nigeria (and across Africa), DSTV satellite TV has legendary error codes:
- **E16**: "Service is currently scrambled"
- **E48**: "No signal detected"
- **E30**: "Smart card not detected"

**The power of these codes**: When your TV shows an error screen with a long explanation, you don't read all that text. You just see "E16" and that's what you tell people.

**Real-world usage**:
- "My DSTV is showing E16" (not "my service is scrambled")
- "I'm getting E48" (not "I have no signal")
- The code becomes the shorthand for technical communication

**Why this works**:
- ✅ **Speakable**: "I got an E-sixteen" rolls off the tongue
- ✅ **Memorable**: Short enough to remember and share
- ✅ **Searchable**: Google "DSTV E16" instantly finds the solution
- ✅ **Universal**: Everyone learns the same vocabulary
- ✅ **Efficient**: Skip reading paragraphs, just say the code

**Why traditional software error codes fail**:
- ❌ Rust's `E0382` is better than most, but still feels like a serial number
- ❌ `hwc::parse::E0001` is too long to speak aloud
- ❌ `0x80070057` (Windows) is completely opaque
- ❌ Long codes don't become part of user vocabulary

---

## Hardware Script Error Code System

**Format**: `[Letter][Digit][Digit]`
- **Letter**: Which subsystem failed (S, C, R, P, M)
- **Digits**: Specific error within that subsystem (01-99)
- **Total length**: 3 characters maximum

**Example**: `P16` - Physics error #16 (Dielectric Breakdown)

**Design goal**: When a user sees an error, they should be able to say "I'm getting P16" and get help immediately, without reading the full error text.

---

## The Five Subsystems

Hardware Script has 5 distinct compilation layers, each with its own error namespace:

### S - Syntax Errors (Parser)

**What it means**: You typed something the language doesn't understand.

**Decades** (grouped by type):
- **S10-S19**: Structure errors (missing colons, wrong indentation)
- **S20-S29**: Value errors (invalid coordinates, unknown units)
- **S30-S39**: Keyword errors (typos in keywords)
- **S40-S49**: Import/module errors (syntax-level import issues)

**Examples**:
- `S11`: Missing colon after block declaration
- `S12`: Unexpected indentation
- `S21`: Invalid coordinate format (expected `[X,Y,Z]`)
- `S22`: Unknown unit (e.g., using `4K7` instead of `4.7kΩ`)
- `S23`: Invalid number format
- `S31`: Unknown keyword or typo
- `S41`: Invalid import syntax

### C - Compiler Errors (IR & Logic)

**What it means**: The syntax is fine, but the logic is flawed.

**Decades**:
- **C10-C19**: Reference errors (component/pin not found)
- **C20-C29**: Import/dependency errors (circular imports, missing packages)
- **C30-C39**: Space definition errors (multiple spaces, missing dimensions)
- **C40-C49**: Type/parameter errors (wrong parameter types)

**Examples**:
- `C11`: Component not found in scope
- `C12`: Pin does not exist on this component
- `C13`: Net name already used
- `C21`: Package not found in registry
- `C22`: Circular dependency detected
- `C23`: Version conflict in dependencies
- `C31`: Multiple Space definitions in one file
- `C32`: Missing dimensions in Space definition
- `C33`: Missing grid in Space definition

### R - Routing & Engine Errors (Physical Placement)

**What it means**: The physical placement or tracing failed.

**Decades**:
- **R10-R19**: Placement errors (collisions, out of bounds)
- **R20-R29**: Routing errors (no path found, trace overlap)
- **R30-R39**: Geometry errors (invalid angles, impossible vias)

**Examples**:
- `R11`: Coordinate out of bounds
- `R12`: Component collision (two parts overlapping)
- `R13`: Component too close to board edge
- `R21`: No route found (trace trapped)
- `R22`: Trace overlapping another trace
- `R23`: Waypoint unreachable
- `R31`: Invalid 90-degree turn on X/Y plane
- `R32`: Diagonal via attempted (not supported)

### P - Physics Errors (Design Rule Check)

**What it means**: The geometry is drawn, but the laws of physics are violated.

**Decades**:
- **P10-P19**: Clearance/voltage errors (dielectric breakdown)
- **P20-P29**: Thermal/current errors (overheating, trace too thin)
- **P30-P39**: Signal integrity errors (impedance, crosstalk)
- **P40-P49**: Electromagnetic errors (antenna effects, EMI)

**Examples**:
- `P16`: **Dielectric Breakdown** (traces too close for voltage difference) - THE FAMOUS ONE
- `P17`: Voltage exceeds material rating
- `P21`: Trace too thin for current (ampacity violation)
- `P22`: Component overheating (thermal violation)
- `P23`: Via current capacity exceeded
- `P31`: Impedance mismatch on transmission line
- `P32`: Traces parallel for too long (crosstalk risk)
- `P33`: Stub length exceeds signal integrity limits

### M - Manufacturing Errors (Export & Fabrication)

**What it means**: The factory physically cannot build what you designed.

**Decades**:
- **M10-M19**: Factory limit errors (trace too thin, hole too small)
- **M20-M29**: Material errors (material not available)
- **M30-M39**: Process errors (incompatible processes)

**Examples**:
- `M11`: Trace thinner than factory minimum
- `M12`: Via drill hole too small for factory
- `M13`: Annular ring too small
- `M14`: Aspect ratio exceeds factory capability
- `M21`: Material not found in database
- `M22`: Material not available from this manufacturer
- `M31`: Incompatible process combination

---

## Error Code Design Principles

### 1. Speakable

Error codes must roll off the tongue naturally.

**Good**: "I got a P-sixteen" (P16)  
**Bad**: "I got hwc-parse-E-zero-zero-zero-one"

### 2. Memorable

Users should be able to remember common errors after seeing them once or twice.

**Good**: P16 (Dielectric Breakdown) becomes legendary like DSTV's E16  
**Bad**: Long codes that require copy-paste to share

### 3. Searchable

Searching "Hardware Script P16" should instantly find documentation.

**Good**: Short, unique codes that search engines love  
**Bad**: Generic text that varies by variable names

### 4. Mental Mapping

The letter prefix should instantly tell users which subsystem failed.

**Good**: "Oh, a P-error means I messed up physics calculations"  
**Bad**: Opaque codes that give no context

### 5. Decade Grouping

Numbers grouped by tens (10s, 20s, 30s) provide intuitive categorization.

**Good**: P10-P19 are all clearance/voltage issues  
**Bad**: Random number assignment with no pattern

---

## How This Looks in Practice

### Example 1: The Famous P16 (Dielectric Breakdown)

```
❌ Error[P16]: Dielectric Breakdown Risk
   ╭─[board.hw:42:1]
 41 │     
 42 │     route Power_120V.out to Relay.in:
   ·           ──────┬───────
   ·                 ╰── High voltage net (120V)
 43 │         path:
 44 │             - [1, 50, 50]
 45 │             - [1, 50, 60]
   ·               ─────┬─────
   ·                    ╰── Approaches within 0.02mm of MCU_5V net here
 46 │ 
   ╰────
  help: The voltage difference is 115V. Through Air, this requires a 
        minimum clearance of 0.08mm to prevent arcing.
        
  hint: Move the waypoint at Line 45 to [1, 50, 62] or explicitly change
        the surrounding material from Air to FR4.
```

**User conversation** (Discord/GitHub):
- "Hey, I'm getting P16 when I compile my board."
- "P16 is clearance. What's your voltage difference?"
- "120V to 5V."
- "Yeah, you need at least 0.08mm clearance through air. Move your traces apart."

**Notice**: The user didn't say "I'm getting a dielectric breakdown risk error." They just said "P16" and everyone knew what to do.

### Example 2: R12 (Component Collision)

```
❌ Error[R12]: Component Collision Detected
   ╭─[board.hw:15:1]
 14 │     
 15 │     add Battery named Power at [1, 10, 10]
   ·     ────────┬────────
   ·             ╰── First component occupies [1,10,10] to [1,15,15]
 16 │     
 17 │     add Capacitor named C1 at [1, 12, 12]
   ·     ─────────┬─────────
   ·              ╰── Second component overlaps here
 18 │ 
   ╰────
  help: Components overlap by 3×3 voxels.
        
  hint: Move Capacitor to [1, 16, 12] or later in the file.
```

**User conversation**:
- "I'm getting R12 when I place my ESP32."
- "R12 means collision. Check if it's overlapping another component."
- "Oh yeah, it's clipping the USB connector. Thanks!"

### Example 3: S22 (Unknown Unit)

```
❌ Error[S22]: Unknown Unit Format
   ╭─[board.hw:28:1]
 28 │     add Resistor (4K7) named R1 at [1, 20, 20]
   ·                   ─┬─
   ·                    ╰── Invalid unit format
   ╰────
  help: Hardware Script uses explicit unit notation, not IEC 60062 shorthand.
        
  hint: Change '4K7' to '4.7kΩ' or '4.7kOhm'
```

**User conversation**:
- "Getting S22 on my resistor values."
- "S22 is unit format. Are you using IEC notation like 4K7?"
- "Yeah, I'll switch to 4.7kΩ. Thanks!"

---

## The Workflow

```
1. User writes .hw file
2. Compiler validates
3. Compiler finds error
4. Compiler outputs beautiful diagnostic with short code (e.g., P16)
5. User sees "Error[P16]" and can immediately:
   a) Search "Hardware Script P16" for documentation
   b) Ask for help: "I'm getting P16"
   c) Read the full explanation if they want to learn
   d) Copy error to LLM (LLM has full context)
```

**The human remains in control.**

---

## Implementation in Rust

### Defining Errors with Short Codes

```rust
use miette::{Diagnostic, SourceSpan};
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
#[error("Dielectric Breakdown Risk")]
#[diagnostic(
    code(P16),  // Short and memorable!
    url("https://docs.hw-script.org/errors/P16"),
    help("The voltage difference is {voltage_diff}V. Through {material}, this requires a minimum clearance of {required_gap}mm to prevent arcing.")
)]
pub struct ClearanceViolation {
    #[source_code]
    pub src: String,
    
    #[label("High voltage net ({voltage_a}V)")]
    pub net_a_span: SourceSpan,
    
    #[label("Approaches within {actual_gap}mm of {net_b_name} net here")]
    pub collision_span: SourceSpan,
    
    pub voltage_diff: f32,
    pub material: String,
    pub required_gap: f32,
    pub actual_gap: f32,
    pub net_b_name: String,
    pub voltage_a: f32,
}
```

### Error Code Constants

Create a central error code registry for each crate:

```rust
// hwc/crates/hwc-parser/src/error_codes.rs
pub mod syntax {
    pub const MISSING_COLON: &str = "S11";
    pub const UNEXPECTED_INDENT: &str = "S12";
    pub const INVALID_COORDINATE: &str = "S21";
    pub const UNKNOWN_UNIT: &str = "S22";
    pub const INVALID_NUMBER: &str = "S23";
    pub const UNKNOWN_KEYWORD: &str = "S31";
    pub const INVALID_IMPORT: &str = "S41";
}

// hwc/crates/hwc-compiler/src/error_codes.rs
pub mod compiler {
    pub const COMPONENT_NOT_FOUND: &str = "C11";
    pub const PIN_NOT_FOUND: &str = "C12";
    pub const NET_NAME_CONFLICT: &str = "C13";
    pub const PACKAGE_NOT_FOUND: &str = "C21";
    pub const CIRCULAR_DEPENDENCY: &str = "C22";
    pub const VERSION_CONFLICT: &str = "C23";
    pub const MULTIPLE_SPACES: &str = "C31";
    pub const MISSING_DIMENSIONS: &str = "C32";
    pub const MISSING_GRID: &str = "C33";
}

// hwc/crates/hwc-engine/src/error_codes.rs
pub mod routing {
    pub const OUT_OF_BOUNDS: &str = "R11";
    pub const COMPONENT_COLLISION: &str = "R12";
    pub const TOO_CLOSE_TO_EDGE: &str = "R13";
    pub const NO_ROUTE_FOUND: &str = "R21";
    pub const TRACE_OVERLAP: &str = "R22";
    pub const WAYPOINT_UNREACHABLE: &str = "R23";
    pub const INVALID_TURN: &str = "R31";
    pub const DIAGONAL_VIA: &str = "R32";
}

pub mod physics {
    pub const DIELECTRIC_BREAKDOWN: &str = "P16";  // THE FAMOUS ONE
    pub const VOLTAGE_EXCEEDS_RATING: &str = "P17";
    pub const TRACE_TOO_THIN: &str = "P21";
    pub const COMPONENT_OVERHEATING: &str = "P22";
    pub const VIA_CURRENT_EXCEEDED: &str = "P23";
    pub const IMPEDANCE_MISMATCH: &str = "P31";
    pub const CROSSTALK_RISK: &str = "P32";
    pub const STUB_TOO_LONG: &str = "P33";
}

// hwc/crates/hwc-export/src/error_codes.rs
pub mod manufacturing {
    pub const TRACE_TOO_THIN_FOR_FAB: &str = "M11";
    pub const VIA_TOO_SMALL: &str = "M12";
    pub const ANNULAR_RING_TOO_SMALL: &str = "M13";
    pub const ASPECT_RATIO_EXCEEDED: &str = "M14";
    pub const MATERIAL_NOT_FOUND: &str = "M21";
    pub const MATERIAL_NOT_AVAILABLE: &str = "M22";
    pub const INCOMPATIBLE_PROCESS: &str = "M31";
}
```

### Using Error Codes

```rust
use crate::error_codes::physics;
use miette::Result;

pub fn validate_clearance(
    board: &Board,
    source: &str
) -> Result<()> {
    for (net_a, net_b) in board.get_net_pairs() {
        let distance = calculate_distance(net_a, net_b);
        let voltage_diff = (net_a.voltage - net_b.voltage).abs();
        let material = board.get_material_between(net_a, net_b);
        let required_gap = calculate_required_clearance(voltage_diff, material);
        
        if distance < required_gap {
            return Err(ClearanceViolation {
                src: source.to_string(),
                net_a_span: net_a.source_span,
                collision_span: net_b.source_span,
                voltage_diff,
                material: material.name.clone(),
                required_gap,
                actual_gap: distance,
                net_b_name: net_b.name.clone(),
                voltage_a: net_a.voltage,
            }.into());
        }
    }
    
    Ok(())
}
```

### Displaying the Error

```rust
// In main.rs or CLI

use miette::{IntoDiagnostic, Result};

fn main() -> Result<()> {
    let source = std::fs::read_to_string("board.hw")
        .into_diagnostic()?;
    
    let board = parse(&source)
        .into_diagnostic()?;
    
    // This will automatically display: Error[P16]: Dielectric Breakdown Risk
    validate_clearance(&board, &source)?;
    
    generate_outputs(&board)?;
    
    Ok(())
}
```

---

## Why This Is Perfect for LLMs

### Rich Context

When the user copies that miette output and pastes it into ChatGPT or Claude, the LLM sees:
- ✅ The error code (P16)
- ✅ The exact source code
- ✅ The exact physics rule that was broken
- ✅ The voltage difference
- ✅ The material
- ✅ The required gap
- ✅ The actual gap
- ✅ A suggested fix

### The LLM Can Actually Fix It

Because the hint explicitly states the mathematics:

```
requires a minimum clearance of 0.08mm
```

The LLM can instantly rewrite the user's waypoints to satisfy the math.

### No Black Boxes

The human remains in complete control.

They:
- ✅ Learn why their hardware design failed
- ✅ Understand the physics rule
- ✅ See the exact location
- ✅ Get actionable suggestions
- ✅ Decide how to fix it

**They are learning hardware engineering in the process.**

---

## Complete Error Code Registry

### S - Syntax Errors (S10-S49)

| Code | Description |
|------|-------------|
| S11  | Missing colon after block declaration |
| S12  | Unexpected indentation |
| S13  | Missing closing bracket |
| S14  | Unexpected token |
| S15  | Unterminated string |
| S21  | Invalid coordinate format |
| S22  | Unknown unit format |
| S23  | Invalid number format |
| S24  | Coordinate out of order (not X,Y,Z) |
| S31  | Unknown keyword |
| S32  | Typo in keyword |
| S41  | Invalid import syntax |
| S42  | Invalid module path |

### C - Compiler Errors (C10-C49)

| Code | Description |
|------|-------------|
| C11  | Component not found in scope |
| C12  | Pin does not exist on component |
| C13  | Net name already used |
| C14  | Duplicate component name |
| C21  | Package not found in registry |
| C22  | Circular dependency detected |
| C23  | Version conflict in dependencies |
| C24  | Import path not found |
| C31  | Multiple Space definitions in one file |
| C32  | Missing dimensions in Space |
| C33  | Missing grid in Space |
| C34  | Invalid dimension values |
| C41  | Invalid parameter type |
| C42  | Missing required parameter |

### R - Routing & Engine Errors (R10-R39)

| Code | Description |
|------|-------------|
| R11  | Coordinate out of bounds |
| R12  | Component collision |
| R13  | Component too close to board edge |
| R14  | Component overlaps substrate |
| R21  | No route found (trace trapped) |
| R22  | Trace overlapping another trace |
| R23  | Waypoint unreachable |
| R24  | Empty waypoints list |
| R31  | Invalid 90-degree turn |
| R32  | Diagonal via attempted |
| R33  | Via on wrong layer |

### P - Physics Errors (P10-P49)

| Code | Description |
|------|-------------|
| P16  | **Dielectric Breakdown** (THE FAMOUS ONE) |
| P17  | Voltage exceeds material rating |
| P18  | Clearance too small for voltage |
| P21  | Trace too thin for current |
| P22  | Component overheating |
| P23  | Via current capacity exceeded |
| P24  | Temperature rise exceeds limit |
| P31  | Impedance mismatch |
| P32  | Crosstalk risk (parallel traces) |
| P33  | Stub length exceeds limit |
| P34  | Signal integrity violation |

### M - Manufacturing Errors (M10-M39)

| Code | Description |
|------|-------------|
| M11  | Trace thinner than factory minimum |
| M12  | Via drill hole too small |
| M13  | Annular ring too small |
| M14  | Aspect ratio exceeds capability |
| M15  | Pad size too small |
| M21  | Material not found in database |
| M22  | Material not available from manufacturer |
| M23  | Material incompatible with process |
| M31  | Incompatible process combination |
| M32  | Layer stack not supported |

---

## The Escape Hatch: Future-Proofing the System

### What If We Run Out of Error Codes?

**Question**: If Hardware Script scales globally and we discover 100+ distinct physics violations, how do we expand without breaking the 3-character rule?

**Wrong approaches** (breaks speakability):
- ❌ Add letters to the end: `P16a`, `P16b` (now 4 characters)
- ❌ Add digits to the end: `P160`, `P161` (now 4 characters)
- ❌ Use special characters: `P16-1`, `P16.1` (not speakable)

**Right approach** (maintains 3-character rule):

**Split the namespace by adding new subsystem letters.**

Right now, `P` handles ALL physics errors (P01-P99 = 99 slots). If it gets too crowded, we split it into specialized subsystems:

- **T** - Thermal errors (T01-T99): Temperature, heat dissipation, thermal runaway
- **E** - Electromagnetic/Signal errors (E01-E99): Impedance, crosstalk, EMI, signal integrity
- **V** - Voltage/Current errors (V01-V99): Ampacity, voltage drop, current capacity

**Example split**:
```
Before (all in P):
  P16 - Dielectric Breakdown
  P21 - Trace too thin for current
  P22 - Component overheating
  P31 - Impedance mismatch
  P32 - Crosstalk risk

After (split into specialized subsystems):
  P16 - Dielectric Breakdown (stays in P for clearance/voltage)
  V21 - Trace too thin for current (moved to V for current)
  T22 - Component overheating (moved to T for thermal)
  E31 - Impedance mismatch (moved to E for signal)
  E32 - Crosstalk risk (moved to E for signal)
```

**Why this works**:
- ✅ Maintains 3-character rule: `T12`, `E44`, `V16`
- ✅ Still speakable: "I'm getting T-twelve"
- ✅ Better mental mapping: "T means thermal, E means electromagnetic"
- ✅ Gives us hundreds of new slots (99 per letter)
- ✅ Backward compatible: Old codes (P16) never change

### The Reality Check

**Do we actually need this?**

Probably not for a very long time. Here's why:

**Fundamental constraint**: There are less than 100 fundamental failure categories per pipeline stage.

**Examples**:
- **Syntax errors**: ~30 fundamental types (missing colon, wrong indent, invalid format, etc.)
- **Compiler errors**: ~40 fundamental types (not found, circular dependency, type mismatch, etc.)
- **Routing errors**: ~30 fundamental types (collision, out of bounds, no path, etc.)
- **Physics errors**: ~50 fundamental types (clearance, ampacity, thermal, signal integrity, etc.)
- **Manufacturing errors**: ~30 fundamental types (too thin, too small, not available, etc.)

**The dynamic payload handles specifics**: The error code is the category, the error message contains the specifics.

**Example**:
```
P16 can represent:
  - 5V to 120V through Air (needs 0.08mm, has 0.02mm)
  - 3.3V to 12V through FR4 (needs 0.01mm, has 0.005mm)
  - 240V to GND through PCB (needs 0.5mm, has 0.3mm)
```

All are "Dielectric Breakdown" (P16), but the error message shows the specific voltages, materials, and gaps.

### The Discipline Benefit

**Limiting codes to 99 per subsystem forces good design**:

1. **Prevents error code inflation**: Can't create a new code for every tiny variation
2. **Creates memorable vocabulary**: Community learns ~50 common codes, not 500
3. **Encourages good error messages**: Specifics go in the text, not the code
4. **Maintains speakability**: "P-sixteen" stays short and memorable

### Migration Strategy (If We Ever Need It)

**If we hit 99 physics errors** (unlikely but possible):

**Step 1**: Analyze the P namespace
```
P10-P19: Clearance/Voltage (10 codes)
P20-P29: Thermal/Current (10 codes)
P30-P39: Signal Integrity (10 codes)
P40-P49: Electromagnetic (10 codes)
... and so on
```

**Step 2**: Identify the most crowded category
```
Example: P20-P29 has 15 thermal errors (overflowing the decade)
```

**Step 3**: Create new subsystem letter
```
Introduce T (Thermal) subsystem:
  T01-T99 for all thermal errors
```

**Step 4**: Deprecate old codes gracefully
```
P21 → T21 (both work for 2 major versions)
P22 → T22 (both work for 2 major versions)

After 2 versions, P21/P22 removed from docs but still recognized
After 4 versions, P21/P22 completely removed
```

**Step 5**: Update documentation
```
Add T to the subsystem list:
  S - Syntax
  C - Compiler
  R - Routing
  T - Thermal (NEW)
  P - Physics (Clearance/Voltage only)
  E - Electromagnetic (if needed later)
  V - Voltage/Current (if needed later)
  M - Manufacturing
```

### Reserved Letters for Future Expansion

**Current subsystems** (v0.1.2):
- S - Syntax
- C - Compiler
- R - Routing
- P - Physics
- M - Manufacturing

**Reserved for future split** (if needed):
- T - Thermal (split from P)
- E - Electromagnetic (split from P)
- V - Voltage/Current (split from P)
- I - Import/Package (split from C)
- L - Layout/Geometry (split from R)

**Avoid using** (too generic or confusing):
- A - Too generic
- B - Confusing with "bug"
- D - Confusing with "debug"
- F - Confusing with "failure"
- X - Too generic

### The Verdict

**For v0.1.2 and beyond**: Stick with the 5 subsystems (S, C, R, P, M) and 99 codes each.

**Why**:
1. We have 495 total error slots (5 × 99)
2. We'll likely use ~150 codes in production
3. The 99-per-subsystem limit enforces discipline
4. If we ever need more, we split gracefully with new letters
5. The 3-character rule is sacred and never breaks

**The system is future-proof without being over-engineered.**

---

## Key Takeaways

1. **Never embed LLMs in the compiler** - Keep it pure, fast, and offline

2. **Short error codes (3 chars max)** - Users say "I'm getting P16" not "I'm getting a dielectric breakdown risk error"

3. **miette + thiserror are perfect** - Rust ecosystem gold standard for beautiful diagnostics

4. **Teach physics through errors** - Users learn while debugging

5. **LLM-friendly by design** - Rich context enables AI assistance

6. **Same quality as rustc** - Hardware errors = Rust borrow checker quality

7. **Human remains in control** - No black boxes, no magic

8. **UNIX philosophy** - Do one thing well (compile), work with text

9. **Speakable codes** - "P-sixteen" rolls off the tongue

10. **Mental mapping** - Letter prefix tells you which subsystem failed

---

## Summary

Hardware Script's error handling philosophy:

**Compiler's job**:
- ✅ Validate physics
- ✅ Generate beautiful diagnostics with short codes
- ✅ Provide actionable hints
- ✅ Teach engineering principles

**Compiler's NOT job**:
- ❌ Call external APIs
- ❌ Require internet
- ❌ Make non-deterministic fixes
- ❌ Hide complexity

**Result**: A pure, offline, incredibly helpful compiler that treats physics violations with the same developer experience as Rust type errors, using short, memorable error codes that become the vocabulary of the Hardware Script community.

**You have the dependencies. You have the architecture. Stick to this plan.**

---

**Document Status**: Developer Experience Guide  
**Last Updated**: March 18, 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite
