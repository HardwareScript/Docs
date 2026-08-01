# Multi-Line Declaration Syntax: Design Rationale

**Version:** 0.2.0  
**Date:** 2026-08-01  
**Status:** Implemented

## The Problem

HardwareScript was originally designed with Ruby-inspired clean syntax, prioritizing whitespace-based structure over explicit delimiters like braces and parentheses. This philosophy worked beautifully for simple declarations:

```hardware
add plane(Aluminum) named Pad_A on layer: metal1:
    net: Signal
    shape: Pad(600nm, 600nm)
```

However, as we introduced **middle-level relational abstractions** (v0.1.9), declaration lines grew significantly longer:

```hardware
# ❌ Too long - hard to read, hard to maintain
add plane(Aluminum) named Metal_Pad align: center_x with Poly_Strip align: center_y with Poly_Strip on layer: metal1:
    net: Signal
    shape: SquarePad(size: 700nm)
```

This 110-character line violates readability principles and makes code review difficult.

## Why Not Just Use Newlines?

We tried allowing implicit multi-line continuation using indentation alone:

```hardware
add plane(Aluminum) named Metal_Pad
    align: center_x with Poly_Strip
    align: center_y with Poly_Strip
    on layer: metal1:
        net: Signal
```

**This failed due to a fundamental ambiguity in indentation-based parsing:**

When the lexer encounters indentation after `named Metal_Pad`, it generates an `INDENT` token. This token signals the start of a **body block** in Python-style syntax. But here, we want **declaration continuation**, not a body block.

The parser cannot distinguish between:
1. Continuation of the declaration (align constraints)
2. Start of the body block (net, shape properties)

Both use indentation, creating **lexical ambiguity** that breaks the parsing model.

## The Solution: Brace-Grouped Constraints

Following the proven design of **Terraform/HCL** and **Kotlin**, we introduced **optional braces `{ }`** to explicitly group multi-line constraint clauses:

```hardware
# ✅ Clean, unambiguous, readable
add plane(Aluminum) named Metal_Pad {
    align: center_x with Poly_Strip
    align: center_y with Poly_Strip
} on layer: metal1:
    net: Signal
    shape: SquarePad(size: 700nm)
```

### Why This Works

1. **Explicit Grouping:** Braces clearly signal "these constraints belong together"
2. **No Ambiguity:** The parser knows indentation inside `{ }` is continuation, not a new block
3. **Familiar Pattern:** Developers coming from Rust, HCL, CSS recognize this immediately
4. **Minimal Overhead:** Only ONE pair of braces, not excessive nesting

## Design Philosophy: Balanced Pragmatism

We didn't abandon Ruby's clean aesthetic — we evolved it pragmatically:

| Context | Syntax Choice | Rationale |
|---------|---------------|-----------|
| **Simple declarations** | No braces (Ruby-style) | `add plane(Aluminum) named Pad on layer: metal1:` |
| **Complex constraints** | Braces (HCL-style) | `{ align: ... \n align: ... }` for readability |
| **Body blocks** | Colon + indent (Python-style) | Universal pattern for nested properties |

**The rule:** Use the simplest syntax that maintains clarity. Braces are **optional** — only use them when declaration lines exceed ~80 characters.

## Comparison with Other DSLs

### Terraform (HCL) — Our Model
```hcl
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Example"
  }
}
```
✅ Braces for multi-line blocks  
✅ Clean and explicit

### Python — Pure Indentation
```python
def long_function(
    argument_one,
    argument_two
):
    pass
```
❌ Requires special continuation rules or parentheses  
❌ Ambiguous with body indentation

### Ruby — No Explicit Delimiters
```ruby
add_plane aluminum named: :pad_a,
          align: { center_x: :poly_strip }
          on: :metal1 do
  net :signal
end
```
❌ Hard to parse unambiguously  
❌ Doesn't scale to complex constraints

## Implementation Details

The parser uses **context-aware brace handling**:

1. After `named ComponentName`, check for `{` token
2. If found, enter **constraint grouping mode**
3. Parse multiple `align:` / `right_of` / etc. clauses
4. Exit on `}` token
5. Continue with `on layer:` or other clauses

```rust
// Simplified parser logic
if self.check(&Token::OpenBrace) {
    self.advance();
    let constraints = self.parse_relational_constraints_block()?;
    self.expect(&Token::CloseBrace)?;
} else {
    // Inline single-line syntax
    let constraints = self.parse_relational_constraints()?;
}
```

## Examples

### Plane with Multiple Alignments
```hardware
add plane(Aluminum) named Metal_Pad {
    align: center_x with Poly_Strip
    align: center_y with Poly_Strip
} on layer: metal1:
    net: Signal
    shape: SquarePad(size: 700nm)
```

### Contact with Placement Constraints
```hardware
add contact(Titanium_Silicide) named Test_Via {
    at: Poly_Strip.center
    spanning layer: poly to metal1
}:
    net: Signal
    diameter: 200nm
    plating_thickness: 5nm
```

### Single-Line (Still Valid)
```hardware
# When constraints are short, inline syntax works fine
add plane(Aluminum) named Pad_A on layer: metal1:
    net: Signal
```

## Conclusion

By introducing **optional brace-grouped constraints**, we achieved:

✅ **Readability:** Long declarations break cleanly across lines  
✅ **Clarity:** No ambiguity between continuation and body blocks  
✅ **Familiarity:** Pattern matches Terraform, Kotlin, CSS  
✅ **Flexibility:** Developers choose single-line or multi-line based on complexity

This is not a departure from clean syntax — it's an **evolution guided by real-world usage**. The best DSLs balance aesthetic ideals with practical engineering constraints. HardwareScript now does both.

---

**See Also:**
- [Spatial Synthesis Abstraction](../v0.1.9/Spatial_Synthesis_Abstraction.md) — Middle-level relational syntax
- [Language Specification](../v0.1/LANGUAGE-SPEC.md) — Core syntax reference
