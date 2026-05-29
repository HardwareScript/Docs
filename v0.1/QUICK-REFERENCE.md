# Hardware Script v0.1 - Quick Reference

**Version**: 0.1 (MVP)  
**One-page syntax guide**

---

## Basic Syntax

### Space Definition

```hw
define Space "BoardName":
    dimensions: <X>mm by <Y>mm by <Z>mm
    grid: <cols> by <rows> by <layers>
```

**Example**:
```hw
define Space "MyBoard":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2
```

### Component Placement

```hw
add <Type> named <Name> at [Z, X, Y] rotated <Direction>
```

**Example**:
```hw
add Transistor_NPN named Switch at [1, 5, 5] rotated North
```

### Routing

```hw
route <Source> to <Destination>:
    path:
        - [Z, X, Y]
        - [Z, X, Y]
        - [Z, X, Y]
```

**Example**:
```hw
route Switch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]
        - [1, 15, 15]
```

### Comments

```hw
# This is a comment
define Space "Board":  # Inline comment
```

---

## Coordinate System

### Format: [Z, X, Y]

- **Z** = Layer (1 = top, 2 = bottom)
- **X** = Column (left to right)
- **Y** = Row (bottom to top)

### Example

```
[1, 5, 10] means:
  Layer 1 (top layer)
  Column 5 (5 voxels from left)
  Row 10 (10 voxels from bottom)
```

### Grid Visualization

```
Y ↑
20│ ┌─────────────┐
  │ │             │
10│ │   [1,10,10] │
  │ │             │
 1│ └─────────────┘
  └─────────────────→ X
   1              20
```

---

## Keywords

### Actions
- `define` - Create definition
- `add` - Place component
- `route` - Create trace

### Structure
- `Space` - Workspace type
- `named` - Assign name
- `at` - Position
- `to` - Destination
- `rotated` - Orientation
- `path` - Waypoint list

### Properties
- `dimensions` - Physical size
- `grid` - Resolution
- `by` - Separator

### Directions
- `North`, `South`, `East`, `West`

---

## Units

- **Distance**: `mm` (millimeters)
- **Electrical**: `V` (volts), `A` (amps)
- **Temperature**: `C` (celsius)

---

## Common Patterns

### Straight Line

```hw
route A to B:
    path:
        - [1, 5, 5]
        - [1, 15, 5]
```

### L-Shape

```hw
route A to B:
    path:
        - [1, 5, 5]
        - [1, 15, 5]
        - [1, 15, 15]
```

### U-Shape

```hw
route A to B:
    path:
        - [1, 5, 5]
        - [1, 5, 15]
        - [1, 15, 15]
        - [1, 15, 5]
```

### Square

```hw
route A to A:
    path:
        - [1, 5, 5]
        - [1, 15, 5]
        - [1, 15, 15]
        - [1, 5, 15]
        - [1, 5, 5]
```

---

## CLI Commands

### Compile

```bash
python hw.py generate <filename.hw>
```

### Output Location

```bash
build/
├── board.gtl    # Gerber file
├── sim.py       # Blender script
└── board.obj    # 3D model
```

---

## Voxel Calculation

```
Voxel Size = Dimension / Grid Cells

Example:
  20mm / 20 cells = 1mm per voxel
  50mm / 100 cells = 0.5mm per voxel
```

---

## Complete Example

```hw
# Define workspace
define Space "First_Board":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2

# Place component
add Transistor_NPN named Switch at [1, 5, 5] rotated North

# Route traces
route Switch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]
        - [1, 15, 15]

route Switch.Emitter to Ground.In:
    path:
        - [1, 5, 4]
        - [1, 10, 4]
```

---

## Troubleshooting

### Error: "list index out of range"
**Cause**: Syntax error  
**Fix**: Check colons, coordinates, keywords

### Error: "No module named 'numpy'"
**Fix**: `pip install numpy`

### Error: "FileNotFoundError: standard-materials.yaml"
**Fix**: Run from `live-test` directory

### No output files
**Fix**: Check for error messages, verify syntax

---

## Tips

1. **Start simple** - Begin with straight lines
2. **Use comments** - Document your design
3. **Check physics** - Resistance should be < 0.1 Ω
4. **Visualize first** - View OBJ before manufacturing
5. **Keep grid simple** - Use even divisions

---

## Limitations (v0.1)

- ❌ Single layer only (Z=1)
- ❌ No component library
- ❌ No imports
- ❌ No collision detection
- ❌ No electrical validation
- ❌ No BOM generation

---

## Next Steps

1. Read [GETTING-STARTED.md](GETTING-STARTED.md)
2. Try the examples
3. Experiment with coordinates
4. View outputs in 3D

---

**Full Documentation**: [INDEX.md](INDEX.md)
