


# The HardwareScript™ Architecture Specification
**Version 1.0 | Absolute System Definition**

HardwareScript is the universal assembler for physical reality. It is a unified, declarative programming language and compiler infrastructure that scales from doping individual silicon P-N junctions to routing 64-bit multi-core GPU motherboards. 

The core philosophy is absolute: **The compiler is the law of physics.** If a HardwareScript project compiles and passes its internal tests, it is mathematically guaranteed to function in physical reality. 

---

## 1. The Unified Ecosystem
The entire hardware design process—materials, manufacturing constraints, logical schematics, physical layout, and behavioral testing—is containerized into exactly three file types.

1. **`hw.toml`**: The package manifest. Defines project metadata, target architecture, and dependencies (e.g., `@standard/materials`).
2. **`hw.lock`**: The cryptographic lockfile ensuring 100% reproducible physical geometries across machines and decades.
3. **`.hw`**: The HardwareScript source file. The universal language of the ecosystem.

---

## 2. The Language: Logical/Physical Duality
HardwareScript utilizes distinct semantic blocks to separate physical assembly from logical intent. This enables true Zero-Cost Abstractions and infinite component reusability.

### A. The Foundation: Materials & Profiles
Before drawing traces, the developer defines the physical atoms and factory constraints.

```ruby
# The physical atoms
define material "TSMC_P_Doped":
    category: semiconductor
    properties:
        resistivity: 1.2e-3 Ω·m
        mobility: 450 cm²/Vs

# The factory capabilities
define profile "TSMC_5nm_Rules":
    trace:
        min_width: 5nm
        min_spacing: 5nm
    clearance:
        safety_factor: 1.5
```

### B. The Physical Assembler (`define space`)
A `space` is a physical boundary where coordinates exist. It is used to arrange raw materials or sub-components in absolute 3D space. **Everything is a box inside a box (Fractal Encapsulation).**

```ruby
# Building a physical Transistor from raw materials
define space "NMOS_Transistor":
    dimensions: 50nm by 50nm by 20nm
    grid: 50 by 50 by 20
    origin: tl by t
    profile: TSMC_5nm_Rules
    
    add substrate(TSMC_P_Doped) spanning [1,1,1] to [50,50,5]
    
    add block(TSMC_N_Doped) named Source at [x:5, y:20, z:5] dimensions: 15nm by 10nm by 5nm
    add block(TSMC_N_Doped) named Drain at[x:30, y:20, z:5] dimensions: 15nm by 10nm by 5nm
    
    # Expose physical blocks as logical pins for higher-level use
    expose Gate_Poly as G
    expose Source as S
    expose Drain as D
```

### C. The Logical Schematic (`define module`)
A `module` represents pure electrical intent. **No physical coordinates are allowed.** It is a reusable schematic that consumes `spaces` or other `modules`.

```ruby
import "nmos.hw"
import "pmos.hw"

# Building a CMOS Inverter (Logic only, no layout)
define module "Inverter_Logic":
    pins:
        In
        Out
        VDD
        VSS
        
    # Instantiate the physical spaces we built above
    add PMOS_Transistor named p1
    add NMOS_Transistor named n1
    
    # Wire the logic
    route In to p1.G
    route In to n1.G
    route p1.D to Out
    route n1.D to Out
    route p1.S to VDD
    route n1.S to VSS
```

### D. Parametric Generation & Signal Bundles
When assembling massive systems (like a GPU motherboard), developers use Comptime Loops and Array Pins to generate millions of physical features instantly.

```ruby
import "alu_logic.hw"
import DDR6_RAM from "@samsung/memory"

define space "GPU_Motherboard":
    dimensions: 300mm by 150mm by 2mm
    origin: tl by t
    
    add Custom_GPU_Chip named MyGPU at [x:150, y:75, z:1]
    
    # Comptime Loop: Generates 8 physical RAM chips
    for i in 0..7:
        # Dynamic coordinate placement
        add DDR6_RAM named VRAM[i] at[x: 50 + (i*20), y: 20, z:1] 
        
        # Signal Bundle: Routes a 32-bit hardware bus in one line
        route MyGPU.MemBus[i] to VRAM[i].DataBus
```

---

## 3. Behavioral SPICE Simulation (System 5)
HardwareScript features a native, in-memory SPICE and physics simulation engine. The compiler traces electrons through P-N junctions, calculates voltage thresholds, evaluates thermal dissipation, and logically flips gates natively.

Testing is written using asynchronous, event-driven syntax directly in `.hw` files.

```ruby
define test "CMOS Inverter Switching Logic":
    setup:
        apply 3.3V to Inverter_Logic.VDD
        apply 0V to Inverter_Logic.VSS
        
    execute:
        # Push a logical 1 (High)
        apply 3.3V to Inverter_Logic.In
        wait 2ns  # Allow propagation delay
        
    assert:
        # Verify logical 0 (Low) output
        Inverter_Logic.Out < 0.1V
        # Verify thermal dynamics
        n1.temperature < 45C
```
If this test fails, compilation halts. **The compiler is the physics law.**

---

## 4. The Compiler Pipeline (MLIR Architecture)
The `hwc` compiler utilizes a Multi-Level Intermediate Representation pipeline to flatten modular code into absolute physical reality.

1. **Pass 1: Symbol Registration.** Lexes and parses `.hw` files natively parsing SI units (`254µm`, `10kΩ`). Populates the Symbol Table with `materials`, `profiles`, and `modules`.
2. **Pass 2: Space Unrolling.** Flattens all `define module` logic and `for` loops into a raw Netlist. Maps all relative/local coordinates to absolute global coordinates based on their parent `define space`.
3. **Pass 3: The Spatial Engine.** Allocates the design into a massive, Data-Oriented 3D Voxel Grid using **Morton Z-Curve Encoding**. Traces and components are mapped to cache-friendly contiguous memory.
4. **Pass 4: Physics & DRC.** The SPICE/Physics engine sweeps the Morton grid, checking for clearance violations, dielectric breakdowns, and running the `define test` behavioral simulations.

---

## 5. The 3-Tier Output Architecture
HardwareScript separates the mathematical truth (the compiler) from the visual representation. 

### Tier 1: The CI/CD Terminal (Source of Truth)
For automated testing and server-side validation.
```bash
$ hwc test main.hw
✅ [Pass] Thermal Dissipation Check (Max: 65°C)
✅ [Pass] CMOS Inverter Switching Logic (Delay: 1.2ns)
✅ Compilation successful. 0 errors.
```

### Tier 2: The `.hwx` Binary & Native Visualizer (Hot Module Reloading)
When running `hwc serve`, the compiler outputs a highly compressed binary format called **`.hwx`** (Hardware Executable). 
* The `.hwx` contains the fully solved physics data, 3D voxel matrices, and electrical states.
* It streams instantly to the **HardwareScript VS Code Extension** (via WebGL/WebGPU) or a standalone Visualizer Engine.
* As the developer edits the `.hw` text, the engine hot-reloads the 3D visualizer in sub-milliseconds, highlighting live voltages, data buses, and temperatures on screen.

### Tier 3: The Exporters (Transpilation)
For interfacing with legacy factories and marketing departments, the `.hwx` binary is translated via custom emitters (`hwc build --target <format>`):
* **`--target pcb`**: Outputs industry-standard Gerber and Excellon Drill files for PCB printing.
* **`--target silicon`**: Outputs GDSII files for TSMC/Intel lithography.
* **`--target viz`**: Outputs native Python scripting for Blender to generate photorealistic, ray-traced marketing animations of the hardware.

---

## 6. Package Management (`hpm`)
The ecosystem is distributed via the Hardware Package Manager (`hpm`). Because the entire language is unified, installing a package brings the logic, the physics, and the 3D geometry simultaneously.

```bash
$ hpm install @samsung/gddr6-memory
$ hpm install @standard/logic-gates
```

Libraries are instantly accessible via standard import paths (`import "@samsung/gddr6-memory" as VRAM`), allowing the global community to collaboratively assemble the world's open-source hardware library.