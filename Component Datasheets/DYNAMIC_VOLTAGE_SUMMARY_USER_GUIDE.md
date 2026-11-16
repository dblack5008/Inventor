# Dynamic Voltage Summary - User Guide

## Problem Solved

✅ **Fixed merged cell error** - The new file has NO merged cells, so you can edit any cell without getting that error

✅ **Made summary dynamic** - Voltage requirements now automatically update when you change components or quantities in the Switch Matrix sheet

## How It Works

### Step-by-Step Usage:

1. **Open the file**: `RF Stage 1 Heat Analysis (DYNAMIC).xlsx`

2. **Go to "Switch Matrix" sheet** (2nd tab)
   - Rows 11-70 are available for components
   - Column A: Select component name (from dropdown or type from Components catalog)
   - Column B: Enter quantity

3. **Go to "Voltage Summary" sheet** (1st tab)
   - The table automatically shows which voltages are needed
   - Current values update in real-time
   - Only voltages with current >0.0000 A are needed for your enclosure

## What the Voltage Summary Shows

### Dynamic Table Columns:

| Column | Description |
|--------|-------------|
| **Voltage Type** | All possible voltages (+3V, +3.3V, +5V, +12V, +15V, +28V, -3V, -3.3V, -5V) |
| **Total Current (A)** | Automatically calculated from your Switch Matrix components |
| **Details** | Shows "NEEDED" or "Not currently used" |

### Summary Statistics:

- **Power supplies needed**: Auto-counts how many different voltage types you need
- **Total power**: Calculates total watts across all voltages

## Current Switch Matrix Setup

Based on your current components (8x TTL-1SP8T-183, 8x BW-S0.5W2+, 8x RFLT8WDC40G):

| Voltage | Current Needed | Components |
|---------|----------------|------------|
| **+5V** | 1.6000 A | TTL-1SP8T-183 (x8) |
| **-5V** | 0.2000 A | TTL-1SP8T-183 (x8) |

All other voltages show 0.0000 A (not needed for this enclosure).

## Understanding Dual-Voltage Components

Some RF components need BOTH a positive AND negative voltage:

### Example: TTL-1SP8T-183
- **Rail 1**: +5V @ 0.2A per unit (digital logic)
- **Rail 2**: -5V @ 0.025A per unit (analog circuitry)

When you have 8 units:
- +5V needs: 8 × 0.2A = 1.6A total
- -5V needs: 8 × 0.025A = 0.2A total

**You need BOTH power supplies!**

## How the Formulas Work

For each voltage type, the sheet uses this logic:

```
For each component in Switch Matrix (rows 11-70):
  1. Get the component name and quantity
  2. Look up that component in the Components catalog
  3. Check if Rail 1 or Rail 2 uses this voltage type
  4. If yes: multiply (quantity × current per unit)
  5. Sum all matching components
```

The formula structure:
- `SUMPRODUCT`: Multiplies quantities by currents
- `INDEX/MATCH`: Looks up component data from Components sheet
- Checks both Rail 1 (column K) and Rail 2 (column P) for voltage types
- Sums currents from columns N (Rail 1) and S (Rail 2)

## Testing the Dynamic Behavior

### Try This:

1. **Open the file** and go to "Switch Matrix" sheet
2. **Change quantity** of TTL-1SP8T-183 from 8 to 4
3. **Go back to "Voltage Summary"**
4. **Watch the values update**:
   - +5V current changes from 1.6A to 0.8A
   - -5V current changes from 0.2A to 0.1A

5. **Add a new component**:
   - In Switch Matrix, row 14, column A: Type "ZFL-1000+"
   - Column B: Enter quantity 2
   - Go to Voltage Summary
   - Now +15V shows 0.2100 A (2 × 0.105A)

### Another Test - Add a 12V Fan:

1. Switch Matrix, any empty row:
   - Column A: "OD3010-12HB fan"
   - Column B: 1
2. Voltage Summary now shows:
   - +12V: 0.1500 A

## Important Notes

### Power Supply Sizing:
- ⚠️ Add 10-20% margin to calculated currents
- ⚠️ Check voltage tolerances in component datasheets
- ⚠️ Some components have narrow voltage ranges (e.g., ±0.5V)

### File Maintenance:
- ✅ NO merged cells - edit freely without errors
- ✅ Original sheets (Switch Matrix, Components) unchanged
- ✅ All formulas reference the Components catalog
- ✅ Add components to Components sheet, then use them in Switch Matrix

### Limitations:
- Formulas reference Components sheet rows 8-107
- If you add components beyond row 107, extend formula ranges
- If you rename sheets, formulas will break (use with caution)

## Common Questions

**Q: Why does it show 0.0000 for most voltages?**
A: Because those voltages aren't used by any components in your Switch Matrix. Only install power supplies for voltages showing >0 current.

**Q: Can I add more rows to Switch Matrix?**
A: The formulas already cover rows 11-70 (60 rows). If you need more, you can extend the formulas.

**Q: What if I want to add a new voltage type?**
A: Add it to the Components sheet for relevant components. You may need to manually add a row to the Voltage Summary for that new voltage.

**Q: Do I need both +5V and -5V?**
A: YES! Some components require both. Check the Components sheet Rail 1 and Rail 2 columns.

## Files Comparison

| File | Description | Use Case |
|------|-------------|----------|
| **RF Stage 1 Heat Analysis.xlsx** | Original file | Has your baseline heat analysis |
| **RF Stage 1 Heat Analysis (IMPROVED).xlsx** | First attempt | Had merged cells (caused errors) - AVOID |
| **RF Stage 1 Heat Analysis (DYNAMIC).xlsx** | Latest version | Use this! Dynamic + no merged cells ✓ |

## Troubleshooting

**Error: "We can't do that to a merged cell"**
→ You're using the old IMPROVED file. Switch to DYNAMIC file.

**Formulas show #REF! error**
→ Sheet names may have been changed. Restore original names: "Switch Matrix", "Components", "Voltage Summary"

**Values don't update**
→ Press F9 to force recalculation in Excel
→ Make sure "Automatic Calculation" is enabled (Excel Options → Formulas)

**A voltage I need shows 0.0000**
→ Check that component is listed in Switch Matrix with quantity >0
→ Verify component name exactly matches Components sheet (case-sensitive)

## Next Steps

1. ✅ Use the DYNAMIC file going forward
2. ✅ Add/remove components in Switch Matrix as your design changes
3. ✅ Use Voltage Summary to spec your power supplies
4. ✅ Add 15-20% margin for safety
5. ✅ Check component datasheets for voltage tolerances

---

**Created**: October 26, 2026  
**Version**: Dynamic v1.0  
**Status**: Production Ready ✓
