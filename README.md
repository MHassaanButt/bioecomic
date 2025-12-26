See config.yaml and fill data CSVs. Run the notebook and use the corrected validator to compare production-only metrics.

## Demo mode
- Set `demo_mode: true` in `config.yaml` to auto-fill blanks **from `data/demo_defaults.yaml`**.
- You can edit `demo_defaults.yaml` to change demo values.
- With `demo_mode:false`, **all blanks are errors** (no fabrication).


# Phase 1 — What’s implemented, why errors are large, and how to fix

This note is a short, plain‑English handoff to explain the current notebook, the source of large validation errors, and exactly what data we need to calibrate it.

---

## 1) What the notebook implements (Phase 1)

**Goal:** single‑year linear program (LP) that chooses crop areas, livestock activity levels, hired labor, and off‑farm labor to **maximize net farm income** subject to land, labor, and calorie constraints.

**Decision variables (per household class _h_):**
- `area[h,c]` — cropped hectares by crop _c_
- `cons[h,c]`, `sold[h,c]`, `stored[h,c]` — crop allocation of production
- `live_units[h,l]` — livestock activity “units” (proxies sellable animals/products)
- `hired[h]` — hired labor days
- `off_farm[h]` — off‑farm labor days

**Objective (maximize net income):**

a.k.a. minimize costs – revenues in `linprog` form.

	net_income = (crop revenue + livestock revenue + off‑farm wages)

	            – (crop input costs + livestock input costs + hired labor cost)

**Key constraints:**
- **Production balance (per crop):** `yield_per_ha * area = cons + sold + stored`
- **Land (per class):** `sum_c area[h,c] ≤ land_available[h]`
- **Labor (per class):** `crop_labor + livestock_labor + off_farm – hired ≤ family_labor_endowment`
- **Hired cap:** `hired[h] ≤ max_hired_labor[h]`
- **Calorie floor:** `Σ_c cons[h,c] * calorie_per_kg ≥ AE[h] * kcal_min * days_per_year`

This is a standard farm household LP. The **math and code are correct** for this scope.

---

## 2) What the validator compares (and why errors are large)

The validator compares **model outputs** to **observed totals** from the survey tables.

### Where the mismatch happens
1) **Scale mismatch** (biggest driver): current data in `crops.csv / livestock.csv / prices.csv / households.csv` are **placeholders** (small yields, prices, land, and wage). Observed Table 10/11 values are in **thousands of Birr**. The model therefore produces outcomes in the **hundreds** → high RMSE/MAPE.

2) **Category mismatch in observed vs model:**
   - **Table 10 “Off‑farm income (C)”** includes **many exogenous items** (gov transfer, PSNP, private transfers, business income, rent‑out). The LP only models **line 7: off‑farm labor** as a decision. If the validator uses **C‑total** instead of **line 7**, the error explodes.
   - **Table 11 crop “A”** includes **consumption purchases** (*crop purchase*, *coffee & oil*) that are **not production inputs**. Counting them as “input cost” overstates costs in validation.
   - **Table 11 livestock “B”** includes **animal products purchase** (consumption). We should exclude it from production inputs.

### Correct mapping for production‑only validation
- **Income side (Table 10):**  
  `rev_crops` = **A total**; `rev_livestock` = **B total**; `off_farm_labor` = **line 7 only**.  
  Treat **(8)–(12)** as **exogenous incomes** (not decisions).

- **Cost side (Table 11):**  
  `cost_crop_inputs` = **sum(lines 1–4)** (rainfed + irrigated + homestead + perennial seedling).  
  `cost_livestock` = **animal feed + animal health/breeding**.  
  `cost_hired_labor` = **hired labor (crop + animal)**.  
  **Exclude**: *crop purchase*, *coffee & oil*, *animal products purchase*, and all **non‑farm expense (C)** from production costs (they’re consumption/transfer items).  
  **Livestock purchase** (line 8) can be treated as investment; keep it **out** of production cost unless we toggle it in config.

When we use the corrected mapping and realistic coefficients, errors should drop sharply.

---

## 3) Why the math is correct but totals differ from Table 10/11

- The LP models **production choices** and **off‑farm labor**.  
- Table 10 includes **exogenous** items (transfers, PSNP, remittances, business income, rent‐out) that are **not determined by production decisions in Phase 1**. We separate these as **exogenous_income** to compare apples‑to‑apples.
- Table 11 includes **household consumption purchases** (e.g., “crop purchase”, “coffee & oil”, “animal products purchase”) — not production inputs — so they should not be used to validate input costs.

---

## 4) What data we still need (to calibrate)
Please provide the following **per household class** and **per activity** so outputs land on the same scale (Birr/HH/year):

### Household class (HFR, HMR, MFIR, MMR, MMIR, MFR)
- `adult_equiv`, `labor_endowment (days)`, `land_available (ha)`, `max_hired_labor (days)`  
- (Optional) `max_off_farm_days`

### Crops (by crop; zone‑specific if needed)
- `calorie_per_kg`, `yield_per_ha (kg/ha)`, `price_sale (Birr/kg)`
- per‑ha input costs: `seed`, `fertilizer`, `chemicals`
- `labor_req_per_ha (days/ha)`
- (Optional) area bounds per class

### Livestock (by activity/product)
- `price_sale (Birr/unit)`, `feed_cost_per_unit`, `vet_cost_per_unit`, `labor_req_per_unit`
- (Optional) activity bounds

### Prices
- `wage (Birr/day)` (crucial for off‑farm vs hired labor trade‑off)

### Exogenous incomes (Table 10 lines 8–12, **already available**)
- Gov transfer (non‑PSNP), Own business, Private transfers/remittances, PSNP, Rent out

---

## 5) What we can show now (to discuss)
- **Model is mathematically sound** and replicable (see equations).  
- **Validation errors are expected** because we’re comparing to totals that include **exogenous and consumption** items and we currently use **placeholder coefficients**.
- We have a **correct production‑only validation mapping** ready (separates exogenous incomes).  
- Once realistic coefficients are inserted, we can target small errors (e.g., <10–15% MAPE) on production metrics.

---

## 6) Immediate fixes to reduce errors quickly
1) **Use the corrected mapping** (production‑only) for validation.
2) **Replace placeholders** with realistic Birr‑scale coefficients:  
   - Crop: raise `price_sale`, `yield_per_ha`; set per‑ha input costs and labor.  
   - Wage: set a realistic `wage` (Birr/day).  
   - Land: set class‑specific `land_available`.
3) Keep exogenous incomes as **reported** (we already ingest Table 10 lines 8–12).

---

## 7) Clear checklist for the confirmation
- Confirm the **validation mapping** above is consistent with how the survey tables should be used in a production model.
- Provide the **coefficient tables** (crops, livestock, wages, land, labor endowment) for the six classes.
- Decide whether **livestock purchase** should be treated as a cost (recurrent) or as **investment** (default: exclude).

---

## 8) References to the code
- **LP build & solve:** `solve_single_year_lp(...)` — objective and constraints exactly as listed.
- **Validator:** merges model outputs with observed (per class) and reports per‑metric diffs + RMSE/MAPE.
- **Observed file:** `data/observed_table10_11.csv` (or the production‑only version if you choose that mapping).

---

### Bottom line
- The **model is correct** for Phase 1 (single‑year production LP).  
- **Large errors** come from **(a)** placeholders vs. real Birr‑scale coefficients, and **(b)** **mixing exogenous/consumption items** from the survey tables into production validation.  
- With the corrected mapping and real coefficients, we should be able to match production‑side aggregates closely and then add exogenous incomes to match household totals for context.

# Phase 2A — Multi-year Bioeconomic LP (NPV)

This bundle extends Phase 1 to a **T-year** horizon with discounting and time-varying scenario multipliers.
No inter-year storage or herd dynamics yet (that comes in Phase 2B/2C). This is Phase 2A: replicate Phase 1 across years with NPV.

## Edit these files
- `data/households.csv`, `data/crops.csv`, `data/livestock.csv`, `data/prices.csv`  (your coefficients)
- `data/scenario_multi.json`  (T, discount_rate, and yearly multipliers arrays)

## Run
Open `Bioeconomic_LP_Phase2A_MultiYear.ipynb` and run cells top-to-bottom.
Outputs are written to `outputs/` as JSON/CSV.

## Validation
If `data/observed_prod_only.csv` exists, the notebook validates **year 1 (t=1)** production-only metrics against the observed table mapping.

# Phase 2B — Multi-year Bioeconomic LP with Inter-year Coupling

Adds **dynamics**:
- Crop storage carryover (inventory balance across years)
- Livestock herd dynamics (herd stock evolves with reproduction and sales)

## New inputs
### crops.csv new columns
- storage_cost_per_kg
- max_storage_kg
- spoilage_rate

### livestock.csv new columns
- reproduction_rate
- max_herd

### initial conditions
- initial_stocks.csv
- initial_herd.csv

## Run
Open `Bioeconomic_LP_Phase2B_Dynamics.ipynb` and run top-to-bottom.

# Phase 3 — Scenario + Policy Runner

Runs Phase 2B (dynamic LP) across scenarios and compares **no-policy** vs **fert subsidy** policy.

Scenarios:
- baseline
- drought (years 2–4 yield multiplier = 0.85)
- fert_shock (fert price multiplier = 1.5)

Policy:
- fert_subsidy: effective fert multiplier = fert_multiplier * (1 - subsidy_rate)

Outputs (generated after running):
- outputs/phase3_scenario_household_npv.csv
- outputs/phase3_scenario_totals.csv
- outputs/phase3_policy_impacts.csv
- outputs/phase3_yearly_all.csv