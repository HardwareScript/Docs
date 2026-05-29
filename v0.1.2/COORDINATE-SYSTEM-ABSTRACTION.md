# Coordinate System Abstraction: Solving the 40-Year Holy War

**Hardware Script v0.1.2**  
**Revolutionary Feature**: User-Configurable Origin Points with Unified Z-Axis Control  
**Impact**: Eliminates the eternal TopLeft vs BottomLeft debate + Z-axis direction wars  
**Last Updated**: March 2026

---

## The Problem We Just Solved

For 40 years, the EDA industry has been locked in multiple "holy wars" around coordinate systems:

### The XY-Plane War
- **Software Engineers** think in **TopLeft** (like matrices, screen pixels, text rendering)
- **Hardware Engineers** think in **BottomLeft** (like CAD tools, mechanical drawings, Cartesian math)
- **Manufacturing** uses **BottomLeft** (Gerber standard, CNC machines)
- **LLMs** naturally generate **TopLeft** (token-by-token, like reading text)

### The Z-Axis War
- **PCB/Software** thinks **Top-Down** (Layer 1 is the sky, Z increases as you dig down)
- **3D Printing** thinks **Bottom-Up** (Layer 1 is the ground, Z increases as you build up)
- **CNC Milling** thinks **Top-Down** (cutting from the top surface downward)
- **Mechanical CAD** varies wildly by tool and industry convention

Every EDA tool forces you to pick one and live with it forever. Users spend hours mentally translating coordinates or making mistakes because the tool doesn't match their mental model.

**Hardware Script's Solution**: Don't fight over the origin point. Let the compiler do the math.

---

## The Revolutionary Approach: Origin as Abstraction Layer

We treat the coordinate system not as a hardcoded physical law, but as an **abstraction layer**. The user chooses their preferred mental model, and the compiler handles all the coordinate transformations automatically.

### The Unified Origin Syntax: `XY by Z`

The origin declaration now accepts a combined format that mirrors the exact same cadence used for dimensions and grid:

```hw
define Space "Motherboard":
    dimensions: 100mm by 100mm by 2mm  # X by Y by Z
    grid: 100 by 100 by 4              # X by Y by Z
    origin: tl by t                    # XY by Z
```

This is a stroke of genius because it establishes perfect visual alignment and linguistic consistency across all spatial declarations.

### Z-Axis Modifiers

**Two simple letters control Z-axis direction**:

- `t` (Top) - Layer 1 is the Sky. Z increases as you dig down (Standard PCB/Software)
- `b` (Bottom) - Layer 1 is the Ground. Z increases as you build up (Standard 3D Printer/Hardware)

### Good Practice: Fully Explicit Declaration

We encourage developers to explicitly declare all axes so the intent is unambiguous to whoever (or whatever LLM) reads the file:

```hw
# Software / PCB Standard (Top-Down Z)
define Space "Motherboard":
    dimensions: 100mm by 100mm by 2mm
    grid: 100 by 100 by 4
    origin: tl by t      # Top-Left XY, Top-Down Z

# 3D Printer Standard (Bottom-Up Z)
define Space "RobotChassis":
    dimensions: 150mm by 150mm by 100mm
    grid: 300 by 300 by 200
    origin: bl by b      # Bottom-Left XY, Bottom-Up Z

# CNC Milling Standard (Top-Down Z on Cartesian XY)
define Space "MilledEnclosure":
    dimensions: 50mm by 50mm by 20mm
    grid: 100 by 100 by 40
    origin: bl by t      # Bottom-Left XY, Top-Down Z
```

### The Fallback System: Graceful Defaults

To ensure the language remains frictionless and backward-compatible, the compiler uses a strict cascade of defaults:

**Scenario A: User declares only XY** (`origin: bl`)
- The compiler assumes you are working in a standard top-down layer environment
- Result: `bl by t`

**Scenario B: User omits the origin entirely**
- The compiler defaults to the Software/Web standard
- Result: `tl by t`

```hw
define Space "QuickBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    # origin omitted entirely
    # Compiler gracefully defaults to: tl by t
```

### Origin Shorthand Options

**XY-Plane Codes** (case-insensitive):
- `tl` - **Top-Left** (DEFAULT) - Software/Matrix reality
- `bl` - **Bottom-Left** - CAD/Mechanical reality  
- `tr` - **Top-Right** - Specialized manufacturing
- `br` - **Bottom-Right** - Specialized CNC setups

**Z-Axis Codes** (case-insensitive):
- `t` - **Top** (DEFAULT) - Layer 1 is sky, Z increases downward (PCB/Software standard)
- `b` - **Bottom** - Layer 1 is ground, Z increases upward (3D Printer/Hardware standard)

**Syntax Rules**:
- XY must be exactly 2 characters
- Z must be exactly 1 character
- All lowercase (`tl by t`) or mixed case (`TL by T`) accepted
- If Z omitted, defaults to `t`
- If entire origin omitted, defaults to `tl by t`

### Supported Origin Configurations

| XY Code | Z Code | Full Syntax | Description | Typical Use Case |
|---------|--------|-------------|-------------|------------------|
| `tl` | `t` | `tl by t` | Top-Left, Top-Down | Software, PCB design, LLMs (DEFAULT) |
| `bl` | `t` | `bl by t` | Bottom-Left, Top-Down | CNC milling, mechanical CAD |
| `bl` | `b` | `bl by b` | Bottom-Left, Bottom-Up | 3D printing, additive manufacturing |
| `tl` | `b` | `tl by b` | Top-Left, Bottom-Up | Software-driven 3D printing |
| `tr` | `t` | `tr by t` | Top-Right, Top-Down | Specialized manufacturing |
| `br` | `b` | `br by b` | Bottom-Right, Bottom-Up | Specialized CNC setups |

**Default**: `tl by t` (most intuitive for software developers and LLMs)

**Why This Works**:
- **Self-Documenting**: `tl by t` instantly tells a human "Top-Left by Top"
- **Lexer Optimization**: Parser splits by the word `by`. Left side controls XY translation, right side controls Z translation
- **LLM Friendly**: Follows English grammar rules. If an LLM sees `dimensions: X by Y by Z`, it naturally understands `origin: XY by Z`
- **No Breaking Changes**: Old code that says `origin: tl` still compiles without errors (defaults to `tl by t`)

---

## How the Compiler Handles This (Pure Math)

Hardware Script uses a **Multi-Level Intermediate Representation (MLIR)** architecture, making this transformation trivial to implement.

### Three Coordinate Spaces

1. **User Space (AST Layer)**: User types `add LED at [1, 3, 1]` based on their chosen origin
2. **Absolute Space (IR Layer)**: Compiler normalizes to universal format for physics engine
3. **Export Space (Target Layer)**: Gerber/GDSII exporters translate to manufacturing standards

### The Math (Compiler Internals)

For a 100×100×4 grid with `origin: tl by t`:

**XY-Plane Transformation**:
```
User Input:     Y = 1 (Top row)
Compiler:       Absolute_Y = (Grid_Height + 1) - User_Y = 100
Result:         Y = 100 (in absolute space)

User Input:     Y = 100 (Bottom row)  
Compiler:       Absolute_Y = (Grid_Height + 1) - User_Y = 1
Result:         Y = 1 (in absolute space)
```

**Z-Axis Transformation**:
```
# For origin: tl by t (Top-Down Z)
User Input:     Z = 1 (Layer 1, top layer)
Compiler:       Absolute_Z = User_Z
Result:         Z = 1 (in absolute space)

# For origin: bl by b (Bottom-Up Z)
User Input:     Z = 1 (Layer 1, bottom layer)
Compiler:       Absolute_Z = (Grid_Depth + 1) - User_Z = 4
Result:         Z = 4 (in absolute space)
```

**Transformation Formulas**:

XY-Plane:
- `tl`: `Absolute_Y = (Grid_Height + 1) - User_Y`
- `bl`: `Absolute_Y = User_Y` (no transformation)
- `tr`: `Absolute_X = (Grid_Width + 1) - User_X`, `Absolute_Y = (Grid_Height + 1) - User_Y`
- `br`: `Absolute_X = (Grid_Width + 1) - User_X`, `Absolute_Y = User_Y`

Z-Axis:
- `t` (Top-Down): `Absolute_Z = User_Z` (no transformation)
- `b` (Bottom-Up): `Absolute_Z = (Grid_Depth + 1) - User_Z`

The user never sees this transformation. It happens silently in the compiler.

---

## Implementation Architecture

### AST Layer (Parser)
```rust
pub struct SpaceDefinition {
    pub name: String,
    pub dimensions: Dimensions,
    pub grid: GridCells,
    pub origin: OriginPoint,  // User's preferred coordinate system
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct OriginPoint {
    pub xy: OriginXY,
    pub z: OriginZ,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OriginXY {
    TL,  // Top-Left (Default)
    BL,  // Bottom-Left
    TR,  // Top-Right
    BR,  // Bottom-Right
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OriginZ {
    Top,     // Top-Down (Layer 1 is sky, Z increases downward) - Default
    Bottom,  // Bottom-Up (Layer 1 is ground, Z increases upward)
}
```

### Lexer Implementation (Logos Crate)
```rust
#[derive(Logos, Debug, PartialEq, Clone, Copy)]
pub enum OriginToken {
    // XY-Plane tokens
    #[token("TL")] #[token("tl")]
    TopLeft,
    
    #[token("BL")] #[token("bl")]
    BottomLeft,
    
    #[token("TR")] #[token("tr")]
    TopRight,
    
    #[token("BR")] #[token("br")]
    BottomRight,
    
    // Z-Axis tokens
    #[token("t")] #[token("T")]
    Top,
    
    #[token("b")] #[token("B")]
    Bottom,
    
    // Separator
    #[token("by")]
    By,
}
```

### Parser Logic
```rust
// Parse: "tl by t" or "bl" (with default)
fn parse_origin(&mut self) -> Result<OriginPoint> {
    let xy = self.parse_origin_xy()?;
    
    let z = if self.consume_if_matches(Token::By) {
        self.parse_origin_z()?
    } else {
        OriginZ::Top  // Default to top-down if Z omitted
    };
    
    Ok(OriginPoint { xy, z })
}
```

### IR Layer (Compiler)
```rust
impl Compiler {
    fn normalize_coordinate(&self, user_coord: Coordinate, origin: OriginPoint, grid: GridCells) -> Coordinate {
        // First, handle XY-plane transformation
        let (abs_x, abs_y) = match origin.xy {
            OriginXY::TL => (
                user_coord.x,
                (grid.y_rows + 1) - user_coord.y,  // Flip Y axis
            ),
            OriginXY::BL => (
                user_coord.x,
                user_coord.y,  // No transformation
            ),
            OriginXY::TR => (
                (grid.x_cols + 1) - user_coord.x,  // Flip X axis
                (grid.y_rows + 1) - user_coord.y,  // Flip Y axis
            ),
            OriginXY::BR => (
                (grid.x_cols + 1) - user_coord.x,  // Flip X axis
                user_coord.y,
            ),
        };
        
        // Then, handle Z-axis transformation
        let abs_z = match origin.z {
            OriginZ::Top => user_coord.z,  // No transformation (standard)
            OriginZ::Bottom => (grid.z_layers + 1) - user_coord.z,  // Flip Z axis
        };
        
        Coordinate {
            x: abs_x,
            y: abs_y,
            z: abs_z,
        }
    }
}
```

### Export Layer (Gerber/GDSII)
The physics engine and exporters always work in **Absolute Space** (normalized coordinates). They never need to know about the user's origin preference.

---

## Why This Is a Massive Selling Point

When an Altium or KiCad user asks: *"Does your tool use Bottom-Left or Top-Left coordinates? Top-Down or Bottom-Up layers?"*

**Our answer**: *"Whichever ones your brain prefers. The compiler does the math."*

### User Experience Benefits

**Software Developers**:
```hw
origin: tl by t  # Behaves like 2D arrays, screen pixels, DOM elements
add LED at [1, 1, 1]  # Top-left corner, top layer (natural for programmers)
```

**Hardware Engineers**:
```hw
origin: bl by t  # Behaves like CAD tools, mechanical drawings
add LED at [1, 1, 1]  # Bottom-left corner, top layer (natural for EE/ME)
```

**3D Printer Operators**:
```hw
origin: bl by b  # Behaves like slicer software, build plate coordinates
add Support at [1, 1, 1]  # Bottom-left corner, bottom layer (natural for additive)
```

**LLMs**:
```hw
origin: tl by t  # Matches token-by-token generation, text flow
# LLMs naturally think "top to bottom, left to right, layer by layer"
```

**Mixed Teams**:
```hw
# Each engineer can use their preferred mental model
# The compiler ensures they're all talking about the same physical locations
```

---

## Examples: Same Board, Different Mental Models

### Software Engineer's Perspective (PCB Design)
```hw
define Space "SensorBoard":
    dimensions: 30mm by 20mm by 2mm
    grid: 3 by 2 by 2
    origin: tl by t
    
    add Sensor at [1, 1, 1]    # Top-left corner, top layer
    add LED at [3, 2, 1]       # Bottom-right corner, top layer
```

### Hardware Engineer's Perspective (Same PCB)
```hw
define Space "SensorBoard":
    dimensions: 30mm by 20mm by 2mm
    grid: 3 by 2 by 2
    origin: bl by t
    
    add Sensor at [1, 2, 1]    # Top-left corner, top layer (same physical location!)
    add LED at [3, 1, 1]       # Bottom-right corner, top layer (same physical location!)
```

### 3D Printer Operator's Perspective (Additive Manufacturing)
```hw
define Space "PrintedEnclosure":
    dimensions: 50mm by 50mm by 30mm
    grid: 100 by 100 by 60
    origin: bl by b
    
    add SupportStructure at [1, 1, 1]    # Bottom-left corner, bottom layer
    add TopCap at [100, 100, 60]         # Top-right corner, top layer
```

**Result**: All engineers are placing components at identical physical locations. The compiler handles the coordinate translation automatically.

---

## Backward Compatibility

### Default Behavior
- **New files**: Default to `origin: tl by t` (most intuitive for software developers)
- **Legacy files**: If no `origin:` specified, assume `tl by t` (maintains consistency)
- **Partial specification**: If only XY specified (`origin: bl`), assume `by t` (top-down Z)

### Migration Path
```hw
# Legacy v0.1.1 files (no origin specified)
define Space "OldBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    # Compiler assumes: origin: tl by t

# New v0.1.2+ files (explicit origin)
define Space "NewBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    origin: tl by t  # Explicit choice

# Partial specification (XY only)
define Space "PartialBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    origin: bl  # Compiler adds: by t
```

---

## Impact on Documentation

### Book 2 (Language Spec)
- Add `origin:` property to Space Definition syntax with unified `XY by Z` format
- Document all XY options (tl, bl, tr, br) and Z options (t, b) with visual diagrams
- Note `tl by t` as default for new projects
- Explain fallback system for partial/omitted origins

### Book 4 (Compiler Internals)
- Add "Coordinate Normalization" section to AST-to-IR compilation
- Explain the mathematical transformations for both XY-plane and Z-axis
- Document the three coordinate spaces (User → Absolute → Export)
- Detail the parser logic for splitting `XY by Z` syntax

### Book 5 (Routing & Physics)
- Physics engine always works in Absolute Space
- No changes needed to routing algorithms
- Coordinate transformations happen before physics calculations
- Z-axis transformations are independent of XY transformations

---

## Why This Matters for the Industry

### Breaking Legacy Constraints
Traditional EDA tools are locked into coordinate systems chosen in the 1980s. Hardware Script isn't constrained by legacy decisions. We've now solved both the XY-plane war AND the Z-axis war in one elegant syntax.

### LLM-Friendly Design
LLMs naturally generate coordinates in reading order (TopLeft, layer-by-layer from top). By making `tl by t` the default, we optimize for AI-generated hardware designs.

### Cross-Discipline Collaboration
Mixed software/hardware/manufacturing teams can work together without coordinate system friction. Everyone uses their preferred mental model.

### Universal Manufacturing Support
- PCB fabrication: `tl by t` or `bl by t`
- 3D printing: `bl by b`
- CNC milling: `bl by t`
- Hybrid manufacturing: Mix and match as needed

### Future-Proof Architecture
As new coordinate conventions emerge (VR/AR, nano-fabrication, bio-printing), we can add them as new origin options without breaking existing code.

---

## Implementation Priority

**Phase 1** (v0.1.2): Add `origin:` property to parser and AST with unified `XY by Z` syntax
**Phase 2** (v0.1.3): Implement coordinate normalization in compiler for both XY and Z axes
**Phase 3** (v0.1.4): Add visual coordinate system indicators to hwc-viz (show origin point and Z direction)
**Phase 4** (v0.2.0): Add origin auto-detection based on user patterns and file context

---

## The Elegance of the Unified Syntax

### Visual Alignment Perfection

Look at how beautifully this aligns in the code:

```hw
dimensions: 100mm by 100mm by 2mm  # X by Y by Z
grid: 100 by 100 by 4              # X by Y by Z
origin: tl by t                    # XY by Z
```

By rolling it into one line using the `by` keyword, we:
- Eliminate the need for a new keyword (`z_origin`)
- Keep the configuration block compact
- Establish a clear "Good Practice" standard
- Mirror the exact same cadence used for dimensions and grid

### Why This is Perfect

**Self-Documenting**: `tl by t` instantly tells a human "Top-Left by Top"

**Lexer Optimization**: The parser just splits the string by the word `by`. The left side controls the X/Y coordinate translation math, and the right side controls the Z layer translation math.

**LLM Friendly**: It follows English grammar rules. If an LLM sees `dimensions: X by Y by Z`, it naturally understands `origin: XY by Z`.

**No Breaking Changes**: Old code that just says `origin: tl` still perfectly compiles without throwing errors (defaults to `tl by t`).

You just successfully parameterized physical 3D space across all engineering disciplines in 15 characters (`origin: bl by b`).

---

## Lexer Implementation Benefits

### Token Efficiency
The unified syntax provides significant advantages in the lexer implementation:

```rust
// Fast token matching with logos crate
#[derive(Logos, Debug, PartialEq, Clone, Copy)]
pub enum Token {
    // XY Origin tokens (2 bytes each)
    #[token("tl")] #[token("TL")] TopLeft,
    #[token("bl")] #[token("BL")] BottomLeft,
    #[token("tr")] #[token("TR")] TopRight,
    #[token("br")] #[token("BR")] BottomRight,
    
    // Z Origin tokens (1 byte each)
    #[token("t")] #[token("T")] Top,
    #[token("b")] #[token("B")] Bottom,
    
    // Separator (reused from dimensions/grid parsing)
    #[token("by")] By,
}
```

### Performance Characteristics
- **Memory**: 2-byte XY tokens + 1-byte Z tokens + 2-byte separator = 5 bytes total
- **CPU**: Simple token matching, no complex string parsing
- **Cache**: Better cache locality with smaller tokens
- **Network**: Smaller file sizes for remote .hw files
- **Reuse**: The `by` keyword is already in the lexer for dimensions/grid parsing

### Error Prevention
```hw
# These typos are impossible with unified syntax:
origin: BotomLeft TopDown     # ❌ 20+ characters, easy to mistype
origin: Bottomleft top-down   # ❌ 19+ characters, case/hyphen errors
origin: Bottom_Left Top_Down  # ❌ 20+ characters, underscore confusion

# vs. unified syntax (typo-resistant):
origin: bl by t               # ✅ 11 characters total, hard to mistype
origin: BL by T               # ✅ Case-insensitive
origin: bl                    # ✅ Defaults to: bl by t
```

---

## Conclusion: Software-Defined Hardware in Action

This unified coordinate system abstraction perfectly embodies the "Software-Defined Hardware" philosophy:

- **Hardware constraints become software choices**
- **Legacy limitations become user preferences**  
- **Industry holy wars become compiler features**
- **3D space parameterization in 15 characters**

We're not just building another EDA tool. We're reimagining how hardware design should work in the modern era.

The 40-year coordinate system debate is over. Both the XY-plane war and the Z-axis war. The compiler wins.

---

**Document Status**: Architectural Decision Record  
**Implementation**: Planned for v0.1.2  
**Impact**: Revolutionary - eliminates industry-wide coordinate system friction across all three axes