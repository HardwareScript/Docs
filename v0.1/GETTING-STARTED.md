# Getting Started with Hardware Script v0.1

**Version**: 0.1 (MVP)  
**Difficulty**: Beginner  
**Time**: 10 minutes

---

## Prerequisites

### Required Software

- **Python 3.x** - [Download](https://www.python.org/downloads/)
- **Text editor** - Any editor (VS Code, Notepad++, etc.)

### Required Python Packages

```bash
pip install numpy pyyaml
```

### Optional Software

- **Blender** - For viewing simulation output ([Download](https://www.blender.org/))
- **3D viewer** - Any OBJ viewer (online or desktop)

## Installation

### 1. Clone or Download

```bash
# If using Git
git clone <repository-url>
cd Hardware-Script/live-test

# Or download and extract ZIP
# Navigate to live-test folder
```

### 2. Verify Installation

```bash
python hw.py
```

**Expected output**:
```
Usage: python hw.py generate <filename.hw>
```

### 3. Check Dependencies

```bash
python -c "import numpy, yaml; print('Dependencies OK')"
```

## Your First Hardware Design

### Step 1: Create a .hw File

Create a file named `my_board.hw`:

```hw
define Space "MyFirstBoard":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2

add Transistor_NPN named Switch at [1, 5, 5] rotated North

route Switch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]
        - [1, 15, 15]
```

### Step 2: Compile

```bash
python hw.py generate my_board.hw
```

**Expected output**:
```
==================================================
🔥 HARDWARE SCRIPT COMPILER (v0.1 MVP)
==================================================

📖 Reading my_board.hw...
📚 Loading Materials Database...
⚙️  Compiling Space: MyFirstBoard
   🛤️ Routing: Switch.Collector_to_Power.Out
      ✅ Physics Check: Trace Resistance = 0.0101 Ω
   ✅ Exported: build/board.gtl
   ✅ Exported: build/sim.py
   ✅ Exported: build/board.obj
🎉 COMPILATION COMPLETE!
```

### Step 3: View Outputs

```bash
ls build/
```

**You should see**:
- `board.gtl` - Gerber file for PCB manufacturing
- `sim.py` - Blender Python script
- `board.obj` - 3D model

## Viewing Your Design

### Option 1: Online 3D Viewer

1. Go to [https://3dviewer.net/](https://3dviewer.net/)
2. Upload `build/board.obj`
3. Rotate and zoom to inspect

### Option 2: Blender

```bash
blender --python build/sim.py
```

**What you'll see**:
- Gray FR4 substrate (20mm × 20mm)
- Copper traces on top (L-shaped path)

### Option 3: Text Editor

Open `build/board.gtl` to see Gerber coordinates:

```
%FSLAX26Y26*%
%MOMM*%
%ADD10C,1.0000*%
D10*
X040000Y050000D01*
X050000Y050000D01*
...
```

## Understanding the Code

### Space Definition

```hw
define Space "MyFirstBoard":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2
```

**What this does**:
- Creates a 20mm × 20mm board
- 2mm thick (standard PCB)
- Divides into 20×20×2 grid
- Each voxel = 1mm × 1mm × 1mm

### Component Placement

```hw
add Transistor_NPN named Switch at [1, 5, 5] rotated North
```

**What this does**:
- Places a transistor component
- Names it "Switch" (for routing)
- Position: Layer 1, Column 5, Row 5
- Rotates to face North

**Note**: Components are placeholders in v0.1

### Routing

```hw
route Switch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]
        - [1, 15, 15]
```

**What this does**:
- Creates copper trace from Switch to Power
- Starts at [1, 5, 6]
- Goes to [1, 15, 6] (horizontal)
- Then to [1, 15, 15] (vertical)
- Compiler fills in all cells between waypoints

## Coordinate System

### Format: [Z, X, Y]

- **Z** = Layer (1 = top, 2 = bottom)
- **X** = Column (left to right)
- **Y** = Row (bottom to top)

### Example

```
[1, 5, 6] means:
  Layer 1 (top)
  Column 5 (5mm from left)
  Row 6 (6mm from bottom)
```

### Grid Visualization

```
Y (rows)
↑
20 ┌─────────────────────┐
   │                     │
   │                     │
10 │         [1,10,10]   │
   │                     │
 1 └─────────────────────┘ → X (columns)
   1                    20
```

## Common Patterns

### Straight Line

```hw
route A to B:
    path:
        - [1, 5, 5]
        - [1, 15, 5]  # Horizontal line
```

### L-Shape

```hw
route A to B:
    path:
        - [1, 5, 5]
        - [1, 15, 5]   # Horizontal
        - [1, 15, 15]  # Vertical
```

### U-Shape

```hw
route A to B:
    path:
        - [1, 5, 5]
        - [1, 5, 15]   # Up
        - [1, 15, 15]  # Across
        - [1, 15, 5]   # Down
```

## Modifying the Example

### Change Board Size

```hw
define Space "BiggerBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 50 by 50 by 2
```

**Result**: 50mm × 50mm board with 1mm voxels

### Change Grid Resolution

```hw
define Space "HighRes":
    dimensions: 20mm by 20mm by 2mm
    grid: 40 by 40 by 2
```

**Result**: 20mm × 20mm board with 0.5mm voxels

### Add More Routes

```hw
route Switch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]

route Switch.Emitter to Ground.In:
    path:
        - [1, 5, 4]
        - [1, 10, 4]
```

**Result**: Two separate copper traces

## Troubleshooting

### Error: "No module named 'numpy'"

**Solution**:
```bash
pip install numpy
```

### Error: "No module named 'yaml'"

**Solution**:
```bash
pip install pyyaml
```

### Error: "FileNotFoundError: standard-materials.yaml"

**Solution**: Make sure you're in the `live-test` directory:
```bash
cd live-test
python hw.py generate my_board.hw
```

### Error: "list index out of range"

**Cause**: Syntax error in .hw file

**Solution**: Check for:
- Missing colons after keywords
- Incorrect coordinate format
- Typos in keywords

### No Output Files

**Check**:
```bash
ls build/
```

If `build/` doesn't exist, the compiler failed. Check error messages.

## Next Steps

### Learn More

- Read [LANGUAGE-SPEC.md](LANGUAGE-SPEC.md) for complete syntax
- Read [ARCHITECTURE.md](ARCHITECTURE.md) for implementation details
- Read [ACHIEVEMENTS.md](ACHIEVEMENTS.md) for what's possible

### Experiment

Try these challenges:

1. **Create a square trace** - Route that forms a closed loop
2. **Make a grid pattern** - Multiple parallel traces
3. **Change voxel size** - Experiment with grid resolution
4. **Measure resistance** - Note how trace length affects resistance

### Example Challenges

**Challenge 1: Square Trace**
```hw
route A to A:
    path:
        - [1, 5, 5]
        - [1, 15, 5]
        - [1, 15, 15]
        - [1, 5, 15]
        - [1, 5, 5]
```

**Challenge 2: Diagonal Approximation**
```hw
route A to B:
    path:
        - [1, 5, 5]
        - [1, 6, 6]
        - [1, 7, 7]
        - [1, 8, 8]
```

## Tips

### 1. Start Simple

Begin with basic straight lines before complex routing.

### 2. Use Comments

```hw
# Power distribution
route Battery.Plus to LED.Anode:
    path:
        - [1, 5, 5]   # Start at battery
        - [1, 15, 5]  # Go to LED
```

### 3. Check Physics Output

The resistance calculation tells you if your trace is reasonable:
- < 0.1 Ω: Good for most applications
- > 1 Ω: May cause voltage drop issues

### 4. Visualize First

Always view the OBJ output to verify your design before manufacturing.

### 5. Keep Grid Simple

Use grid sizes that divide evenly into dimensions:
- 20mm / 20 = 1mm ✅
- 20mm / 17 = 1.176mm ❌ (harder to work with)

## File Organization

```
my-project/
├── my_board.hw           # Your design
├── hw.py                 # Compiler (copy from live-test)
├── standard-materials.yaml  # Materials DB (copy from live-test)
└── build/                # Generated outputs
    ├── board.gtl
    ├── sim.py
    └── board.obj
```

## Getting Help

### Check Documentation

- [README.md](README.md) - Overview
- [LANGUAGE-SPEC.md](LANGUAGE-SPEC.md) - Syntax reference
- [ARCHITECTURE.md](ARCHITECTURE.md) - How it works

### Common Questions

**Q: Can I use this for real PCB manufacturing?**  
A: v0.1 generates valid Gerber files, but they're minimal. For production, you'll need drill files, silkscreen, and solder mask (coming in v0.2+).

**Q: Why is my trace resistance so high?**  
A: Long traces have higher resistance. Try shorter paths or wider traces (not yet supported in v0.1).

**Q: Can I import components?**  
A: Not in v0.1. Component library coming in v0.2.

**Q: How do I make multi-layer boards?**  
A: Not yet supported. Coming in v0.2.

---

**You're ready to start designing hardware with text!**

Try the example, experiment with coordinates, and see what you can create.
