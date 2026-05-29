# Hardware Script v0.1 - MVP Release

**Status**: ✅ Working MVP  
**Release Date**: March 2026  
**Paradigm**: Text-Based Hardware Design Language

---

## What We Built

Hardware Script v0.1 is a working proof-of-concept compiler that transforms plain-text hardware descriptions into multiple output formats:

- **Gerber files** for PCB manufacturing
- **Blender Python scripts** for 3D visualization
- **OBJ 3D models** for universal viewing

## Core Achievement

We proved that hardware can be described in human-readable text and compiled deterministically into physical manufacturing files using a discrete 3D tensor grid system.

## What Works in v0.1

### ✅ Implemented Features

1. **Space Definition** - Define physical board dimensions and grid resolution
2. **Component Placement** - Add components with rotation support
3. **Waypoint Routing** - Explicit copper trace routing with automatic line interpolation
4. **Physics Validation** - Calculate trace resistance using materials database
5. **Multi-Format Export** - Generate Gerber, Blender, and OBJ files
6. **Materials Database** - YAML-based material properties (conductors, insulators, semiconductors)

### 📋 Language Features

```hw
define Space "BoardName":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2

add Transistor_NPN named MainSwitch at [1, 5, 5] rotated North

route MainSwitch.Collector to Power.Out:
    path:
        - [1, 5, 6]
        - [1, 15, 6]
        - [1, 15, 15]
```

### 🔧 Compiler Pipeline

1. **Lexer** - Tokenizes .hw source code
2. **Parser** - Builds Abstract Syntax Tree (AST)
3. **Grid Engine** - Populates 3D numpy tensor with component/trace data
4. **Physics Engine** - Validates electrical properties
5. **Export Engine** - Generates output files

### 📦 Output Files

- `build/board.gtl` - Gerber top layer (industry standard)
- `build/sim.py` - Blender Python script (executable)
- `build/board.obj` - Universal 3D model (viewable anywhere)

## Quick Start

```bash
# Compile a .hw file
python hw.py generate test_board.hw

# View outputs
ls build/
# board.gtl  board.obj  sim.py

# Open in Blender
blender --python build/sim.py
```

## Verified Test Case

**Input**: `test_board.hw` (20mm × 20mm board with L-shaped copper trace)

**Output**: 
- ✅ Gerber file with 20 copper coordinates
- ✅ Blender script with FR4 substrate + 20 copper voxels
- ✅ OBJ model with 168 vertices (verified in online viewer and Blender)
- ✅ Physics calculation: 0.0101 Ω trace resistance

## Architecture Highlights

### Discrete 3D Tensor Grid

The core innovation: treating physical space as a discrete matrix instead of continuous geometry.

```python
# Board represented as 3D numpy array
tensor = np.ones((z_layers, x_cells, y_cells), dtype=np.int8)

# Cell states
EMPTY = 0    # Bare substrate
COPPER = 2   # Routed trace
```

**Benefits**:
- O(1) collision detection
- Deterministic output
- AI-native coordinate system
- Direct mapping to voxel rendering

### Bresenham Line Interpolation

Waypoints are automatically connected using Bresenham's algorithm:

```python
route MainSwitch.Collector to Power.Out:
    path:
        - [1, 5, 6]    # Start
        - [1, 15, 6]   # Waypoint
        - [1, 15, 15]  # End
# Compiler fills in all intermediate cells
```

### Materials Database

YAML-based physical properties from Materials Project API:

```yaml
conductors:
  copper:
    resistivity_ohm_m: 1.68e-08
    max_current_density_a_mm2: 35
    melting_point_c: 1085
    thermal_conductivity_w_mk: 401
```

## Known Limitations (v0.1)

### Not Yet Implemented

- ❌ Multi-layer routing (only Z=1 supported)
- ❌ Via generation (layer changes)
- ❌ Component library (only placeholder components)
- ❌ Import system (no standard library yet)
- ❌ Substrate spanning syntax
- ❌ Advanced routing parameters (trace width, clearance)
- ❌ Electrical validation (voltage/current checks)
- ❌ BOM generation
- ❌ Drill file export

### Parser Limitations

- Simple regex-based lexer (no error recovery)
- Limited syntax validation
- No semantic analysis
- Hardcoded component types

### Export Limitations

- Gerber: Only top layer, no drill files
- Blender: Individual voxels (not optimized meshes)
- OBJ: No material properties or colors

## File Structure

```
Hardware-Script/
├── Docs/
│   ├── v0.1/
│   │   ├── README.md           (this file)
│   │   ├── LANGUAGE-SPEC.md    (syntax reference)
│   │   └── ARCHITECTURE.md     (implementation details)
├── live-test/
│   ├── hw.py                   (compiler implementation)
│   ├── test_board.hw           (example source)
│   ├── standard-materials.yaml (materials database)
│   └── build/                  (generated outputs)
└── engine-test/                (development phases)
```

## Technical Stack

- **Language**: Python 3.x
- **Dependencies**: 
  - `numpy` - 3D tensor operations
  - `pyyaml` - Materials database parsing
  - `re` - Lexer pattern matching

## What's Next (v0.2 Roadmap)

1. **Multi-layer support** - Full Z-axis routing with vias
2. **Component library** - Standard parts (resistors, capacitors, ICs)
3. **Import system** - Modular component definitions
4. **Advanced routing** - Trace width, clearance, impedance control
5. **Complete Gerber export** - All layers, drill files, silkscreen
6. **Electrical validation** - Voltage/current/power checks
7. **BOM generation** - Component lists for manufacturing

## Success Metrics

✅ **Proof of Concept**: Text-to-hardware compilation works  
✅ **Multi-Format Export**: Single source → 3 output formats  
✅ **Physics Integration**: Real material properties in calculations  
✅ **3D Visualization**: Viewable in multiple tools  
✅ **Industry Standards**: Valid Gerber output format  

## Documentation

- [Language Specification](LANGUAGE-SPEC.md) - Complete syntax reference
- [Architecture Guide](ARCHITECTURE.md) - Implementation details
- [Materials Database](../../live-test/standard-materials.yaml) - Physical properties

---

**Hardware Script v0.1** - Proving that hardware design can be as simple as writing text.
