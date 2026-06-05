# SCHEMA_TRUTH.md

**Source of truth for the AMS Floors database schema, cost-column rules, and save-path contract.**

> **How to read this document.** Every fact is tagged:
> - **[VERIFIED 2026-06-04]** — confirmed directly against the live Supabase `public` schema (column inventory of all 18 tables, 295 columns). The live DB wins over this doc if they ever disagree; re-verify after any migration.
> - **[RULE]** — a decision/contract carried from prior project work. Not derivable from a column dump; must be honored by code.
> - **[TO CONFIRM]** — not yet verified; flagged honestly rather than assumed. Do not treat as fact.
>
> Verification method: `select table_name, column_name, data_type, is_nullable from information_schema.columns where table_schema='public'`. Run via the **Supabase SQL editor** (the REST API / `information_schema` route does NOT work for column detection — forbidden trap). The editor caps at 100 rows by default; the full result is 296 rows incl. header — export must capture all of them.

---

## 0. Session updates (read this first)

**[VERIFIED 2026-06-05 — Module 7 cutover DONE]**
- The command center (`index.html`) now reads **only the live Supabase tables**. It deploys via **Netlify** (project `dapper-faun-fb14a0`, URL `dapper-faun-fb14a0.netlify.app`), auto-building from GitHub `main`. (Note: deploy host is Netlify, not GitHub Pages.)
- Live read confirmed in-app: **$13.17M open pipeline / 154 active bids**, "Live sync" on.
- **Removed:** `parseBidsheetFile` (retired parser) and `BulkIntakeView` (in-app bulk import UI). Both wrote `costPerUnit` into `estimate_items` — the v2/v10 corruption pattern. Gone wholesale.
- **`_SEED` kept** as the offline fallback + rollback snapshot. Its auto-insert-if-empty branch was **neutralized** — an empty query no longer writes `_SEED` into `proposals`; it surfaces an error and shows `_SEED` read-only.
- **Reversible:** revert the commit on GitHub `main`; Netlify redeploys. No DB rows/schema were changed by the cutover.

**[VERIFIED 2026-06-05 — migration data quality]**
- `estimate_items`: **2,722** line items; **2,628** have a real `unitCost` (avg $41.90, range $0–$4,718.05). The ~94 without are header/labor lines (expected). `materialActual` populated on 2,659; `lineMarkupRate` on 2,300.
- Markup varies per file as expected (e.g. AFC Urgent Care #1313 = 30%; U.S. Foodservice #1535.3 = 23%). Engine totals reconcile to the penny off the stored `unitCost`.

**[RULE — STATUS SOURCE OF TRUTH] (added 2026-06-05)**
Proposal **status is NOT stored in the bidsheets/proposals**, and **NOT reliably in `_SEED`** (that snapshot is a hand-kept pipeline list and may be ~2 months stale). The authoritative source is **which OneDrive folder the project file lives in**:
```
OneDrive - AMS Floors\AMS Floors - General\
  1. Current Projects      -> Active
  2. Currently Bidding     -> Negotiating / Open
  3. Submitted Proposals   -> Submitted
  4. Completed Projects    -> Won / Closed
  5. Dead Projects         -> Dead / Lost
```
- **All 154 migrated proposals currently carry `status = "Submitted"` in the live DB** (placeholder; real statuses not yet assigned).
- Proposal **numbers and names are both unique**, and filenames carry the number — so folder→file→`proposals.id` mapping is near-automatic (match on number; name as cross-check).
- **[RULE]** When assigning status, derive it from the **folder**, not from `_SEED` or the bidsheet. Use `_SEED` only as a cross-check; where folder and `_SEED` disagree, flag for Peter — never silently pick. Do NOT bulk-map `_SEED` statuses onto the DB.

---

## 1. The cost-column law (prime directive — never re-litigate)

These are the facts whose violation caused the v2/v10 silent corruption. All four are **[VERIFIED 2026-06-04]** against the live schema and **re-confirmed 2026-06-05**:

| Fact | Status |
|---|---|
| `estimate_items.unitCost` **exists** (numeric) | ✅ VERIFIED |
| `estimate_items.costPerUnit` **does NOT exist** | ✅ VERIFIED ABSENT |
| `purchase_order_items.costPerUnit` **exists** (numeric) | ✅ VERIFIED |
| `purchase_order_items.unitCost` **does NOT exist** | ✅ VERIFIED ABSENT |

**[RULE]** POs **read** `unitCost` (from `estimate_items`) and **write** `costPerUnit` (to `purchase_order_items`). Writing `costPerUnit` to `estimate_items` = silent corruption (the v2/v10 bug). The schema enforces this structurally — the wrong column does not exist on either table — but code must still never attempt it.

**[RULE]** Never use `information_schema` via the REST API for column detection. Use the SQL editor.

### 1a. Module 12 finding — COST column reads the wrong field [VERIFIED 2026-06-05]
The command center reads `item.costPerUnit` (which does not exist on migrated rows) in several places, so the **COST column renders blank** and the **PO generator would read $0** — even though `unitCost` holds real data and line **totals already reconcile** (the math fn falls back to `unitCost`). Only the display/PO layer is affected.

Spots reading `item.costPerUnit` that need a `?? item.unitCost` fallback (re-grep for exact lines before editing):
- COST cell — the editable input (`<ACInput value={item.costPerUnit} onChange={...updateField('costPerUnit',...)}`). **Also a write hazard:** its `onChange` writes `costPerUnit` back to `estimate_items` (a non-existent column) — editing a cost silently fails. Fix must redirect the write to **`unitCost`**.
- PO generator `perOrderCost(i)` — reads `i.costPerUnit` only (why POs read $0).
- Catalog price-drift hint history (two spots).
- A PO calc helper (`var calc=...costPerUnit...`).

**[RULE]** This is Module 12 work. Fixing cost math/reads requires **re-passing all 5 fixtures (§7)** to prove totals still reconcile before trusting it.

---

## 2. Peter's locked math (proven to the penny — never recompute differently)

**[RULE]** These formulas are fixed. Any engine/save change must re-pass the fixture suite (§7).

```
AMOUNT   = ORDER × COST
MATERIAL = AMOUNT + (AMOUNT × tax) + freight + (AMOUNT × markup)
LABOR    = QTY × laborRate
```

- **Labor uses QTY; material uses ORDER.** (Do not cross them.)
- **Store:** `unitCost` (raw COST), `materialActual` (computed MATERIAL), `lineMarkupRate` (markup decoded from the file's formula).
- **Never write** the baked-cost columns: `usedMaterial`, `multiplier`, `deduction`, `height`. (They exist on `estimate_items` — see §4 — but are forbidden to write.)
- **Markup VARIES per file** (10% Northside, 20% others; live examples 30% / 23%). Read it from the column header / decode from the formula. **Never assume a markup rate.**

---

## 3. Save-path contract

**[RULE]** Write order: `proposals` → `estimates` → `estimate_items`. Change orders / addendums attach as `proposal_revisions`; VE / alternate scenarios as `proposal_scenarios`. Generated proposal snapshots go to `proposal_documents`.

**[RULE]** Idempotency: define a stable key (e.g. source filename + projectNumber) so re-running the migration creates zero duplicates. (Specific key field **[TO CONFIRM]** — no natural unique constraint visible in the column inventory; FK/constraint dump needed.)

**[VERIFIED 2026-04]** Linking columns present: `estimates.proposalId`, `estimate_items.estimateId`, `estimate_items.proposalId`, `estimate_items.scenarioId`, `purchase_orders.proposalId`/`estimateId`, `purchase_order_items.purchaseOrderId`/`estimateItemId`, `proposal_revisions.proposalId`, `proposal_scenarios.proposalId`.
**[TO CONFIRM]** Actual foreign-key *constraints* (vs. just columns) — the column inventory does not include FK relationships. Run a separate `key_column_usage` / `table_constraints` query to confirm enforced FKs and any unique constraints.

---

## 4. Verified table inventory [VERIFIED 2026-06-04]

18 tables in `public`. Columns below are exact live names. `(type)` shown; all are nullable=YES except `id` (NO) unless noted.

### proposals (35 cols) — pipeline/CRM record
`id, project, location, gc, contact, email, phone, submitted(date), bidAmount, contractAmountActual, status, owner, notes, sharepoint, retainage, estMaterial, estLabor, estOther, actMaterial, actLabor, actOther, lastTouched(date), createdAt, updatedAt, nextAction, nextActionDate(date), lostReason, lostToPrice, wonByPrice, estimator, bdOwner, pm, projectNumber, architectId, plansDate(date)`
- **`status`** lives here. All 154 migrated rows currently = `"Submitted"` (placeholder). Real status source = OneDrive folder (see §0).

### estimates (38 cols) — proposal header / project + GC + prepared-by + rates
`id, proposalId, proposalNumber, projectName, projectAddressLine1/2, projectCity/State/Zip, projectNumber, plansDated(date), addendum, gcId, gcName, gcAddressLine1/2, gcCity/State/Zip, gcContactName/Email/Phone, architectName/Address/Email/Phone, preparedByName/Email/Phone, taxRate, markupRate, baseBidNotes, alternatesNotes, proposalNotes, baseBidTotal, alternatesTotal, createdAt, updatedAt`
- **Note:** `taxRate` and `markupRate` live here at the estimate level. Per §2, markup varies per file — this is where the decoded rate is stored.

### estimate_items (44 cols) — the atomic line items
`id, estimateId, proposalId, parentItemId, sortOrder, sectionCode, sectionName, isAlternate, alternateGroupName, itemCode, size, manufacturer, description, color, qty, unit, sfPerCtn, syPerCtn, orderQty, orderUnit, orderQtyOverride, `**`unitCost`**`, freight, laborRate, displayMode, supplierId, supplierName, installerId, installerName, isVeAlternative, veNote, notes, createdAt, updatedAt, `**`usedMaterial, multiplier, deduction, height`**` (FORBIDDEN TO WRITE), source, coveragePerUnit, alternateType, scenarioId, `**`materialActual, lineMarkupRate`**` (STORE THESE)`
- Cost column: **`unitCost`** ✅. No `costPerUnit` ✅. (App reads `costPerUnit` first — see §1a, the Module 12 fix.)

### purchase_orders (29 cols)
`id, poNumber, proposalId, estimateId, type, supplierId, supplierName, installerId, installerName, shipToName, shipToAddressLine1/2, shipToCity/State/Zip, shipToContact, shipToPhone, status, subtotalAmount, taxAmount, freightAmount, totalAmount, neededByDate(date), sentAt, confirmedAt, receivedAt, notes, createdAt, updatedAt`
- **Three-party ship-to** fields present (manufacturer → job site) for Module 12.

### purchase_order_items (14 cols)
`id, purchaseOrderId, estimateItemId, sortOrder, itemCode, manufacturer, description, size, color, qty, unit, `**`costPerUnit`**`, amount, notes`
- Cost column: **`costPerUnit`** ✅. No `unitCost` ✅.

### gcs (14 cols) — general contractors
`id, name, addressLine1/2, city, state, zip, phone, email, website, paymentTerms, notes, createdAt, updatedAt`

### suppliers (16 cols)
`id, name, contactName/Email/Phone, addressLine1/2, city/state/zip, paymentTerms, leadTimeDays(int), defaultFreightRate, notes, createdAt, updatedAt`

### installers (11 cols)
`id, name, contactName/Email/Phone, specialties, defaultLaborRates(jsonb), paymentTerms, notes, createdAt, updatedAt`

### proposal_revisions (9 cols) — addendums/COs
`id, proposalId, revisionNumber(int), addendumRef, addendumDate(date), summary, acknowledgedAt, acknowledgedBy, createdAt`

### proposal_scenarios (11 cols) — VE / alternates
`id, proposalId, scenarioName, isSubmitted(bool), isBaseScenario(bool), detectedFrom, detectionText, priceDelta, notes, createdAt, updatedAt`

### proposal_documents (10 cols) — generated snapshots
`id, estimateId, proposalId, version(int), generatedBy, snapshot(jsonb), sentTo, sentAt, notes, createdAt`

### ve_packages (6 cols)
`id, estimateId, name, description, sortOrder, createdAt`

### ve_package_items (3 cols)
`id, vePackageId, estimateItemId`

### line_items (13 cols) — est-vs-actual tracking (Phase 4)
`id, proposalId, type, item, vendor, qty, unit, estUnit, estTotal, actUnit, actTotal, status, createdAt`

### products (17 cols) — catalog
`id, itemCode, manufacturer, supplierId, size, description, unit, sfPerCtn, syPerCtn, lastKnownCost, lastUsedDate(date), category, csiDivision, useCount(int), notes, createdAt, updatedAt`
- Catalog "living statistics": `lastKnownCost`, `lastUsedDate`, `useCount`.

### architects (15 cols)
`id, name, addressLine1/2, city/state/zip, phone, email, contactName/Email/Phone, notes, createdAt, updatedAt`

### proposal_note_templates (6 cols)
`id, name, content, isDefault(bool), sortOrder, createdAt`

### app_settings (5 cols)
`id(int), slaSubmitted(int), slaNegotiating(int), targetMargin, riskMarginFloor`

---

## 5. Slice dimensions — corrected against live schema

| Dimension | Status |
|---|---|
| GC | ✅ `proposals.gc`, `estimates.gcId`/`gcName`, table `gcs` |
| Estimator | ✅ **`proposals.estimator` EXISTS** — NOT missing. Also `pm`, `bdOwner`. |
| Material / supplier | ✅ `estimate_items.manufacturer`/`supplierName`, table `suppliers` |
| CSI section | ✅ `estimate_items.sectionCode`/`sectionName`, `products.csiDivision` |
| Crew / installer | ✅ `estimate_items.installerName`, table `installers` |
| Time | ✅ `submitted`, `createdAt`, `lastTouched`, etc. |
| Status | ✅ `proposals.status` — **but source of truth is the OneDrive folder; all 154 currently "Submitted" (placeholder).** See §0. |
| Margin | ✅ derivable from cost/markup; `app_settings.targetMargin` |
| **Project type** | ❌ **GENUINELY ABSENT** — no `projectType` column anywhere. Real gap to add at migration. |

**[RULE]** Project type is the one slice dimension missing from the schema. Add it (likely `proposals.projectType`) before/at Module 5 migration so the intelligence layer can slice by it.

---

## 6. Engine & forbidden traps

- **[RULE]** `ams_xlsx_reader.py` is the single trusted parser. The command center's `parseBidsheetFile` is **REMOVED as of 2026-06-05** (it corrupted — wrote `costPerUnit` to `estimate_items`). Salvage only UI/presentation from old versions — never parser/save code.
- **[RULE] Do NOT reintroduce an in-app bidsheet parser/writer.** Historical files load via the verified Python migration pipeline; forward takeoffs come via Module 14 (Bluebeam CSV).
- **[RULE] Forbidden:** `costPerUnit` on `estimate_items`; per-file parser heuristics; `information_schema` via REST API; "write both column names"; trusting on-screen reconciliation without checking actual DB rows; silent formula auto-adoption; auto-acted suggestions; **bulk-mapping `_SEED` statuses onto the DB** (status comes from the folder).

---

## 7. Fixture suite (regression — re-pass on every engine/save change)

| File | Expected |
|---|---|
| PROPOSAL_AMS_Crash_Champions.xlsx | $25,537.38 |
| PROPOSAL_AMS_Homewood_Suites_Cheshire.xlsx | base $533,954.08 / alt $552,533.90 |
| PROPOSAL_AMS_FLOORS_Northside_Christian.xlsx | $68,534.35 (header 15%, **actual markup 10%**) |
| PROPOSAL_AMS_FLOORS_Teachers_FCU.xlsx | $36,351.60 |
| PROPOSAL_AMS_Aspinwall.xlsx | computed $232,023 / stated $232,110 (+$87 rounding) — **verify against the real file (`onedrive_live\1320 Aspinwall\`), not a screenshot** |

---

## 8. Outstanding to confirm (do not treat as done)

1. **Foreign-key constraints & unique constraints** — column inventory only; run `table_constraints` + `key_column_usage` queries.
2. **Idempotency key** — no natural unique constraint visible; decide and document in Module 3/4.
3. **Authoritative reader** — `ams_xlsx_reader.py` vs `ams_reader_v2.py` (Module 2).
4. **`projectType` column** — absent; add at migration.
5. **Status assignment** — all 154 currently "Submitted." Real source = OneDrive folder (§0). Next task: dump folder→file listing, match on proposal number, generate `UPDATE`. `_SEED` is cross-check only.
6. **Module 12 COST/PO fix** — app reads `costPerUnit`, data is in `unitCost`; fix ~5 spots + redirect the edit-write to `unitCost`; re-pass fixtures (§1a, §7).
7. This doc reflects schema as of **2026-06-04**, with session updates **2026-06-05**. Re-verify after any migration or DDL change. Live DB wins.
