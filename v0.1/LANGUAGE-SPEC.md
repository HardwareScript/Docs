# Hardware Script v0.1 - Language Specification

**Version**: 0.1 (MVP)  
**Status**: Implemented and Working  
**Document Type**: Syntax Reference

---

## Language Overview

Hardware Script v0.1 implements a minimal subset of the planned language specification. This document describes what actually works in the current implementation.

## Implemented Syntax

### 1. Space Definition

Define the physical workspace with dimensions and grid resolution.

**Syntax**:
```hw
define Space "SpaceName":
    dimensions: <width>mm by <height>mm by <depth>mm
    grid: <x_cells> by <y_cells> by <z_cells>
```

**Example**:
```hw
define Space "First_MVP_Board":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2
```

**Rules**:
- Dimensions must include `mm` unit
- Grid values are integers
- Voxel size calculated as: `dimension / grid_cells`

**Calculation**:
```
20mm / 20 cells = 1mm per voxel
```

### 2. Component Placement

Add components to the board at specific grid coordinates.

**Syntax**:
```hw
add <ComponentType> named <Name> at [Z, X, Y] rotated <Direction>
```

**Example**:
```hw
add Transistor_NPN named MainSwitch at [1, 5, 5] rotated North
```

**Coordinate System**:
- `[Z, X, Y]` format (layer, column, row)
- Z=1 is top layer (only layer supported in v0.1)
- X and Y are 1-indexed grid positions

**Rotation**:
- `North`, `South`, `East`, `West`
- Currently parsed but not used in rendering

**Supported Components** (v0.1):
- `Transistor_NPN` - Placeholder component type
- Component definitions not yet implemented

### 3. Routing

Define copper traces between points using explicit waypoints.

**Syntax**:
```hw
route <Source> to <Destination>:
    path:
        - [Z, X, Y]
        - [Z, X, Y]
        - [Z, X, Y]
```

**Example**:
```hw
route MainSwitch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]
        - [1, 15, 15]
```

**Behavior**:
- Compiler interpolates straight lines between consecutive waypoints
- Uses Bresenham's line algorithm for 2D paths
- All intermediate cells marked as COPPER in tensor grid

**Limitations** (v0.1):
- Source/destination names are parsed but not validated
- No connection to actual component pins
- Only same-layer routing (no vias)

### 4. Comments and Documentation

**Two types of comments**:

#### 4.1 Human Comments (`#`)
Comments start with `#` and are ignored by the compiler.

**Example**:
```hw
# This is a comment for humans
define Space "Board":  # Inline comment
    dimensions: 50mm by 50mm by 2mm
```

#### 4.2 Documentation Comments (`##`)
Documentation comments start with `##` and are extracted by `hwsd` (Hardware Script Documentation) to generate auto-documentation.

**Example**:
```hw
## A complete 5V voltage regulator module
## Input: 7-12V DC
## Output: Regulated 5V at up to 1A
## WARNING: Requires heatsink above 500mA load
define Space "5V_Regulator":
    dimensions: 20mm by 15mm by 2mm
    grid: 40 by 30 by 2
    
    ## Main voltage regulator IC
    add IC_LM7805 named Regulator at [1, 20, 15]
    
    ## Input filter capacitor (reduces ripple)
    add Capacitor (10uF) named C1 at [1, 10, 15]
```

**Documentation Generation**:
```bash
hwsd generate 5v_regulator.hw
# Outputs: HTML documentation with all ## comments formatted
```

**Why This Matters**:
- Auto-generated documentation (like Rust's rustdoc or Elixir's HexDocs)
- LLMs can read hwsd output to learn how to use libraries
- No separate docs to maintain (docs live in code)
- Community libraries are self-documenting

## Token Types

The lexer recognizes these token patterns:

| Token | Pattern | Example |
|-------|---------|---------|
| KEYWORD | `define`, `Space`, `add`, `route`, etc. | `define` |
| STRING | `"..."` | `"BoardName"` |
| COORD | `[n, n, n]` | `[1, 5, 10]` |
| MEASURE | `n` + unit | `20mm`, `5V` |
| NUMBER | Integer | `100` |
| IDENTIFIER | Name or path | `Power.Out` |
| COLON | `:` | `:` |
| LIST_ITEM | `- [n, n, n]` | `- [1, 5, 6]` |

## Complete Working Example

```hw
define Space "First_MVP_Board":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2

add Transistor_NPN named MainSwitch at [1, 5, 5] rotated North

route MainSwitch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]
        - [1, 15, 15]
```

**What this generates**:
- 20mm × 20mm board with 1mm voxel resolution
- L-shaped copper trace from (5,6) → (15,6) → (15,15)
- 20 copper voxels in the 3D tensor grid
- Trace resistance: 0.0101 Ω

## Parser Implementation

### Lexer

Regex-based tokenizer using Python's `re` module:

```python
TOKEN_TYPES = [
    ('KEYWORD',   r'\b(define|Space|dimensions|grid|add|named|at|rotated|by|route|to|path)\b'),
    ('STRING',    r'"[^"]*"'),
    ('COORD',     r'\[\d+,\s*\d+,\s*\d+\]'),
    ('MEASURE',   r'\b\d+(?:\.\d+)?(?:mm|cm|V|A)\b'),
    ('NUMBER',    r'\b\d+\b'),
    ('IDENTIFIER',r'[a-zA-Z_][a-zA-Z0-9_]*(\.[a-zA-Z0-9_]+)?'),
    ('COLON',     r':'),
    ('LIST_ITEM', r'-\s*\[\d+,\s*\d+,\s*\d+\]'),
    ('SKIP',      r'[ \t\n\r]+'),
]
```

### AST Structure

Parser produces this Abstract Syntax Tree:

```python
{
    "Space": {
        "name": "First_MVP_Board",
        "dim": (20.0, 20.0, 2.0),
        "grid": (20, 20, 2)
    },
    "Components": [
        {
            "type": "Transistor_NPN",
            "name": "MainSwitch",
            "coords": [1, 5, 5],
            "rotation": "North"
        }
    ],
    "Routes": [
        {
            "name": "MainSwitch.Collector_to_Power.Out",
            "waypoints": [[1, 5, 6], [1, 15, 6], [1, 15, 15]]
        }
    ]
}
```

## Indentation Rules

**Current Implementation**: Indentation is parsed but not strictly enforced.

The lexer skips whitespace, so these are equivalent:

```hw
# Properly indented (recommended)
define Space "Board":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2

# No indentation (also works)
define Space "Board":
dimensions: 20mm by 20mm by 2mm
grid: 20 by 20 by 2
```

**Future**: Strict indentation-based scoping will be enforced in v0.2+

## Error Handling

**Current State**: Minimal error handling in v0.1

**What happens on errors**:
- Invalid syntax: Python exception with regex mismatch
- Missing tokens: IndexError when parser expects more tokens
- Invalid coordinates: Python eval() error

**Example Error**:
```
IndexError: list index out of range
```

**Future**: Proper error messages with line numbers and suggestions.

## Not Yet Implemented

These features are in the specification but not in v0.1:

### ❌ Import System
```hw
import Copper from standard.materials  # Not working
```

### ❌ Material Definitions
```hw
define Material "CustomCopper":  # Not working
    type: Conductor
```

### ❌ Component Definitions
```hw
define Component "MyTransistor":  # Not working
    grid: 3 by 3
```

### ❌ Substrate Spanning
```hw
add Substrate(FR4) spanning all  # Not working
```

### ❌ Advanced Routing Parameters
```hw
route A to B:
    path: [...]
    trace_width: 2mm      # Not working
    clearance: 0.5mm      # Not working
```

### ❌ Multi-Layer Routing
```hw
route A to B:
    path:
        - [1, 5, 5]  # Top layer
        - [2, 5, 5]  # Via - Not working
```

## Grammar (Simplified)

```ebnf
program = space_def component_list route_list

space_def = "define" "Space" STRING ":"
            "dimensions" ":" measure "by" measure "by" measure
            "grid" ":" number "by" number "by" number

component_list = { component_def }

component_def = "add" IDENTIFIER "named" IDENTIFIER 
                "at" coordinate "rotated" direction

route_list = { route_def }

route_def = "route" identifier "to" identifier ":"
            "path" ":" waypoint_list

waypoint_list = { "-" coordinate }

coordinate = "[" number "," number "," number "]"

direction = "North" | "South" | "East" | "West"

measure = number "mm"

identifier = IDENTIFIER [ "." IDENTIFIER ]
```

## Usage

```bash
# Compile a .hw file
python hw.py generate test_board.hw

# Output
📖 Reading test_board.hw...
📚 Loading Materials Database...
⚙️  Compiling Space: First_MVP_Board
   🛤️ Routing: MainSwitch.Collector_to_Power.Out
      ✅ Physics Check: Trace Resistance = 0.0101 Ω
   ✅ Exported: build/board.gtl
   ✅ Exported: build/sim.py
   ✅ Exported: build/board.obj
🎉 COMPILATION COMPLETE!
```

## Coordinate System Details

### Grid Coordinates (Discrete)

```
[Z, X, Y] where:
  Z ∈ [1, z_layers]    (1-indexed)
  X ∈ [1, x_cells]     (1-indexed)
  Y ∈ [1, y_cells]     (1-indexed)
```

### Physical Coordinates (Continuous)

Converted during export:

```python
physical_x = (grid_x - 1) * voxel_size
physical_y = (grid_y - 1) * voxel_size
physical_z = (grid_z - 1) * layer_thickness
```

### Tensor Indexing (0-indexed)

Internal representation:

```python
tensor[z-1, x-1, y-1] = COPPER
```

## Line Interpolation Algorithm

Bresenham's algorithm connects waypoints:

```python
def draw_line(p1, p2):
    z1, x1, y1 = p1[0]-1, p1[1]-1, p1[2]-1
    z2, x2, y2 = p2[0]-1, p2[1]-1, p2[2]-1
    
    dx, dy = abs(x2 - x1), abs(y2 - y1)
    x, y = x1, y1
    sx = 1 if x1 < x2 else -1
    sy = 1 if y1 < y2 else -1
    
    if dx > dy:
        err = dx / 2.0
        while x != x2:
            points.append((z1, x, y))
            err -= dy
            if err < 0:
                y += sy
                err += dx
            x += sx
    else:
        err = dy / 2.0
        while y != y2:
            points.append((z1, x, y))
            err -= dx
            if err < 0:
                x += sx
                err += dy
            y += sy
    
    points.append((z1, x, y))
    return points
```

## Reserved Keywords (v0.1)

**Implemented**:
- `define` - Start definition block
- `Space` - Define workspace
- `dimensions` - Physical size
- `grid` - Tensor resolution
- `by` - Separator in dimensions/grid
- `add` - Place component
- `named` - Assign name
- `at` - Specify position
- `rotated` - Set orientation
- `route` - Define trace
- `to` - Route destination
- `path` - Waypoint list

**Parsed but unused**:
- `North`, `South`, `East`, `West` - Rotation directions

## File Extension

`.hw` - Hardware Script source files

---

**Document Status**: v0.1 Implementation Reference  
**Last Updated**: March 2026  
**Next Version**: v0.2 will add imports, components, and multi-layer support
