# Hardware Script v0.1.6 - Lazy Realization Architecture

**Document Type**: Instantaneous Development Loop  
**Status**: Sub-Second Iteration Speed Blueprint  
**Last Updated**: April 2026

---

## The Core Insight: Logical Truth vs. Physical Realization

**The Problem**: Current `hwc build` tries to realize the entire physical universe (Bit-Blit every atom into the grid) during development. For a 100M transistor design, this takes 40-50 seconds and uses 1.6 GB RAM.

**The Solution**: Separate logical validation from physical realization. Developers don't need to know where the copper is to verify their logic is sound.

---

## The Three-Command Architecture

### Command 1: `hwc check` (The Instant Loop)

**Target**: < 500ms for 100M transistors

**What It Does**:
1. **Syntax Pass**: Validates .hw text
2. **Symbol Pass**: Registers components and modules
3. **Logical Synthesis**: Proves that A + B works and fits in bit-width
4. **Netlist Alignment**: Checks if M1.G is connected to a net

**The Secret**: **ZERO VOXELS**. The compiler works entirely in the Netlist Arena (integer math). No 3D grid allocation.

**Result**: Instant "Syntax OK" or "Logic Error" feedback.

```bash
$ hwc check processor.hw
✓ Syntax valid
✓ All modules resolved
✓ Netlist connectivity verified
✓ 100,234,567 transistors logically sound

Completed in 0.42s
```

### Command 2: `hwc simulate --digital` (The Functional Loop)

**Target**: < 1s to start

**What It Does**:
- Runs Rust-native boolean evaluator on synthesized logic
- Simulates the "Soft IP" (the logic)
- Ignores physical distance between gates
- Verifies functional correctness (e.g., CPU computes 2+2=4)

**The Secret**: Pure logical simulation. No physical geometry involved.

```bash
$ hwc simulate --digital processor.hw --test add_test.hw
Running digital simulation...
✓ Test: 2 + 2 = 4 (PASS)
✓ Test: 255 + 1 = 0 with overflow (PASS)
✓ All 1,000 test vectors passed

Completed in 0.8s
```

### Command 3: `hwc build --physical` (The Tape-Out)

**Target**: 40-60s (Foundry Prep)

**What It Does**:
1. **Bit-Blit Unrolling**: Stamps millions of transistors into 3D grid
2. **Physical DRC**: Checks for atom-level collisions
3. **Parasitic Extraction**: Calculates RC delays
4. **Manufacturing Prep**: Generates intermediate binary

**The Change**: This is **no longer the default**. This is what you do when ready to see the 3D model or send to factory.

```bash
$ hwc build --physical processor.hw
⚙️  Physical realization mode
🔧 Stamping 100,234,567 component instances...
   └─ Completed in 25.3s
🔍 Running physical DRC...
   └─ Completed in 8.7s
💾 Writing hardware simulation binary (.hwsb)...
   └─ Completed in 2.1s

Total: 36.1s
Memory: 1.6 GB
```

---

## Why We Saw 1.6 GB and 40 Seconds

The 1.6 GB is **not your source code** — it's the **Physical Memory Map** of the chip:

```
100M instances × 16 bytes per instance = 1.6 GB RAM
```

The 40 seconds is the time to OR-together billions of bits into the Morton-indexed grid.

**The Fix**: Lazy Realization. The Voxel Grid remains **empty** during development. It only fills up when:

1. You run Physical DRC (`hwc build --physical`)
2. You open 3D Viewport and zoom in
3. You trigger final export (`hwc export gdsii`)

---

## The Netlist-First Rule

### Old Architecture (v0.1.5)

```
Parse → Compile → Bit-Blit ALL atoms → Export ALL formats
                  ↑
                  Bottleneck: 40s + 1.6 GB
```

### New Architecture (v0.1.6)

```
Parse → Compile → Netlist Arena (instant)
                  ↓
                  Lazy Realization (on-demand)
                  ↓
                  Physical Grid (only when needed)
```

**Key Change**: `HardwareSpace` stores **Instance Pointers**, not Flat Pours.

```rust
// Old Way (v0.1.5): Metadata explosion
pub struct HardwareSpace {
    pub pours: Vec<PourMetadata>,  // 500M pours = 100 GB
}

// New Way (v0.1.6): Structural proxying
pub struct HardwareSpace {
    pub component_instances: Vec<ComponentInstance>,  // 100M instances = 1.6 GB
    pub voxel_grid: LazyVoxelGrid,  // Empty until realized
}
```

---

## The Instant 3D Viewport Architecture

### The Proxy Pattern

When you open the 3D viewer with 100M transistors:

**LOD 0 (Far Away)**:
- IDE draws 100M tiny dots (one per transistor)
- GPU handles via instanced rendering
- Time: **Instant** (< 16ms per frame = 60 FPS)

**LOD 1 (Zooming In)**:
- You zoom into a 10µm box containing 1,000 transistors
- IDE asks compiler: "What's in this box?"
- **Reactive Unrolling**: Compiler unrolls only those 1,000 transistors
- Time: **< 1ms**

**LOD 2 (Close-Up)**:
- You zoom into a single transistor
- Compiler Bit-Blits that one stamp into a temporary buffer
- Renderer shows gate poly, source/drain diffusion, metal contacts
- Time: **< 0.1ms**

### Implementation

```rust
// hwc-engine/src/lazy_voxel_grid.rs

pub struct LazyVoxelGrid {
    /// Realized chunks (only populated on-demand)
    realized_chunks: FxHashMap<ChunkKey, VoxelChunk>,
    
    /// Component instances (always in memory)
    instances: Vec<ComponentInstance>,
    
    /// Spatial index for fast queries
    spatial_index: ComponentSpatialIndex,
}

impl LazyVoxelGrid {
    /// Get material at position (lazy realization)
    pub fn get_material_at(&mut self, pos: Point3D) -> MaterialId {
        let chunk_key = self.pos_to_chunk_key(pos);
        
        // Check if chunk is already realized
        if let Some(chunk) = self.realized_chunks.get(&chunk_key) {
            return chunk.get_voxel(pos);
        }
        
        // Chunk not realized - check component instances
        let instances_in_chunk = self.spatial_index.query_chunk(chunk_key);
        
        for &instance_idx in instances_in_chunk {
            let instance = &self.instances[instance_idx];
            
            if instance.bbox().contains(pos) {
                // Found instance containing this position
                // Return material from stamp (no realization needed)
                let stamp = instance.stamp();
                let rel_pos = pos - instance.origin;
                return stamp.get_material_at_relative(rel_pos);
            }
        }
        
        // Empty space
        AIR_MATERIAL_ID
    }
    
    /// Realize a chunk (on-demand)
    pub fn realize_chunk(&mut self, chunk_key: ChunkKey) {
        if self.realized_chunks.contains_key(&chunk_key) {
            return;  // Already realized
        }
        
        // Create new chunk
        let mut chunk = VoxelChunk::new();
        
        // Find all instances overlapping this chunk
        let instances = self.spatial_index.query_chunk(chunk_key);
        
        // Bit-Blit each instance into the chunk
        for &instance_idx in instances {
            let instance = &self.instances[instance_idx];
            let stamp = instance.stamp();
            
            chunk.stamp_bitmask(instance.origin, stamp);
        }
        
        // Store realized chunk
        self.realized_chunks.insert(chunk_key, chunk);
    }
}
```

---

## Export Separation

### Old: `hwc build` Does Everything

```bash
$ hwc build processor.hw
# Generates:
# - build/processor.glb (3D model)
# - build/processor.dxf (CAD)
# - build/processor.gerber (PCB)
# - build/processor.gds (Silicon)
# - build/processor.spice (Netlist)
# Total time: 50s
```

### New: Separate `hwc export`

```bash
# Fast development loop (< 1s)
$ hwc check processor.hw
$ hwc simulate --digital processor.hw

# When ready for visualization
$ hwc export glb processor.hw
# Generates: build/processor.glb
# Time: 5s (only realizes visible geometry)

# When ready for manufacturing
$ hwc export gdsii processor.hw --physical
# Generates: build/processor.gds
# Time: 40s (full physical realization)
```

### The `.hwsb` Format (Hardware Simulation Binary)

`hwc build` now produces a lightweight intermediate format:

```rust
// Hardware Simulation Binary (.hwsb)
pub struct HardwareSimulationBinary {
    /// Netlist (connectivity)
    pub netlist: NetlistArena,
    
    /// Component instances (pointers to stamps)
    pub instances: Vec<ComponentInstance>,
    
    /// Component stamps (shared geometry)
    pub stamps: Vec<ComponentStamp>,
    
    /// Metadata
    pub dimensions: Dimensions,
    pub grid: GridCells,
    pub voxel_size: VoxelSize,
}
```

**Size**: ~100 MB for 100M transistors (vs. 1.6 GB for realized grid)

**Load Time**: < 0.5s (memory-mapped file)

---

## Performance Comparison

### Developer's Reality

| Step | Current (v0.1.5) | Proposed (v0.1.6) | Speedup | Why It's Faster |
|------|------------------|-------------------|---------|-----------------|
| **Logic Check** | 1.2s | **0.05s** | 24× | No voxel grid allocation |
| **Full Build** | 12s | **0.8s** | 15× | No GDSII/GLB file writing |
| **Physical DRC** | N/A | **8s** | N/A | Bit-parallel sweeps on grid |
| **Export GDSII** | 50s | **40s** | 1.25× | One-time binary streaming |

### Memory Usage

| Phase | Current (v0.1.5) | Proposed (v0.1.6) | Reduction |
|-------|------------------|-------------------|-----------|
| **Check** | 500 MB | **50 MB** | 10× |
| **Build** | 1.6 GB | **100 MB** | 16× |
| **Physical** | 1.6 GB | **1.6 GB** | 1× |

---

## The Development Loop

### Typical Workflow

```bash
# 1. Write code
vim processor.hw

# 2. Instant validation (< 500ms)
hwc check processor.hw

# 3. Functional verification (< 1s)
hwc simulate --digital processor.hw --test tests/

# 4. Iterate rapidly
# (Repeat steps 1-3 hundreds of times)

# 5. When ready to visualize
hwc export glb processor.hw
open build/processor.glb

# 6. When ready for tape-out
hwc build --physical processor.hw
hwc export gdsii processor.hw
```

**Result**: The compiler feels like a **text editor**, not a heavy CAD tool.

---

## Implementation Checklist

### Phase 1: Command Separation
- [ ] Create `hwc check` command (syntax + netlist only)
- [ ] Create `hwc simulate --digital` command
- [ ] Modify `hwc build` to skip exports by default
- [ ] Add `hwc build --physical` flag for full realization
- [ ] Create `hwc export <format>` command
- [ ] Update CLI help text

### Phase 2: Lazy Voxel Grid
- [ ] Implement `LazyVoxelGrid` struct
- [ ] Add `get_material_at()` with on-demand realization
- [ ] Add `realize_chunk()` for viewport queries
- [ ] Implement chunk-based spatial index
- [ ] Test lazy realization correctness
- [ ] Benchmark memory usage (should be < 100 MB)

### Phase 3: Hardware Simulation Binary (.hwsb)
- [ ] Define `.hwsb` binary format
- [ ] Implement serialization (netlist + instances + stamps)
- [ ] Implement deserialization with memory mapping
- [ ] Add compression (zstd or lz4)
- [ ] Test load time (should be < 500ms)
- [ ] Document format specification

### Phase 4: Export Refactoring
- [ ] Move GLB export to `hwc export glb`
- [ ] Move DXF export to `hwc export dxf`
- [ ] Move Gerber export to `hwc export gerber`
- [ ] Move GDSII export to `hwc export gdsii`
- [ ] Add `--physical` flag to force full realization
- [ ] Test each export format independently

### Phase 5: IDE Integration
- [ ] Implement frustum-based chunk queries
- [ ] Add distance-based LOD selection
- [ ] Implement reactive unrolling for viewport
- [ ] Test 60 FPS with 100M transistors
- [ ] Profile GPU instanced rendering
- [ ] Add debug visualization for realized chunks

---

## The "Instantaneous One-Second Guarantee"

With Lazy Realization, Hardware Script achieves:

### For Developers (99% of time)
- **Check**: < 500ms
- **Simulate**: < 1s
- **Memory**: < 100 MB
- **Iteration**: Instant

### For Manufacturing (1% of time)
- **Physical DRC**: ~8s
- **Export GDSII**: ~40s
- **Memory**: ~1.6 GB
- **Frequency**: Once per tape-out

**Result**: Hardware Script feels like **writing software**, not designing hardware.

---

## Summary

The Lazy Realization architecture achieves **instantaneous development** through:

1. **Netlist-First**: Logical validation without physical realization
2. **Structural Proxying**: Store instances, not atoms
3. **On-Demand Realization**: Only Bit-Blit what's needed
4. **Command Separation**: `check` → `simulate` → `build --physical` → `export`
5. **Lazy Voxel Grid**: Empty until queried

**The Vision**: A compiler that feels like a text editor, with the power to design 100-billion transistor chips.

---

**Document Status**: Instantaneous Development Loop Blueprint  
**Last Updated**: April 2026  
**Part of**: Hardware Script v0.1.6 Documentation Suite

