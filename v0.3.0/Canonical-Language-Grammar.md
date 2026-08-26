# HardwareScript v0.3.0: Canonical Language Grammar & Lexical Specification

**Document Type:** Authoritative Language Specification  
**Target Version:** v0.3.0  
**Status:** Approved for Implementation (Milestone 1)  
**Focus:** Lexical Tokens, Formal EBNF Grammar, Explicit Block Scoping, Comptime Synthesis, Natural Boolean Keywords, and Cascade-Proof Parsing  

---

## 1. The Executive Audit: What Went Wrong and Why We Must Pivot

Between versions v0.1.0 and v0.2.2, HardwareScript suffered from continuous compiler redesigns, language churn, and usability friction. A rigorous post-mortem identifies two fundamental architectural flaws that paralyzed development:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       THE TWO FATAL ARCHITECTURAL FLAWS                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. THE SIGNIFICANT-WHITESPACE CASCADING ERROR TRAP                         │
│     • The lexer maintained an internal indentation stack (INDENT / DEDENT). │
│     • Single syntax errors failed to emit or consume synthetic DEDENTs.     │
│     • The parser remained trapped inside nested blocks, misinterpreting     │
│       every subsequent top-level statement.                                 │
│     • Result: 1 missing expression caused 40+ cascading downstream errors. │
│                                                                             │
│  2. THE DECLARATIVE KEYWORD "WHACK-A-MOLE" TRAP                             │
│     • Complex layouts were forced into static declarative keywords.         │
│     • Every new physical pattern required adding specialized tokens         │
│       (`align`, `right_of`, `matrix`, `fill`, `chain_x`, `overlap`).        │
│     • The compiler tried to solve placement via opaque geometric heuristics │
│       (`FixedTransform2D`, `relational_resolver.rs`).                       │
│     • Result: Designers spent hours tweaking numbers by 50nm to satisfy an  │
│       opaque constraint solver instead of writing deterministic code.       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What Would Have Happened If We Kept the Trend?
1. **Total Compiler Gridlock:** Every new semiconductor layout topology (such as interdigitated differential pairs, R-2R DAC ladders, multi-finger RF gates, or BGA escape fans) would have required modifying the Rust lexer, parser, and AST lowering passes.
2. **Unusable Tooling & IDE Support:** Language Server Protocols (LSP) cannot perform reliable incremental parsing, autocomplete, or diagnostics on ambiguous, significant-whitespace grammars with relational guessing solvers.
3. **Failure to Tape Out:** Designers would continue spending 90% of their time fighting syntax bugs and layout solver drifts rather than designing, verifying, and taping out physical silicon and PCBs.

---

## 2. The v0.3.0 Paradigm Shift: Compile-Time Turing-Complete HDL

HardwareScript v0.3.0 transforms from a **fragile markup DSL** into a **Compile-Time Generative Hardware Description Language (`comptime` HDL)**:

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                   HARDWARESCRIPT SOURCE CODE (.hw)                      │
 │  • Explicit curly brace scoping `{ ... }` (Zero Indentation Cascades)   │
 │  • Turing-complete compile-time evaluation (`fn`, `let`, `for`, `if`)   │
 │  • First-class physical dimensional units (`10um`, `1.8V`, `20mA`)      │
 │  • Algorithmic layout generators (PCells) defined in `@std`             │
 │  • Readable English boolean logic (`and`, `or`, `not`)                  │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │
                                      ▼ [hwc compile / eval]
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                COMPTIME INTERPRETER / VM (hwc-eval)                     │
 │  • Step-bounded execution ($10^7$ step limit prevents infinite loops)   │
 │  • Deterministic integer DBU picometer math ($1\text{ pm} = 10^{-12}\text{ m}$)│
 │  • Emits discrete physical primitives directly into master EntityGraph  │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │              SINGLE MASTER DATABASE (EntityGraph / DBU)                 │
 │  • Flat 2D/3D polygons, contact pillars, wire vectors                   │
 │  • Topological netlist (Node-to-pin connectivity)                       │
 │  • Semiconductor device contracts (SPICE model hooks)                   │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                 PHYSICAL SYNTHESIS & TAPE-OUT GATES                     │
 │  • Topological Line-Search Router & QP Legalizer                        │
 │  • G-Cell Sweep-Line DRC & PIVB Connectivity Solver                     │
 │  • Sakurai BEM Parasitic Extraction (.sp)                               │
 │  • Native GDSII / OASIS / Gerber / DXF / GLB Stream                     │
 └─────────────────────────────────────────────────────────────────────────┘
```

### Why Compile-Time Turing Completeness with Strategic Boundaries?
* **Full Expressiveness Upstream:** Designers can use loops, math, array transformations, trigonometry, and user-defined functions to algorithmically generate physical layouts.
* **Deterministic Simplicity Downstream:** The compiler core no longer makes guesses. It accepts explicit picometer polygons, contacts, and devices.
* **Hermetic Sandbox:** The language execution has no access to real-time OS clocks, random seeds, or network sockets, guaranteeing bit-identical GDSII and SPICE builds across all platforms.

---

## 3. Complete Lexical Specification

In HardwareScript v0.3.0, **significant indentation is completely eliminated.** All block scoping is delimited explicitly by curly braces (`{` and `}`). Whitespace (spaces, tabs, newlines) serves strictly as token separators.

### 3.1 Lexical Tokens Overview

```
                      HARDWARESCRIPT v0.3.0 TOKEN SET
                      
 ┌───────────────────────────────────┬───────────────────────────────────┐
 │ COMPTIME ENGINE KEYWORDS          │ PHYSICAL SYNTHESIS KEYWORDS       │
 ├───────────────────────────────────┼───────────────────────────────────┤
 │ `fn`        `let`       `mut`     │ `space`     `module`    `device`  │
 │ `struct`    `enum`      `if`      │ `material`  `profile`   `route`   │
 │ `else`      `for`       `in`      │ `test`      `nets`      `pins`    │
 │ `return`    `assert`    `match`   │                                   │
 │ `import`    `export`    `true`    │                                   │
 │ `false`     `const`               │                                   │
 ├───────────────────────────────────┼───────────────────────────────────┤
 │ BOOLEAN OPERATOR KEYWORDS         │ OPERATORS & DELIMITERS            │
 ├───────────────────────────────────┼───────────────────────────────────┤
 │ `and`       `or`        `not`     │ `{` `}` `(` `)` `[` `]`           │
 │                                   │ `:` `,` `.` `->` `=>`             │
 │                                   │ `=` `+` `-` `*` `/` `%`           │
 │                                   │ `==` `!=` `<` `>` `<=` `>=`       │
 │                                   │ `..` (exclusive) `..=` (inclusive)│
 └───────────────────────────────────┴───────────────────────────────────┘
```

---

### 3.2 Reserved Keyword Registry

HardwareScript reserves exactly **26 core keywords**:

```
and         assert      const       device      else        enum        
export      false       fn          for         if          import      
in          let         match       material    module      mut         
nets        not         or          pins        profile     return      
route       space       struct      test        true
```

#### Purged Keywords (Deleted from Grammar)
The following keywords and legacy tokens from earlier versions are permanently deleted from the language grammar:
* **Purged Prepositions & Modifiers:** `align`, `with`, `inside`, `right_of`, `left_of`, `above`, `below`, `center_between`, `center_x`, `center_y`, `origin`, `resolution`, `grid`, `absolute`, `last`, `after`, `spanning`, `by`, `matrix`, `fill`, `chain_x`, `shared_gate`, `device_nets`, `prefer`, `require`.
* **Purged Punctuation:** Cryptic C-style boolean operators `&&`, `||`, `!` are permanently replaced by the readable keywords `and`, `or`, `not`.

---

### 3.3 Numeric & Physical Measurement Literals

HardwareScript provides native, first-class lexical support for engineering measurements with physical units:

#### A. Integer Literals
```
Decimal:      0, 42, 1000000
Hexadecimal:  0x1F, 0xDEADBEEF
Binary:       0b1010_0011
Octal:        0o755
```

#### B. Floating-Point Literals
```
Standard:     0.0, 3.14159265, 0.001
Scientific:   1.68e-8, 2.5E+3, 1e-12
```

#### C. Physical Measurement Literals
A measurement literal consists of a numeric prefix (integer or float) immediately followed by a case-sensitive physical unit suffix with no intervening whitespace:

$$\text{Measurement} ::= (\text{IntegerLiteral} \mid \text{FloatLiteral}) \times \text{UnitSuffix}$$

Supported standard physical units:

| Category | Unit Suffixes | Canonical Internal Unit (i64/i128) |
| :--- | :--- | :--- |
| **Length / Distance** | `pm`, `nm`, `um`, `mm`, `cm`, `m`, `mil`, `in` | Picometers ($1\text{ pm} = 10^{-12}\text{ m}$) |
| **Voltage / Potential** | `nV`, `uV`, `mV`, `V`, `kV` | Nanovolts ($1\text{ nV} = 10^{-9}\text{ V}$) |
| **Current** | `pA`, `nA`, `uA`, `mA`, `A`, `kA` | Picoamperes ($1\text{ pA} = 10^{-12}\text{ A}$) |
| **Resistance** | `uOhm`, `mOhm`, `Ohm`, `kOhm`, `MOhm`, `GOhm` | Micro-ohms ($1\,\mu\Omega = 10^{-6}\,\Omega$) |
| **Capacitance** | `aF`, `fF`, `pF`, `nF`, `uF`, `mF`, `F` | Attofarads ($1\text{ aF} = 10^{-18}\text{ F}$) |
| **Inductance** | `pH`, `nH`, `uH`, `mH`, `H` | Picohenries ($1\text{ pH} = 10^{-12}\text{ H}$) |
| **Time / Delay** | `fs`, `ps`, `ns`, `us`, `ms`, `s` | Femtoseconds ($1\text{ fs} = 10^{-15}\text{ s}$) |
| **Frequency** | `Hz`, `kHz`, `MHz`, `GHz`, `THz` | Hertz ($1\text{ Hz}$) |
| **Power** | `pW`, `nW`, `uW`, `mW`, `W`, `kW` | Picowatts ($1\text{ pW} = 10^{-12}\text{ W}$) |
| **Conductivity** | `S_m`, `kS_m`, `MS_m` | Siemens per meter |
| **Resistivity** | `ohm_m`, `ohm_cm` | Ohm-meters |
| **Angle** | `deg`, `rad` | Micro-degrees / Micro-radians |
| **Temperature** | `degC`, `K` | Millikelvin |

---

### 3.4 String Literals & Interpolation

String literals are enclosed in double quotes (`"..."`) and support standard escape sequences (`\n`, `\t`, `\r`, `\\`, `\"`, `\0`).

HardwareScript natively supports compile-time string interpolation using inline curly braces `{expression}`:

```hardware
let stage = 3
let width = 2.5um
let net_name = "NODE_{stage}_A"                  // Resolves to "NODE_3_A"
println("Stage {stage}: Channel Width = {width}") // Prints: "Stage 3: Channel Width = 2.5um"
```

---

## 4. Formal Grammar Specification (EBNF)

Below is the formal Extended Backus-Naur Form (EBNF) grammar for HardwareScript v0.3.0.

### 4.1 Program & Top-Level Declarations

```ebnf
Program ::= ( TopLevelItem )* EOF

TopLevelItem ::= ImportDecl
               | ExportDecl
               | StructDecl
               | EnumDecl
               | FunctionDecl
               | MaterialDecl
               | ProfileDecl
               | DeviceDecl
               | ModuleDecl
               | SpaceDecl
               | TestDecl

ImportDecl ::= "import" ( ImportSymbols "from" StringLiteral 
                        | StringLiteral )

ImportSymbols ::= "*" 
                | "{" IdentifierList "}"
                | Identifier

ExportDecl ::= "export" ( FunctionDecl 
                        | StructDecl 
                        | EnumDecl 
                        | MaterialDecl 
                        | DeviceDecl 
                        | ProfileDecl )

IdentifierList ::= Identifier ( "," Identifier )* ( "," )?
```

---

### 4.2 Comptime Types & Functions

```ebnf
StructDecl ::= "struct" Identifier "{" ( StructField ( "," StructField )* ( "," )? )? "}"
StructField ::= Identifier ":" TypeExpr

EnumDecl ::= "enum" Identifier "{" ( EnumVariant ( "," EnumVariant )* ( "," )? )? "}"
EnumVariant ::= Identifier ( "(" TypeExprList ")" | "{" StructFieldList "}" )?

FunctionDecl ::= "fn" Identifier "(" ( ParameterList )? ")" ( "->" TypeExpr )? Block

ParameterList ::= Parameter ( "," Parameter )* ( "," )?
Parameter ::= Identifier ":" TypeExpr ( "=" Expression )?

TypeExpr ::= Identifier ( "[" TypeExprList "]" )?
           | "(" TypeExprList ")"
           | "fn" "(" TypeExprList ")" ( "->" TypeExpr )?

TypeExprList ::= TypeExpr ( "," TypeExpr )* ( "," )?
```

---

### 4.3 Statements & Control Flow

```ebnf
Block ::= "{" ( Statement )* "}"

Statement ::= LetStmt
            | AssignmentStmt
            | IfStmt
            | ForStmt
            | ReturnStmt
            | AssertStmt
            | ExpressionStmt

LetStmt ::= "let" ( "mut" )? Identifier ( ":" TypeExpr )? "=" Expression ( ";" )?

AssignmentStmt ::= TargetExpr ( "=" | "+=" | "-=" | "*=" | "/=" ) Expression ( ";" )?

IfStmt ::= "if" Expression Block ( "else" ( IfStmt | Block ) )?

ForStmt ::= "for" ( Identifier | Identifier "," Identifier ) "in" Expression Block

ReturnStmt ::= "return" ( Expression )? ( ";" )?

AssertStmt ::= "assert" "(" Expression ( "," StringLiteral ( "," ExpressionList )? )? ")" ( ";" )?

ExpressionStmt ::= Expression ( ";" )?
```

---

### 4.4 Hardware Domain Declarations

```ebnf
MaterialDecl ::= "material" Identifier "{" ( MaterialProperty )* "}"
MaterialProperty ::= Identifier ":" Expression ( ";" )?

ProfileDecl ::= "profile" Identifier "{" ( ProfileBlock )* "}"
ProfileBlock ::= Identifier ( Identifier )? "{" ( ProfileField )* "}"
ProfileField ::= Identifier ":" Expression ( ";" )?

DeviceDecl ::= "device" Identifier "{" ( DeviceSection )* "}"
DeviceSection ::= Identifier ":" Expression ( ";" )?
                | Identifier "{" ( DeviceField )* "}"
DeviceField ::= Identifier ":" Expression ( ";" )?

ModuleDecl ::= "module" Identifier "{" ( ModuleItem )* "}"
ModuleItem ::= "pins" ":" "[" PinDeclList "]" ( ";" )?
             | RouteStatement

PinDeclList ::= PinDecl ( "," PinDecl )* ( "," )?
PinDecl ::= ( "input" | "output" | "inout" | "power" | "ground" )? Identifier

SpaceDecl ::= "space" Identifier ( "implements" Identifier )? "{" ( SpaceItem )* "}"
SpaceItem ::= "dimensions" ":" "[" Expression "," Expression "]" ( ";" )?
            | "profile" ":" Identifier ( ";" )?
            | "nets" "{" ( NetDefItem )* "}"
            | Statement
            | RouteStatement

NetDefItem ::= Identifier ":" "{" ( NetProperty ( "," NetProperty )* )? "}" ( ";" )?
NetProperty ::= Identifier ":" Expression

RouteStatement ::= "route" Expression "to" Expression ( "with" "intent" ":" Identifier | Block )?

TestDecl ::= "test" Identifier "for" Identifier "{" ( TestConfig )* "}"
TestConfig ::= Identifier ":" "{" ( TestParam ( "," TestParam )* )? "}" ( ";" )?
TestParam ::= Identifier ":" Expression
```

---

### 4.5 Expression Grammar & Operator Precedence

HardwareScript utilizes a Pratt parsing precedence hierarchy for all arithmetic, dimensional, logical, and member expressions.

| Precedence | Operators | Associativity | Description |
| :--- | :--- | :--- | :--- |
| **1 (Lowest)** | `or` | Left-to-right | Logical OR |
| **2** | `and` | Left-to-right | Logical AND |
| **3** | `==`, `!=`, `<`, `<=`, `>`, `>=` | Non-associative | Comparison & Equality |
| **4** | `..`, `..=` | Non-associative | Range Formation |
| **5** | `+`, `-` | Left-to-right | Addition, Subtraction |
| **6** | `*`, `/`, `%` | Left-to-right | Multiplication, Division, Modulo |
| **7** | `not`, `-` (unary), `+` (unary) | Right-to-left | Logical Not, Negation |
| **8 (Highest)**| `.`, `()`, `[]`, `{}` | Left-to-right | Member Access, Call, Index, Struct Init |

```ebnf
Expression ::= LogicalOrExpr

LogicalOrExpr ::= LogicalAndExpr ( "or" LogicalAndExpr )*

LogicalAndExpr ::= EqualityExpr ( "and" EqualityExpr )*

EqualityExpr ::= RelationalExpr ( ( "==" | "!=" ) RelationalExpr )*

RelationalExpr ::= RangeExpr ( ( "<" | "<=" | ">" | ">=" ) RangeExpr )*

RangeExpr ::= AdditiveExpr ( ( ".." | "..=" ) AdditiveExpr )?

AdditiveExpr ::= MultiplicativeExpr ( ( "+" | "-" ) MultiplicativeExpr )*

MultiplicativeExpr ::= UnaryExpr ( ( "*" | "/" | "%" ) UnaryExpr )*

UnaryExpr ::= ( "not" | "-" | "+" ) UnaryExpr 
            | PostfixExpr

PostfixExpr ::= PrimaryExpr ( CallSuffix 
                            | FieldAccessSuffix 
                            | IndexSuffix )*

CallSuffix ::= "(" ( ExpressionList )? ")"
FieldAccessSuffix ::= "." Identifier
IndexSuffix ::= "[" Expression "]"

PrimaryExpr ::= IntegerLiteral
              | FloatLiteral
              | MeasurementLiteral
              | StringLiteral
              | "true"
              | "false"
              | Identifier
              | ArrayLiteral
              | StructInstanceExpr
              | "(" Expression ")"

ArrayLiteral ::= "[" ( ExpressionList )? "]"
ExpressionList ::= Expression ( "," Expression )* ( "," )?

StructInstanceExpr ::= Identifier "{" ( FieldInitList )? "}"
FieldInitList ::= FieldInit ( "," FieldInit )* ( "," )?
FieldInit ::= Identifier ( ":" Expression )?
```

---

## 5. Concrete Code Examples

### 5.1 Parameterized CMOS Inverter Layout (`cmos_inverter.hw`)

```hardware
import { sky130_nmos, sky130_pmos, sky130_tap, pad, route_strap } from @std/layout/sky130
import * from @std/primitives/units

module CMOS_Inverter {
    pins: [input In, output Out, power VDD, ground VSS]
}

space CMOS_Inverter_Space implements CMOS_Inverter {
    dimensions: [20.0um, 18.0um]
    profile: SKY130_1V8_CMOS

    nets {
        VDD: { classification: power,  potential: 1.8V, current: 20.0uA }
        VSS: { classification: ground, potential: 0.0V, current: 20.0uA }
        In:  { classification: signal, potential: 1.8V, current: 0.1uA }
        Out: { classification: signal, current: 20.0uA }
    }

    // 1. Parametric Cell Instantiation (Comptime Evaluation)
    let nmos = sky130_nmos(
        name: "M_NMOS", W: 1.0um, L: 150nm, at: [10.0um, 5.0um],
        source: VSS, drain: Out, gate: In, bulk: VSS
    )

    let pmos = sky130_pmos(
        name: "M_PMOS", W: 2.0um, L: 150nm, at: [10.0um, 10.5um],
        source: VDD, drain: Out, gate: In, bulk: VDD
    )

    // 2. Substrate & Well Taps
    let sub_tap  = sky130_tap(type: TapType::P_Sub,  at: [nmos.source.x, nmos.source.y - 2.2um], net: VSS)
    let well_tap = sky130_tap(type: TapType::N_Well, at: [pmos.source.x, pmos.source.y + 2.2um], net: VDD)

    // 3. Direct Gate Interconnect & Output Drain Strap
    space.add_polygon(
        layer: "poly", 
        net: In, 
        points: rect_between(nmos.gate_head, pmos.gate_head, width: 150nm)
    )

    route_strap(
        from: nmos.drain, 
        to: pmos.drain, 
        net: Out, 
        layer: "li1", 
        bridge_layer: "metal1"
    )

    // 4. Power Delivery Routing
    route sub_tap.port to nmos.source { intent: Power }
    route well_tap.port to pmos.source { intent: Power }

    // 5. Test Bonding Pads
    let pad_vss = pad(name: "VSS_Pad", at: [nmos.source.x - 2.5um, nmos.source.y], net: VSS)
    let pad_vdd = pad(name: "VDD_Pad", at: [pmos.source.x - 2.5um, pmos.source.y], net: VDD)
    let pad_in  = pad(name: "In_Pad",  at: [pmos.gate.x, pmos.gate.y + 2.5um],     net: In)
    let pad_out = pad(name: "Out_Pad", at: [pmos.drain.x + 2.5um, (nmos.drain.y + pmos.drain.y)/2], net: Out)

    route pad_vss to nmos.source { intent: Power }
    route pad_vdd to pmos.source { intent: Power }
    route pad_in to pmos.gate_head { intent: Signal }
    route pad_out to pmos.drain { intent: Signal }
}

test CMOS_Inverter_VTC_Test for CMOS_Inverter_Space {
    dc: { sweep: In, start: 0.0V, stop: 1.8V, step: 0.02V }
    tran: { step: 5ps, stop: 10ns }
}
```

---

### 5.2 Parametric Transistor Generator (`@std/layout/sky130/nmos.hw`)

```hardware
import * from @std/primitives/units

export struct TransistorPort {
    center: Point2D,
    x: Measurement,
    y: Measurement,
    layer: String,
    net: Net,
}

export struct NMOSLayout {
    source: TransistorPort,
    drain: TransistorPort,
    gate: TransistorPort,
    gate_head: TransistorPort,
    num_vias: Int,
}

export fn sky130_nmos(
    name: String,
    W: Measurement,
    L: Measurement,
    at: Point2D,
    source: Net,
    drain: Net,
    gate: Net,
    bulk: Net,
    sd_len: Measurement = 750nm
) -> NMOSLayout {
    let diff_len = (2 * sd_len) + L
    let poly_overhang = 200nm
    let gate_head_size = 400nm
    let via_diameter = 170nm
    let via_pitch = 400nm

    // Logical check using natural English boolean keywords
    if W > 5.0um and L == 150nm {
        println("Wide driver instance '{name}' detected: synthesizing high-density via arrays...")
    }

    // 1. Bind compact model for SPICE extraction
    space.add_device(
        type: DeviceType::NMOS,
        name: name,
        terminals: { S: source, D: drain, G: gate, B: bulk },
        params: { W: W, L: L }
    )

    // 2. Active Channel & Implant Masks
    space.add_polygon(
        layer: "diff",
        net: source,
        rect: [at.x - diff_len/2, at.y - W/2, diff_len, W]
    )

    space.add_polygon(
        layer: "nsdm",
        rect: [at.x - diff_len/2 - 130nm, at.y - W/2 - 130nm, diff_len + 260nm, W + 260nm]
    )

    // 3. Polysilicon Gate & Off-Channel Safe Landing Head
    let gate_x = at.x
    space.add_polygon(
        layer: "poly",
        net: gate,
        rect: [gate_x - L/2, at.y - W/2 - poly_overhang, L, W + (2 * poly_overhang)]
    )

    let head_y = at.y + W/2 + poly_overhang
    space.add_polygon(
        layer: "poly",
        net: gate,
        rect: [gate_x - gate_head_size/2, head_y, gate_head_size, gate_head_size]
    )

    space.add_polygon(
        layer: "npc",
        rect: [gate_x - gate_head_size/2 - 90nm, head_y - 90nm, gate_head_size + 180nm, gate_head_size + 180nm]
    )

    // 4. Source & Drain Contact Arrays
    let src_x = at.x - diff_len/2 + sd_len/2
    let drn_x = at.x + diff_len/2 - sd_len/2

    space.add_polygon(layer: "li1", net: source, rect: [src_x - 200nm, at.y - W/2, 400nm, W])
    space.add_polygon(layer: "li1", net: drain,  rect: [drn_x - 200nm, at.y - W/2, 400nm, W])

    let num_vias = max(1, (W - 200nm) / via_pitch)
    let via_offset = (num_vias - 1) * via_pitch / 2

    for i in 0..num_vias {
        let vy = at.y - via_offset + (i * via_pitch)
        space.add_contact(from: "diff", to: "li1", at: [src_x, vy], diameter: via_diameter, net: source)
        space.add_contact(from: "li1", to: "metal1", at: [src_x, vy], diameter: via_diameter, net: source)
        space.add_contact(from: "diff", to: "li1", at: [drn_x, vy], diameter: via_diameter, net: drain)
    }

    return NMOSLayout {
        source:    TransistorPort { center: [src_x, at.y], x: src_x, y: at.y, layer: "metal1", net: source },
        drain:     TransistorPort { center: [drn_x, at.y], x: drn_x, y: at.y, layer: "li1",    net: drain },
        gate:      TransistorPort { center: [gate_x, at.y], x: gate_x, y: at.y, layer: "poly", net: gate },
        gate_head: TransistorPort { center: [gate_x, head_y + gate_head_size/2], x: gate_x, y: head_y + gate_head_size/2, layer: "poly", net: gate },
        num_vias:  num_vias,
    }
}
```

---

## 6. Error Recovery & Parser Synchronization Model

With curly-brace delimited scoping, the recursive descent parser in `hwc-parser` implements deterministic **Panic Mode Synchronization**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PARSER SYNCHRONIZATION ALGORITHM                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ 1. Parse Token Stream: TopLevelItem → Block → Statement                     │
│                                                                             │
│ 2. If Syntax Error Encountered on Statement S:                              │
│    ├── Emit diagnostic with exact SourceSpan (Line, Column, Snippet)        │
│    ├── Increment compile error counter                                      │
│    └── Enter synchronize_statement():                                       │
│         Skip tokens until Token::Semicolon OR next statement keyword        │
│         (`let`, `for`, `if`, `return`, `assert`, `route`)                   │
│                                                                             │
│ 3. If Block Boundary Corrupted:                                             │
│    └── Enter synchronize_block():                                           │
│         Skip tokens matching braces balance until matching Token::CloseBrace│
│                                                                             │
│ 4. Resume Clean Parsing at Next Top-Level Declaration                       │
│    └── GUARANTEE: Zero downstream cascading error storms!                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Migration & Grammar Change Summary

| Language Element | Old Paradigm (v0.1.0 – v0.2.2) | Canonical Paradigm (v0.3.0) |
| :--- | :--- | :--- |
| **Block Delimiters** | Colon + Indentation (`space SpaceName:`) | Explicit Braces (`space SpaceName { ... }`) |
| **Cascading Errors** | Severe ($1\text{ typo} \rightarrow 40\text{ errors}$) | **Zero Cascades** (Hard synchronization on `}`) |
| **Boolean Operators** | C-style `&&`, `||`, `!` or mixed keywords | **Natural Keywords** (`and`, `or`, `not`) |
| **Layout Generators** | Hardcoded compiler tokens (`matrix`, `chain_x`) | **Comptime Functions** (`fn`, loops, `@std`) |
| **Coordinates** | Relational guessing solver (`align: with:`) | **Deterministic Arithmetic** (`pad.right + 200nm`) |
| **Device Wiring** | Fake $1\text{nm}$ `Air` dummy pours | Direct compact model terminal binding |
| **Language Class** | Static Configuration DSL | **Turing-Complete Generative HDL** |

