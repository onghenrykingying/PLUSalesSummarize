# PLU Sales Summarizer — Project Summary

A pure client-side, single-file web app (`index.html`) that turns raw retail POS exports
into per-SKU **Profit & Loss** reports for a store in Manila, Philippines. No server, no
build step, no backend — everything runs in the browser.

---

## 1. Architecture

- **One file:** all HTML, CSS, and JavaScript live in `index.html`.
- **Libraries (CDN):** SheetJS `xlsx` 0.20.3 (read/write Excel & CSV), Chart.js 4.4.0,
  jsPDF 2.5.1 + jspdf-autotable 3.8.2, Google Fonts (Plus Jakarta Sans).
- **Reference data:** `Reference/MASTERLIST.CSV` — a sample POS SKU master list.
- **Everything is local:** files are read in-browser via `FileReader`/`arrayBuffer`;
  outputs are generated client-side and downloaded.

---

## 2. Pipeline (4 steps)

| Step | Name | Input | Output |
|------|------|-------|--------|
| **1** | **SKU Cleaner** | Raw POS master list (CSV/Excel) | Clean SKU list (8 essential columns) |
| **2** | **Weekly** | Raw weekly sales files **+ SKU list (required)** | Weekly P&L (2 files) |
| **3** | **Monthly** | Multiple Weekly P&L export files | Aggregated monthly P&L (2 files) |
| **4** | **Re-price** *(optional)* | A month's sales + an **updated** SKU list | Re-priced P&L (2 files) |

**Flow logic:**
- **Step 1** trims a messy POS master down to: `prod_code, prod_desc1, unit_cost, sell_price,
  dept_code, dept_desc, sup_code, sup_desc`.
- **Step 2 (Weekly)** is where the profit calculation happens: it aggregates raw sales by PLU
  and joins each SKU's **unit cost & selling price from that week's SKU list** to produce a
  full P&L. The SKU list is **required** here.
- **Step 3 (Monthly)** simply **compiles** the weekly P&L files into one month. It needs **no
  SKU list** — prices already live in the weekly files. It is the *real* monthly P&L.
- **Step 4 (Re-price)** is a **"what-if"** tool: apply a *different/updated* SKU list to a
  month's sales to recompute the P&L at new prices. Distinct from Step 3 (which keeps each
  week's actual prices).

---

## 3. P&L methodology (two pricing methods, shown side by side)

Every SKU is costed two ways so you can compare *catalog* vs *reality*:

**PRIMARY — SKU List price method** (the main basis):
- `list_revenue = sell_price × qty`
- `list_profit  = list_revenue − COGS`

**SECONDARY — Realized method** (what actually rang up at the till):
- `revenue (realized) = amt` (actual sales amount from the POS)
- `avg_price (realized) = amt ÷ qty`
- `profit (realized) = amt − COGS`

**Shared:** `COGS = unit_cost × qty`

**Deviation between the two methods:**
- `price_dev = avg_price − sell_price` (per-unit gap: realized − list)
- `variance  = amt − list_revenue` (revenue/profit gap: realized − list)

> Use case: the **Primary** answers "what *should* we have made at catalog prices?"; the
> **Realized** answers "what *did* we actually make?"; the **Deviation** shows discounting,
> price overrides, or stale catalog prices.

---

## 4. Per-SKU data fields

Produced for every SKU row in the P&L:

```
plu, desc, qty, amt,
unit_cost, sell_price, cogs,
list_revenue, list_profit, list_margin %, list_sales_mix %, list_profit_contrib %,
avg_price, profit (realized), margin %, sales_mix %, profit_contrib %,
price_dev, variance,
abc, matched, matchType (exact|stripped|special), noPrice, isDiscount,
sup_code, sup_desc, dept_code, dept_desc, uom_code, wholeprice
```

---

## 5. On-screen views (Weekly, Monthly, Re-price)

After processing, each step shows a single **Total Amount** stat card, a **Data Quality**
panel, and a 3-tab P&L preview:

1. **P&L · SKU List Price** (primary, 14 cols): PLU, Description, ABC, Department, Supplier,
   Total Qty, Unit Cost, Sell Price, COGS, Revenue, Profit, Margin %, Sales Mix %, Profit Contrib %.
2. **Realized & Deviation** (12 cols): PLU, Description, Total Qty, Sell Price·List,
   Avg Price·Realized, Price Dev, Revenue·List, Revenue·Realized, Profit·List, Profit·Realized,
   Profit Variance, Realized Margin %.
3. **Non-Moving SKUs** (9 cols): PLU, Description, Sup Code, Supplier, Dept Code, Department,
   Unit Cost, UOM, Whole Price — SKUs in the master that had **no sales**.

All columns are sortable; tabs operate independently per step.

---

## 6. Matching logic

- Match is by **SKU/PLU number only** (`prod_code`) — **not** by item name/description.
- Two-step: **exact** code match, then a **leading-zero-stripped** fallback
  (e.g. `004806…` ↔ `4806…`).
- Stripped-only matches are flagged for review ("Matched only by zero-stripping").

---

## 7. Special handling

### Missing selling price (Option B)
A matched SKU with no/zero selling price → `list_revenue` treated as **0**, the row is **kept**
and flagged with a ⚠ marker, and counted under "SKUs Missing Price". (Avoids fake 100% margins.)

### Discounts — built-in special codes (NOT in the SKU list)
Cashier-applied social-benefit discounts are recognized automatically across all steps:

| Code | Meaning |
|------|---------|
| **DISC1** | S.C.D. (Senior Citizen Discount) |
| **DISC2** | P.W.D. (Person With Disability Discount) |

- Treated as `unit_cost = 0`, `sell_price = 0`, `COGS = 0`; the **(negative) amount flows
  through both methods**, reducing revenue and profit (i.e. net sales). Grouped under "Discounts".
- **Excluded** from error checks (never flagged as unmatched / zero-cost / zero-price /
  qty-amount mismatch / loss-maker). Tagged `DISCOUNT` in the table.
- The Summary sheet shows a **"Total Discounts (S.C.D./P.W.D.)"** line + per-code breakdown.

### ABC classification
By realized revenue, cumulative buckets: **A = top 70%**, **B = next 20%** (to 90%), **C = rest**.

---

## 8. Monthly aggregation & weekly price changes

Prices can change week to week, so the same SKU may carry **different unit/sell prices in each
weekly file**. The Monthly step handles this correctly:

- Each weekly file is **already priced** at that week's prices; Monthly **sums**
  Revenue / Profit / COGS — so totals faithfully reflect week-by-week pricing.
- The displayed **Unit Cost / Sell Price are weighted-average effective prices**:
  - `Effective Sell Price = Monthly Revenue ÷ Monthly Qty`
  - `Effective Unit Cost  = Monthly COGS    ÷ Monthly Qty`
  - (e.g. 60 units @ ₱10 + 40 @ ₱12 → effective ₱10.80)
- SKUs whose price changed across weeks are flagged in Data Quality.

---

## 9. Data Quality panel + Issues export

Each step lists triggered checks with a severity badge and a count, plus an **Export Issues**
button (Excel = Summary + one sheet per issue listing offending rows; CSV = one flat sheet).

| Severity | Checks |
|----------|--------|
| **Blocker** | SKU not matched · Missing/zero unit cost · Missing/zero sell price · *(Monthly)* file had no usable P&L data |
| **Warning** | Sell price below unit cost · Negative qty · Inconsistent qty/amount · Realized profit negative (loss-maker) · **Realized price ≥ 20% off list price** · Duplicate PLU in SKU list · *(Monthly)* cost/sell price changed across weeks |
| **Info** | Matched only by zero-stripping · Negative cost/price · Blank PLU rows skipped |

---

## 10. File outputs & naming

Each step produces **two files** (P&L + Non-Moving), plus an optional **Issues** file.
The **P&L Analysis** workbook has 3 sheets:

- **Page 1 — P&L (SKU List Price):** primary per-SKU P&L. Also carries `Revenue (Realized)` &
  `Profit (Realized)` columns so the Monthly step can re-aggregate from this one sheet.
- **Page 2 — Realized & Deviation:** realized price/profit vs list, with deviation/variance.
- **Page 3 — Summary:** KPIs (on SKU-list basis), discounts, department rollup,
  top/bottom 10 SKUs by profit.

**Naming uses the selected period (From/To dates), not an auto time-stamp.** Files are named
by period with spaces; you file them into your own period-named folder.

| Step | Period format | Example files |
|------|---------------|---------------|
| 1 · SKU Cleaner | `YYYY-MM-DD` (list version date) | `2026-05-01 SKU-List.xlsx` |
| 2 · Weekly | `YYYY-MM-DD_DD` (same month) or `YYYY-MM-DD_YYYY-MM-DD` | `2026-05-05_11 Weekly PnL.xlsx` · `2026-05-05_11 Weekly NonMoving.xlsx` · `2026-05-05_11 Weekly Issues.xlsx` |
| 3 · Monthly | `YYYY-MM` (from the From date) | `2026-05 Monthly PnL.xlsx` · `2026-05 Monthly NonMoving.xlsx` · `2026-05 Monthly Issues.xlsx` |
| 4 · Re-price | `YYYY-MM` | `2026-05 Reprice PnL.xlsx` · `2026-05 Reprice NonMoving.xlsx` · `2026-05 Reprice Issues.xlsx` |

- A **Quick range** preset dropdown (This/Last week, This/Last month, Last 7/30 days, YTD;
  weeks = Mon–Sun) fills the From/To dates.
- Blank dates fall back to **today** (Step 1 / Weekly) or **today's month** (Monthly / Re-price).
- **CSV** format exports the primary sheet only — multi-sheet output requires **Excel**.

---

## 11. Input expectations

**SKU master list** (Step 1 / SKU list uploads) — recognized columns (flexible header match):
`prod_code` (or plu_code/plu/code), `prod_desc1`/`desc`, `unit_cost`, `sell_price`,
`sup_code`, `sup_desc`, `dept_code`, `dept_desc`, `whole_code`, `uom_code`, `wholeprice`.

**Sales files** — need at least: a **PLU** column, **Total Qty**, and an **amount** column
(plain summaries use `Total Amount`; P&L exports use `Revenue (Realized)`).

**Step 4 (Re-price)** accepts either a plain monthly summary **or** a P&L export as the
"Month's Sales" input.

---

## 12. Robust CSV reading (important fix)

Real POS descriptions often contain stray double-quotes — e.g. inch marks like `1/2"` or
`24"`. SheetJS's CSV parser treats each `"` as a quoted-field delimiter, which made it
**merge/drop rows** (e.g. a 10,610-row list read as 7,943).

**Fix:** all CSV uploads in **every step** now go through a custom, **line-safe tolerant CSV
parser** — one physical line per row, so a stray quote can never spill across rows. A `"` in
the middle of a field is kept as a literal character. `.xls/.xlsx` files are unaffected and
still parsed by SheetJS. Row counts are now correct even with stray quotes.

---

## 13. Step relationship (Step 3 vs Step 4)

- **Step 3 (Monthly)** = *what actually happened* — sums the weekly P&L files, preserving each
  week's real prices. This is the authoritative monthly P&L.
- **Step 4 (Re-price)** = *what-if* — re-prices a whole month against a single (updated) SKU
  list. The **Realized & Deviation** tab then shows how the new prices compare to what actually
  sold. Use only when prices changed or to test new pricing.

---

*Single-file app — all changes are made in `index.html`. No build, no dependencies to install.*
