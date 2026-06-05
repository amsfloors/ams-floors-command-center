# Module 7 — Cutover to Live Data (2026-06-05)

**Status: DONE.** The command center (`index.html`) reads only the live Supabase
tables. The corrupting in-app bidsheet import path has been removed.

---

## Deploy / hosting (corrected)
- The app deploys via **Netlify**, project **`dapper-faun-fb14a0`**
  (`dapper-faun-fb14a0.netlify.app`), auto-building from GitHub `main`.
  (Not GitHub Pages.)

## What was verified before any edit
- Live `estimate_items` cost columns per SCHEMA_TRUTH: `unitCost`, `materialActual`,
  `lineMarkupRate` present; **no `costPerUnit`** on the table.
- Migration clean: 2,722 line items; 2,628 have a real `unitCost`
  (avg $41.90, range $0–$4,718.05). The ~94 without are header/labor lines.
- App already read live (proposals, estimates, estimate_items, line_items, suppliers,
  installers, gcs, app_settings) + realtime subscription. `_SEED` was bootstrap/offline only.

## What changed in index.html (4 edits)
1. **Removed `parseBidsheetFile`** — retired parser; built `costPerUnit` for `estimate_items`.
2. **Removed `BulkIntakeView`** — in-app bulk import UI; its `approve()` wrote `costPerUnit`
   into `estimate_items`. Removed wholesale (UI + render in Settings).
3. **Trimmed Settings "Import / Export"** — dropped the bulk-import button + panel;
   kept **Add Single Project** and **Export Everything to Excel**.
4. **Neutralized the `_SEED` auto-insert** — empty query no longer inserts `_SEED`;
   logs an error and shows `_SEED` read-only. The `_SEED` array is kept (offline + rollback).

## Kept on purpose (do not remove)
- `_SEED` array (offline fallback + rollback snapshot)
- `window.genSupplierPOs` (PO generator, Module 12)
- Installer Work Order generator (Module 13)
- All live Supabase reads + realtime subscription

## Live verification (post-deploy)
- Netlify published `main` commit at 10:51 AM, 0 errors.
- Dashboard: $13.17M open pipeline / 154 active bids; "Live sync" on.
- Pipeline: 154 rows · $13.17M, real names/amounts.
- Estimate totals reconcile to the penny (#1535.3 = $49,928; #1313 = $12,179).

## Open follow-ups (NOT part of this cutover)

### A. Status assignment — source of truth is the OneDrive FOLDER
- Status is **not** in the bidsheets and **not** reliably in `_SEED`. It is encoded by
  **which OneDrive folder a project file lives in**:
  `1. Current Projects`→Active · `2. Currently Bidding`→Negotiating ·
  `3. Submitted Proposals`→Submitted · `4. Completed Projects`→Won ·
  `5. Dead Projects`→Dead/Lost.
- All 154 currently = "Submitted" (placeholder).
- Proposal numbers AND names are unique; filenames carry the number → folder→id match
  is near-automatic. Use `_SEED` as cross-check only; never bulk-map `_SEED` onto the DB.
- Next-task first step (run in its own chat, confirm paths first):
  ```
  Get-ChildItem -Path "<...>\1. Current Projects","<...>\2. Currently Bidding",
    "<...>\3. Submitted Proposals","<...>\4. Completed Projects","<...>\5. Dead Projects"
    -Recurse -File |
    Select-Object @{n='Folder';e={$_.Directory.Name}}, Name |
    Export-Csv -Path "$HOME\status_map.csv" -NoTypeInformation
  ```

### B. Module 12 — COST column / PO generator read the wrong field
- App reads `item.costPerUnit` (absent on migrated rows) → COST column blank, POs would
  read $0. Data is in `unitCost`; line totals already reconcile.
- Fix ~5 spots to fall back to `unitCost`, AND redirect the COST cell's edit-write to
  `unitCost` (it currently writes the non-existent `costPerUnit`). Re-pass all 5 fixtures.

## Reversibility
This cutover is one commit. Revert it on GitHub `main`
(`https://github.com/amsfloors/ams-floors-command-center/commits/main`) → Netlify
redeploys. No DB schema/rows changed, so reverting `index.html` fully restores prior behavior.
