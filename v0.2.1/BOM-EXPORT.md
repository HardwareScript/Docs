# Bill of Materials (BOM) Export

**Version**: v0.3.0  
**Status**: Production Ready  
**Philosophy**: Dual-format support for PCB assembly and ASIC fabrication tracking

---

## Overview

HardwareScript generates Bill of Materials (BOM) in CSV format with a **dual-table structure**:

1. **Discrete Component BOM** - For PCB/SMT assembly (Digi-Key, Mouser part orders)
2. **Material Usage Report** - For ASIC/semiconductor foundry fabrication tracking

---

## Output Format

### Top Section: Discrete Components

```csv
Reference,Type,Value,Package,Manufacturer,Part Number,Description,Quantity
Wafer,Substrate,0.02x0.01x0.00mm,,,,,1
```

Used for:
- PCB assembly (discrete resistors, ICs, connectors)
- Monolithic die footprint (for ASIC designs)

### Middle Section: Material Usage (Fabrication)

```csv
# MATERIAL USAGE (Fabrication)
Reference,Type,Material,Layer,Area_nm2,Volume_nm3
Resistor_Body_A,Pour,Polysilicon,polyres (z:0nm),2000000,400000000
Resistor_Body_B,Pour,Polysilicon,polyres (z:0nm),2000000,400000000
Contact_A,Pour,Aluminum,d1 (z:500nm),160000,134400000
Contact_B,Pour,Aluminum,d1 (z:500nm),160000,134400000
In_Pad,Pour,Aluminum,d1 (z:500nm),1000000,840000000
Out_Pad,Pour,Aluminum,d1 (z:500nm),1000000,840000000
```

**Columns:**
- **Reference** - Pour name from layout
- **Type** - Always "Pour" for material deposits
- **Material** - Material name (Polysilicon, Aluminum, Tungsten, etc.)
- **Layer** - Stackup layer name with Z-coordinate (e.g., `polyres (z:0nm)`)
- **Area_nm2** - Pour area in square nanometers
- **Volume_nm3** - Pour volume in cubic nanometers (Area × Thickness)

### Bottom Section: Aggregated Material Totals

```csv
# AGGREGATED MATERIAL TOTALS (Foundry Fabrication Summary)
Material,Total_Area_nm2,Total_Volume_nm3,Layer_Count,Coverage_Percentage
Aluminum,2320000,1948800000,4,1.2%
Polysilicon,4000000,800000000,2,2.0%
```

**Columns:**
- **Material** - Material name
- **Total_Area_nm2** - Sum of all pour areas for this material
- **Total_Volume_nm3** - Sum of all pour volumes for this material
- **Layer_Count** - Number of pours using this material
- **Coverage_Percentage** - (Total Area / Die Area) × 100%

---

## Filtering Rules (v0.3.0)

### Virtual Entities Excluded

The following pours are **automatically filtered out** from material usage:

1. **Air material** - Virtual/phantom pours for bookkeeping
2. **Zero volume** - Pours with no physical volume
3. **Zero area** - Pours with no physical area

**Example:**
```hw
# This virtual pour for device terminal binding is NOT in BOM
add pour(Air) named Resistor_Bulk:
    device: R1.BULK
    net: GND
    dimensions: 1nm by 1nm  # Virtual, filtered out
```

---

## Layer Name Resolution

Layer names are resolved from the **stackup definition** by matching Z-coordinates:

```hw
profile Resistor_3D:
    stackup:
        polyres: [material: Polysilicon, thickness: 200nm, routable: true]
        d1:      [material: Silicon_Dioxide, thickness: 300nm, routable: false]
        metal1:  [material: Aluminum, thickness: 840nm, routable: true]
```

**Output:**
- Pour at Z=0nm → `polyres (z:0nm)`
- Pour at Z=500nm → `d1 (z:500nm)` or `metal1 (z:500nm)` depending on layer match

**Fallback:** If no layer matches, shows raw Z-coordinate: `z:500nm`

---

## Coverage Percentage Calculation

```
Coverage % = (Total Material Area / Die Area) × 100%

Example:
Die Area = 20μm × 10μm = 200,000,000 nm²
Aluminum Total Area = 2,320,000 nm²
Coverage = (2,320,000 / 200,000,000) × 100% = 1.2%
```

**Used for:**
- Metal density compliance (CMP requirements)
- DRC validation
- Foundry fabrication cost estimation

---

## Use Cases

### PCB Assembly
- Order discrete components from distributors
- Track wafer/substrate footprint
- Generate assembly instructions

### ASIC Fabrication
- Calculate sputtering target consumption
- Verify metal density rules for CMP
- Estimate deposition time and material costs
- Track layer-specific material usage

---

## Design Principles

1. **Zero Hardcoding** - All layer names from stackup, no magic strings
2. **No Fallbacks** - If data unavailable, shows raw values transparently
3. **Geometry-Driven** - All volumes calculated from bounding boxes
4. **Clean Filtering** - Simple boolean logic (Air OR volume==0 OR area==0)
5. **Sorted Output** - Materials alphabetically sorted for consistency

---

## Mathematical Accuracy

All volume calculations are **100% exact**:

```
Volume = Area × Thickness

Polysilicon Example:
Area = 2.0μm × 1.0μm = 2,000,000 nm²
Thickness = 200nm (from stackup)
Volume = 2,000,000 nm² × 200nm = 400,000,000 nm³ ✓
```

---

## Example Output

```csv
Reference,Type,Value,Package,Manufacturer,Part Number,Description,Quantity
Wafer,Substrate,0.02x0.01x0.00mm,,,,,1

# MATERIAL USAGE (Fabrication)
Reference,Type,Material,Layer,Area_nm2,Volume_nm3
Resistor_Body_A,Pour,Polysilicon,polyres (z:0nm),2000000,400000000
Resistor_Body_B,Pour,Polysilicon,polyres (z:0nm),2000000,400000000
Contact_A,Pour,Aluminum,d1 (z:500nm),160000,134400000
Contact_B,Pour,Aluminum,d1 (z:500nm),160000,134400000
In_Pad,Pour,Aluminum,d1 (z:500nm),1000000,840000000
Out_Pad,Pour,Aluminum,d1 (z:500nm),1000000,840000000

# AGGREGATED MATERIAL TOTALS (Foundry Fabrication Summary)
Material,Total_Area_nm2,Total_Volume_nm3,Layer_Count,Coverage_Percentage
Aluminum,2320000,1948800000,4,1.2%
Polysilicon,4000000,800000000,2,2.0%
```

---

## Summary

The BOM export provides:
- ✅ Discrete component listing for procurement
- ✅ Layer-specific material usage with named layers
- ✅ Aggregated totals for foundry fabrication
- ✅ Coverage percentages for DRC/CMP compliance
- ✅ 100% mathematically exact volume calculations
- ✅ Clean filtering of virtual/bookkeeping entities

**Status**: Production-ready for foundry handoff and assembly manufacturing.
