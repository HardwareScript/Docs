# Hardware Script v0.1.2 - Materials Database Philosophy

**Document Type**: Technical Foundation  
**Status**: Architectural Explanation  
**Last Updated**: March 2026

---

## Purpose of This Document

This document addresses the fundamental question of how to represent all materials in a computationally efficient way and explains the LLM-compiler feedback loop.

**Key questions**:
1. How do we define materials without maintaining an infinite library?
2. Are materials just combinations from the periodic table?
3. How does the compiler-LLM feedback loop work?

---

## The Materials Question

### The Problem Statement

**From the notes**:

> "For every material we can't just define the height, the mass, and the space. We have to define the material, but keeping up a library of all the materials will be computationally expensive. If we can define them as a combination of a percentage of a universal set of rules, that's what I'm thinking about the periodic table."

### The Answer: Macro-Properties, Not Atomic Composition

**Hardware Script doesn't need to model atomic structure.**

**It needs to model electrical, thermal, and mechanical properties.**

### The Approach

**Instead of**:
```yaml
copper:
  atomic_number: 29
  electron_configuration: [Ar] 3d10 4s1
  crystal_structure: face_centered_cubic
```

**Use macro-properties**:
```yaml
copper:
  # Electrical
  resistivity_ohm_m: 1.68e-08
  max_current_density_a_mm2: 35
  
  # Thermal
  thermal_conductivity_w_mk: 401
  melting_point_c: 1085
  
  # Mechanical
  density_kg_m3: 8960
  
  # Visual
  color_hex: '#B87333'
```

### Why This Works

**For hardware design, you don't need to know**:
- Atomic structure
- Quantum mechanics
- Crystal lattice
- Electron orbitals

**You need to know**:
- Will it conduct electricity? (resistivity)
- Will it melt? (melting point, thermal conductivity)
- How much does it weigh? (density)
- What does it look like? (color)

### The Standard Library

**Hardware Script ships with a standard materials database**:

```yaml
# standard-materials.yaml

conductors:
  copper: {...}
  aluminum: {...}
  gold: {...}
  silver: {...}

insulators:
  fr4: {...}
  air: {...}
  silicon_dioxide: {...}
  polyimide: {...}

semiconductors:
  silicon: {...}
  gallium_nitride: {...}
  gallium_arsenide: {...}
```

**~50-100 common materials cover 99% of use cases.**

### Custom Materials

**Users can define custom materials**:

```yaml
# my_materials.yaml

conductors:
  custom_alloy:
    name: "My Custom Copper Alloy"
    resistivity_ohm_m: 2.0e-08
    thermal_conductivity_w_mk: 350
    density_kg_m3: 8500
    color_hex: '#AA6633'
```

**The compiler validates that all required properties are present.**

### Computational Efficiency

**Memory usage**:
```
Standard library: ~50 materials × ~10 properties = 500 values
Custom materials: User-defined, typically <10 materials
Total: <1KB of data
```

**Lookup time**: O(1) hashmap lookup by material name

**No atomic simulation required.**

---

## The Compilation Validation System

### The Vision

**From the notes**:

> "After the LLM generates it for you, compile it. The compilation will find the error inside for you, not just the LLM."

### How It Works

**Traditional EDA tools**:
```
1. Design in GUI
2. Run separate checks (ERC, DRC, SPICE)
3. Manually interpret results
4. Fix issues
5. Repeat
```

**Hardware Script**:
```
1. Write .hw file (or LLM generates it)
2. Run: hws build board.hw
3. Compiler validates everything automatically
4. Get clear error messages
5. Fix and recompile
```

### What the Compiler Validates

**1. Electrical Rules Check (ERC)**:
```
- Output pins not connected to output pins
- Power nets have proper voltage
- All pins connected
- No floating nets
```

**2. Design Rules Check (DRC)**:
```
- Trace widths meet manufacturing limits
- Clearances maintained
- Via sizes valid
- Component spacing adequate
```

**3. Physics Validation**:
```
- Traces won't melt (current capacity)
- Voltage drops acceptable
- Thermal limits not exceeded
- Signal integrity maintained
```

**4. Logic Validation** (for behavioral blocks):
```
- Boolean expressions valid
- State machines complete
- Timing constraints met
```

### Example Error Messages

**Electrical error**:
```
Error: hwc::electrical::short_circuit (E0051)
  × Short Circuit: Via contacts copper plane without clearance
   ╭─[board.hw:48:17]
 48 │             - [2, 30, 30]  # Via through ground plane
   ·               ─────┬─────
   ·                    ╰── Via contacts GroundPlane (0V) = SHORT!
   ╰────
  help: Via drills through solid copper ground plane.
        Add 'clearance: 0.5mm' to create an anti-pad.
```

**Physics error**:
```
Error: hwc::physics::ampacity_violation (E0043)
  × Trace Width Insufficient: Trace will overheat under specified current
   ╭─[board.hw:28:1]
 28 │     route Motor.Power to Battery.Plus:
   ·           ─────┬─────
   ·                ╰── This net carries 10A
 31 │             - [1, 80, 20]
   ·               ─────┬─────
   ·                    ╰── Trace width: 0.5mm (too thin)
   ╰────
  help: For 10A through 1oz copper with 10°C rise, minimum width is 2.5mm.
  hint: Add 'width: 3mm' to the route block.
```

---

## The LLM-Compiler Feedback Loop

### The Magic Loop

**This is where Hardware Script becomes revolutionary.**

**The workflow**:

```
1. User: "Design a 5V to 3.3V regulator"
   ↓
2. LLM: Generates .hw file
   ↓
3. Compiler: Validates physics
   ↓
4. Compiler: Finds error (trace too thin)
   ↓
5. System: Feeds error back to LLM
   ↓
6. LLM: "Ah, my mistake, increasing trace width"
   ↓
7. LLM: Regenerates .hw file
   ↓
8. Compiler: Validates again
   ↓
9. Repeat until: Build Successful: 0 Errors
```

### The Implementation

**Agentic loop**:

```python
def design_with_llm(prompt: str, max_iterations: int = 10):
    for i in range(max_iterations):
        # LLM generates design
        hw_code = llm.generate(prompt)
        
        # Save to file
        with open("design.hw", "w") as f:
            f.write(hw_code)
        
        # Compile
        result = subprocess.run(
            ["hws", "build", "design.hw"],
            capture_output=True,
            text=True
        )
        
        if result.returncode == 0:
            print("✅ Build successful!")
            return hw_code
        
        # Feed errors back to LLM
        errors = result.stderr
        prompt = f"""
        The previous design had errors:
        
        {errors}
        
        Please fix these issues and regenerate the .hw file.
        Original request: {prompt}
        """
    
    raise Exception("Failed to generate valid design after max iterations")
```

### Why This Works

**Because everything is text**:
- LLM can read the .hw file
- LLM can read the error messages
- LLM can understand the physics
- LLM can fix the issues

**The compiler provides**:
- Clear error messages
- Exact line numbers
- Suggested fixes
- Physics explanations

**The LLM provides**:
- Design generation
- Error interpretation
- Automatic fixes
- Iterative improvement

### The Result

**Compiling, failing, rewriting, and recompiling a motherboard 1,000 times a minute until it perfectly passes physics checks.**

**This is only possible because everything is text.**

---

## The "Bypass Raspberry Pi" Philosophy

### The Problem with Current Approach

**Amateur developers use a $35 Raspberry Pi** (full computer with OS) just to:
- Turn on a sprinkler when it gets dark
- Read a temperature sensor
- Control a relay

**This is massive waste**:
- Money ($35 vs $0.50)
- Power (5W vs 0.01W)
- Space (credit card vs fingernail)
- Complexity (Linux OS vs discrete logic)

### The Hardware Script Approach

**Direct discrete logic without CPU overhead.**

**Example: Light-activated switch**

**Traditional approach**:
```
Raspberry Pi ($35)
    +
Python script
    +
GPIO library
    +
Photoresistor
    +
Relay
    =
$50, 5W, complex
```

**Hardware Script approach**:
```
Photoresistor
    +
Transistor (acts as switch)
    +
Relay
    =
$0.50, 0.01W, simple
```

### The Design Process

**User prompt**:
```
"Make a switch that turns on when the sun goes down"
```

**LLM generates**:
```hw
define Space "LightSwitch":
    dimensions: 20mm × 20mm × 2mm
    
    add Photoresistor named Sensor at [1, 5, 5]
    add Transistor_NPN named Switch at [1, 10, 10]
    add Relay named Output at [1, 15, 15]
    
    # When light is low, photoresistor resistance is high
    # This turns on transistor, which activates relay
    Sensor.out -> Switch.base
    Switch.collector -> Output.coil
```

**Compiler validates**:
```
✓ Photoresistor output voltage: 0.7V (low light)
✓ Transistor base threshold: 0.6V (will activate)
✓ Relay coil current: 50mA (within transistor limit)
✓ Build successful!
```

**Result**: $0.50 board the size of a fingernail.

---

## Key Takeaways

1. **Materials use macro-properties** - Not atomic composition

2. **Standard library covers 99% of cases** - ~50-100 common materials

3. **Custom materials are easy** - Just define the properties you need

4. **Computationally efficient** - <1KB data, O(1) lookup

5. **Compiler validates everything** - Electrical, physical, thermal, logic

6. **Clear error messages** - Like Rust compiler, with suggestions

7. **LLM feedback loop** - Automatic error fixing through iteration

8. **Bypass unnecessary complexity** - Direct discrete logic, no CPU needed

9. **Text-based enables automation** - LLM can read, write, and fix

10. **Revolutionary workflow** - Design → Compile → Fix → Repeat (automated)

---

## Summary

**Materials database philosophy**:
- Use macro-properties (resistivity, thermal conductivity, density)
- Not atomic composition (too complex, unnecessary)
- Standard library + custom materials
- Computationally efficient (<1KB, O(1) lookup)

**Compilation validation**:
- Electrical rules (ERC)
- Design rules (DRC)
- Physics validation (thermal, current, voltage)
- Logic validation (behavioral blocks)
- Clear, actionable error messages

**LLM-compiler feedback loop**:
- LLM generates design
- Compiler validates
- Errors fed back to LLM
- LLM fixes and regenerates
- Repeat until successful
- Enables fully automated hardware design

**Bypass Raspberry Pi philosophy**:
- Direct discrete logic
- No CPU overhead
- $0.50 vs $35
- 0.01W vs 5W
- Fingernail vs credit card size

**This is the foundation for AI-native hardware design.**

---

**Document Status**: Technical Foundation  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite
