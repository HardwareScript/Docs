# Book 3: The Ecosystem & Project Structure

**Hardware Script v0.1.4**  
**Target Audience**: Project architects setting up workspaces  
**Last Updated**: March 2026

---

## The 3-File Architecture

```
Project Root
├── hw.toml                    # Project manifest & dependencies
├── hw.lock                    # Dependency version lockfile
├── materials.hw               # Material definitions
├── profiles.hw                # Factory rules
├── components.hw              # Atomic component definitions
├── modules.hw                 # Logical schematics (no coordinates)
├── constraints.hw             # Mechanical/Signal rules
├── interface.hw               # Firmware bindings
├── tests.hw                   # CI/CD physics tests
├── main.hw                    # Physical assembly/entry point
└── build/                     # Generated outputs
    ├── board.hwx              # Compiled simulation executable
    ├── robot_controller.h     # Auto-generated firmware header
    ├── gerber/                # PCB manufacturing files
    ├── assembly/              # Pick-and-place files
    ├── bom/                   # Bill of materials
    └── viz/                   # 3D models (.glb, .obj)
```

Hardware Script uses exactly **3 file extensions**:
- `hw.toml` - Project manifest
- `hw.lock` - Dependency lockfile
- `.hw` - Hardware Script source (everything)

---

## Stakeholder Map

| Stakeholder | Definition Block | Domain |
|-------------|------------------|--------|
| System Architect | `hw.toml` | Project definition and dependencies |
| Logic Designer | `define module` | Electrical schematics (pure logic, no coordinates) |
| Layout Engineer | `define space` | Physical placement and trace routing |
| Component Engineer | `define component` | Atomic component definitions with parameters |
| Mechanical Engineer | `define mechanical` | Physical boundaries and keep-outs |
| Materials Scientist | `define material` | Custom alloys and thermal properties |
| RF / High-Speed Eng | `define signal_group` | Signal integrity and impedance rules |
| Firmware Developer | `define interface` | Hardware-software API bindings |
| Test/QA Engineer | `define test` | Bare-metal physics validation |
| Manufacturing Eng | `define profile` | Factory capabilities and fabrication limits |

---

## File Types

### 1. hw.toml (Project Manifest)

Project metadata, build scripts, and package dependencies.

```toml
[project]
name = "robot-controller"
version = "1.0.0"
scale = "PCB"
author = "Your Name"
license = "MIT"

[dependencies]
"@power/5v-regulator" = "1.2.0"
"@sensors/imu-module" = "2.0.1"
"@motor/h-bridge" = "3.0.0"

[build]
default_target = "pcb"
optimization_level = 2

[features]
pro_model = ["extra_ram", "bluetooth", "wifi"]
lite_model = []
default = ["lite_model"]
```

### 2. hw.lock (Lockfile)

Auto-generated file that locks exact versions and cryptographic hashes of component libraries. Ensures reproducible builds.

### 3. .hw (Hardware Script Source)

Universal language file. Everything that defines physical reality goes here.

---

## Project Organization

Professional projects separate concerns by file name, following the Logical/Physical Duality principle:

**Logical Layer** (Pure Intent):
- `modules.hw` - Electrical schematics with no coordinates

**Physical Layer** (Absolute Reality):
- `main.hw` - Physical assembly with coordinates
- `materials.hw` - Atomic properties
- `profiles.hw` - Factory rules
- `components.hw` - Atomic parts with parameters

**Constraints Layer** (Rules):
- `constraints.hw` - Mechanical and signal rules
- `interface.hw` - Firmware bindings
- `tests.hw` - Physics validation

### materials.hw

Defines atomic physical and electrical properties:

```hw
define material "Silver_Alloy_X":
    category: conductor
    symbol: "AgX"
    
    properties:
        density: 10490 kg/m3
        thermal_conductivity: 429 W/mK
        resistivity: 1.59e-8 Ω/m
        max_current_density: 40 A/mm2
        color: "#C0C0C0"

define material "Polyimide_Kapton":
    category: insulator
    symbol: "PI"
    
    properties:
        dielectric_strength: 303 kV/mm
        relative_permittivity: 3.4
        max_operating_temp: 360 C
        color: "#FFA500"
```

### profiles.hw

Defines physical limits of the factory:

```hw
define profile "JLCPCB_4Layer":
    description: "Standard via-in-pad limits and trace widths"
    
    trace:
        min_width: 127µm
        min_spacing: 127µm
        
    via:
        min_diameter: 300µm
        min_annular_ring: 150µm
        
    layer:
        max_count: 4
        min_thickness: 35µm
```

### components.hw

Defines mathematical, electrical, and physical layout of atomic parts with parametric generics:

```hw
# Parametric component - accepts arguments for reusability
define component "Resistor_0805" (val: Measurement, tol: Measurement):
    metadata:
        manufacturer: "Yageo"
        package: "0805"
        
    pins:
        Pin1
        Pin2
        
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.5mm)
        pin_positions:
            Pin1 at [x:0, y:0]
            Pin2 at [x:2.0mm, y:0]
            
    electrical:
        resistance: val        # Direct parameter reference
        tolerance: tol         # No {} interpolation needed
        max_voltage: 150V
        max_power: 0.125W

# Usage with keyword arguments for clarity
add Resistor_0805 (val: 10kΩ, tol: 1%) named R1
```

### modules.hw

Defines logical schematics with pure electrical intent (no physical coordinates):

```hw
import "components.hw"

# A logical module - pure schematic, no coordinates
define module "LED_Driver":
    pins:
        VCC
        GND
        
    add Resistor_0805 (val: 100Ω, tol: 5%) named R1
    add LED_0805 ("Red") named Light
    
    # Pure electrical connections - no [x, y, z] coordinates!
    route VCC to R1.Pin1
    route R1.Pin2 to Light.Anode
    route Light.Cathode to GND

# Comptime parametric generation - 64-bit ALU
define module "64Bit_ALU":
    pins:
        Bus_A[64]      # Array pin (hardware bus)
        Bus_B[64]
        Bus_Out[64]
        
    # Comptime loop - unrolls at compile time
    for i in 0..63:
        add SingleBit_ALU named Bit[i]
        
        route Bus_A[i] to Bit[i].In_A
        route Bus_B[i] to Bit[i].In_B
        route Bit[i].Out to Bus_Out[i]
        
        # Comptime conditional - evaluates at compile time
        if i > 0:
            route Bit[i-1].CarryOut to Bit[i].CarryIn
```

### constraints.hw

Defines enclosure limits and speed-of-light timing rules:

```hw
define mechanical "RobotEnclosure":
    dimensions: 150mm by 100mm by 50mm
    
    mounting_holes:
        - at [x:5, y:5] diameter 3mm
        - at [x:145, y:95] diameter 3mm
        
    keepout:
        - region [x:20, y:20] to [x:60, y:60] height 15mm

define signal_group "USB_Data":
    type: differential_pair
    target_impedance: 90Ω
    impedance_tolerance: 5%
    max_length_mismatch: 0.15mm
```

### interface.hw

Binds physical hardware nets to software variables:

```hw
define interface "RobotController":
    target: "ESP32_WROOM_32"
    
    bindings:
        Motor_PWM = DriverIC.Pin_4
        Status_LED = LED1.Anode
        Temp_Sensor = Thermistor.Out
```

### tests.hw

Automated scripts for bare-metal CI/CD physics validation:

```hw
define test "Short Circuit Protection":
    setup:
        apply 12V to PowerSource.VIN
        apply 0V to PowerSource.GND
        
    execute:
        short Regulator.VOUT to GND
        wait 1ms
        
    assert:
        Regulator.VOUT < 0.5V
        Regulator.temperature < 100 C
        PowerSource.current < 2A
```

### main.hw

The entry point that imports everything and maps logical modules to physical space:

```hw
import "materials.hw"
import "profiles.hw"
import "components.hw"
import "modules.hw"
import "constraints.hw"

define space "MotorController":
    dimensions: 50mm by 50mm by 2mm
    grid: 500 by 500 by 2
    origin: tl by t
    
    profile: JLCPCB_4Layer
    mechanical: RobotEnclosure
    
    add substrate(Polyimide_Kapton) spanning [1, 1, 1] to [500, 500, 2]
    
    # Instantiate a logical module
    add LED_Driver named Driver1
    
    # Map the module's internal components to physical coordinates
    layout Driver1:
        R1 at [x:20, y:20, z:1]
        Light at [x:25, y:20, z:1]
    
    # Comptime loop for physical placement
    for i in 0..7:
        add DDR6_RAM named VRAM[i] at [x: 50 + (i*20), y: 20, z:1]
    
    route Power.Plus to Driver1.VCC:
        path:
            - [x:10, y:10, z:1]
            - [x:20, y:10, z:1]
            - [x:20, y:20, z:1]
```

---

## Package Management (hpm)

### The Two-Tier System

Hardware Script has a **two-tier package architecture**:

**Tier 1: Standard Library (`@std/`)** - Ships with compiler, works offline
```hw
import Copper from "@std/materials"
import Resistor_0805 from "@std/components"
import NAND from "@std/logic"
```

**Tier 2: HPM Registry** - Cloud-hosted, downloaded on-demand
```hw
import ESP32 from "@espressif/microcontrollers"
import MotorDriver from "@adafruit/drivers"
import TempSensor from "@sensors/environmental"
```

### Installation

```bash
curl -sSL https://hw-script.org/install.sh | bash
```

### Commands

```bash
hpm init my-project                    # Initialize new project
hpm search temperature                 # Search registry
hpm install @sensors/dht22             # Install package
hpm install @espressif/esp32_wroom_32  # Install vendor chip
hpm update                             # Update hw.lock
```

### Registry Security & CI/CD

The HPM Registry uses **automated validation** for all submitted packages:

**Security Check**: Scans `.hw` files for malicious content (HardwareScript is declarative text with no system-level APIs like `fs.readFile()` or `os.execute()`)

**Physics Check**: Compiles the component in a sandbox and runs Design Rule Check (DRC)

**Unit Validation**: Ensures packages don't attempt to redefine core physics units (mm, V, A, C)

**Auto-Approval**: If the chip compiles and obeys physics laws, the registry automatically publishes it

**Reserved Namespaces**: `@std` and `@standard` are hard-blocked at the registry level

### Community Unit Packages

The HPM ecosystem enables domain-specific unit extensions without bloating the compiler:

```bash
# RF Engineering
hpm install @rf-engineering/microwave-units

# Nuclear Sensors
hpm install @nuclear/radiation-units

# Automotive Standards
hpm install @automotive/can-bus-units

# Mechanical Engineering
hpm install @mechanical/torque-units
```

**Example usage**:
```hw
import "@rf-engineering/microwave-units"

define component "WiFi_Antenna" (freq: Measurement, power: Measurement):
    electrical:
        operating_frequency: freq  # Now understands GHz
        max_power: power           # Now understands dBm

add WiFi_Antenna (freq: 2.4GHz, power: 10dBm) named Antenna1
```

This architecture prevents maintainer burnout by delegating domain-specific standards to the community while keeping the core compiler focused on the 4 essential physics units.

### Self-Hosted Registries

Companies can host private, internal HPM registries for proprietary silicon:

```toml
# hw.toml
[registry]
url = "https://hpm.internal.tesla.com"
token = "${TESLA_HPM_TOKEN}"
```

Each package includes `.hw` files defining geometry and physics, plus associated 3D models (`.glb`) in an `assets/` directory.

### LLM Integration

Every project initialized with `hpm init` includes a `.hw-llm-index.md` file that serves as a "bootloader" for AI assistants (ChatGPT, Claude, Cursor).

**The Problem**: AI assistants have zero pre-training data on Hardware Script.

**The Solution**: The `.hw-llm-index.md` provides:
- Minimal syntax rules
- Tri-Fold Case Sensitivity rules
- Command references for querying full documentation via CLI (`hpm doc read <name>`)

This enables zero-shot Hardware Script code generation entirely offline.

---

## IDE Integration

### VS Code Extension

**Features**:
- Syntax highlighting for all `.hw` definition blocks
- IntelliSense autocomplete for components, pins, and SI units
- Real-time error checking via Language Server Protocol (LSP)
- Integrated terminal for `hwc verify` and `hwc build`
- Native 3D preview panel

**Installation**:
```bash
code --install-extension hardware-script.hw-language-support
```

### Language Server Protocol (LSP)

The unified grammar allows the Hardware Script LSP to provide:
- Semantic analysis and validation
- Go-to-definition (Cmd+Click a material in `main.hw` to jump to `materials.hw`)
- Find all references for routing nets
- Code actions for quick fixes

---

## CI/CD Integration

### GitHub Actions

```yaml
name: Hardware Verification

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Install Hardware Script
        run: curl -sSL https://hw-script.org/install.sh | bash
      
      - name: Verify Physics and Constraints
        run: hwc verify main.hw
      
      - name: Run Test Benches
        run: hwc test tests.hw
      
      - name: Generate Manufacturing Files
        run: hwc build main.hw --target pcb
```

### Manufacturing Integration

```bash
# Generate manufacturing files
hwc build main.hw --target pcb

# Get automated fabrication quote
hwc quote --service jlcpcb --quantity 10

# Place order (requires API key)
hwc order --service jlcpcb --quantity 10 --shipping express
```

---

## Compilation Targets

Hardware Script uses explicit abstraction blocks inside `define component` to separate different levels of hardware description. The same `.hw` codebase compiles to different targets based on the `--target` CLI flag.

| Target | Reads | Ignores | Primary Output |
|--------|-------|---------|----------------|
| `pcb` (Default) | `pins:`, `layout:` | `behavior:`, `electrical:` | Gerber files, Drill files |
| `fpga` | `pins:`, `behavior:` | `layout:`, `render:` | Verilog/VHDL |
| `asic` | `pins:`, `behavior:`, `layout:` | `render:` | GDSII layout files |
| `spice` | `pins:`, `electrical:` | `layout:`, `render:` | SPICE netlist |
| `viz` | `pins:`, `layout:`, `render:` | `behavior:`, `electrical:` | OBJ/GLB 3D models |

---

## Generated Outputs

### .hwx (Simulation Executable)

Compiled binary containing 3D grid, material states, and physics data. Fed into simulation engine or Blender plugin.

```bash
hwc build board.hw --target sim --output board.hwx
```

### Manufacturing Files

**Gerber** (`.gtl`, `.gbl`) - PCB manufacturing  
**GDSII** (`.gds`) - Silicon chip manufacturing  
**3D models** (`.obj`, `.glb`, `.step`) - Visualization and mechanical CAD  
**Blender scripts** (`.py`) - Photorealistic rendering  
**BOM files** (`.csv`) - Bill of materials  
**Drill files** (`.drl`) - Via specifications

---

## Project Type Examples

Different projects use different combinations of `define` blocks. Here's what you typically define for each project type:

| Project Type | Typical Files | Definition Blocks Used |
|-------------|---------------|------------------------|
| **Simple LED Circuit** | `main.hw` | `define space`, `add`, `route` |
| **Custom PCB** | `main.hw`, `components.hw`, `profiles.hw` | `define space`, `define component`, `define profile` |
| **Modular PCB** | `main.hw`, `modules.hw`, `components.hw` | `define space`, `define module`, `define component`, `layout` |
| **IoT Device** | `main.hw`, `modules.hw`, `components.hw`, `interface.hw`, `tests.hw` | `define space`, `define module`, `define component`, `define interface`, `define test` |
| **High-Speed Board** | `main.hw`, `constraints.hw`, `profiles.hw` | `define space`, `define signal_group`, `define profile` |
| **RF/Antenna Design** | `main.hw`, `materials.hw`, `constraints.hw` | `define space`, `define material`, `define signal_group`, `add polygon` |
| **Robotics Controller** | `main.hw`, `modules.hw`, `components.hw`, `constraints.hw`, `interface.hw`, `tests.hw` | `define space`, `define module`, `define component`, `define mechanical`, `define interface`, `define test` |
| **Power Supply** | `main.hw`, `components.hw`, `tests.hw` | `define space`, `define component`, `define test` |
| **FPGA Development** | `main.hw`, `modules.hw`, `components.hw`, `constraints.hw` | `define space`, `define module`, `define component`, `define signal_group` (timing) |
| **Silicon Chip (ASIC)** | `main.hw`, `materials.hw`, `profiles.hw`, `components.hw`, `modules.hw` | `define space`, `define module`, `define material`, `define profile` (PDK), `define component` |
| **GPU/Multi-Core CPU** | `main.hw`, `modules.hw`, `components.hw` | `define space`, `define module` (with `for` loops), `layout`, `define component` |
| **Automotive ECU** | `main.hw`, `modules.hw`, `components.hw`, `constraints.hw`, `interface.hw`, `tests.hw` | `define space`, `define module`, `define component`, `define mechanical` (vibration), `define interface`, `define test` (environmental) |

**Key Insights**: 
- The same `.hw` extension is used for everything, but the `define` blocks inside determine what you're building
- A simple project might have everything in `main.hw`, while a complex project organizes definitions across multiple `.hw` files
- The Logical/Physical Duality (`define module` vs `define space`) enables fractal encapsulation and infinite reusability
- Comptime features (`for` loops, `if` conditionals, parameters) enable parametric generation of massive systems from minimal code

---

## Conclusion

Hardware Script's unified 3-File Architecture (`hw.toml`, `hw.lock`, `.hw`) solves the fragmentation problem of electronic design automation.

The **two-tier package system** mirrors world-class languages:
- **Standard Library** (`@std/`): Bundled with compiler, works offline, irreducible baseline
- **HPM Registry**: Cloud-hosted ecosystem with automated security validation

The Logical/Physical Duality (`define module` vs `define space`) enables true separation of concerns:
- Logic designers write pure schematics without coordinates
- Layout engineers map logic to physical space
- The compiler flattens modules into spaces during Pass 2

Comptime features (`for` loops, `if` conditionals, parametric generics) enable massive systems to be generated from minimal code, scaling from simple LED circuits to 64-bit GPUs.

With secure package management (hpm), IDE integration, CI/CD support, and LLM collaboration, Hardware Script provides the ultimate modern developer experience for physical engineering.
