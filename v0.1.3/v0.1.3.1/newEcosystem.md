This document has been thoroughly rewritten to reflect the v0.1.4 "Rust Moment" paradigm shift. It replaces the old "10-File Architecture" with the elegant **3-File Architecture**, transforming the section on file extensions into a guide on project organization and definition blocks.

---

# Book 3: The Ecosystem & Project Structure

**Hardware Script v0.1.4**  
**Target Audience**: Project architects setting up workspaces  
**Last Updated**: March 2026

---

## Foundation

This document consolidates the following source materials:

**Source Files**:
- `Docs/v0.1.4/LANGUAGE-BREAKING-CHANGES.md` — The transition from 10 extensions to the unified `.hw` ecosystem
- `Docs/v0.1.1/FILE-EXTENSIONS.md` — The original separation of domains (now updated to block types)
- `Docs/v0.1.2/FILE-EXTENSIONS-VALIDATION.md` — The domain separation rationale
- `Docs/v0.0.1/TOOLING.md` — The package registry, standard library paths, and CLI commands
- `Docs/v0.1.2/IDE-INTEGRATION-AND-LAUNCH.md` — VS Code extension setup and LSP architecture

---

## Quick Reference: The 3-File Architecture

Before v0.1.4, Hardware Script used 10 different file extensions (`.hwx`, `.hwp`, `.hwmat`, etc.). We realized this recreated the exact fragmentation problem we were trying to solve. 

Now, Hardware Script uses exactly **3 file extensions** for everything:

```
Project Root
├── hw.toml                    # 1. Project manifest & dependencies
├── hw.lock                    # 2. Dependency version lockfile
├── materials.hw               # 3. Hardware Script Source (Material definitions)
├── profiles.hw                # 3. Hardware Script Source (Factory rules)
├── components.hw              # 3. Hardware Script Source (Custom ICs)
├── constraints.hw             # 3. Hardware Script Source (Mechanical/Signal rules)
├── interface.hw               # 3. Hardware Script Source (Firmware bindings)
├── tests.hw                   # 3. Hardware Script Source (CI/CD physics tests)
├── main.hw                    # 3. Hardware Script Source (The assembly/entry point)
└── build/                     # Generated outputs
    ├── board.hwx              # Compiled simulation executable
    ├── robot_controller.h     # Auto-generated firmware header
    ├── gerber/                # PCB manufacturing files
    ├── assembly/              # Pick-and-place files
    ├── bom/                   # Bill of materials
    └── viz/                   # 3D models (.glb, .obj)
```

---

## Part I: Philosophy and Design Principles

### The Rust Moment: Unifying the Ecosystem

In C and C++, you have `.c`, `.cpp`, `.h`, `.hpp`—a scattered mess of headers and implementations. Rust looked at that and said: **"No. Everything is just `.rs`. What matters is what you declare inside of it."**

Hardware Script v0.1.4 embraces this exact philosophy. We do not use YAML for configurations and `.hw` for routing. **Everything that defines physical reality or engineering intent is written in Hardware Script natively.**

### The Stakeholder Map: Perfect Domain Separation

Hardware engineering requires collaboration across many disciplines. Instead of siloing these experts into different proprietary tools and file formats, Hardware Script brings them together into **one language with dedicated definition blocks**.

| Stakeholder | Definition Block | Domain Expertise |
|-------------|------------------|------------------|
| System Architect | `hw.toml` | Project definition and package dependencies |
| Logic/Layout Engineer | `define space` | Trace routing and component placement |
| Component Engineer | `define component` | Atomic component definitions & layout |
| Mechanical Engineer | `define mechanical` | Physical boundaries and keep-outs |
| Materials Scientist | `define material` | Custom alloys and thermal properties |
| RF / High-Speed Eng | `define signal_group` | Signal integrity and impedance rules |
| Firmware Developer | `define interface` | Hardware-software API bindings |
| Test/QA Engineer | `define test` | Bare-metal physics validation |
| Manufacturing Eng | `define profile` | Factory capabilities and fabrication limits |

**No one is missing. You have successfully containerized physical reality under one grammar.**

---

## Part II: The 3 File Types

### 1. The Manifest (`hw.toml`)

**What it is**: Project metadata, build scripts, and package dependencies.

**Analogy**: `package.json` (Node.js) or `Cargo.toml` (Rust)

**Why we need it**: Tells the `hpm` (Hardware Package Manager) what libraries to download, defines the project name, and sets the default target scale. **This is the only file that does not use `.hw` syntax**, as it must be readable by standard CI/CD tooling and registry servers.

**Example**:
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
pro_model =["extra_ram", "bluetooth", "wifi"]
lite_model = []
default = ["lite_model"]
```

### 2. The Lockfile (`hw.lock`)

**What it is**: Auto-generated file that locks exact versions and cryptographic hashes of component libraries.

**Analogy**: `package-lock.json` (Node.js) or `Cargo.lock` (Rust)

**Why we need it**: Hardware is expensive. If you compile a board today and re-compile it 5 years from now, you must generate the exact same geometry. The lockfile ensures a dependency update doesn't accidentally move a component by 0.1mm and ruin your board.

### 3. Hardware Script Source (`.hw`)

**What it is**: The universal language file. Everything that defines physical reality goes here. 

**Analogy**: `.rs` (Rust) or `.ts` (TypeScript)

**Why we need it**: By consolidating all physical definitions into a single file extension, we enable:
- **Flawless IDE integration** (one grammar for syntax highlighting)
- **Sub-millisecond compilation** (the parser loads all tokens into memory upfront)
- **Native SI Units** (no more YAML `min_width_nm` hacks; everything natively understands `254µm`)
- **LLM generation** (an AI can output an entire hardware ecosystem in one code block)

---

## Part III: Project Organization Best Practices

While you *can* write a 10,000-line `main.hw` file containing all your materials, components, rules, and routing, **standard practice** is to separate concerns logically by file name. 

Here is how a professional team organizes a Hardware Script project using the unified `.hw` language.

### 1. `materials.hw` (Physical Properties)
Defines the atomic physical and electrical properties of conductors, insulators, and semiconductors. The engine uses this for thermal and electrical validation.

```hw
## Custom Silver Alloy for High-Frequency RF
define material "Silver_Alloy_X":
    category: conductor
    symbol: "AgX"
    
    properties:
        density: 10490 kg/m3
        thermal_conductivity: 429 W/mK
        resistivity: 1.59e-8 Ω/m
        max_current_density: 40 A/mm2
        color: "#C0C0C0"

## Flexible Substrate
define material "Polyimide_Kapton":
    category: insulator
    symbol: "PI"
    
    properties:
        dielectric_strength: 303 kV/mm
        relative_permittivity: 3.4
        max_operating_temp: 360 C
        color: "#FFA500"
```

### 2. `profiles.hw` (Manufacturing Rules)
Defines the physical limits of the factory you are using (e.g., JLCPCB, PCBWay, TSMC). The compiler throws an error if your routing violates these rules.

```hw
## JLCPCB 4-Layer Standard Constraints
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

### 3. `components.hw` (Atomic Parts)
Defines the mathematical, electrical, and physical layout of specific parts.

```hw
## Standard 0805 SMD Resistor
define component "Resistor_0805":
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
            Pin2 at[x:2.0mm, y:0]
            
    electrical:
        max_voltage: 150V
        max_power: 0.125W
```

### 4. `constraints.hw` (Mechanical Boundaries & Speeds)
Defines enclosure limits and speed-of-light timing rules.

```hw
## Physical boundaries for the IoT Enclosure
define mechanical "RobotEnclosure":
    dimensions: 150mm by 100mm by 50mm
    
    mounting_holes:
        - at[x:5, y:5] diameter 3mm
        - at [x:145, y:95] diameter 3mm
        
    keepout:
        - region [x:20, y:20] to[x:60, y:60] height 15mm

## USB 2.0 Differential Pair Rules
define signal_group "USB_Data":
    type: differential_pair
    target_impedance: 90Ω
    impedance_tolerance: 5%
    max_length_mismatch: 0.15mm
```

### 5. `interface.hw` (Firmware Contract)
Binds physical hardware nets to software variables. Generates C/Rust header files for the firmware team.

```hw
## API Contract for the Firmware Team
define interface "RobotController":
    target: "ESP32_WROOM_32"
    
    bindings:
        Motor_PWM = DriverIC.Pin_4
        Status_LED = LED1.Anode
        Temp_Sensor = Thermistor.Out
```

### 6. `main.hw` (The Assembly)
The entry point. Imports your modular files and routes the board together.

```hw
import "materials.hw"
import "profiles.hw"
import "components.hw"
import "constraints.hw"

define space "MotorController":
    dimensions: 50mm by 50mm by 2mm
    grid: 500 by 500 by 2
    origin: tl by t
    
    profile: JLCPCB_4Layer
    mechanical: RobotEnclosure
    
    add substrate(Polyimide_Kapton) spanning [1, 1, 1] to[500, 500, 2]
    
    add Resistor_0805 (10kΩ) named PullUp at [x:20, y:20, z:1] rotated 90
    
    route Power.Plus to PullUp.Pin1:
        path:
            - [x:10, y:10, z:1]
            -[x:20, y:10, z:1]
            -[x:20, y:20, z:1]
```

---

## Part IV: Package Management (hpm)

### The Hardware Package Ecosystem

Hardware Script implements a package management system analogous to NPM or Cargo. 

**Example standard imports in `main.hw`**:
```hw
import BluetoothModule from "@hardware/comms"
import TemperatureSensor from "@sensors/environmental"
import LithiumBattery from "@power/rechargeable"
```

Each package download includes the `.hw` files that define the component's geometry and physics, along with associated 3D models (`.glb`) stored in an `assets/` directory.

### Package Manager CLI (hpm)

**Installation**:
```bash
# Install Hardware Script CLI
curl -sSL https://hw-script.org/install.sh | bash
```

**Package Commands**:
```bash
hpm init my-project          # Initialize new project (creates hw.toml and main.hw)
hpm search temperature       # Search registry for sensors
hpm install @sensors/dht22   # Install package
hpm update                   # Update hw.lock
```

### LLM Integration (AI-Assisted Development)

Hardware Script is designed to be **LLM-native** from the ground up. Every new project initialized with `hpm init` includes a `.hw-llm-index.md` file that serves as a "bootloader" for AI assistants (ChatGPT, Claude, Cursor).

**The Problem**: AI assistants have zero pre-training data on Hardware Script. Without guidance, they will hallucinate syntax from Verilog or Python.

**The Solution**: The `.hw-llm-index.md` provides:
- Minimal syntax rules to get started
- Tri-Fold Case Sensitivity rules (lowercase keywords, SI units)
- Command references for querying full documentation via the CLI (`hpm doc read <name>`)

This enables **zero-shot Hardware Script code generation** entirely offline, without requiring the AI to have pre-training data on the language.

---

## Part V: IDE Integration

Because Hardware Script uses a single `.hw` extension for everything, IDE integration is phenomenally fast and stable. 

### VS Code Extension

**Features**:
- Syntax highlighting for all `.hw` definition blocks
- IntelliSense autocomplete for components, pins, and SI units
- Real-time error checking via Language Server Protocol (LSP)
- Integrated terminal for `hws verify` and `hws build`
- Native 3D preview panel for visual debugging

**Installation**:
```bash
code --install-extension hardware-script.hw-language-support
```

### Language Server Protocol (LSP)

The unified grammar allows the Hardware Script LSP to provide:
- Semantic analysis and validation (e.g., verifying `254µm` fits inside the constraints of `profiles.hw`)
- Go-to-definition (Cmd+Click a material in `main.hw` to jump straight to `materials.hw`)
- Find all references for routing nets
- Code actions for quick fixes (e.g., suggesting proper SI unit capitalization)

---

## Part VI: CI/CD & Manufacturing Integration

### Continuous Integration (GitHub Actions)

Because Hardware Script is entirely text-based, hardware validation runs natively in CI/CD pipelines just like software testing.

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
        run: hws verify main.hw
      
      - name: Run Test Benches
        run: hws test tests.hw
      
      - name: Generate Manufacturing Files
        run: hws build main.hw --target pcb
```

### Supported Fabrication Services

The compiler natively outputs industry-standard formats (Gerber, Excellon Drill, GDSII) which can be sent directly to manufacturers. 

```bash
# Generate manufacturing files
hws build main.hw --target pcb

# Get automated fabrication quote via API integration
hws quote --service jlcpcb --quantity 10

# Place order (requires API key)
hws order --service jlcpcb --quantity 10 --shipping express
```

---

## Conclusion

Hardware Script's transition to the unified 3-File Architecture (`hw.toml`, `hw.lock`, `.hw`) fundamentally solves the fragmentation problem of electronic design automation. 

The ecosystem provides complete coverage of the hardware design workflow from concept to manufacturing, without forcing the user to switch between YAML configurations, JSON schemas, and proprietary GUI binaries.

With package management (hpm), perfect IDE integration, CI/CD support, and a syntax designed specifically for LLM collaboration, Hardware Script provides the ultimate, modern developer experience for physical engineering.