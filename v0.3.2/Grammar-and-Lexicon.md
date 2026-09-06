# HardwareScript v0.3.2: Formal Grammar & Lexical Specification

**Document Type:** Technical Language Reference Manual  
**Target Version:** v0.3.2 (Production-Locked Standard)  
**Status:** Authoritative & Canonical  
**Target Crates:** `hwc-parser`, `hwc-compiler`  
**Date:** September 2026  

---

## 1. Lexical Architecture & Scanner Specification

The `hwc-parser` lexer is implemented as a high-speed, deterministic finite-state scanner (via `logos`). Whitespace serves solely as token delimiters; significant indentation, synthetic `INDENT`/`DEDENT` tokens, and line continuations are permanently eradicated.

### 1.1 Reserved Keywords
HardwareScript v0.3.2 reserves the canonical lexical keywords for declarations, physical entities, and natural boolean logic:

```text
and         assert      bundle      const       else        enum        
export      false       fn          footprint   for         if          
impl        implements  import      in          let         match       
module      mut         nets        not         or          pour        
profile     return      route       space       struct      true
```

Contextual block identifiers include: `dimensions`, `masks`, `stackup`, `routing`, `drc`, `parasitics`, `pins`, `netlist`.

### 1.2 Comments
- Single-line comments begin with `#` and extend to the end of the line:
  ```hardware
  # Silicon wafer floorplan boundary
  ```
- Multi-line block comments use C-style `/* ... */` and can be nested arbitrarily:
  ```hardware
  /* Multi-line block comment /* Nested comment */ */
  ```

### 1.3 Identifiers
An identifier begins with an ASCII letter (`a-z`, `A-Z`) or an underscore (`_`), followed by any combination of ASCII letters, digits (`0-9`), or underscores:

$$\text{Identifier} ::= [\text{a-zA-Z\_}][\text{a-zA-Z0-9\_}]^*$$

### 1.4 Numeric & Physical Measurement Literals
Physical measurements are first-class lexical tokens consisting of an integer or floating-point literal directly followed by a case-sensitive unit suffix without whitespace:

$$\text{MeasurementLiteral} ::= (\text{IntegerLiteral} \mid \text{FloatLiteral}) \times \text{UnitSuffix}$$

#### Regular Expressions & SI Exponents Table

| Category | Suffixes | Lexer Regex | SI Exponent Vector $[L, M, T, I, \Theta, N, J]$ |
| :--- | :--- | :--- | :--- |
| **Length** | `pm`, `nm`, `um`, `mm`, `m`, `mil` | `(pm\|nm\|um\|mm\|m\|mil)` | $[+1, 0, 0, 0, 0, 0, 0]$ |
| **Area** | `pm2`, `nm2`, `um2`, `mm2`, `m2` | `(pm2\|nm2\|um2\|mm2\|m2)` | $[+2, 0, 0, 0, 0, 0, 0]$ |
| **Time** | `fs`, `ps`, `ns`, `us`, `ms`, `s` | `(fs\|ps\|ns\|us\|ms\|s)` | $[0, 0, +1, 0, 0, 0, 0]$ |
| **Voltage** | `nV`, `uV`, `mV`, `V`, `kV` | `(nV\|uV\|mV\|V\|kV)` | $[+2, +1, -3, -1, 0, 0, 0]$ |
| **Current** | `pA`, `nA`, `uA`, `mA`, `A` | `(pA\|nA\|uA\|mA\|A)` | $[0, 0, 0, +1, 0, 0, 0]$ |
| **Resistance** | `uOhm`, `mOhm`, `Ohm`, `kOhm`, `MOhm` | `(uOhm\|mOhm\|Ohm\|kOhm\|MOhm)` | $[+2, +1, -3, -2, 0, 0, 0]$ |
| **Sheet Res** | `Ohm_sq`, `mOhm_sq` | `(Ohm_sq\|mOhm_sq)` | $[0, +1, -3, -2, 0, 0, 0]$ |
| **Capacitance** | `aF`, `fF`, `pF`, `nF`, `uF` | `(aF\|fF\|pF\|nF\|uF)` | $[-2, -1, +4, +2, 0, 0, 0]$ |
| **Cap Density** | `fF_um2`, `aF_um2` | `(fF_um2\|aF_um2)` | $[-4, -1, +4, +2, 0, 0, 0]$ |
| **Inductance** | `pH`, `nH`, `uH`, `mH`, `H` | `(pH\|nH\|uH\|mH\|H)` | $[+2, +1, -2, -2, 0, 0, 0]$ |

### 1.5 String Literals & String Interpolation
Double-quoted strings support compile-time expression evaluation using curly braces:
```hardware
"Channel {name} offset: {at.x + 200nm}"
```

---

## 2. Pratt Operator Precedence Hierarchy

The expression grammar is implemented via Pratt recursive descent parsing using 8 distinct precedence tiers:

| Tier | Precedence | Operators | Associativity | Parsing Rule |
| :---: | :--- | :--- | :---: | :--- |
| **1** | Logical OR | `or`, `\|\|` | Left | Lowest precedence |
| **2** | Logical AND | `and`, `&&` | Left | Logical conjunction |
| **3** | Equality / Relational | `==`, `!=`, `<`, `<=`, `>`, `>=` | None | Non-associative comparisons |
| **4** | Range Formation | `..`, `..=` | None | Half-open and closed ranges |
| **5** | Additive | `+`, `-` | Left | Addition and subtraction |
| **6** | Multiplicative | `*`, `/`, `%` | Left | Multiplication, division, modulo |
| **7** | Unary Prefix | `not`, `!`, `-`, `+` | Right | Logical NOT, arithmetic negation |
| **8** | Postfix / Member | `.`, `()`, `[]` | Left | Field access, calls, indexing |

---

## 3. Formal EBNF Grammar Specification

```ebnf
(* ===========================================================================
   TOP-LEVEL PROGRAM & DECLARATIONS
   =========================================================================== *)

Program ::= ( TopLevelItem )* EOF

TopLevelItem ::= ImportDecl
               | ExportDecl
               | StructDecl
               | ImplBlock
               | EnumDecl
               | BundleDecl
               | FunctionDecl
               | ModuleDecl
               | ProfileDecl
               | FootprintDecl
               | SpaceDecl

ImportDecl ::= "import" ( "*" | ImportSymbolList ) "from" ( StringLiteral | Identifier )

ImportSymbolList ::= "{" Identifier ( "," Identifier )* ( "," )? "}"
                   | Identifier

ExportDecl ::= "export" ( StructDecl | EnumDecl | BundleDecl | FunctionDecl | ProfileDecl | FootprintDecl | SpaceDecl )

(* ===========================================================================
   STRUCTS, ENUMS & METHODS
   =========================================================================== *)

StructDecl ::= "struct" Identifier "{" ( StructField ( "," StructField )* ( "," )? )? "}"
StructField ::= Identifier ":" TypeExpr

EnumDecl ::= "enum" Identifier "{" ( EnumVariant ( "," EnumVariant )* ( "," )? )? "}"
EnumVariant ::= Identifier ( "(" TypeExprList ")" )?

ImplBlock ::= "impl" Identifier "{" ( FunctionDecl )* "}"

BundleDecl ::= "bundle" Identifier "{" ( BundleField ( "," BundleField )* ( "," )? )? "}"
BundleField ::= Identifier ":" ( "signal" | "power" | "ground" | TypeExpr )

FunctionDecl ::= "fn" Identifier "(" ( ParameterList )? ")" ( "->" TypeExpr )? Block

ParameterList ::= Parameter ( "," Parameter )* ( "," )?
Parameter ::= Identifier ":" TypeExpr ( "=" Expression )?

TypeExpr ::= Identifier ( "[" TypeExprList "]" )?
           | "(" TypeExprList ")"
           | "fn" "(" TypeExprList ")" ( "->" TypeExpr )?

TypeExprList ::= TypeExpr ( "," TypeExpr )* ( "," )?

(* ===========================================================================
   SCHEMATIC MODULES & NETLISTS
   =========================================================================== *)

ModuleDecl ::= "module" Identifier "{" ( ModuleItem )* "}"

ModuleItem ::= "pins" ":" "[" ( PinDecl ( "," PinDecl )* )? "]" ( ";" )?
             | "netlist" "{" ( NetlistEntry )* "}"

PinDecl ::= ( "input" | "output" | "inout" | "power" | "ground" )? Identifier

NetlistEntry ::= Identifier ":" Identifier "(" ( NetlistArg ( "," NetlistArg )* )? ")" ( ";" )?
NetlistArg ::= Identifier ":" Expression

(* ===========================================================================
   HARDWARE PROFILES, FOOTPRINTS & SPACES
   =========================================================================== *)

ProfileDecl ::= "profile" Identifier "implements" Identifier "{" ( ProfileSection )* "}"

ProfileSection ::= Identifier ( Identifier )? "{" ( ProfileField )* "}"
                 | Identifier ":" Expression ( ";" )?

ProfileField ::= ( Identifier )+ ":" Expression ( ";" )?
               | Identifier ":" "{" ( ProfileField )* "}" ( ";" )?

FootprintDecl ::= "footprint" Identifier "for" Identifier "{" ( FootprintSection )* "}"

FootprintSection ::= "body_size" ":" "[" Expression "," Expression "]" ( ";" )?
                   | "pitch" ":" Expression ( ";" )?
                   | "parasitics" "{" ( ParasiticField )* "}"
                   | "pins" "{" ( FootprintPinEntry )* "}"

ParasiticField ::= Identifier ":" Expression ( ";" )?

FootprintPinEntry ::= "pin" IntegerLiteral ":" Expression "{" ( PinProperty ( "," PinProperty )* )? "}" ( ";" )?
PinProperty ::= Identifier ":" Expression

SpaceDecl ::= "space" Identifier ( "implements" Identifier )? "{" ( SpaceItem )* "}"

SpaceItem ::= "dimensions" ":" "[" Expression "," Expression "]" ( ";" )?
            | "profile" ":" Identifier ( ";" )?
            | "nets" "{" ( NetDeclaration )* "}"
            | PourStatement
            | RouteStatement
            | Statement

NetDeclaration ::= Identifier ":" "{" ( NetProperty ( "," NetProperty )* )? "}" ( ";" )?
NetProperty ::= Identifier ":" Expression

PourStatement ::= "pour" Identifier "{" ( PourProperty ( "," PourProperty )* )? "}" ( ";" )?
PourProperty ::= Identifier ":" Expression

RouteStatement ::= "route" Expression "to" Expression ( "{" ( RouteProperty )* "}" )?
RouteProperty ::= Identifier ":" Expression ( "," | ";" )?

(* ===========================================================================
   STATEMENTS & CONTROL FLOW
   =========================================================================== *)

Block ::= "{" ( Statement )* "}"

Statement ::= LetStmt
            | AssignmentStmt
            | IfStmt
            | ForStmt
            | ReturnStmt
            | AssertStmt
            | ExpressionStmt

LetStmt ::= "let" ( "mut" )? BindingPattern ( ":" TypeExpr )? "=" Expression ( ";" )?
BindingPattern ::= Identifier | "(" Identifier ( "," Identifier )* ")"

AssignmentStmt ::= PostfixExpr ( "=" | "+=" | "-=" | "*=" | "/=" ) Expression ( ";" )?

IfStmt ::= "if" Expression Block ( "else" ( IfStmt | Block ) )?

ForStmt ::= "for" Identifier ( "," Identifier )? "in" Expression Block

ReturnStmt ::= "return" ( Expression )? ( ";" )?

AssertStmt ::= "assert" "(" Expression ( "," StringLiteral )? ")" ( ";" )?

ExpressionStmt ::= Expression ( ";" )?

(* ===========================================================================
   EXPRESSIONS
   =========================================================================== *)

Expression ::= LogicalOrExpr

LogicalOrExpr ::= LogicalAndExpr ( ( "or" | "||" ) LogicalAndExpr )*

LogicalAndExpr ::= EqualityExpr ( ( "and" | "&&" ) EqualityExpr )*

EqualityExpr ::= RelationalExpr ( ( "==" | "!=" ) RelationalExpr )*

RelationalExpr ::= RangeExpr ( ( "<" | "<=" | ">" | ">=" ) RangeExpr )*

RangeExpr ::= AdditiveExpr ( ( ".." | "..=" ) AdditiveExpr )?

AdditiveExpr ::= MultiplicativeExpr ( ( "+" | "-" ) MultiplicativeExpr )*

MultiplicativeExpr ::= UnaryExpr ( ( "*" | "/" | "%" ) UnaryExpr )*

UnaryExpr ::= ( "not" | "!" | "-" | "+" ) UnaryExpr
            | PostfixExpr

PostfixExpr ::= PrimaryExpr ( CallSuffix 
                            | MethodCallSuffix 
                            | FieldAccessSuffix 
                            | IndexSuffix )*

CallSuffix ::= "(" ( ArgumentList )? ")"
MethodCallSuffix ::= "." Identifier "(" ( ArgumentList )? ")"
FieldAccessSuffix ::= "." Identifier
IndexSuffix ::= "[" Expression "]"

ArgumentList ::= Argument ( "," Argument )* ( "," )?
Argument ::= ( Identifier ":" )? Expression

PrimaryExpr ::= IntegerLiteral
              | FloatLiteral
              | MeasurementLiteral
              | StringLiteral
              | "true"
              | "false"
              | Identifier
              | ArrayLiteral
              | StructInitExpr
              | IfExpr
              | "(" Expression ")"

ArrayLiteral ::= "[" ( Expression ( "," Expression )* ( "," )? )? "]"

StructInitExpr ::= Identifier "{" ( FieldInit ( "," FieldInit )* ( "," )? )? "}"
FieldInit ::= Identifier ":" Expression

IfExpr ::= "if" Expression Block "else" Block
```
