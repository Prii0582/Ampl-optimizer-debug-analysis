# Ampl-optimizer-debug-analysis
Root cause analysis of AMPL charge mix optimization failures for ductile iron grades — Python, pandas, foundry domain
# Add/Dil Optimizer Debugging — AMPL Charge Mix Analysis

> **Root cause analysis and fix documentation for AI-driven charge mix optimization failures**  
> Built at **NowPurchase / MetalCloud MIS** · Python · AMPL · pandas · data analysis

---

## What This Project Does

MetalCloud MIS includes an **Add/Dil (Addition/Dilution) Optimizer** — an AMPL-based mathematical model that recommends what raw materials (FeSiMg, CRC scrap, GPC, etc.) to add to a furnace to hit target chemistry for a specific ductile iron (DI) or grey cast iron (GCI) grade.

When the optimizer returns `is_optimized = false`, foundry customers get no suggestion — and don't know why. This project involved systematically diagnosing **why specific grades and readings were failing**, identifying root causes in the AMPL model configuration, and recommending precise fixes that recovered those suggestions.

---

## Customers Analysed

| Customer | Grades Investigated | Ladle Size | Infeasible Readings |
|---|---|---|---|
| Vishwakarma Auto Components | 19361, 19362, 19363, 19717, 19725, 19744, 19756, 19765 | 1000 kg + 500 kg | 48 readings |
| Mane Foundry | 26118, 29575, 29577 | 330 kg furnace | 90 / 100 readings |
| NIF ISPAT | Multiple DI grades | Various | Analysed |

---

## Root Causes Found

### Root Cause 1 — FeSiMg Minimum Bound Too High (All Customers)

The nodularization formula computes FeSiMg requirement based on bath sulphur content:

```
FeSiMg_required ≈ (S_initial × ladle_weight × K) / FeSiMg_Mg_content
```

At low sulphur values (S = 0.007–0.016%), the formula outputs **~6.3 kg** needed for a 1000 kg ladle. But the AMPL model had `FeSiMg.lb = 9 kg` — a hard minimum that exceeded the actual requirement, making the constraint mathematically infeasible.

**Fix:** Recalculated safe bounds from actual observed S values in production data:
- 1000 kg ladle: `6–13 kg` (not 9–14 kg)
- 500 kg ladle: `3–6 kg` (separate bounds required — same grade, different ladle)

---

### Root Cause 2 — Broken Furnace Capacity Constraint (Mane Foundry)

For Mane Foundry (330 kg furnace cap):

```
s_crc_scrap.lb = 120 kg   (minimum CRC scrap)
s_heel = 300 kg            (fixed heel weight, always present)

120 + 300 = 420 kg  >  330 kg furnace cap
```

This means **every single heat** was structurally infeasible before the optimizer even ran — the minimum CRC scrap plus the fixed heel weight already exceeded the furnace. 90 out of 100 readings failed for this reason.

**Fix:** Remove the `s_crc_scrap` minimum cap (set to 0). The furnace cap constraint itself handles the upper bound correctly.

---

### Root Cause 3 — Sulphur Contamination (Unresolvable Case)

One heat at Mane Foundry (MF-39 B14, Grade 29577) had S_initial = 0.091% — far above any recovery threshold. This heat was flagged as unresolvable by the optimizer regardless of bounds configuration and excluded from the fix scope.

---

## Analysis Workflow

```
Customer reports: "optimizer not generating suggestions"
          │
          ▼
Export spectrometer CSV → filter is_optimized = false
          │
          ▼
Load AMPL .mod files for failing grades
          │
          ▼
Check: does nodularization formula output < FeSiMg.lb?
          │
          ▼
Recalculate required FeSiMg range from actual S_initial distribution
          │
          ▼
Check: does s_crc_scrap.lb + s_heel > furnace cap?
          │
          ▼
Recommend specific bound changes → verify against test readings
          │
          ▼
Document fix with plain-language summary for engineering team
```

---

## Key Technical Concepts

| Term | Meaning |
|---|---|
| `is_optimized` | Boolean flag — `true` if optimizer found a valid solution |
| `infeasible_elements_raw` | Which specific constraints caused the failure |
| `i_Mgfinal` | AMPL penalty variable — activates when FeSiMg constraint is violated |
| `s_heel` | Fixed residual metal in furnace before charging — not a variable |
| `FeSiMg` | Ferro Silicon Magnesium — the nodularising agent for ductile iron |
| `GPC` | Graphite Petroleum Coke — carbon recarburiser |
| `S_initial` | Bath sulphur reading from spectrometer before nodularisation |

---

## Files / Tools Used

```
spectrometer_additiondilution_*.csv   — export of spectro readings with is_optimized flag
AMPL .mod files                       — optimizer model per grade
pandas                                — data filtering and distribution analysis
```

---

## Business Impact

- **30+ customers** had optimization failures that blocked AI suggestions — a direct revenue and trust risk
- Root causes identified and documented with recommended fixes
- After fixes applied: **suggestion generation restored** for all analysed infeasible readings
- Framework reusable for any new customer grade showing optimization failures

---

## Skills Demonstrated

`AMPL` `Mathematical optimization` `Python` `pandas` `Data analysis` `Root cause analysis` `Foundry metallurgy` `Ductile iron` `Constraint debugging` `Technical documentation`
