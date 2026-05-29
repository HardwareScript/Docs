# Compiler Changes (v0.1.6)

**Base Documentation**: [v0.1.5 COMPILER-INTERNALS.md](../v0.1.5/COMPILER-INTERNALS.md)  
**Status**: Implementation guide for syntax unification  
**Version**: 0.1.6

---

## Overview

This document outlines the specific changes needed in the `hwc` compiler to implement the v0.1.6 Unified Grammar. The changes are designed to **simplify** the compiler, not complicate it.

**Key Insight**: By unifying the syntax, we can delete hundreds of lines of "exception" code and special cases. The compiler becomes "dumb and fast."

---

## Lexer Changes

### 1. Remove Soft Keyword Hacks and Add Logic Operators

**Problem in v0.1.5**: The lexer treated property names like `tolerance`, `trace`, `via`, `clearance` as reserved keywords, requiring complex `expect_identifier_or_keyword()` helpers in the parser.

**Solution in v0.1.6**: These are just identifiers. Remove them from the keyword list. Add logic operator keywords.

**File**: `hwc/crates/hwc-parser/src/lexer/token.rs`

```rust
// ❌ REMOVE these from Token enum:
Token::Tolerance,
Token::Trace,
Token::Via,
Token::Clearance,
// ... and many others

// ✅ They are now just Token::Identifier(String)

// ✅ ADD logic operator keywords:
#[token("and")]
And,

#[token("or")]
Or,

#[token("not")]
Not,

#[token("xor")]
Xor,

#[token("reg")]  // Lowercase register primitive
Reg,
```

**Impact**: Reduces the Token enum by ~20 variants, simplifies lexer logic, adds logic operators.

### 2. Add New Keywords

**New first-class keywords** for the Type-as-Keyword paradigm:

```rust
// ✅ ADD these to Token enum:
#[token("component")]
Component,

#[token("space")]
Space,

#[token("material")]
Material,

#[token("profile")]
Profile,

#[token("module")]
Module,

#[token("enum")]
Enum,

#[token("struct")]
Struct,

#[token("unit")]
Unit,
```

**Note**: These replace the old pattern of `define` + type string.

### 3. Remove `define` Keyword

```rust
// ❌ REMOVE:
#[token("define")]
Define,
```

**Impact**: The `define` keyword is no longer needed. Definitions start directly with the type keyword.

---

## Parser Changes

### 1. Top-Level Definition Parser

**Old Pattern (v0.1.5)**:
```rust
// Parser sees: define component "Name":
fn parse_definition(&mut self) -> Result<Definition, ParseError> {
    self.expect(&Token::Define)?;
    
    match self.current()?.token {
        Token::Component => {
            self.advance();
            let name = self.expect_string()?;  // Requires quotes
            // ...
        }
        // ...
    }
}
```

**New Pattern (v0.1.6)**:
```rust
// Parser sees: component Name:
fn parse_definition(&mut self) -> Result<Definition, ParseError> {
    match self.current()?.token {
        Token::Component => {
            self.advance();
            let name = self.expect_identifier()?;  // Bare identifier
            self.parse_component(name)
        }
        Token::Space => {
            self.advance();
            let name = self.expect_identifier()?;
            self.parse_space(name)
        }
        Token::Material => {
            self.advance();
            let name = self.expect_identifier()?;
            self.parse_material(name)
        }
        Token::Module => {
            self.advance();
            let name = self.expect_identifier()?;
            self.parse_module(name)
        }
        Token::Enum => {
            self.advance();
            let name = self.expect_identifier()?;
            self.parse_enum(name)
        }
        Token::Struct => {
            self.advance();
            let name = self.expect_identifier()?;
            self.parse_struct(name)
        }
        Token::Unit => {
            self.advance();
            let name = self.expect_identifier()?;
            self.parse_unit(name)
        }
        Token::Profile => {
            self.advance();
            let name = self.expect_identifier()?;
            self.parse_profile(name)
        }
        _ => Err(ParseError::ExpectedDefinition)
    }
}
```

**Impact**: 
- Simpler control flow (no nested match)
- No string literal parsing for names
- Each type has a dedicated parser function

### 2. Property Block Parser (Declarative World)

**Key Change**: Property assignments use `:`, not `=`.

```rust
// ✅ v0.1.6 - Parse property blocks
fn parse_property_block(&mut self) -> Result<Vec<Property>, ParseError> {
    let mut properties = Vec::new();
    
    while !self.check(&Token::Dedent) {
        // Property key is just an identifier (no special keyword handling)
        let key = self.expect_identifier()?;
        
        // Expect colon for declarative properties
        self.expect(&Token::Colon)?;
        
        // Parse the value (measurement, string, number, etc.)
        let value = self.parse_value()?;
        
        properties.push(Property { key, value });
    }
    
    Ok(properties)
}
```

**Impact**: 
- No more `expect_identifier_or_keyword()` hack
- Consistent parsing across all property blocks
- Same logic for `electrical:`, `mechanical:`, `thermal:`, `metadata:`, etc.

### 3. Logic Block Parser (Behavioral World)

**Key Change**: Logic assignments use `=`, and single `=` for comparison too.

```rust
// ✅ v0.1.6 - Parse logic blocks
fn parse_logic_block(&mut self) -> Result<Vec<LogicStatement>, ParseError> {
    let mut statements = Vec::new();
    
    while !self.check(&Token::Dedent) {
        match self.current()?.token {
            Token::Let => {
                // let wire = expression
                self.advance();
                
                // Check for 'mut' keyword
                let is_mutable = if self.check(&Token::Mut) {
                    self.advance();
                    true
                } else {
                    false
                };
                
                let name = self.expect_identifier()?;
                self.expect(&Token::Equals)?;  // Behavioral uses =
                let expr = self.parse_expression()?;
                statements.push(LogicStatement::Wire { name, expr, is_mutable });
            }
            Token::Identifier(_) => {
                // wire = expression  OR  reg.next = expression
                let target = self.parse_target()?;
                self.expect(&Token::Equals)?;  // Behavioral uses =
                let expr = self.parse_expression()?;
                statements.push(LogicStatement::Assignment { target, expr });
            }
            // ...
        }
    }
    
    Ok(statements)
}

// Parse expressions - single = for comparison
fn parse_comparison(&mut self) -> Result<Expression, ParseError> {
    let left = self.parse_additive()?;
    
    if self.check(&Token::Equals) {  // Single = for comparison
        self.advance();
        let right = self.parse_additive()?;
        return Ok(Expression::Comparison {
            op: ComparisonOp::Equal,
            left: Box::new(left),
            right: Box::new(right),
        });
    }
    
    // Other comparison operators: !=, <, >, <=, >=
    // ...
    
    Ok(left)
}
```

**Impact**: 
- Clear separation between declarative (`:`) and behavioral (`=`)
- Single `=` eliminates `==` confusion
- Context determines whether `=` is assignment or comparison
- Easier error messages

### 4. List Parser (Universal `[]` Syntax)

**Key Change**: All lists support bracket notation.

```rust
// ✅ v0.1.6 - Universal list parser
fn parse_list<T, F>(&mut self, parse_item: F) -> Result<Vec<T>, ParseError>
where
    F: Fn(&mut Self) -> Result<T, ParseError>
{
    // Check for bracket notation
    if self.check(&Token::OpenBracket) {
        self.advance();
        let mut items = Vec::new();
        
        while !self.check(&Token::CloseBracket) {
            items.push(parse_item(self)?);
            
            if self.check(&Token::Comma) {
                self.advance();
            } else if !self.check(&Token::CloseBracket) {
                return Err(ParseError::ExpectedCommaOrBracket);
            }
        }
        
        self.expect(&Token::CloseBracket)?;
        return Ok(items);
    }
    
    // Fallback: comma-separated or newline-separated (backward compatibility)
    let mut items = Vec::new();
    loop {
        items.push(parse_item(self)?);
        
        if self.check(&Token::Comma) {
            self.advance();
        } else if self.check(&Token::Newline) || self.check(&Token::Dedent) {
            break;
        } else {
            return Err(ParseError::ExpectedCommaOrNewline);
        }
    }
    
    Ok(items)
}
```

**Usage**:
```rust
// Parse pins: [A, B, C]
fn parse_pins(&mut self) -> Result<Vec<Pin>, ParseError> {
    self.expect(&Token::Pins)?;
    self.expect(&Token::Colon)?;
    self.parse_list(|parser| parser.parse_pin())
}

// Parse enum values: [Red, Green, Blue]
fn parse_enum_values(&mut self) -> Result<Vec<String>, ParseError> {
    self.expect(&Token::Values)?;
    self.expect(&Token::Colon)?;
    self.parse_list(|parser| parser.expect_identifier())
}
```

**Impact**: 
- One list parser for all contexts
- Backward compatible with comma/newline syntax
- Bracket notation is now canonical

### 5. Flexible Metadata/Profile Parser

**Key Change**: Accept arbitrary key-value pairs in `metadata` and `profile` blocks.

```rust
// ✅ v0.1.6 - Flexible dictionary parser
fn parse_metadata(&mut self) -> Result<HashMap<String, String>, ParseError> {
    self.expect(&Token::Metadata)?;
    self.expect(&Token::Colon)?;
    self.expect(&Token::Indent)?;
    
    let mut metadata = HashMap::new();
    
    while !self.check(&Token::Dedent) {
        // Accept ANY identifier as a key
        let key = self.expect_identifier()?;
        self.expect(&Token::Colon)?;
        
        // Value must be a string
        let value = self.expect_string()?;
        
        // No validation - accept any key
        metadata.insert(key, value);
    }
    
    self.expect(&Token::Dedent)?;
    Ok(metadata)
}
```

**Impact**: 
- No more "Unknown metadata field" errors
- Users can add custom tracking fields
- Future-proof for BOM generation

### 6. Coordinate Origin Parser

**Key Change**: Parse terse origin syntax with `by` keyword.

```rust
// ✅ v0.1.6 - Terse origin syntax
fn parse_origin(&mut self) -> Result<Origin, ParseError> {
    self.expect(&Token::Origin)?;
    self.expect(&Token::Colon)?;
    
    // Parse horizontal origin (tl, tr, bl, br, c)
    let horizontal = match self.expect_identifier()?.as_str() {
        "tl" => HorizontalOrigin::TopLeft,
        "tr" => HorizontalOrigin::TopRight,
        "bl" => HorizontalOrigin::BottomLeft,
        "br" => HorizontalOrigin::BottomRight,
        "c" => HorizontalOrigin::Center,
        other => return Err(ParseError::InvalidOrigin(other.to_string())),
    };
    
    self.expect(&Token::By)?;
    
    // Parse vertical origin (t, b)
    let vertical = match self.expect_identifier()?.as_str() {
        "t" => VerticalOrigin::TopDown,
        "b" => VerticalOrigin::BottomUp,
        other => return Err(ParseError::InvalidOrigin(other.to_string())),
    };
    
    Ok(Origin { horizontal, vertical })
}
```

**Impact**: 
- Maintains visual rhythm with `dimensions` and `grid`
- Terse and professional
- Industry-standard shorthand

### 7. Struct Parser (Simplified)

**Key Change**: Structs are bare bit-width tables (no `fields:` keyword).

```rust
// ✅ v0.1.6 - Simplified struct parser
fn parse_struct(&mut self, name: Identifier) -> Result<StructDefinition, ParseError> {
    self.expect(&Token::Colon)?;
    self.expect(&Token::Indent)?;
    
    let mut fields = Vec::new();
    
    // Parse fields directly (no 'fields:' keyword)
    while !self.check(&Token::Dedent) {
        let field_name = self.expect_identifier()?;
        self.expect(&Token::OpenBracket)?;
        let width = self.expect_integer()?;
        self.expect(&Token::CloseBracket)?;
        
        fields.push(StructField {
            name: field_name,
            width: width as usize,
        });
    }
    
    self.expect(&Token::Dedent)?;
    
    Ok(StructDefinition { name, fields })
}
```

**Example**:
```hw
struct Instruction:
    opcode[4]
    rs1[5]
    rs2[5]
    rd[5]
```

**Impact**: 
- Cleaner syntax (no boilerplate `fields:` keyword)
- Structs look like clean bit-width tables
- Simpler parser logic

---

## AST Changes

### 1. Definition Nodes

**Old AST (v0.1.5)**:
```rust
pub enum Definition {
    Component {
        name: String,  // Was a string literal
        // ...
    },
    Material {
        name: String,  // Was a string literal
        // ...
    },
}
```

**New AST (v0.1.6)**:
```rust
pub enum Definition {
    Component {
        name: Identifier,  // Now a bare identifier
        // ...
    },
    Material {
        name: Identifier,  // Now a bare identifier
        // ...
    },
}

pub struct Identifier {
    pub name: String,
    pub span: Span,
}
```

**Impact**: 
- Type safety (identifiers vs string literals)
- Better error messages (can point to identifier location)

### 2. Property Nodes

**Old AST (v0.1.5)**:
```rust
pub struct Property {
    pub key: String,
    pub value: Value,
    pub operator: PropertyOperator,  // Could be : or =
}

pub enum PropertyOperator {
    Colon,
    Equals,
}
```

**New AST (v0.1.6)**:
```rust
// Declarative properties (always use :)
pub struct Property {
    pub key: Identifier,
    pub value: Value,
    // No operator field - always colon in declarative context
}

// Behavioral assignments (always use =)
pub struct Assignment {
    pub target: Target,
    pub value: Expression,
    // No operator field - always equals in behavioral context
}
```

**Impact**: 
- Clear separation at AST level
- No ambiguity about which operator to use
- Type system enforces the boundary

---

## Symbol Table Changes

### 1. Identifier Storage

**Old (v0.1.5)**:
```rust
// Names were stored as strings
symbol_table.insert("Resistor_0805".to_string(), Symbol::Component { ... });
```

**New (v0.1.6)**:
```rust
// Names are stored as identifiers (no quotes)
symbol_table.insert(Identifier::new("Resistor_0805"), Symbol::Component { ... });
```

**Impact**: 
- Consistent with source code (no quote stripping)
- Better error messages (can show where identifier was defined)

---

## Error Message Improvements

### 1. Context-Aware Errors

**Old (v0.1.5)**:
```
Error: Expected ':' or '='
```

**New (v0.1.6)**:
```
Error: Expected ':' in property block (use '=' only in logic blocks)
  --> main.hw:15:20
   |
15 |     resistance = 10kΩ
   |                ^ Expected ':' here
   |
   = help: Properties use ':' for assignment. Use '=' only in logic: blocks.
```

**Impact**: 
- Teaches users the boundary rule
- Reduces confusion about when to use `:` vs `=`

### 2. Identifier vs String Errors

**Old (v0.1.5)**:
```
Error: Expected string literal
```

**New (v0.1.6)**:
```
Error: Expected identifier (no quotes needed)
  --> main.hw:5:11
   |
5  | component "Resistor":
   |           ^^^^^^^^^^^ Remove quotes
   |
   = help: In v0.1.6, type names are bare identifiers: component Resistor:
```

**Impact**: 
- Guides users through migration
- Explains the v0.1.6 syntax change

---

## Testing Strategy

### 1. Lexer Tests

```rust
#[test]
fn test_type_keywords() {
    let input = "component space material module enum struct unit profile";
    let tokens = lex(input);
    
    assert_eq!(tokens[0], Token::Component);
    assert_eq!(tokens[1], Token::Space);
    assert_eq!(tokens[2], Token::Material);
    // ...
}

#[test]
fn test_no_soft_keywords() {
    let input = "tolerance trace via clearance";
    let tokens = lex(input);
    
    // All should be identifiers, not keywords
    assert!(matches!(tokens[0], Token::Identifier(_)));
    assert!(matches!(tokens[1], Token::Identifier(_)));
    assert!(matches!(tokens[2], Token::Identifier(_)));
    assert!(matches!(tokens[3], Token::Identifier(_)));
}
```

### 2. Parser Tests

```rust
#[test]
fn test_component_definition() {
    let input = r#"
component Resistor:
    pins: [A, B]
    electrical:
        resistance: 10kΩ
"#;
    
    let ast = parse(input).unwrap();
    
    match &ast.definitions[0] {
        Definition::Component { name, .. } => {
            assert_eq!(name.name, "Resistor");  // No quotes
        }
        _ => panic!("Expected component definition"),
    }
}

#[test]
fn test_property_colon() {
    let input = r#"
component Test:
    electrical:
        resistance: 10kΩ
"#;
    
    // Should parse successfully with colon
    assert!(parse(input).is_ok());
}

#[test]
fn test_logic_equals() {
    let input = r#"
module Test:
    logic:
        let sum = A + B
        count.next = count + 1
"#;
    
    // Should parse successfully with equals
    assert!(parse(input).is_ok());
}

#[test]
fn test_universal_list_syntax() {
    let input = r#"
component Test:
    pins: [A, B, C, D]
    
enum State:
    values: [Idle, Active, Done]
"#;
    
    let ast = parse(input).unwrap();
    
    // Verify bracket notation works
    match &ast.definitions[0] {
        Definition::Component { pins, .. } => {
            assert_eq!(pins.len(), 4);
        }
        _ => panic!("Expected component"),
    }
}
```

### 3. End-to-End Tests

```rust
#[test]
fn test_standard_library_units() {
    let input = include_str!("../stdlib/units.hw");
    
    // All unit definitions should parse successfully
    assert!(parse(input).is_ok());
}

#[test]
fn test_verilog_killer_stress_test() {
    let input = include_str!("../stdlib/logic/registers.hw");
    
    // Complex logic should parse and compile
    let result = compile(input);
    assert!(result.is_ok());
}
```

---

## Migration Tool

A syntax transformer will automatically migrate v0.1.5 code to v0.1.6.

```rust
pub fn migrate_v015_to_v016(input: &str) -> String {
    let mut output = input.to_string();
    
    // 1. Remove 'define' keyword
    output = output.replace("define component", "component");
    output = output.replace("define space", "space");
    output = output.replace("define material", "material");
    output = output.replace("define module", "module");
    output = output.replace("define enum", "enum");
    output = output.replace("define struct", "struct");
    output = output.replace("define unit", "unit");
    output = output.replace("define profile", "profile");
    
    // 2. Remove quotes from type names (regex-based)
    // component "Name": -> component Name:
    let re = Regex::new(r#"(component|space|material|module|enum|struct|unit|profile)\s+"([^"]+)":"#).unwrap();
    output = re.replace_all(&output, "$1 $2:").to_string();
    
    // 3. Origin syntax stays unchanged (tl by t)
    // No change needed
    
    // 4. Replace Reg with reg in logic blocks
    output = output.replace("Reg(", "reg(");
    
    // 5. Replace == with = in logic blocks (context-aware)
    // This requires proper parsing, but simple regex for common cases:
    let re_comparison = Regex::new(r"if\s+(\w+)\s+==\s+").unwrap();
    output = re_comparison.replace_all(&output, "if $1 = ").to_string();
    
    // 6. Remove 'fields:' from struct definitions
    output = output.replace("    fields:\n", "");
    
    // 7. Property assignments stay as colons (no change needed)
    // 8. Logic assignments stay as equals (no change needed)
    
    output
}
```

---

## Performance Impact

**Expected Performance Improvements**:

1. **Lexer**: ~5-10% faster (fewer token variants to check)
2. **Parser**: ~10-15% faster (simpler control flow, no nested matches)
3. **Memory**: ~10% reduction (smaller AST nodes, no operator enums)

**Benchmark Results** (to be measured):
```
v0.1.5: 1000 files in 2.5s
v0.1.6: 1000 files in 2.1s (16% faster)
```

---

## Summary

The v0.1.6 compiler changes are designed to **simplify**, not complicate:

1. **Lexer**: Remove soft keywords, add type keywords, remove `define`
2. **Parser**: Unified list parser, clear `:` vs `=` boundary, flexible metadata
3. **AST**: Cleaner node types, type-safe identifiers
4. **Errors**: Context-aware messages that teach the boundary rule
5. **Performance**: Faster compilation, smaller memory footprint

**The compiler becomes "dumb and fast"—it doesn't need special cases, just universal rules.**
