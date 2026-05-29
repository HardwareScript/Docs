# Hardware Script v0.1.5 - The Software Heritage

**Document Type**: Architectural Philosophy  
**Status**: Official Release  
**Version**: 0.1.5  

---

## The Shoulders of Giants

Hardware Script is not an evolution of traditional EDA tools (like Altium or Cadence). It is an evolution of **Software Engineering**. To build a language capable of defining physical reality, we looked at the last 50 years of software language design and applied their greatest breakthroughs to the physical domain.

Here is the "Software Heritage" of Hardware Script, and how concepts from world-class compilers ensure the physical safety and scalability of hardware.

### 1. From C & C++: The Universal Assembler & Zero-Cost Abstractions

**The Universal Assembler (Portability):**
Just as C abstracted the CPU (allowing one codebase to compile to x86 or ARM), Hardware Script abstracts the **Factory**. A developer writes one `.hw` file representing their circuit's intent. By changing the target profile (`hwc build main.hw --profile tsmc_5nm` vs `--profile jlcpcb_2layer`), the compiler recalculates the trace geometries to fit the specific manufacturer.

**Zero-Cost Abstractions:**
In C++, high-level templates compile down to raw, lightning-fast assembly. In Hardware Script, high-level `define module` blocks (like a 64-bit ALU) are completely flattened during Pass 2 of the compiler. The Voxel Engine only sees raw atoms and copper traces. The user gets beautiful architecture; the factory gets a perfectly optimized, flat Gerber/GDSII file.

### 2. From Rust: The Electrical Borrow Checker & Strict Safety

**The Electrical Borrow Checker:**
In Rust, multiple threads cannot mutably write to the same memory address (Data Races). In hardware, this is called a **Short Circuit**. 
Hardware Script tracks pin directions (`Input`, `Output`, `Passive`). A single physical net can have infinite `Input` pins reading from it, but the compiler enforces that it can have **exactly one** `Output` pin driving it. Connecting two `Outputs` results in a strict compiler error, preventing physical fires.

**No "Null Pointers" (Floating Pins):**
In hardware, a disconnected input pin acts as an antenna, picking up random RF noise and causing unpredictable bugs. If a component defines a pin, Hardware Script requires it to be routed. If the user intentionally wants to leave it disconnected, they must explicitly mark it (e.g., `leave MCU.GPIO4 floating`), forcing intentionality.

### 3. From Zig: No Hidden Magic & `comptime`

**Zero Hidden Power Nets:**
Legacy EDA tools often auto-connect `VCC` and `GND` pins silently behind the scenes. Hardware Script adopts Zig's philosophy: **What You See Is What You Get.** If a microchip needs power, the user must explicitly route it. If it is not in the text, it does not exist in physical reality.

**Compile-Time Execution (`comptime`):**
Hardware Script rejects messy pre-processor macros. Generating a 128-bit data bus is done using standard `for` loops. Because routing is declarative, these loops execute strictly at compile-time, stamping out physical geometry into the Voxel Grid without runtime overhead.

### 4. From F#: Compile-Time Physics & Exhaustive Matching

**Units of Measure:**
Hardware Script's parser natively understands physics types (`10V`, `5mA`, `50Ω`). If a user attempts to route a 5000V signal through a trace rated for 50V, the compiler catches it during the mathematical DRC (Design Rule Check) pass. The compiler does the algebra; the user gets infinite physical safety.

**Exhaustive Pattern Matching:**
In traditional HDLs, forgetting a state in a state machine accidentally synthesizes a physical memory latch, ruining the silicon. Hardware Script enforces exhaustive matching for behavioral logic. If an `enum` has 4 states, the `match` statement must handle all 4, guaranteeing predictable silicon synthesis.

### 5. From Java & JavaScript: Interfaces & Selectors

**Hardware Interfaces (Polymorphism):**
To combat global supply chain shortages, Hardware Script uses interfaces. You define a contract (`define interface "I2C_Sensor"`). As long as a physical component implements that interface, you can swap the chip at compile-time (`--use Sensor_B`), and the compiler will automatically recalculate the physical routing for the new footprint.

**The POM (Physical Object Model):**
Inspired by CSS selectors querying the DOM, Hardware Script allows developers to query physical reality. You can apply constraints to massive groups of nets instantly:
`apply profile HighCurrent to nets: match "*_VCC"`.
