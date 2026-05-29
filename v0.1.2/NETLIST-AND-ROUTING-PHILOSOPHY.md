# Hardware Script v0.1.2 - Netlist and Routing Philosophy

**Document Type**: Fundamental Architecture Guide  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## The "Make or Break" Question

Should the automatic system decide what component connects to what?

**Answer**: NO. Absolutely not.

The automatic system should never decide what component connects to what. That is 100% determined by the user (or the logic schematic).

Your algorithm's only job is to figure out the **physical geometry** of how to connect them.

---

## Understanding "The Web of Connections"

### The Question

Does the battery connect directly to the camera? Or does everything just go to the CPU?

### The Answer: Nets (Networks)

In electronics, connections are not thought of as individual wires between two components. They are grouped into things called **"Nets"** (short for Networks).

**A Net is simply a list of pins that must all share the same electrical signal.**


### Two Main Types of Nets

#### 1. Data Nets (Point-to-Point or Buses)

**Characteristics**:
- Usually 1-to-1 or 1-to-a-few connections
- Specific signal paths
- Order matters

**Examples**:
```
Net "Camera_Data":
  - Camera.Pin_5 → CPU.Pin_42

Net "CPU_to_RAM":
  - CPU.Pin_10 → RAM.Pin_3
  - CPU.Pin_11 → RAM.Pin_4
  - CPU.Pin_12 → RAM.Pin_5
```

#### 2. Power & Ground Nets (The Global Web)

**Characteristics**:
- Connect to EVERY component
- Order doesn't matter
- Massive fanout

**The Reality**:

Yes, the battery (usually going through a power regulator chip first) must connect to every single component on the board:
- Camera needs power
- CPU needs power
- RAM needs power
- They all share the "VCC" (Power) Net and the "GND" (Ground) Net

**Example**:
```
Net "VCC_5V":
  - Battery.Plus
  - Regulator.Out
  - Camera.VCC
  - CPU.VCC
  - RAM.VCC
  - LED.Anode
  - Sensor.VCC

Net "GND":
  - Battery.Minus
  - Regulator.GND
  - Camera.GND
  - CPU.GND
  - RAM.GND
  - LED.Cathode
  - Sensor.GND
```


### How This Affects Complexity

**The Problem**:

If your algorithm tries to draw individual, thin wires from the battery to every single component, you will instantly create a massive, tangled bowl of spaghetti that is impossible to route.

**The Solution**:

Use different strategies for different net types:
- **Data Nets**: Route individual traces
- **Power/Ground Nets**: Use copper planes (entire layers of solid copper)

---

## The Division of Labor: Human vs. Auto-Router

In professional EDA (Electronic Design Automation) software like Altium or KiCad, the division of labor between the Human and the Auto-Router is very strict.

**Here is the blueprint you should copy for your voxel engine.**

### A. The User Defines the "Netlist" (The Logic)

Before your 3D water algorithm does anything, the user must provide a **"Netlist"**.

This is just a simple data list that says:

```
Net 1: Connect CPU.Pin_4 to Camera.Pin_2
Net 2: Connect CPU.Pin_5 to RAM.Pin_1
Net 3 (Ground): Connect Battery.Pin_Neg, Camera.Pin_1, CPU.Pin_10, RAM.Pin_8
```

**In Hardware Script syntax**:
```hw
# User explicitly defines connections
Battery.Plus -> Regulator.VIN
Regulator.VOUT -> CPU.VCC
Regulator.VOUT -> Camera.VCC
Regulator.VOUT -> RAM.VCC

CPU.Data_Out -> Camera.Data_In
CPU.Data_Out -> RAM.Data_In
```


### B. The Auto-Router Handles Data (The Geometry)

For Net 1 and Net 2 (data connections), your deterministic algorithm takes over.

**What it does**:
1. Looks at Pin A and Pin B
2. Uses the rules we discussed (45-degree turns, Manhattan routing on layers)
3. Finds the best path
4. Respects clearance and trace width constraints

**What it does NOT do**:
- Decide which pins to connect
- Understand what the signal means
- Know why the camera needs to talk to the CPU

### C. The "Plane" Cheat for Power (How to Handle the Battery)

For something like Net 3 (Power/Ground), auto-routers do not route wires.

Instead, the software allows the user to dedicate an entire Z-Layer of the board as a **"Copper Plane"**.

**Layer Strategy**:
```
Layer 1: X-axis routing (horizontal traces)
Layer 2: Solid copper sheet dedicated to Ground
Layer 3: Solid copper sheet dedicated to Power (5V)
Layer 4: Y-axis routing (vertical traces)
```

**How Components Connect to Planes**:

If the Camera needs Ground, the algorithm doesn't draw a line to the battery. It just drops a single vertical **Via** (straight down 90 degrees) directly into Layer 2.

**Boom, it's connected to the battery.**

**Example**:
```
Camera @ [1, 50, 50]
Camera.GND needs ground

Algorithm:
  1. Drill via from [1, 50, 50] to [2, 50, 50]
  2. Layer 2 is solid copper (Ground plane)
  3. Ground plane connects to Battery.Minus
  4. Done. Camera is grounded.
```


---

## Where Your Automatic System Should STOP Dead

Your auto-router should be a **"slave" to the user's constraints**.

It should stop dead and ask for user intervention in these scenarios:

### Scenario 1: The "Trapped" Scenario (100% Cost)

**What happens**:

If your algorithm tries to route from A to B, but is completely blocked by other wires and cannot find a path without violating spacing rules, it stops.

**What it does**:
```
❌ ROUTING FAILED

Cannot route Net "USB_Data" from CPU.Pin_5 to Camera.Pin_2
  - All paths blocked by existing routes
  - No valid path found within constraints

Suggestions:
  1. Move components closer together
  2. Add another routing layer
  3. Manually route this connection with waypoints
  4. Remove or reroute blocking traces
```

**What it does NOT do**:
- Violate clearance rules to "force" a path
- Delete other routes to make space
- Give up silently and leave it unrouted

### Scenario 2: Critical High-Speed Signals

**The Problem**:

In the real world, connections between a CPU and RAM must be exact, matching lengths to the millimeter so data arrives at the exact same picosecond.

An auto-router usually ruins this.

**The Solution**:

The system should allow the user to **manually route the "VIP" wires first**, lock them in place, and then tell the auto-router to fill in the rest around them.

**In Hardware Script**:
```hw
# User manually routes critical signal with exact waypoints
CPU.DDR_CLK -> RAM.CLK:
    path:
        - [1, 50, 50]
        - [1, 55, 50]
        - [1, 55, 60]
        - [1, 60, 60]
    locked: true  # Auto-router cannot modify this

# Auto-router handles the rest
CPU.Data_0 -> RAM.Data_0  # Algorithm finds path
CPU.Data_1 -> RAM.Data_1  # Algorithm finds path
```


### Scenario 3: Waypoints (User Guidance)

**The Problem**:

If the auto-router takes a path the user thinks is ugly or inefficient, the system should allow the user to provide hints.

**The Solution**:

The system should allow the user to click a few **"checkpoints"** or **"waypoints"** in the 3D space.

The auto-router then merely connects the dots:
```
Start → Waypoint 1 → Waypoint 2 → End
```

**In Hardware Script**:
```hw
# Fully automatic (algorithm decides entire path)
Battery.Plus -> LED.Anode

# Semi-automatic (user provides waypoints, algorithm fills in details)
Battery.Plus -> LED.Anode:
    waypoints:
        - [1, 10, 10]  # Start (Battery position)
        - [1, 50, 10]  # User hint: go right first
        - [1, 50, 90]  # User hint: then go up
        - [1, 90, 90]  # End (LED position)
    # Algorithm fills in the exact path between waypoints

# Fully manual (user specifies every voxel)
Battery.Plus -> LED.Anode:
    path:
        - [1, 10, 10]
        - [1, 11, 10]
        - [1, 12, 10]
        # ... every single voxel
        - [1, 90, 90]
    locked: true
```

---

## Summary: The Architecture for Your Engine

### Keep Your Automatic System Mathematically Simple

**Input**: An array of pins that need to be connected (defined by the user)

**Execution**: The algorithm connects them using pathfinding (like A* or Dijkstra's algorithm, heavily weighted by your physical manufacturing rules)

**The Line**: If the pathfinding algorithm cannot find a valid route, or if a rule is broken, it stops dead. It leaves a "ghost line" (a straight, unrouted visual line) between the two components and waits for the user to:
- Move components around
- Add a layer
- Manually route it
- Provide waypoints


### What Your Engine Needs to Know

**Your engine does NOT need to know**:
- How a camera works
- Why it needs power
- What the data signal means
- The logic of the circuit

**Your engine ONLY needs to know**:
- Point A coordinates
- Point B coordinates
- Physical constraints (clearance, trace width, layers)
- Existing obstacles (other routes, components)

---

## Implementation Checklist

### Phase 1: Netlist Parsing (High Priority)

- [ ] Parse connection statements from .hw files
- [ ] Build a graph of all connections (nets)
- [ ] Identify net types (data vs. power/ground)
- [ ] Store voltage and current requirements per net

### Phase 2: Routing Strategies (Current Work)

- [ ] Implement point-to-point routing for data nets
- [ ] Implement copper plane strategy for power/ground
- [ ] Add via generation for plane connections
- [ ] Support manual waypoints
- [ ] Support locked routes

### Phase 3: Failure Handling (Next Sprint)

- [ ] Detect trapped/blocked routes
- [ ] Generate helpful error messages
- [ ] Suggest solutions (move components, add layers)
- [ ] Visualize unrouted connections as "ghost lines"

---

## Key Takeaways

1. **User defines WHAT connects** - The netlist is user-specified, not automatic

2. **Router defines HOW to connect** - The algorithm only handles geometry

3. **Two net types** - Data (point-to-point) vs. Power/Ground (planes)

4. **Copper planes for power** - Don't route individual wires for power distribution

5. **Stop on failure** - Don't violate rules to force a path

6. **Support manual override** - Waypoints and locked routes for critical signals

7. **Be a slave to constraints** - The router serves the user, not the other way around

---

## Summary: Your Next Steps

### Right Now
- ✅ Understand the netlist concept
- ✅ Recognize the difference between data nets and power nets

### Next Sprint
- 🔄 Implement netlist parsing
- 🔄 Build connection graph
- 🔄 Identify net types automatically

### Next Month
- 📋 Implement copper plane strategy
- 📋 Add waypoint support
- 📋 Add failure detection and helpful errors

**You don't need to teach your engine how a camera works or why it needs power. You only need to teach it how to connect Point A to Point B perfectly within the rules of physics.**

---

**Document Status**: Fundamental Architecture Guide  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite

