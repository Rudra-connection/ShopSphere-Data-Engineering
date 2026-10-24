# E-commerce star schema – KPI dashboard

**Deliverables (this folder):** `Ecommerce_Star_Schema_Dashboard.xlsx` (dashboard + KPI sheets + profiling, data model, assumptions, AI log; original tables included), `pipeline.py` (raw→curated→KPIs), `build_workbook.py` (workbook builder), `schema.sql` (DDL + KPI 4 window SQL).

Run: `python3 pipeline.py <input.xlsx> <outdir>` then `python3 build_workbook.py <input.xlsx> <outdir>/kpis.pkl <output.xlsx>`.

## Headline results
Net revenue 19.16M (after 0.82M approved refunds) over 223 realised orders · 90-day repeat rate 9.2% (8.6% on customers with a closed window) · 218 of 328 order/payment records carry a reconciliation flag · 163 of 758 lines sold below cost, est. loss 1.80M.

## Assumptions & edge cases
See the `Assumptions_README` sheet (A1–A12). Most important:
- Realised sales = Completed + Returned orders; only Approved returns reduce revenue; refunds sit in the order month because 29/59 returns pre-date their order.
- Unknown customer/product keys are kept in revenue totals but excluded from customer/product-level KPIs.
- KPI 5 uses a 30-unit minimum; KPI 6 weights 40/30/30 (editable); KPI 7 uses ≤90 days between distinct orders.
- The source file is already a star schema, so "raw → curated" is mostly structural cleanup (16,365 empty columns), integrity checks and derived measures – no duplicate products or non-standard labels were found.

## SCD approach (customers)
Type 2: surrogate key + effective_from/to + is_current. Today every customer has one current row; a change in segment/city closes the row and inserts a new one.

## Trade-offs
Formulas are used where cheap and auditable (tiles, KPI 1, MoM, scores, counts); ranking/window/join logic is precomputed by the pipeline.

## AI usage log
Claude (Sonnet 5.5) wrote the pipeline and workbook. Validation: KPI 1 formula total = pipeline total (diff 0), net = gross − discount on all lines, recalculation with 0 formula errors, spot checks of payment reconciliation. Full log in the `AI_Usage_Log` sheet.
