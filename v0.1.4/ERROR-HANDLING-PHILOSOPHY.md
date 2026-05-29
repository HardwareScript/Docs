# Hardware Script v0.1.4 - Error Handling Philosophy

**Document Type**: Developer Experience Guide  
**Status**: Updated for Logic Synthesis (v0.1.4)  
**Last Updated**: March 31, 2026

---

## Document History

**v0.1.2** (March 2026): Established 3-character error code system (S, C, R, P, M) based on DSTV insight.

**v0.1.4** (March 2026): Added Logic synthesis error codes (L01-L39) for hardware design validation.

---

## The Six Subsystems (v0.1.4)

Hardware Script now has 6 distinct compilation layers, each with its own error namespace:

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

### L - Logic Errors (Logic Synthesis & Hardware Design)

**What it means**: Your logic block has design errors (unbound wires, width mismatches, combinational loops).

**Decades**:
- **L01-L09**: Variable and wire errors
- **L10-L19**: Operator and expression errors
- **L20-L29**: Control flow errors
- **L30-L39**: Register and timing errors

**Examples**:
- `L01`: **Unbound wire** - Variable not declared in pins or let statements
- `L02`: **Width mismatch** - Assigning 8-bit value to 4-bit wire
- `L03`: **Combinational loop** - Feedback without register
- `L04`: **Clock domain crossing** - Missing synchronizer
- `L05`: **Multiple drivers** - Wire assigned in multiple places
- `L06`: **Uninitialized register** - Register without init value
- `L07`: **Invalid enum variant** - Unknown enum value
- `L08`: **Struct field mismatch** - Wrong field name
- `L30`: **Missing clock signal** - Register without clock
- `L31`: **Missing reset signal** - Register without reset

### C - Compiler Errors (IR & General Compilation)

**What it means**: The syntax is fine, but the compilation logic is flawed.

**Decades**:
- **C10-C19**: Reference errors (component/pin not found)
- **C20-C29**: Import/dependency errors (circular imports, missing packages)
- **C30-C39**: Space definition errors (multiple spaces, missing dimensions)
- **C40-C49**: Type/parameter errors (wrong parameter types)

**Examples**:
- `C11`: Component not found in scope
- `C12`: Pin does not exist on this component
- `C24`: Import path not found
- `C31`: Multiple Space definitions in one file

### R - Routing & Engine Errors (Physical Placement)

**What it means**: The physical placement or tracing failed.

**Decades**:
- **R10-R19**: Placement errors (collisions, out of bounds)
- **R20-R29**: Routing errors (no path found, trace overlap)
- **R30-R39**: Geometry errors (invalid angles, impossible vias)

**Examples**:
- `R11`: Coordinate out of bounds
- `R12`: Component collision (two parts overlapping)
- `R21`: No route found (trace trapped)
- `R22`: Trace overlapping another trace

### P - Physics Errors (Design Rule Check)

**What it means**: The geometry is drawn, but the laws of physics are violated.

**Decades**:
- **P10-P19**: Clearance/voltage errors (dielectric breakdown)
- **P20-P29**: Thermal/current errors (overheating, trace too thin)
- **P30-P39**: Signal integrity errors (impedance, crosstalk)
- **P40-P49**: Electromagnetic errors (antenna effects, EMI)

**Examples**:
- `P16`: **Dielectric Breakdown** (traces too close for voltage difference) - THE FAMOUS ONE
- `P21`: Trace too thin for current (ampacity violation)
- `P22`: Component overheating (thermal violation)
- `P31`: Impedance mismatch on transmission line

### M - Manufacturing Errors (Export & Fabrication)

**What it means**: The factory physically cannot build what you designed.

**Decades**:
- **M10-M19**: Factory limit errors (trace too thin, hole too small)
- **M20-M29**: Material errors (material not available)
- **M30-M39**: Process errors (incompatible processes)

**Examples**:
- `M11`: Trace thinner than factory minimum
- `M12`: Via drill hole too small for factory
- `M21`: Material not found in database

---

## Logic Error Examples (NEW in v0.1.4)

### Example 1: L01 (Unbound Wire)

```
❌ Error[L01]: Unbound wire 'result' in logic block
   ╭─[alu.hw:15:1]
 14 │     logic:
 15 │         let output = result + 1
   ·                      ───┬───
   ·                         ╰── Wire 'result' not found
 16 │ 
   ╰────
  Available wires: A, B, CarryIn, Sum, CarryOut
  
  help: Declare 'result' as a pin in the module definition or as an 
        internal wire with 'let result = ...'
        
  hint: Did you mean 'Sum'?
```

**User conversation**:
- "I'm getting L01 on my ALU module."
- "L01 is unbound wire. Check your pin declarations."
- "Oh, I misspelled 'Sum' as 'result'. Thanks!"

### Example 2: L02 (Width Mismatch)

```
❌ Error[L02]: Width mismatch: Cannot assign 8-bit value to 4-bit wire 'nibble'
   ╭─[counter.hw:22:1]
 21 │     logic:
 22 │         let nibble[4] = counter[7..0]
   ·             ───┬───      ──────┬──────
   ·                │                ╰── 8-bit value (bits 7 down to 0)
   ·                ╰── 4-bit wire declared here
 23 │ 
   ╰────
  help: The source expression is 8 bits wide, but the destination wire 
        is only 4 bits wide.
        
  hint: Use slicing to truncate: counter[3..0]
        Or extend the destination: let nibble[8] = ...
```

**User conversation**:
- "Getting L02 when I try to assign my counter."
- "L02 is width mismatch. What are you trying to do?"
- "I want the lower 4 bits. I'll use counter[3..0]. Thanks!"

### Example 3: L03 (Combinational Loop)

```
❌ Error[L03]: Combinational loop detected: feedback → adder → feedback
   ╭─[oscillator.hw:18:1]
 17 │     logic:
 18 │         let feedback = adder_out + 1
   ·             ────┬────
   ·                 ╰── Wire 'feedback' depends on itself
 19 │         let adder_out = feedback + input
   ·             ─────┬─────
   ·                  ╰── Creates a loop back to 'feedback'
 20 │ 
   ╰────
  help: Combinational logic cannot have feedback loops. This creates 
        an unstable circuit that will oscillate.
        
  hint: Insert a register to break the loop:
        let feedback = Reg(clock: Clk, reset: Rst, init: 0)
        feedback.next = adder_out + 1
```

**User conversation**:
- "I'm getting L03 in my state machine."
- "L03 is combinational loop. You need a register to break it."
- "Ah, I forgot to add the Reg(). Thanks!"

### Example 4: L07 (Invalid Enum Variant)

```
❌ Error[L07]: Unknown enum variant 'Running' for enum 'State'
   ╭─[fsm.hw:25:1]
 24 │     logic:
 25 │         state.next = State.Running
   ·                      ──────┬──────
   ·                            ╰── Variant 'Running' does not exist
 26 │ 
   ╰────
  Available variants: Idle, Active, Paused, Stopped
  
  help: The enum 'State' does not have a variant named 'Running'.
        
  hint: Did you mean 'Active'?
```

**User conversation**:
- "Getting L07 on my state machine."
- "L07 is invalid enum variant. Check your enum definition."
- "Oh, I renamed it to 'Active'. Thanks!"

---

## Complete Error Code Registry (v0.1.4)

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

### L - Logic Errors (L01-L39) [NEW in v0.1.4]

| Code | Description |
|------|-------------|
| L01  | **Unbound wire** - Variable not declared |
| L02  | **Width mismatch** - Bit width incompatibility |
| L03  | **Combinational loop** - Feedback without register |
| L04  | **Clock domain crossing** - Missing synchronizer |
| L05  | **Multiple drivers** - Wire assigned multiple times |
| L06  | **Uninitialized register** - Register without init |
| L07  | **Invalid enum variant** - Unknown enum value |
| L08  | **Struct field mismatch** - Wrong field name |
| L09  | **Type mismatch** - Incompatible types |
| L10  | Invalid operator for type |
| L11  | Invalid operand type |
| L12  | Division by zero |
| L20  | Unreachable code |
| L21  | Missing match arm |
| L22  | Duplicate match arm |
| L30  | **Missing clock signal** - Register needs clock |
| L31  | **Missing reset signal** - Register needs reset |
| L32  | Invalid clock edge |

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

## Implementation in Rust (v0.1.4)

### Logic Error Code Constants

```rust
// hwc/crates/hwc-compiler/src/error_codes.rs

/// Logic synthesis error codes for Hardware Script logic blocks.
pub mod logic {
    // L01-L09: Variable and wire errors
    pub const UNBOUND_WIRE: &str = "L01";
    pub const WIDTH_MISMATCH: &str = "L02";
    pub const COMBINATIONAL_LOOP: &str = "L03";
    pub const CLOCK_DOMAIN_CROSSING: &str = "L04";
    pub const MULTIPLE_DRIVERS: &str = "L05";
    pub const UNINITIALIZED_REGISTER: &str = "L06";
    
    // L07-L09: Type system errors
    pub const INVALID_ENUM_VARIANT: &str = "L07";
    pub const STRUCT_FIELD_MISMATCH: &str = "L08";
    pub const TYPE_MISMATCH: &str = "L09";
    
    // L10-L19: Operator and expression errors
    pub const INVALID_OPERATOR: &str = "L10";
    pub const INVALID_OPERAND_TYPE: &str = "L11";
    pub const DIVISION_BY_ZERO: &str = "L12";
    
    // L20-L29: Control flow errors
    pub const UNREACHABLE_CODE: &str = "L20";
    pub const MISSING_MATCH_ARM: &str = "L21";
    pub const DUPLICATE_MATCH_ARM: &str = "L22";
    
    // L30-L39: Register and timing errors
    pub const MISSING_CLOCK_SIGNAL: &str = "L30";
    pub const MISSING_RESET_SIGNAL: &str = "L31";
    pub const INVALID_CLOCK_EDGE: &str = "L32";
}
```

### Using Logic Error Codes

```rust
use crate::error_codes::logic as error_codes;
use thiserror::Error;

#[derive(Error, Debug, Clone)]
pub enum ElectricalSymbolError {
    #[error("Error [{code}]: Unbound wire '{name}' in logic block\n  Available wires: {available}\n  Hint: Check your pin declarations and let statements", 
        code = error_codes::UNBOUND_WIRE)]
    UnboundWire { 
        name: String, 
        available: String 
    },

    #[error("Error [{code}]: Width mismatch: Cannot assign {src_width}-bit value to {dst_width}-bit wire '{name}'\n  Hint: Use slicing to truncate: {name}[{dst_width_minus_1}..0] or extend the destination width", 
        code = error_codes::WIDTH_MISMATCH,
        dst_width_minus_1 = dst_width.saturating_sub(1))]
    WidthMismatch {
        name: String,
        src_width: usize,
        dst_width: usize,
    },
    
    #[error("Error [{code}]: Wire '{name}' already declared\n  Hint: Each wire name must be unique within a logic block", 
        code = error_codes::MULTIPLE_DRIVERS)]
    DuplicateWire { 
        name: String 
    },
}
```

---

## Why Logic Errors (L) Are Separate from Compiler Errors (C)

### Different Mental Models

**Compiler errors (C)** are about:
- Symbol resolution (component not found)
- Import management (circular dependencies)
- Space definition (missing dimensions)
- General compilation issues

**Logic errors (L)** are about:
- Hardware design (unbound wires, width mismatches)
- Digital logic (combinational loops, clock domains)
- Type safety at the hardware level (enum variants, struct fields)
- Register and timing (clock signals, reset signals)

### Different Audiences

**C errors** are for:
- General hardware engineers
- PCB designers
- System integrators

**L errors** are for:
- Digital logic designers
- FPGA engineers
- Hardware description language (HDL) users
- People writing state machines and ALUs

### Different Documentation

**C error docs** explain:
- How to structure Hardware Script files
- How to import libraries
- How to define spaces and components

**L error docs** explain:
- Digital logic design principles
- Bit-width inference rules
- Combinational vs sequential logic
- Clock domain crossing techniques

---

## The Escape Hatch: Future-Proofing (Updated for v0.1.4)

### Current Subsystems (v0.1.4)

- S - Syntax (99 slots)
- L - Logic (99 slots) [NEW]
- C - Compiler (99 slots)
- R - Routing (99 slots)
- P - Physics (99 slots)
- M - Manufacturing (99 slots)

**Total**: 594 error slots (6 × 99)

### Reserved for Future Split (if needed)

- T - Thermal (split from P)
- E - Electromagnetic (split from P)
- V - Voltage/Current (split from P)
- I - Import/Package (split from C)
- G - Geometry (split from R)

### The Reality

We now have 594 total error slots. We'll likely use ~200 codes in production. The system remains future-proof without being over-engineered.

---

## Key Takeaways (v0.1.4)

1. **Logic errors (L) are now first-class citizens** - Hardware design validation gets the same treatment as physics validation

2. **Short error codes remain sacred** - "I'm getting L01" is just as speakable as "I'm getting P16"

3. **Mental mapping is clear** - L means logic design, P means physics, C means compilation

4. **Same quality everywhere** - Logic errors = Physics errors = Rust borrow checker quality

5. **Teach hardware design through errors** - Users learn digital logic principles while debugging

6. **LLM-friendly by design** - Rich context enables AI assistance for logic synthesis

7. **Human remains in control** - No black boxes, no magic, just clear explanations

8. **UNIX philosophy maintained** - Compiler stays pure, fast, and offline

---

## Summary

Hardware Script v0.1.4 extends the error handling philosophy to logic synthesis:

**New in v0.1.4**:
- ✅ Logic error codes (L01-L39)
- ✅ Hardware design validation
- ✅ Bit-width inference errors
- ✅ Combinational loop detection
- ✅ Clock domain crossing warnings
- ✅ Enum and struct type checking

**Maintained from v0.1.2**:
- ✅ 3-character error codes
- ✅ Speakable and memorable
- ✅ Beautiful miette diagnostics
- ✅ Actionable hints
- ✅ No embedded LLMs
- ✅ Pure, offline compiler

**Result**: A comprehensive error system that covers the entire hardware design flow, from syntax to logic to physics to manufacturing, all with the same developer experience quality.

---

**Document Status**: Developer Experience Guide  
**Last Updated**: March 31, 2026  
**Part of**: Hardware Script v0.1.4 Documentation Suite
