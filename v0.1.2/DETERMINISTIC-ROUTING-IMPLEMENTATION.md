# Hardware Script v0.1.2 - Deterministic Routing Implementation

**Document Type**: Rust Implementation Guide  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## The Critical Problem: Non-Deterministic Data Structures

### What Happens Without Determinism

**The Problem**:

In Rust, standard `HashMap` and `HashSet` iterate in a **randomized order** for security reasons (to prevent hash collision attacks).

If your water algorithm loops over a standard `HashSet` to decide where to flow next, **the route will change every time you compile!**

**Example of Non-Deterministic Behavior**:

```rust
use std::collections::HashSet;

let mut frontier = HashSet::new();
frontier.insert((1, 10, 10));
frontier.insert((1, 10, 11));
frontier.insert((1, 11, 10));

// Iteration order is RANDOM!
for cell in frontier.iter() {
    // Process cell...
    // Different order each run = different routes!
}
```

**Result**:
- Run 1: Routes through (1, 10, 11) first → Path A
- Run 2: Routes through (1, 11, 10) first → Path B
- Run 3: Routes through (1, 10, 10) first → Path C

**Same input, different output every time!**

This breaks the fundamental promise of Hardware Script: **deterministic compilation**.

---

## The Solution: Stable Data Structures

### Use VecDeque Instead of HashSet

**VecDeque** is a First-In-First-Out (FIFO) queue that guarantees:
- Elements are processed in the order they were added
- Same input always produces same output
- Perfect for breadth-first search (BFS) algorithms

### Implementation in Rust

```rust
use std::collections::VecDeque;

// WRONG: Non-deterministic
let mut frontier = HashSet::new();

// RIGHT: Deterministic
let mut frontier = VecDeque::new();
```

### Complete Water Algorithm with VecDeque

```rust
use std::collections::{VecDeque, HashSet};

fn route_water_deterministic(
    start: (usize, usize, usize),
    end: (usize, usize, usize),
    grid: &Grid
) -> Option<Vec<(usize, usize, usize)>> {
    // VecDeque for processing order (deterministic)
    let mut frontier = VecDeque::new();
    frontier.push_back(start);
    
    // HashSet for visited tracking (order doesn't matter here)
    let mut visited = HashSet::new();
    visited.insert(start);
    
    // HashMap for path reconstruction
    let mut came_from = HashMap::new();
    
    while let Some(current) = frontier.pop_front() {  // FIFO order!
        if current == end {
            return Some(reconstruct_path(came_from, start, end));
        }
        
        // Get neighbors in STABLE order
        let neighbors = get_neighbors_stable(current, grid);
        
        for neighbor in neighbors {
            if !visited.contains(&neighbor) {
                visited.insert(neighbor);
                came_from.insert(neighbor, current);
                frontier.push_back(neighbor);  // Add to back of queue
            }
        }
    }
    
    None  // No path found
}
```

### Key Points

1. **VecDeque for frontier**: Guarantees FIFO processing order
2. **HashSet for visited**: Order doesn't matter, just membership testing
3. **Stable neighbor ordering**: Even the neighbor generation must be deterministic

---

## Ensuring Neighbor Order Stability

### The Problem

Even with VecDeque, if you generate neighbors in random order, you'll get non-deterministic results.

**Bad (Non-Deterministic)**:
```rust
fn get_neighbors(cell: (usize, usize, usize)) -> Vec<(usize, usize, usize)> {
    let mut neighbors = HashSet::new();
    // Add neighbors...
    neighbors.into_iter().collect()  // Random order!
}
```

### The Solution

Always generate neighbors in a **fixed, predictable order**.

**Good (Deterministic)**:
```rust
fn get_neighbors_stable(
    cell: (usize, usize, usize),
    grid: &Grid
) -> Vec<(usize, usize, usize)> {
    let (z, x, y) = cell;
    let mut neighbors = Vec::new();
    
    // FIXED ORDER: North, South, East, West, Up, Down
    
    // North (Y+1)
    if y + 1 < grid.height {
        neighbors.push((z, x, y + 1));
    }
    
    // South (Y-1)
    if y > 0 {
        neighbors.push((z, x, y - 1));
    }
    
    // East (X+1)
    if x + 1 < grid.width {
        neighbors.push((z, x + 1, y));
    }
    
    // West (X-1)
    if x > 0 {
        neighbors.push((z, x - 1, y));
    }
    
    // Up (Z+1)
    if z + 1 < grid.depth {
        neighbors.push((z + 1, x, y));
    }
    
    // Down (Z-1)
    if z > 0 {
        neighbors.push((z - 1, x, y));
    }
    
    neighbors  // Always in same order!
}
```

---

## Why This Matters

### Determinism = Trust

**With deterministic routing**:
- Same .hw file always produces same Gerber output
- Version control diffs are meaningful
- Debugging is possible (reproducible bugs)
- CI/CD works reliably
- Users can trust the compiler

**Without deterministic routing**:
- Every compile produces different board
- Can't reproduce bugs
- Can't verify changes
- Users lose trust in the tool

### Git-Friendly Compilation

**Deterministic**:
```bash
$ git diff board.gtl
- X050000Y050000D01*
+ X050000Y060000D01*

Clear: Route moved 1mm North
```

**Non-Deterministic**:
```bash
$ git diff board.gtl
Entire file changed!
Can't tell what actually changed vs. random variation
```

---

## Implementation Checklist

### Phase 1: Core Determinism (Critical)

- [x] Replace HashSet with VecDeque for frontier
- [x] Implement stable neighbor ordering
- [ ] Test: Compile same file 100 times, verify identical output
- [ ] Add determinism tests to CI/CD

### Phase 2: Verification (High Priority)

- [ ] Add hash of output files to build log
- [ ] Detect non-deterministic behavior automatically
- [ ] Warn if output changes without input changes

### Phase 3: Documentation (Next Sprint)

- [ ] Document determinism guarantee in language spec
- [ ] Add examples showing reproducible builds
- [ ] Create troubleshooting guide for determinism issues

---

## Testing Determinism

### Simple Test

```rust
#[test]
fn test_deterministic_routing() {
    let hw_source = r#"
        space "TestBoard" {
            dimensions: 100mm × 100mm × 2mm
            grid: 100 × 100 × 2
        }
        
        Battery @ [1, 10, 10]
        LED @ [1, 90, 90]
        
        Battery.Plus -> LED.Anode
    "#;
    
    // Compile 10 times
    let mut outputs = Vec::new();
    for _ in 0..10 {
        let output = compile(hw_source);
        outputs.push(output);
    }
    
    // All outputs must be identical
    for i in 1..outputs.len() {
        assert_eq!(outputs[0], outputs[i], 
            "Non-deterministic output detected!");
    }
}
```

### Hash Verification

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

fn hash_output(gerber: &str) -> u64 {
    let mut hasher = DefaultHasher::new();
    gerber.hash(&mut hasher);
    hasher.finish()
}

#[test]
fn test_output_hash_stability() {
    let hw_source = load_test_file("test_board.hw");
    
    let output1 = compile(hw_source);
    let output2 = compile(hw_source);
    
    let hash1 = hash_output(&output1.gerber);
    let hash2 = hash_output(&output2.gerber);
    
    assert_eq!(hash1, hash2, 
        "Output hash changed between compilations!");
}
```

---

## Key Takeaways

1. **Use VecDeque, not HashSet** - For frontier/queue in routing algorithm

2. **Fixed neighbor order** - Always generate neighbors in same sequence

3. **Determinism is critical** - Same input must always produce same output

4. **Test extensively** - Compile same file multiple times, verify identical output

5. **Git-friendly** - Deterministic output makes version control meaningful

6. **Trust through reproducibility** - Users can rely on consistent behavior

---

## Summary

**The Problem**: HashSet/HashMap have random iteration order in Rust

**The Solution**: Use VecDeque for FIFO processing order

**The Benefit**: Guaranteed deterministic routing - same input always produces same output

**The Result**: Trustworthy, reproducible, version-control-friendly compilation

---

**Document Status**: Rust Implementation Guide  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite

