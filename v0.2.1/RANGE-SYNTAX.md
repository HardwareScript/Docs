# Range Syntax Specification

**Version:** 0.2.1  
**Status:** Implemented  
**Decision:** Final (No Further Debate)

---

## Executive Summary

HardwareScript uses **Rust/Swift-style explicit range syntax** to eliminate off-by-one errors in parametric hardware generation:

- **`0..N`** (exclusive): Runs **N times** — count-driven iteration
- **`0..=N`** (inclusive): Runs **N+1 times** — bound-driven iteration

This design provides mathematical determinism for both parametric arrays (via counts) and explicit hardware bounds (bit slices, layer ranges).

---

## Syntax

### Exclusive Range: `..` (Half-Open)

```hardware
for i in 0..3:
    # Runs 3 times: i = 0, 1, 2
    add via(...) named Via_{i}
```

**Use Cases:**
- Parametric via/component counts: `let via_count = 3; for i in 0..via_count:`
- Array generation where N means "N items"
- Prevents the `- 1` off-by-one bug

**Iteration Count:** `end - start`

---

### Inclusive Range: `..=` (Closed)

```hardware
for i in 0..=3:
    # Runs 4 times: i = 0, 1, 2, 3
    add via(...) named Via_{i}
```

**Use Cases:**
- Hardware bounds: "From bit 0 up to and including bit 3"
- Matching Verilog/VHDL bit slice notation: `wire [7:0]` = 8 bits
- Explicit terminal indices in datasheets

**Iteration Count:** `end - start + 1`

---

## Design Rationale

### The Problem We Solved

**Before (Implicit Inclusive - Ruby/Hardware Style):**
```hardware
let via_count = 3
for i in 0..via_count:  # Generated 4 vias (0,1,2,3) — bug!
```

Engineers were forced to write `0..(via_count - 1)` everywhere, causing off-by-one errors.

**After (Explicit Syntax):**
```hardware
let via_count = 3
for i in 0..via_count:   # Generates 3 vias (0,1,2) — correct!
```

No mental arithmetic. No `- 1` bugs.

---

### Why This Choice is Mathematically Optimal

**Dijkstra's Proof (EWD831):** Half-open ranges `[start, end)` are superior because:

1. **Element count is trivial:** `count = end - start` (no +1 needed)
2. **Ranges concatenate perfectly:** `0..M` followed by `M..N` covers `0..N` with no gaps or overlaps
3. **Empty ranges are natural:** `5..5` is empty (0 iterations), not an error

**When You Need Inclusive:** Use `..=` explicitly (hardware bit slices, layer bounds).

---

## Index Base: 0-Based (Non-Negotiable)

HardwareScript uses **0-based indexing** universally:

- Binary addressing: LSB is always bit 0 (2⁰)
- Memory offsets: Base pointer + 0 bytes
- Modulo arithmetic: `(row + col) % 2` works for checkerboards starting at (0,0)
- 98% of modern engineers expect 0-based indexing

**Not up for debate.** This is engineering reality.

---

## Comparison with Other Languages

| Language       | Range Syntax      | Behavior                      | HardwareScript Equivalent |
|----------------|-------------------|-------------------------------|---------------------------|
| **Rust**       | `0..3` / `0..=3`  | Explicit (3 items / 4 items)  | **Identical** ✅          |
| **Swift**      | `0..<3` / `0...3` | Explicit (3 items / 4 items)  | Same semantics            |
| **Python**     | `range(0, 3)`     | Exclusive only (3 items)      | Our `0..3`                |
| **MATLAB**     | `0:3`             | Inclusive (4 items)           | Our `0..=3`               |
| **Verilog**    | `[7:0]`           | Inclusive (8 bits)            | Our `7..=0` (bit slices)  |
| **Ruby (Old)** | `0..3`            | Inclusive (4 items)           | **Rejected** ❌           |

---

## Examples

### Via Array (Count-Driven)
```hardware
space Test_Substrate:
    dimensions: [10mm, 10mm, 1.6mm]
    
    # Place 3 vias exactly
    for i in 0..3:
        add via(Copper, 0.3mm) named Via_{i} at [x: i * 2mm, y: 5mm, z: 0]:
            from_layer: substrate
            to_layer: metal1
            net: GND
```
**Output:** Via_0, Via_1, Via_2 (3 vias)

---

### Power Grid (Explicit Bounds)
```hardware
space Power_Mesh:
    dimensions: [5mm, 5mm, 1.6mm]
    
    # Row 0 through row 4 inclusive (5 rows)
    for row in 0..=4:
        for col in 0..=4:
            add via(Copper, 0.2mm) named GND_R{row}_C{col} at [...]:
                net: GND
```
**Output:** 5×5 = 25 vias (rows 0,1,2,3,4 × cols 0,1,2,3,4)

---

### Bit Slice (Hardware Notation)
```hardware
logic decoder:
    let opcode = instruction[7..=4]  # Bits 4,5,6,7 (4 bits inclusive)
    let immediate = instruction[3..0] # ERROR: Use 0..4 or 0..=3
```

For bit slices, `..=` matches Verilog `[high:low]` notation.

---

## Migration from Old Code

**If you have code that assumes inclusive `..`:**

Replace `for i in 0..N:` with one of:
- `for i in 0..=N:` (if you want N+1 iterations)
- `for i in 0..(N+1):` (count-driven with explicit +1)
- `for i in 0..N:` and update loop body expectations

**The compiler will not guess.** Explicit is correct.

---

## Decision Authority

This specification is **final and non-negotiable**:

- **Inclusive `..` was rejected** because it causes parametric count bugs
- **Explicit syntax mirrors Rust/Swift**, languages proven at scale (Mozilla, Apple)
- **Supported by formal proof** (Dijkstra EWD831)
- **Matches 98% of modern engineering practice** (C, C++, Rust, Python, JavaScript)

No further debate will be entertained. Engineers who disagree may fork the language.

---

## Summary Table

| Syntax    | Name           | Behavior        | Iterations for `0..3` or `0..=3` | Use Case                  |
|-----------|----------------|-----------------|----------------------------------|---------------------------|
| `0..3`    | Exclusive      | Half-open `[0,3)` | 3 (values: 0,1,2)                | Parametric counts         |
| `0..=3`   | Inclusive      | Closed `[0,3]`    | 4 (values: 0,1,2,3)              | Hardware bounds, bit slices |

---

**End of Specification.**  
If you encounter ambiguity in for-loop behavior, this document is the authoritative reference.
