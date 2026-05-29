# Hardware Script (.hw) Architecture Specification

**Version**: 0.2 (Draft)  
**Document Type**: System Architecture and Implementation Guide  
**Core Innovation**: The Universal Spatial Engine with Discrete 3D Tensor Grid

---

## 1. System Architecture Overview

### 1.1 The Universal Spatial Engine

Hardware Script replaces traditional GUI-first EDA (Electronic Design Automation) tools with a plain-English, indentation-based text format. It explicitly rejects the necessity of CPUs/OS loops (e.g., Raspberry Pi) for basic logic, instead allowing a **Synthesizer** to transpile textual logic into discrete physical components (transistors/logic gates), 3D simulation scripts, and manufacturing blueprints.

**Core Innovation**: Hardware Script rejects unpredictable "black-box" auto-routing and continuous 3D space. Instead, it treats physical reality as a **Discrete 3D Tensor Grid** (Matrix of Voxels). Because the system is purely mathematical, it is inherently provable, O(1) memory-efficient, and natively fluent for Large Language Models (LLMs).

### 1.2 Fractal Hardware Philosophy

Hardware is fractal: a microchip, a motherboard, and a data center are all just bounded physical volumes. Hardware Script unifies them under a single concept—the **Space**.

By replacing the traditional "PCB Board" with a universally scalable "Space" and a deterministic "3D Tensor Grid," Hardware Script scales from:
- Nanometer silicon wafer design
- Millimeter PCB circuits
- Centimeter embedded systems
- Meter-wide mechanical chassis
- Industrial infrastructure

All using identical syntax.

### 1.3 Project Structure

A standard project contains two core elements:

- **hw.config.yaml**: The build target configuration
- **\*.hw files**: The declarative hardware source code

---

## 2. The Synthesizer CLI

The system does not "compile" an executable (.exe). It is a **Synthesizer** and **Linter** that checks the laws of physics and generates targeted output files.

### 2.1 `hw verify`

**Purpose**: Reads .hw code and runs the Physics & Logic Linter.

**Action**: Validates electrical rules (Voltage/Amperage limits) and Design Rules (3D spatial collisions, out-of-bounds placement).

**Output**: Generates no files. Outputs success/failure directly to the terminal.

**Examples**:
```
❌ Error: 3D Collision at (x:10, y:10)
❌ Error: Voltage Mismatch at Line 15
✅ Verification Successful: 0 Errors
```

### 2.2 `hw generate`

**Purpose**: Executes the instructions mapped out in hw.config.yaml.

**Action**: Translates the validated .hw text into the specified formats (e.g., Python scripts for Blender, or Gerber/BOM files for factories).

**Output**: Files written to paths specified in configuration.

---

## 3. Configuration (hw.config.yaml)

Because .hw files are purely descriptive, the .yaml file tells the Synthesizer what the user actually wants to produce.

### 3.1 Configuration Schema

```yaml
name: "ProjectName"
version: "1.0.0"

targets:
  - type: simulation
    engine: blender
    export_path: "./build/sim.py"
  
  - type: manufacturing
    format: gerber_X2
    layers: 4
    export_path: "./build/factory/"
```

### 3.2 Target Types

**simulation**
- `engine`: 3D simulation platform (blender, unity, custom)
- `export_path`: Output location for simulation scripts

**manufacturing**
- `format`: Output format (gerber_X2, gerber_RS274X, kicad)
- `layers`: Number of PCB layers
- `export_path`: Output directory for manufacturing files

---

## 4. The Discrete 3D Tensor Grid System

### 4.1 Core Philosophy: Rejecting Black-Box Auto-Routing

Traditional EDA software relies on unpredictable, "black-box" auto-routing algorithms operating in continuous 3D space. Hardware Script rejects this approach.

Instead, .hw treats the physical circuit board as a **Discrete 3D Tensor Grid** (a matrix of voxels). This makes physical hardware generation:
- **Mathematically provable**
- **Completely deterministic**
- **Highly optimized for LLM (AI) generation**
- **Memory efficient with O(1) collision detection**

### 4.2 The Voxel Grid System

#### Grid Definition

When a Space is defined, the user declares its absolute physical dimensions and its grid resolution:

- **Dimensions**: The real-world size (e.g., 50mm by 50mm by 2mm)
- **Grid**: The mathematical subdivision of the board (e.g., 50 by 50 by 2)
- **Smallest Unit (The Voxel)**: Calculated by dividing dimensions by grid

Example: A 50mm board with a 50×50 grid means 1 Grid Cell = 1mm

All physical components and copper traces perfectly snap to these grid cells.

#### Mathematical Calculation

```
Voxel Width = Board Width / Grid Columns
Voxel Height = Board Height / Grid Rows
Voxel Depth = Board Depth / Grid Layers

Example:
  Dimensions: 50mm by 50mm by 2mm
  Grid: 50 by 50 by 2
  
  Voxel = 1mm × 1mm × 1mm
```

### 4.3 Memory Representation

The Synthesizer represents the board as a 3D tensor in memory:

```
Space[Z][X][Y] where:
  Z ∈ [1, layers]
  X ∈ [1, grid_columns]
  Y ∈ [1, grid_rows]
```

#### Cell State Enumeration

Each cell can hold exactly one state:

```
enum CellState {
    EMPTY = 0,    // Bare substrate (FR4)
    COPPER = 1,   // Routed trace
    PAD = 2,      // Component pin/pad
    HOLE = 3      // Drilled via
}
```

### 4.4 Explicit Waypoint Routing & Line Generation

Users must explicitly declare waypoints for all copper traces. Hardware Script uses **Waypoint Routing** (Vector Interpolation) between the declared vertices.

If the code states `[1, 1, 1] -> [1, 10, 1]`, the Synthesizer automatically generates and claims `[1,2,1]`, `[1,3,1]` ... `[1,9,1]` as copper cells.

#### Interpolation Algorithm

```
For each consecutive waypoint pair (P1, P2):
    If P1.Z == P2.Z:
        // Same layer - draw trace
        Interpolate line from P1 to P2
        Mark all cells along path as COPPER
    Else:
        // Layer change - drill via
        If P1.X == P2.X AND P1.Y == P2.Y:
            Mark cell at [P1.Z, P1.X, P1.Y] as HOLE
            Mark cell at [P2.Z, P2.X, P2.Y] as HOLE
        Else:
            Throw Error: "Via requires same X,Y coordinates"
```

### 4.5 Collision Detection (O(1) Complexity)

Before marking a cell as COPPER, the Synthesizer checks:

```
If Space[Z][X][Y] != EMPTY:
    Throw Error: "Collision at [Z, X, Y]"
```

This provides **instant collision detection** without complex 3D geometry calculations.

**Example Error**:
```
❌ Fatal Error at Line 23: Route Collision
  Attempted to route through [1, 10, 10]
  Cell already occupied by: Component "Sensor" (PAD)
  Current route: Power.out → Motor.in
```

### 4.6 Multi-Layer PCB Architecture

#### 4.6.1 The Physical "Sandwich" Structure

Multi-layer PCBs are constructed as a sandwich of alternating substrate and copper layers:

- **Outer Layers** (Z=1, Z=max): Physical component placement + copper routing
- **Inner Layers** (Z=2, Z=3, etc.): Pure copper routing, power planes, ground planes

#### 4.6.2 Layer Usage Rules

**Top Layer (Z=1) & Bottom Layer (Z=max)**:
- ✅ **Permitted**: Components, pins, copper traces, vias
- **Purpose**: Component mounting and primary signal routing

**Inner Layers (Z=2, Z=3, etc.)**:
- ✅ **Permitted**: Copper traces, copper planes, vias
- ❌ **Prohibited**: Physical components (transistors, resistors, ICs)
- **Purpose**: Signal crossing, power distribution, ground planes

#### 4.6.3 Common Inner Layer Assignments

```hw
define Space "ComplexBoard":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 4
    
    # Layer 1: Component layer (top)
    add Substrate(FR4) spanning all
    
    # Layer 2: Ground plane (solid copper sheet)
    add CopperPlane(GND) at layer 2 spanning all
    
    # Layer 3: Power plane (5V distribution)
    add CopperPlane(5V) at layer 3 spanning all
    
    # Layer 4: Component layer (bottom)
```

#### 4.6.4 Engine A Layer Validation

Engine A enforces strict Z-axis rules based on physical manufacturing constraints:

**Error Example**:
```hw
add Transistor named T1 at [2, 50, 50]  # Inner layer placement
```

**Engine A Response**:
```
❌ Fatal Physical Error at Line 15: Invalid Component Placement
  Cannot place physical component inside inner substrate layer Z=2
  Components only permitted on outer layers: Z=1, Z=4
  Suggestion: Move component to [1, 50, 50] or [4, 50, 50]
```

### 4.7 Via Generation Rules

A Via is automatically generated when:

1. **Z coordinate changes** between consecutive waypoints
2. **X and Y coordinates remain identical**

```hw
# Valid Via
- [1, 10, 5]  # Top layer
- [3, 10, 5]  # Bottom layer (Via drilled at X=10, Y=5)

# Invalid Via
- [1, 10, 5]  # Top layer
- [3, 12, 7]  # Bottom layer (X and Y changed - not a vertical via)
```

**Error for Invalid Via**:
```
❌ Error: Invalid Via at waypoints [1,10,5] → [3,12,7]
  Via requires identical X,Y coordinates
  Suggestion: Add intermediate waypoint [1,12,7] before layer change
```

### 4.8 Trace Width Calculation

The Synthesizer calculates minimum trace width based on current requirements:

```
Width (mm) = (Current (A) × 0.048) / (Temperature_Rise (°C) × Thickness (oz))^0.44

For 1oz copper at 10°C rise:
  1A requires ~0.25mm
  2A requires ~0.45mm
  5A requires ~1.0mm
```

If a waypoint path is only 1 cell wide but requires 2mm trace width, the Synthesizer automatically expands to adjacent cells or throws an error if space is unavailable.

---

## 5. The Compilation Pipeline

The Hardware Script compiler implements a three-stage pipeline that transforms declarative text into manufacturable hardware:

### 5.1 Engine A: Logic and Physics Validation

The first compilation stage performs comprehensive electrical and physical validation:

- **Electrical Rule Checking**: Validates voltage compatibility, current capacity, and power distribution
- **Thermal Analysis**: Ensures components operate within thermal limits
- **Physical Constraint Verification**: Validates trace widths, component spacing, and layer stack-up

When violations are detected, the compiler generates structured error messages:

```
Error at Line 12: Voltage Mismatch
  PowerSource (5V) connected to Sensor (max 3.3V)
  Suggestion: Add voltage regulator or use 3.3V power source
```

This text-based error format enables LLMs to iteratively refine designs in an automated feedback loop until all constraints are satisfied.

### 5.2 Engine B: 3D Simulation and Visualization (Blender Export)

Physical hardware requires spatial reasoning and visual debugging. The Synthesizer generates fully executable Python scripts for 3D engines (e.g., Blender) that dynamically construct the physical hardware model.

#### 5.2.1 Translation Process

The Synthesizer translates the internal 3D tensor matrix into Blender Python API calls:

1. **Substrate Generation**: Reads Space dimensions and generates a base mesh (e.g., FR4 fiberglass or Silicon wafer)
2. **Copper Trace Optimization**: Iterates through all cells marked as COPPER, mathematically merges adjacent cells into continuous polygon meshes for performance
3. **Component Placement**: Locates PAD cells, retrieves the associated component's `render_mesh` identifier, and places the corresponding .obj or .blend file at the exact real-world coordinate
4. **Via Visualization**: Renders HOLE cells as cylindrical vias connecting layers

#### 5.2.2 Example Output (build/sim.py)

```python
import bpy

# Synthesized from Space: Dimensions 50x50x2mm
bpy.ops.mesh.primitive_cube_add(size=(50, 50, 2), location=(25, 25, 1))

# Synthesized from route: Power.out to Light.in
create_copper_trace(waypoints=[(5, 5, 2), (50, 5, 2), (50, 49, 2)])

# Synthesized from Component Instantiation
import_and_place_mesh(
    name="TO92_Transistor",
    filepath="~/.hwm/standard/switches/to92.obj",
    location=(50, 50, 2)
)
```

When executed, Blender opens and the exact physical hardware materializes on screen instantly. Engineers can manipulate environmental conditions (e.g., light levels, temperature) and observe system behavior before physical prototyping.

### 5.3 Engine C: Manufacturing Output Generation

The final compilation stage produces industry-standard manufacturing files. The Synthesizer supports multiple output formats depending on the target manufacturing process.

#### 5.3.1 PCB Manufacturing (Gerber X2)

For printed circuit boards, the Synthesizer generates:

- **Gerber Files**: PCB copper layer definitions (.gtl, .gbl, .g2, .g3)
- **Excellon Drill Files**: Via and hole specifications (.xln)
- **Pick-and-Place Files**: Component placement coordinates (.cpl)
- **Bill of Materials (BOM)**: Component specifications and quantities (.bom.csv)

**Translation Process**:

1. **Copper Layers**: Scans Z=1 (top layer) and Z=3 (bottom layer), converts all COPPER cells to Gerber "Draw" commands
2. **Drill File**: Locates all HOLE cells, multiplies grid coordinates by voxel size to get real-world positions, outputs Excellon CNC commands
3. **BOM Generation**: Counts instantiated components from the abstract syntax tree, groups identical components automatically

**Example Output (build/factory/top_layer.gtl)**:
```
%FSLAX26Y26*%  (Format Statement)
%MOMM*%        (Units: Millimeters)
%ADD10C,0.5*%  (Define a circular aperture/trace width of 0.5mm)
D10*           (Select aperture 10)
X050000Y050000D02* (Move to Start Waypoint without drawing)
X500000Y050000D01* (Draw line to Corner Waypoint)
X500000Y490000D01* (Draw line to End Waypoint)
M02*           (End of File)
```

The Synthesizer automatically converts grid coordinates `[1, 5, 5] -> [1, 50, 5]` into CNC G-Code by multiplying by voxel size.

#### 5.3.2 Silicon Manufacturing (GDSII)

For integrated circuits, the Synthesizer generates GDSII files (Graphic Data System) used by silicon foundries like TSMC, Intel, and Samsung.

**Translation Process**:

1. **Silicon Layer (Z=1)**: Locates all silicon logic gates, calculates 2D bounding boxes, exports as GDSII polygon structures representing N-well and P-well masks
2. **Metal Layers (Z=2+)**: Converts all COPPER cells to GDSII polygons for "Metal Layer 1", "Metal Layer 2", etc.
3. **Via Masks**: Instead of drill files, generates GDSII "Contact/Via Mask" layers so the foundry knows where to etch holes with acid or lasers

**Configuration Example (hw.config.yaml)**:
```yaml
name: "CustomSilicon"
version: "1.0.0"

targets:
  - type: manufacturing
    format: gds_II          # Outputs Silicon Foundry Stencils
    process_node: "130nm"   # TSMC manufacturing rules
    export_path: "./build/foundry/"
```

These outputs are immediately compatible with PCB fabrication services, assembly houses, and silicon foundries, enabling direct transition from design to manufacturing.

---

## 6. Error Handling and Validation

### 6.1 Electrical Validation Errors

```
Error at Line X: Voltage Mismatch
  Source: PowerSource (5V)
  Destination: Sensor (max 3.3V)
  Suggestion: Add voltage regulator
```

### 6.2 Physical Validation Errors

```
Error at Line Y: 3D Collision Detected
  Component A: Battery at [1, 10, 10]
  Component B: Capacitor at [1, 11, 10]
  Overlap: 1 voxel
  Suggestion: Adjust position or rotation
```

### 6.3 Logic Validation Errors

```
Error at Line Z: Undefined Component Reference
  Rule references 'Sensor.temperature'
  Component 'Sensor' not found in scope
```

### 6.4 Routing Validation Errors

```
Error at Line W: Invalid Routing Path
  Path segment [1, 5, 5] → [1, 20, 5]
  Crosses occupied cells at [1, 12, 5]
  Suggestion: Add waypoint to route around obstacle
```

---

## 7. Build Output Formats

### 7.1 Simulation Output (Blender Python)

Generated Python script includes:
- 3D mesh geometry for all components
- Material properties and textures
- Physics simulation for electrical flow
- Interactive controls for environmental variables

**Example Output Structure**:
```python
import bpy

# Create board substrate
bpy.ops.mesh.primitive_cube_add(size=50, location=(0,0,0))

# Add components with physics
def add_battery(position):
    # Component mesh and electrical properties
    pass

# Simulate electrical flow
def simulate_circuit():
    # Real-time voltage/current visualization
    pass
```

### 7.2 Manufacturing Output (Gerber)


Generated files include:
- `.gtl` - Top copper layer
- `.gbl` - Bottom copper layer
- `.g2` - Inner layer 2 (if multi-layer)
- `.g3` - Inner layer 3 (if multi-layer)
- `.gto` - Top silkscreen
- `.gbs` - Bottom silkscreen
- `.gko` - Board outline
- `.xln` - Drill file (Excellon format)
- `.bom` - Bill of Materials (CSV)
- `.cpl` - Component Placement List (CSV)

**Example BOM Output**:
```csv
Reference,Value,Footprint,Quantity,Manufacturer
U1,ATmega328,DIP-28,1,Microchip
R1,10K,0805,1,Generic
C1,100nF,0805,1,Generic
```

---

## 8. Benefits of Deterministic Grid System

1. **Predictable Output**: Same .hw file always generates identical board
2. **AI-Native**: Tensor coordinates are natural for LLM reasoning
3. **Fast Validation**: O(1) collision detection vs O(n²) geometry checks
4. **Version Control Friendly**: Small coordinate changes = small diffs
5. **Simulation Ready**: Direct mapping to 3D voxel rendering
6. **Manufacturing Precision**: Grid snapping eliminates floating-point errors
7. **Provable Correctness**: Purely mathematical system with deterministic behavior

---

## 9. Component Scaling Architecture

### 9.1 Components as Nested Local Spaces

Components are defined with their own local grid system that dynamically scales to the parent Space's voxel resolution.

**Component Definition** (Local Space):
```hw
define Component "Transistor_NPN":
    grid: 3 by 3               # Local 3x3 matrix
    
    body:
        spans: [0, 0] to [2, 2]  # Claims all 9 cells
    
    pins:
        Collector at [0, 1]
        Base      at [1, 0]
        Emitter   at [2, 1]
```

**Component Instantiation** (Global Space):
```hw
define Space "PCB":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    # Voxel = 0.5mm
    
    add Transistor_NPN named T1 at [1, 10, 10]
```

### 9.2 Dynamic Scaling Algorithm

When a component is added to a Space:

1. Calculate parent Space voxel size: `50mm / 100 = 0.5mm`
2. Component local grid is `3 by 3`
3. Component physical footprint: `3 × 0.5mm = 1.5mm × 1.5mm`
4. Map local pins to global coordinates:
   - Local `[0, 1]` → Global `[1, 10, 11]`
   - Local `[1, 0]` → Global `[1, 11, 10]`
   - Local `[2, 1]` → Global `[1, 12, 11]`

This allows the same component definition to work at any scale.

---

## 10. Future Architecture Extensions

### 10.1 Planned Features (v0.3)

- **Parametric Components**: Variables and expressions in component definitions
- **Loop Constructs**: Array placement with iteration
- **Conditional Compilation**: Platform-specific directives
- **Analog Signal Processing**: Continuous signal rules
- **Thermal Simulation**: Real-time heat dissipation modeling

### 10.2 Under Consideration

- **Multi-file Projects**: Import system for splitting large designs
- **Real-time Collaborative Editing**: Conflict resolution for team development
- **Hardware Testing Framework**: Automated validation and test generation
- **Cost Optimization Engine**: Suggest component alternatives based on price/availability
- **Supply Chain Integration**: Direct API connections to distributors

---

## 11. Synthesizer Implementation Notes

### 11.1 Core Data Structures

```python
class Space:
    dimensions: Tuple[float, float, float]  # Physical size in mm
    grid: Tuple[int, int, int]              # Tensor subdivision
    voxel_size: Tuple[float, float, float]  # Calculated resolution
    cells: np.ndarray                        # 3D tensor of CellState
    components: List[Component]              # Instantiated components
    routes: List[Route]                      # Electrical connections

class Component:
    name: str
    local_grid: Tuple[int, int]
    body: List[Tuple[int, int]]
    pins: Dict[str, Tuple[int, int]]
    global_position: Tuple[int, int, int]

class Route:
    source: str                              # "Component.pin"
    destination: str                         # "Component.pin"
    waypoints: List[Tuple[int, int, int]]
    trace_width: float
    max_current: float
```

### 11.2 Routing Algorithm Pseudocode

```python
def interpolate_route(waypoints):
    for i in range(len(waypoints) - 1):
        p1, p2 = waypoints[i], waypoints[i+1]
        
        if p1[0] == p2[0]:  # Same layer
            # Bresenham's line algorithm for 2D path
            cells = bresenham_line(p1[1:], p2[1:])
            for cell in cells:
                if space.cells[p1[0]][cell[0]][cell[1]] != EMPTY:
                    raise CollisionError(p1[0], cell[0], cell[1])
                space.cells[p1[0]][cell[0]][cell[1]] = COPPER
        else:  # Layer change
            if p1[1] == p2[1] and p1[2] == p2[2]:
                # Valid via
                space.cells[p1[0]][p1[1]][p1[2]] = HOLE
                space.cells[p2[0]][p2[1]][p2[2]] = HOLE
            else:
                raise InvalidViaError(p1, p2)
```

### 11.3 Electrical Validation Algorithm

```python
def validate_electrical_rules(space):
    for route in space.routes:
        source_voltage = get_component_voltage(route.source)
        dest_voltage = get_component_voltage(route.destination)
        
        if source_voltage > dest_voltage:
            raise VoltageError(route, source_voltage, dest_voltage)
        
        trace_width = calculate_trace_width(route.max_current)
        if route.trace_width < trace_width:
            raise TraceWidthError(route, trace_width)
```

---

**Document Status**: Draft Specification  
**Last Updated**: March 2026  
**Part of**: Hardware Script (.hw) Documentation Suite

---
