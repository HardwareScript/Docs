# Hardware Script (.hw) Tooling and Ecosystem

**Version**: 0.2 (Draft)  
**Document Type**: Development Tools, Package Management, and Ecosystem Guide

---

## 1. The Hardware Package Ecosystem

### 1.1 Component Registry Architecture

Hardware Script implements a package management system analogous to NPM (Node Package Manager) for software:

```hw
import BluetoothModule from @hardware/comms
import TemperatureSensor from @sensors/environmental
import LithiumBattery from @power/rechargeable
```

Each package includes:
- Complete electrical specifications
- 3D mechanical models and footprints
- Thermal characteristics
- Reference designs and usage examples
- Version history and compatibility information

### 1.2 Package Structure

A standard hardware package contains:

```
package-name/
├── package.hw.json          # Package metadata
├── component.hw             # Component definition
├── models/
│   ├── mesh.obj            # 3D model
│   └── footprint.kicad     # PCB footprint
├── docs/
│   ├── README.md           # Usage documentation
│   └── datasheet.pdf       # Manufacturer datasheet
└── examples/
    └── basic-usage.hw      # Example implementations
```

### 1.3 Package Metadata (package.hw.json)

```json
{
  "name": "@sensors/temperature-ds18b20",
  "version": "1.2.0",
  "description": "Digital temperature sensor with 1-Wire interface",
  "author": "Hardware Script Community",
  "license": "MIT",
  "keywords": ["sensor", "temperature", "1-wire"],
  "electrical": {
    "voltage_min": "3.0V",
    "voltage_max": "5.5V",
    "current_typical": "1.5mA",
    "current_max": "4mA"
  },
  "physical": {
    "footprint": "TO-92",
    "dimensions": "5.2mm x 4.1mm x 2.5mm"
  },
  "dependencies": {
    "@standard/materials": "^1.0.0"
  }
}
```

### 1.4 Ecosystem Benefits

1. **Reusability**: Proven designs can be shared and reused across projects
2. **Standardization**: Common interfaces and footprints reduce integration complexity
3. **Rapid Prototyping**: Complex subsystems can be integrated without deep domain expertise
4. **Version Management**: Track component revisions and maintain compatibility
5. **Community Contribution**: Open-source hardware development at scale

---

## 2. Standard Library Reference

### 2.1 standard.materials

**Conductors**
- `Copper` - Standard PCB copper (1oz, 2oz variants)
- `Gold` - Contact plating and high-reliability connections
- `Silver` - High-conductivity applications
- `Aluminum` - Heat sinks and chassis

**Insulators**
- `FR4` - Standard PCB substrate
- `Rogers` - High-frequency RF applications
- `Polyimide` - Flexible PCB substrate
- `Ceramic` - High-temperature applications

**Semiconductors**
- `Silicon` - Standard semiconductor material
- `GalliumArsenide` - High-frequency applications
- `SiliconCarbide` - High-power applications

### 2.2 standard.power

**Power Sources**
- `Battery(voltage)` - Generic battery with voltage specification
- `LithiumIon(capacity, voltage)` - Rechargeable lithium battery
- `SolarCell(voltage, current)` - Photovoltaic cell
- `ACAdapter(voltage, current)` - Wall power adapter

**Power Management**
- `VoltageRegulator(input, output)` - Linear voltage regulator
- `BuckConverter(input, output, current)` - Step-down switching regulator
- `BoostConverter(input, output, current)` - Step-up switching regulator
- `Capacitor(value, voltage)` - Energy storage and filtering

### 2.3 standard.sensors

**Environmental**
- `TemperatureSensor(range, accuracy)` - Temperature measurement
- `HumiditySensor(range, accuracy)` - Humidity measurement
- `PressureSensor(range, accuracy)` - Barometric pressure
- `LightSensor(range, spectrum)` - Light intensity measurement

**Motion**
- `Accelerometer(axes, range)` - Linear acceleration
- `Gyroscope(axes, range)` - Angular velocity
- `Magnetometer(axes, range)` - Magnetic field
- `IMU(dof)` - Inertial measurement unit (combined sensors)

**Proximity**
- `UltrasonicSensor(range)` - Distance measurement
- `InfraredSensor(range)` - IR proximity detection
- `LidarSensor(range, resolution)` - Laser distance measurement

### 2.4 standard.switches

**Transistors**
- `Transistor_NPN(voltage, current)` - NPN bipolar junction transistor
- `Transistor_PNP(voltage, current)` - PNP bipolar junction transistor
- `MOSFET_N(voltage, current)` - N-channel MOSFET
- `MOSFET_P(voltage, current)` - P-channel MOSFET

**Relays**
- `Relay(coil_voltage, contact_rating)` - Electromechanical relay
- `SolidStateRelay(voltage, current)` - Solid-state switching

**Mechanical**
- `Button(type)` - Momentary or latching button
- `Switch(poles, throws)` - Mechanical switch

### 2.5 standard.logic

**Basic Gates**
- `AND(inputs)` - Logical AND gate
- `OR(inputs)` - Logical OR gate
- `NOT()` - Logical NOT gate (inverter)
- `XOR(inputs)` - Logical XOR gate
- `NAND(inputs)` - Logical NAND gate
- `NOR(inputs)` - Logical NOR gate

**Complex Logic**
- `Multiplexer(inputs, select)` - Data selector
- `Demultiplexer(outputs, select)` - Data distributor
- `FlipFlop(type)` - Memory element (D, JK, SR, T)
- `Counter(bits, direction)` - Digital counter

### 2.6 standard.comms

**Serial Protocols**
- `UART(baud_rate)` - Universal asynchronous receiver-transmitter
- `SPI(clock_speed)` - Serial peripheral interface
- `I2C(clock_speed)` - Inter-integrated circuit

**Wireless**
- `Bluetooth(version, range)` - Bluetooth module
- `WiFi(standard, frequency)` - WiFi module
- `LoRa(frequency, range)` - Long-range radio
- `Zigbee(frequency)` - Low-power mesh networking

**Wired Networking**
- `Ethernet(speed)` - Ethernet PHY
- `CAN(speed)` - Controller area network
- `USB(version, speed)` - Universal serial bus

---

## 3. IDE Integration

### 3.1 VS Code Extension

**Features**
- Syntax highlighting for .hw files
- IntelliSense autocomplete for components and properties
- Real-time error checking and linting
- Integrated terminal for `hw verify` and `hw generate`
- 3D preview panel for visual debugging
- Component library browser

**Installation**
```bash
code --install-extension hardware-script.hw-language-support
```

### 3.2 Language Server Protocol (LSP)

The Hardware Script LSP provides:
- Semantic analysis and validation
- Go-to-definition for components and materials
- Find all references for routing and connections
- Hover documentation for standard library components
- Code actions for quick fixes

### 3.3 Syntax Highlighting

**Supported Editors**
- Visual Studio Code
- Sublime Text
- Atom
- Vim/Neovim
- Emacs

**TextMate Grammar** (for editor integration):
```json
{
  "scopeName": "source.hw",
  "patterns": [
    {
      "name": "keyword.control.hw",
      "match": "\\b(define|import|from|add|connect|route|rule|if|else)\\b"
    },
    {
      "name": "entity.name.type.hw",
      "match": "\\b(Material|Component|Space)\\b"
    },
    {
      "name": "constant.numeric.hw",
      "match": "\\b\\d+(\\.\\d+)?(mm|cm|m|V|A|mA|Ohms|C|F)\\b"
    }
  ]
}
```

---

## 4. Testing and Validation Tools

### 4.1 Unit Testing Framework

```hw
# test_sprinkler.hw
import test from standard.testing

test "Power Distribution":
    given:
        Space "TestBoard"
        Battery (5V) named Power at [1, 1, 1]
        LED named Light at [1, 10, 10]
        connect Power.out to Light.in
    
    assert:
        Light.voltage == 5V
        Light.current < 20mA

test "Logic Rules":
    given:
        LightSensor named Eye
        Valve named Output
        rule "Trigger":
            if Eye.light_level < 50%:
                Output state is ON
    
    when:
        Eye.light_level = 30%
    
    assert:
        Output.state == ON
```

### 4.2 Simulation Testing

```bash
# Run simulation tests
hw test --mode simulation test_sprinkler.hw

# Output:
✓ Power Distribution (12ms)
✓ Logic Rules (8ms)
✓ Thermal Limits (15ms)

3 tests passed, 0 failed
```

### 4.3 Manufacturing Validation

```bash
# Validate design for manufacturing
hw validate --drc --erc main.hw

# Design Rule Check (DRC)
✓ Trace width meets minimum (0.15mm)
✓ Clearance meets minimum (0.15mm)
✓ Via size meets minimum (0.3mm)

# Electrical Rule Check (ERC)
✓ No floating nets
✓ All power pins connected
✓ Voltage levels compatible
```

---

## 5. CI/CD Integration

### 5.1 GitHub Actions Workflow

```yaml
name: Hardware Verification

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Install Hardware Script
        run: |
          curl -sSL https://hw-script.org/install.sh | bash
          echo "$HOME/.hw/bin" >> $GITHUB_PATH
      
      - name: Verify Design
        run: hw verify main.hw
      
      - name: Run Tests
        run: hw test tests/*.hw
      
      - name: Generate Manufacturing Files
        run: hw generate --target manufacturing
      
      - name: Upload Gerbers
        uses: actions/upload-artifact@v2
        with:
          name: gerber-files
          path: build/factory/
```

### 5.2 GitLab CI Pipeline

```yaml
stages:
  - verify
  - test
  - build

verify_design:
  stage: verify
  script:
    - hw verify main.hw

run_tests:
  stage: test
  script:
    - hw test tests/*.hw

generate_outputs:
  stage: build
  script:
    - hw generate --target simulation
    - hw generate --target manufacturing
  artifacts:
    paths:
      - build/
```

---

## 6. Package Management CLI

### 6.1 Installation

```bash
# Install Hardware Script CLI
curl -sSL https://hw-script.org/install.sh | bash

# Or using package managers
brew install hardware-script    # macOS
apt install hardware-script     # Ubuntu/Debian
choco install hardware-script   # Windows
```

### 6.2 Package Commands

```bash
# Initialize new project
hw init my-project

# Search for packages
hw search temperature sensor

# Install package
hw install @sensors/temperature-ds18b20

# Update packages
hw update

# List installed packages
hw list

# Publish package (requires authentication)
hw publish
```

### 6.3 Project Initialization

```bash
hw init sprinkler-controller

# Creates:
# sprinkler-controller/
# ├── hw.config.yaml
# ├── main.hw
# ├── .gitignore
# └── README.md
```

---

## 7. Community and Ecosystem

### 7.1 Package Registry

**Official Registry**: https://registry.hw-script.org

**Package Categories**
- Power Management
- Sensors
- Communication Modules
- Display Drivers
- Motor Controllers
- Audio Processing
- RF/Wireless
- Development Boards

### 7.2 Community Resources

**Documentation**: https://docs.hw-script.org
**Forum**: https://community.hw-script.org
**GitHub**: https://github.com/hwsl-lang
**Discord**: https://discord.gg/G9VBxKpW
**Twitter**: @hwsl_lang
**Email**: hardwarescript@gmail.com

### 7.3 Contributing

```bash
# Fork and clone repository
git clone https://github.com/your-username/hw-component.git

# Create component package
hw init --type component my-sensor

# Test locally
hw test my-sensor.hw

# Publish to registry
hw publish --tag latest
```

---

## 8. Manufacturing Integration

### 8.1 Supported Fabrication Services

- **PCBWay**: Direct API integration
- **JLCPCB**: Automated quote and order
- **OSH Park**: Community PCB service
- **Seeed Studio**: Fusion PCB service

### 8.2 Order Workflow

```bash
# Generate manufacturing files
hw generate --target manufacturing

# Get fabrication quote
hw quote --service jlcpcb --quantity 10

# Place order (requires API key)
hw order --service jlcpcb --quantity 10 --shipping express
```

### 8.3 Assembly Services

```bash
# Generate assembly files (BOM + CPL)
hw generate --target assembly

# Check component availability
hw check-stock build/factory/bom.csv

# Get assembly quote
hw quote --service jlcpcb --assembly --quantity 10
```

---

## 9. Advanced Tooling

### 9.1 Design Rule Checker (DRC)

```bash
# Run design rule check
hw drc --rules oshpark main.hw

# Custom rules file
hw drc --rules custom-rules.yaml main.hw
```

**custom-rules.yaml**:
```yaml
trace_width_min: 0.15mm
trace_spacing_min: 0.15mm
via_diameter_min: 0.3mm
via_drill_min: 0.2mm
copper_to_edge_min: 0.5mm
```

### 9.2 Bill of Materials (BOM) Optimizer

```bash
# Optimize BOM for cost
hw optimize-bom --target cost build/factory/bom.csv

# Optimize for availability
hw optimize-bom --target availability build/factory/bom.csv

# Suggest alternatives
hw suggest-alternatives R1 build/factory/bom.csv
```

### 9.3 3D Viewer

```bash
# Launch interactive 3D viewer
hw view main.hw

# Export 3D model
hw export-3d --format step main.hw
hw export-3d --format obj main.hw
```

---

**Document Status**: Draft Specification  
**Last Updated**: March 2026  
**Part of**: Hardware Script (.hw) Documentation Suite

---
