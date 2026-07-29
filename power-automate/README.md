# Power Automate flow packages (best-effort, placeholder-based)

This folder contains two importable flow package attempts, plus their raw
flow definitions, for syncing an on-prem file share (via On-Premises Data
Gateway) into a SharePoint document library. See
`../docs/power-automate-sharepoint-to-fileshare-sync.md` for the full
narrative walkthrough these are based on.

## Files

| File | Contents |
|---|---|
| `FileShareToSharePointSync-EventDriven.zip` | Import package for **Option A** (trigger on file created, single flat folder) |
| `FileShareToSharePointSync-Scheduled.zip` | Import package for **Option B** (recurrence + list/compare/write, skips already-current files) |
| `eventdriven-definition.json` | Raw Workflow Definition Language JSON for Option A (same content as inside the zip) |
| `scheduled-definition.json` | Raw Workflow Definition Language JSON for Option B (same content as inside the zip) |

## Important caveat

Power Automate's package `.zip` format is not officially published by
Microsoft — it's reverse-engineered here from known real-world exports
(`manifest.json` + `Microsoft.Flow/flows/<guid>/definition.json` +
`apiConnectionReferences.json`). The `definition.json` payload itself uses
the well-documented Workflow Definition Language (shared with Azure Logic
Apps Consumption), so the trigger/action/expression logic is trustworthy —
but the surrounding package wrapper may not match exactly what the "Import
Package (Legacy)" feature expects, and specific connector `operationId`s
inside `definition.json` were written from memory of the **File System**
and **SharePoint** connector schemas rather than validated against a live
tenant.

**What to do:**
1. Try importing the `.zip` first (see below). Importing is non-destructive
   — if it fails validation, nothing is created and you just fall back to
   step 2.
2. If import fails (or actions show up broken/unrecognized), open the
   corresponding `*-definition.json` file as a reference and manually
   recreate the trigger/actions in the designer using
   `docs/power-automate-sharepoint-to-fileshare-sync.md` — the JSON tells
   you exactly which connector, operation, and expression to use for each
   step, which is faster than building from scratch.
3. Either way, **before/after import**, replace these placeholders (find &
   replace across the files, or edit the corresponding fields directly in
   the designer after import):

   | Placeholder | Replace with |
   |---|---|
   | `<<SOURCE_FOLDER>>` | Path on the file share, relative to your File System connection's root folder, e.g. `/ExportFolder` |
   | `<<SITE_URL>>` | Your SharePoint site address, e.g. `https://yourtenant.sharepoint.com/sites/YourSite` |
   | `<<DEST_FOLDER>>` | Destination library/folder in SharePoint, e.g. `/Shared Documents/ExportFolder` |

   Also note: on import, Power Automate will always prompt you to select or
   create the **File System** connection (through your On-Premises Data
   Gateway) and the **SharePoint** connection — this step is normal and
   expected even for a "real" exported package, not a symptom of anything
   being wrong with the file.

## How to attempt the import

1. Go to [make.powerautomate.com](https://make.powerautomate.com) → **My flows**.
2. Click **Import** → **Import Package (Legacy)**.
3. Upload the `.zip` file.
4. Under the package's resource list, set connections for `shared_filesystem`
   (pick your gateway-backed File System connection) and
   `shared_sharepointonline`.
5. Click **Import**.
6. Open the imported flow in edit mode, replace the `<<...>>` placeholders in
   each action's inputs with your real values, and save.
7. Follow **Step 6/7** of the main doc (error handling, testing) before
   turning on/relying on the flow.

## What's intentionally left out of the generated definitions

- **Recursive subfolder traversal** (Option B, doc step 4B.2) is not encoded
  here — `List_files_in_folder` only lists the single `<<SOURCE_FOLDER>>`
  level. Add the child-flow or `Do until` recursion described in the doc if
  you need nested folders.
- **Delete reconciliation** (removing files from SharePoint that were
  deleted on the file share) is not included — add per doc step 4B.4.
- **Error-handling Scope/Catch blocks and notifications** (doc step 6) are
  not included, to keep the generated definition focused on the core
  copy logic.
