[SCHEMA_TRUTH (2).md](https://github.com/user-attachments/files/28477451/SCHEMA_TRUTH.2.md)
# AMS Floors — Module 1: SCHEMA TRUTH (verified single source of truth)

**Status:** VERIFIED against `Supabase_Snippet_Public_table_schema_summary.csv` (live schema, 18 tables) on the Module-1 audit. Every claim below names the columns it was checked against. This document RETIRES any prior schema description that disagrees with it. Project 3 does NOT rebuild this schema; it writes to it correctly.

---

## The 18 tables (confirmed count)
app_settings, architects, estimate_items, estimates, gcs, installers, line_items, products, proposal_documents, proposal_note_templates, proposal_revisions, proposal_scenarios, proposals, purchase_order_items, purchase_orders, suppliers, ve_package_items, ve_packages.

---

## The tables that carry the migration (verified column lists)

### estimate_items (44 cols) — the line-item home; ALL bidsheet lines land here
Capture fields confirmed present: `id, estimateId, proposalId, parentItemId, sortOrder, sectionCode, sectionName, isAlternate, alternateGroupName, itemCode, size, manufacturer, description, color, qty, unit, sfPerCtn, syPerCtn, orderQty, orderUnit, orderQtyOverride, unitCost, freight, laborRate, displayMode, supplierId, supplierName, installerId, installerName, isVeAlternative, veNote, notes, createdAt, updatedAt, usedMaterial, multiplier, deduction, height, source, coveragePerUnit, alternateType, scenarioId, materialActual, lineMarkupRate`.
- Cost-per-unit column here is **`unitCost`** (NOT `costPerUnit`).
- Multi-block/scenario support: `scenarioId`, `isAlternate`, `alternateGroupName`, `sectionCode`, `sectionName`.
- Computed-with-raw-inputs support: `unitCost` (raw) + `materialActual` (computed) + `lineMarkupRate` can coexist — store all three, never collapse.

### purchase_order_items (14 cols) — PO line home
`id, purchaseOrderId, estimateItemId, sortOrder, itemCode, manufacturer, description, size, color, qty, unit, costPerUnit, amount, notes`.
- Cost-per-unit column here is **`costPerUnit`** (NOT `unitCost`).

### purchase_orders (29 cols) — full ship-to confirmed
`shipToName, shipToAddressLine1, shipToAddressLine2, shipToCity, shipToState, shipToZip, shipToContact, shipToPhone` all present, plus `supplierId/Name`, `installerId/Name`, `freightAmount, taxAmount, subtotalAmount, totalAmount`. Supports the three-party PO (manufacturer → job site).

### proposals (35 cols)
`architectId, plansDate, bidAmount, contractAmountActual` confirmed. Also already present: labor-actuals fields `estMaterial/estLabor/estOther` + `actMaterial/actLabor/actOther` (Module 8), and follow-up fields `status, nextAction, nextActionDate, lostReason, lostToPrice, wonByPrice, lastTouched` (Modules 7/9). These modules need no new columns.

### estimates (38 cols)
Flat header fields incl. `projectAddressLine1..Zip` (ship-to source), `gcContactName/Email/Phone`, and the FLAT architect fields `architectName, architectAddress, architectEmail, architectPhone`. Also `taxRate, markupRate, baseBidTotal, alternatesTotal, plansDated, projectNumber`.

### architects / proposal_scenarios / proposal_revisions
- `architects`: full firm record (name, address, contact*) — authoritative architect home, linked via `proposals.architectId`.
- `proposal_scenarios`: `scenarioName, isSubmitted, isBaseScenario, detectedFrom, detectionText, priceDelta` — multi-block AND multi-GC support (Modules 4/5).
- `proposal_revisions`: `revisionNumber, addendumRef, addendumDate, summary, acknowledgedAt` — change-order/addendum support (Module 4).

---

## MANDATORY column-name mapping facts (every module obeys)

1. **unitCost ≠ costPerUnit.** Cost-per-unit is `estimate_items.unitCost` but `purchase_order_items.costPerUnit`. PO generation READS `unitCost`, WRITES `costPerUnit`. Assuming they match = silent zero-cost POs = v10's bug. (Verified: `estimate_items` has no `costPerUnit`; `purchase_order_items` has no `unitCost`.)

2. **Architect data lives in TWO places.** `architects` table (FK `proposals.architectId`) AND flat `estimates.architectName/architectAddress/architectEmail/architectPhone`. RESOLVED: do NOT pick authoritative up front — capture both with provenance, defer the choice — see Authoritative-Source Policy (D1) below. Never let them diverge silently; a divergence is a Review-List flag.

3. **plansDate naming divergence (NEW, found in Module 1).** `proposals.plansDate` AND `estimates.plansDated` both exist (note the trailing "d"). Same divergence risk as the architect duplication. RESOLVED: capture both with provenance, defer authoritative choice — see Authoritative-Source Policy (D2) below.

4. **projectNumber duplicated (NEW, found in Module 1).** Present on both `estimates.projectNumber` and `proposals.projectNumber`. Store as TEXT (malformed values like `51977.00`, `R1.204/28/23`). RESOLVED: capture both with provenance, defer authoritative choice — see Authoritative-Source Policy (D3) below.

---

## RETIREMENTS (do not write to these)

- **`line_items` ghost table — RETIRED.** Confirmed live (12 cols: `type, item, vendor, qty, unit, estUnit, estTotal, actUnit, actTotal, status`) as a parallel/competing line store alongside `estimate_items`. Project 3 uses ONLY `estimate_items`. Nothing writes to `line_items`.
- **Baked-cost columns — RETIRED (live but never written).** `estimate_items.usedMaterial, multiplier, deduction, height`. Confirmed present; treat as dead. Store raw `unitCost` + computed `materialActual` + `lineMarkupRate` instead.

---

## AUTHORITATIVE-SOURCE POLICY (RESOLVED — capture both, decide on data)

**Decision (Peter, Module 1):** Do NOT choose an authoritative source up front. Choosing before we have seen how the values behave across ~500 files would be guessing, which the Prime Directive forbids. Instead, for each of the three duplicated concepts, CAPTURE BOTH SIDES with full provenance on each (file/sheet/cell/read-or-computed/timestamp), then let the data decide:

- **D1 — Architect:** capture BOTH the `architects` table linkage (`proposals.architectId`) AND the flat `estimates.architectName/architectAddress/architectEmail/architectPhone`, each with its own provenance.
- **D2 — Plans date:** capture BOTH `proposals.plansDate` AND `estimates.plansDated`, each with provenance.
- **D3 — Project number:** capture BOTH `proposals.projectNumber` AND `estimates.projectNumber` (TEXT), each with provenance.

**Per-file behavior:**
- Both sides AGREE → no action, silent.
- Both sides DISAGREE → raise a Review-List flag for that file showing BOTH values and their provenance. The accumulating pattern of disagreements across the batch is the EVIDENCE that tells Peter which source is reliable — the authoritative-source decision is then made on data, not assumption.

**Non-blocking, always.** These three never stop a record from importing: both sides are stored regardless, so a disagreement is an ENRICHMENT-lane flag, not a BLOCKING quarantine. They are recorded as three OPEN DECISIONS in the Review List (Module 3), not as gate failures.

---

## Module 1 done-criteria check
- Live schema documented and trusted: YES — every fact verified against the LIVE database (all 18 tables, full column inventory), not just the CSV snapshot. (Earlier "only 4 tables" reading was a 100-row truncation of the column query; `Supabase_Snippet_List_Public_Tables.csv` confirms all 18 exist.)
- `line_items` retired: YES (documented; only `estimate_items` used).
- Column-name mapping facts recorded for every module to obey: YES (the facts above), with 2 newly discovered divergence traps added.
- **Live reconciliation result:** 16/16 documented facts verified TRUE against the live DB, including the critical one: `estimate_items` has `unitCost` and NO `costPerUnit`; `purchase_order_items` has `costPerUnit` and NO `unitCost`.

---

# SAVE-PATH HAND-OFF CONTRACT (Module 4 deliverable)

How a record produced by `ams_xlsx_reader.py` maps into the live tables the Command Center reads. This is the SPEC only — no writes happen here; the save path is built and dry-run as a separate, reviewable step. The reader stays the single trusted engine; the app's `parseBidsheetFile` is RETIRED (see "Why the app's parser retires" below).

## Rule 0 — only verified records are written
- `outcome == IMPORT` (both gates passed) → write per the mapping below.
- `outcome == QUARANTINE` → write NOTHING to the data tables. The file becomes Review-List entries (Module 3) instead. No partial writes, ever.

## A verified BASE proposal writes three things
1. **one `proposals` row** — pipeline/operational fields (status, bidAmount, follow-up + labor-actuals fields already present). `plansDate` and `projectNumber` captured here per D2/D3.
2. **one `estimates` row** — header + tax/markup; flat architect block (`architectName/Address/Email/Phone`), `plansDated`, `projectNumber` — each captured WITH PROVENANCE per D1–D3 (both sides stored; disagreement = non-blocking Review flag, never a silent pick).
3. **N `estimate_items` rows** — one per captured bidsheet line. Cost fields:
   - WRITE **`unitCost`** = the raw per-unit cost read from the sheet. (Verified: estimate_items has `unitCost`, NOT `costPerUnit`.)
   - WRITE `materialActual`, `lineMarkupRate` = the decoded-from-FORMULA markup rate (NOT the lying label).
   - WRITE `qty`, `unit`, `itemCode`, `manufacturer` (registry-resolved), `description`, `sectionCode`, `sectionName`, `freight`, `laborRate`, `sortOrder`, and `source` (flag = migrated, reversible).
   - NEVER write `costPerUnit` (does not exist on this table — silent insert rejection = v10 bug).
   - NEVER write the baked-cost columns `usedMaterial / multiplier / deduction / height` (they exist but are RETIRED; writing baked math is the v1/v10 corruption pattern).

## A verified CHANGE-ORDER file (Module 4) writes the base PLUS revisions
- Write the base proposal exactly as above (the base block reconciles internally to the penny).
- For EACH change order, write **one `proposal_revisions` row**, linked by `proposalId`, carrying:
  - the CO number (e.g. `1703-01`) — DISTINCT per CO even when sheet names collide;
  - the CO's captured total (`co_total`) and its line provenance;
  - `projectNumber` + `plansDate` from the CO sheet.
- COs are NEVER merged into the base and NEVER summed across each other. Base + each CO are distinct linked records.
- **CO reconciliation caveat:** these CO sheets carry NO stated CO total cell. `co_total` is an honest captured sum (section-row lump where present, else sum of item rows), flagged `recon_note = "no stated CO total on sheet — Peter confirms"`. CO totals import as PROVISIONAL pending Peter's confirmation — an enrichment-lane flag, not a claimed verification. If a section shows BOTH a section-row lump AND nonzero itemized rows that disagree, the conflict is surfaced for Peter (lump preferred).

## Idempotency (enforced by the save path, not this spec)
- Re-importing the same file MUST NOT create duplicate rows (hardening rule 9). Keying/upsert strategy is defined when the save path is built; flagged here as a REQUIRED property to prove in the dry-run before any real write.

## Sequencing (decided)
- Populate the database FIRST; the Command Center keeps running on its in-browser `_SEED` until a DELIBERATE, reversible cutover from `_SEED` to live tables. Migrated rows are distinguishable via `estimate_items.source`.

## Why the app's parser retires (evidence, not preference)
- The deployed Command Center inserts `costPerUnit` into `estimate_items`. The live table has NO such column → PostgREST rejects the insert → the app's bidsheet import has been SILENTLY FAILING to store costs. This is the v10 bug, live in production. The verified reader writing `unitCost` is the corrected path. The app keeps its operational UI (pipeline, follow-ups, PO/WO, PDFs) but STOPS parsing bidsheets; it reads clean data, it does not manufacture it.

## Module 4 done-criteria check
- CO file imports base + N CO as DISTINCT LINKED records: YES (Residence Inn → base 293,983.10 + CO 1703-01 = 1,200.00 + CO 1703-02 = 2,970.00).
- Each CO captured with provenance, none merged/summed-across: YES.
- CO recon honesty (no stated total → Peter-confirms flag): YES.
- 12-fixture regression after CO code added: outcomes UNCHANGED (5 IMPORT / 7 QUARANTINE).
- Hand-off field mapping locked against LIVE schema (`unitCost`, not `costPerUnit`): YES.
