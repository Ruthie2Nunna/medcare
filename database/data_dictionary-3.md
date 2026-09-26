# MedCare E1 + P1 — Data Dictionary

**Owner:** Ayush · **Reference date ("today"):** 2026-09-23 · **Demand/sales history:** 2025-09-23 → 2026-09-22 (365 days)

## 1. What is in the package

| Path | Purpose |
|---|---|
| `data/demand.csv` | 65,700 rows — daily demand per SKU × warehouse (P1 forecasting input) |
| `data/inventory.csv` | 180 rows — current inventory state per SKU × warehouse (E1 monitoring input) |
| `data/batches.csv` | 428 rows — batch-level stock and expiry (E1 transfer / near-expiry / expired-stock input) |
| `data/sales.csv` | 65,700 rows — daily units sold, price and revenue per SKU × warehouse |
| `database/medcare.db` | SQLite version — same 4 tables + 7 indexes |
| `database/medcare_mysql.sql` | MySQL version — same 4 tables + 7 indexes, as a `.sql` dump (`CREATE DATABASE`, `ENGINE=InnoDB`, batched `INSERT`s) |
| `database/schema.sql` | SQLite DDL used to build `medcare.db` |
| `documentation/validation_report.md` | Result of the validation checklist (53/53 passing) |
| `scripts/` *(optional)* | `generate_data.py` → `add_sales.py` → `export_mysql.py` (all seeded, reproducible), and `validate.py` |

**Not included on purpose** (calculated by the application): forecasts, alerts, recommendations, stockout risk, replenishment, promotions, seasonality tables, current-stock summary, model metrics.

**Note on table count:** the original spec calls for exactly 3 tables (demand, inventory, batches). This package has 4 — `sales` was added afterward at the team's request. Worth mentioning proactively if judges ask.

## 2. Master lists (used identically in all four datasets)

**Warehouses** — 6 IDs. `W002` = Madurai and `W004` = Mumbai are fixed by the demo story. The other city names are documentation-only labels and are *not* stored in any dataset.

| ID | City | ID | City |
|---|---|---|---|
| W001 | Delhi | W004 | Mumbai *(excess demo)* |
| W002 | Madurai *(shortage demo)* | W005 | Kolkata |
| W003 | Bengaluru | W006 | Hyderabad |

**SKUs** — 30 IDs (`M001`–`M030`). Names are documentation-only labels; the CSVs carry `sku_id` only.

| Group (drives `season_flag`) | SKUs |
|---|---|
| Fever — seasons 15 Aug–31 Oct and 1 Dec–15 Feb | M001 Paracetamol 500 mg · M002 Ibuprofen 400 mg |
| Winter flu / respiratory — 20 Nov–20 Feb | M003 Cetirizine 10 mg · M004 Cough Syrup 100 ml · M005 Azithromycin 500 mg · M006 Vitamin C 500 mg · M007 Xylometazoline Nasal Spray · M008 Salbutamol Inhaler |
| Monsoon — 15 Jun–30 Sep | M009 ORS Sachets · M010 Loperamide 2 mg · M011 Artemether-Lumefantrine · M012 Doxycycline 100 mg · M013 Clotrimazole Cream |
| Non-seasonal (chronic / general) | M014 Metformin · M015 Amlodipine · M016 Atorvastatin · M017 Losartan · M018 Levothyroxine · M019 Omeprazole · M020 Pantoprazole · M021 Aspirin 75 mg · M022 Insulin Glargine · M023 Amoxicillin · M024 Diclofenac · M025 Vitamin D3 · M026 Multivitamin · M027 Calcium + D3 · M028 Antacid Gel · M029 Povidone-Iodine · M030 Hydrocortisone Cream |

## 3. `demand.csv` / table `demand`

Grain: one row per **date × sku_id × warehouse_id**. Every SKU/warehouse pair has 365 continuous days.

| Column | Type | Nullable | Rule | Description |
|---|---|---|---|---|
| `date` | DATE (`YYYY-MM-DD`) | No | 2025-09-23 … 2026-09-22, no gaps | Date of demand |
| `sku_id` | TEXT | No | `M001`–`M030` | Product identifier |
| `warehouse_id` | TEXT | No | `W001`–`W006` | Distribution center |
| `demand_qty` | INTEGER | No | ≥ 0 (observed 15–180) | Actual demand / consumption that day |
| `promotion_flag` | INTEGER | No (default 0) | 0 or 1 | 1 = promotion active that day |
| `season_flag` | INTEGER | No (default 0) | 0 or 1 | 1 = seasonal demand period active for that SKU's group |

**Patterns built into the data:** base level per SKU × warehouse (25–55 units/day), weekly cycle (weekends lower), mild yearly growth (~+6%), promotion blocks (1.2–1.4× uplift), seasonal uplift (1.5–1.7×, ramping over 7 days), autocorrelated noise (~±6–8%). The last 21 days of M001 ramp up further (up to +45% at W002) to give a visible rising trend.

## 4. `inventory.csv` / table `inventory`

Grain: one row per **sku_id × warehouse_id** (180 rows).

| Column | Type | Nullable | Rule | Description |
|---|---|---|---|---|
| `sku_id` | TEXT | No | must exist in `demand` | Product identifier |
| `warehouse_id` | TEXT | No | must exist in `demand` | Distribution center |
| `current_stock` | INTEGER | No | ≥ 0; ≤ `capacity`; equals the sum of that pair's batch quantities | Current available stock |
| `min_threshold` | INTEGER | No | ≥ 0; ≥ `safety_stock` | Minimum inventory threshold |
| `safety_stock` | INTEGER | No | ≥ 0 | Desired buffer stock |
| `capacity` | INTEGER | No | > 0 | Storage capacity for this SKU at this warehouse |
| `lead_time_days` | INTEGER | No | > 0 (observed 3–8) | Expected replenishment lead time |

Generated mix (non-M001): ≈ 55% healthy, ≈ 18% low, ≈ 17% high-cover/excess, ≈ 10% moderate.

## 5. `batches.csv` / table `batches`

Grain: one row per batch (428 rows). 1–4 batches per SKU × warehouse.

| Column | Type | Nullable | Rule | Description |
|---|---|---|---|---|
| `batch_id` | TEXT (PK) | No | unique, format `B00001` | Unique batch identifier |
| `sku_id` | TEXT | No | must exist in `inventory` | Product identifier |
| `warehouse_id` | TEXT | No | must exist in `inventory` | Where the batch is stored |
| `quantity` | INTEGER | No | > 0 | Units in the batch |
| `expiry_date` | DATE (`YYYY-MM-DD`) | No | valid date | Batch expiry |

Expiry mix: **6 batches (1.4%) are deliberately already expired** — dead stock, 7 to 46 days past expiry, spread across 6 different non-M001 SKU/warehouse pairs, to demonstrate an "already expired" alert distinct from the near-expiry one. None of the 6 involve M001, so the shortage/excess demo story is untouched. Beyond that: ≈ 4% of batches expire within 30 days (including the M001 near-expiry batch), ≈ 10% in 31–90 days, the rest have 3–24 months left.

**Why this matters for the demo:** a full year of history means some stock realistically goes unsold and expires unnoticed. This gives the E1 alerts screen two distinct alert types to show — "near expiry" (act soon) and "already expired" (write-off / dead stock) — rather than just one.

## 6. `sales.csv` / table `sales`

Grain: one row per **date × sku_id × warehouse_id** — exactly one sales row for every `demand` row (65,700 rows), joining 1-to-1 on those three columns.

| Column | Type | Nullable | Rule | Description |
|---|---|---|---|---|
| `date` | DATE (`YYYY-MM-DD`) | No | same 365 days as `demand` | Date of the sales |
| `sku_id` | TEXT | No | must exist in `inventory` | Product identifier |
| `warehouse_id` | TEXT | No | must exist in `inventory` | Distribution center |
| `units_sold` | INTEGER | No | ≥ 0 and ≤ `demand_qty` for the same row | Units actually sold / fulfilled that day |
| `unit_price` | REAL (₹) | No | > 0, 2 decimals | Effective selling price per unit that day |
| `revenue` | REAL (₹) | No | = `units_sold` × `unit_price` (2 decimals) | Revenue for the row |

**How it relates to demand:** `demand_qty` is what customers wanted; `units_sold` is what was actually delivered. Equal on most days. On ~2.7% of rows, part of the demand couldn't be filled (5–30% short), and locations already below their `min_threshold` lose sales more often (~8% of days vs ~1.5%). Overall fill rate ≈ 99.5%.

**Price logic:** illustrative INR prices per SKU (M001 ₹18, M022 insulin pen ₹780), a +4% price revision from 2026-04-01, and a 10% discount on promotion days.

**What it is not:** no sales targets, growth rates or KPIs are stored. The forecasting model should train on `demand_qty`, not `units_sold`, since sales hide unfilled demand.

## 7. Relationships

```
demand (sku_id, warehouse_id) ──┐
                                ├── inventory (sku_id, warehouse_id)   ← one row per pair
batches (sku_id, warehouse_id) ─┘         Σ batches.quantity = inventory.current_stock

sales (date, sku_id, warehouse_id)  ←→  demand (date, sku_id, warehouse_id)   1-to-1, units_sold ≤ demand_qty
```

No foreign keys are declared (kept simple); consistency is enforced by the generator and confirmed in `validation_report.md`.

## 8. Demo scenario (data conditions only)

| Item | Data condition |
|---|---|
| **M001 @ W002 Madurai — shortage** | stock 120, safety 80, min 200, lead time 5 days; demand rising (+23% last 14d vs prior 14d), last-7-day average ≈ 83/day |
| **M001 @ W004 Mumbai — excess** | stock 850, safety 100, min 250, lead time 5 days; demand ≈ 49/day, requirement ≈ 350, surplus ≈ 500 |
| **Near-expiry batch** | `B00006`: M001 @ W004, 250 units, expires 2026-10-14 (21 days from 2026-09-23) |
| **Seasonal increase** | M001 fever season 15 Aug–31 Oct (`season_flag = 1`), season demand ≈ 1.6× normal |
| **Already-expired batches** | 6 batches (M004, M005, M008, M017, M026, M030), 7–46 days past expiry — dead stock, distinct from near-expiry |
| **Reorder-required (no transfer possible)** | M002, M011, M021 are short at **every** warehouse, with **no surplus anywhere** in the network — the only fix is a supplier reorder |
| **Transfer-fixable** | 21 SKUs (including M001) are short somewhere, but another warehouse has genuine surplus to draw on |

The final recommendation (transfer, reorder, etc.) is **not** in the data — the decision engine derives it.

### Why the reorder scenario matters
Without it, every shortage in the dataset could be solved by moving stock around, which would make a "reorder from supplier" code path in the decision engine untested and unnecessary to build. M002, M011 and M021 were deliberately pushed into network-wide shortage — every one of their 6 warehouses sits below both `min_threshold` and a naive required-stock calculation (`lead_time_days × recent demand + safety_stock`) — so the decision engine has a genuine case where transfer isn't an option and reordering is the only real fix.

## 9. Notes for teammates

- **Nishit (ML):** forecast from `demand.csv` (not `sales.csv`); `promotion_flag` and `season_flag` are the exogenous features. Last observed day is 2026-09-22.
- **Decision engine:** "today" = 2026-09-23; use `expiry_date − today ≤ 30` for near-expiry.
- **Gaurav (dashboard):** read from `medcare.db` (SQLite) or `medcare_mysql.sql` (MySQL) — same data, either engine.
- **Sales:** use `sales` for revenue / fill-rate views. Train the ML model on `demand_qty`, not `units_sold`.
- **Reproducibility:** run `generate_data.py` (seed 42) → `add_sales.py` (seed 7) → `export_mysql.py` to regenerate everything.
