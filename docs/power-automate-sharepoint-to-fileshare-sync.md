# Power Automate: Sync a SharePoint Path to an On-Prem File Share

This guide walks through building a Power Automate flow that keeps a file share
(reachable only via an on-premises data gateway) in sync with a document
library/folder in SharePoint Online.

## Architecture overview

```
SharePoint Online (source folder)
        │  (SharePoint connector - cloud, no gateway needed)
        ▼
   Power Automate flow
        │  (File System connector - routed through On-Premises Data Gateway)
        ▼
On-prem / network file share (\\server\share\path)
```

- **SharePoint side**: uses the built-in **SharePoint** connector (cloud-to-cloud, no gateway required).
- **File share side**: uses the **File System** connector, which requires the
  **On-premises Data Gateway** because the share is not internet-reachable.

---

## Prerequisites

1. **Gateway host machine**: A Windows machine/VM that has network access to the
   target file share (`\\server\sharename\folder`) and stays powered on/online.
2. **On-premises Data Gateway installed** on that machine, registered under the
   same Microsoft 365 / Entra ID tenant as your Power Automate environment.
   - Download: Power Automate portal → **Settings (gear) → On-premises data
     gateway → Install gateway**.
   - During setup you'll create/join a gateway and set a **recovery key** —
     store this somewhere safe (needed to add more admins or recover config).
3. **Service account** with read/write NTFS permissions on the target share,
   used to run the gateway service and the File System connection.
4. **Permissions**:
   - You (or the flow owner) must have **Read** access to the SharePoint
     library/folder being synced.
   - You must be listed as an **admin or user** of the gateway resource in
     Power Automate (Data → Gateways).
5. **Premium license**: The **File System** connector is a **Premium**
   connector, so a Power Automate per-user/per-flow plan (or a Microsoft 365
   plan that includes premium connectors) is required for the flow owner.

---

## Step 1 — Install and register the On-Premises Data Gateway

1. On the gateway machine, download and run the installer from
   `https://aka.ms/on-premises-data-gateway-installer`.
2. Choose **Register a new gateway on this computer**.
3. Sign in with the Microsoft 365/Entra account that owns/manages this
   environment.
4. Name the gateway (e.g. `GW-FileShareSync-Prod`) and set the recovery key.
5. Confirm it shows **Online/Connected** in the local gateway app.
6. In the Power Automate portal, go to **Data → Gateways** and verify the
   gateway appears there with status **Online**.
7. Under the gateway → **Admins**, add any additional users who need to
   create connections through it, or share it via **Manage users**.

## Step 2 — Create the File System connection (through the gateway)

1. In Power Automate, go to **Data → Connections → New connection**.
2. Search for **File System** and select it.
3. Fill in:
   - **Root folder**: the UNC path the gateway machine can reach, e.g.
     `\\fileserver01\SyncedDocs`
   - **Gateway**: choose the gateway you registered in Step 1.
   - **Username / Password**: the service account with read/write access to
     that share (format `DOMAIN\svc-account`).
4. Save. The connection will validate against the gateway; fix any
   credential/network errors before continuing.

## Step 3 — Create/confirm the SharePoint connection

- The **SharePoint** connector uses your normal Microsoft 365 sign-in — no
  gateway needed. It's created automatically the first time you add a
  SharePoint trigger/action in the flow (Step 4).

## Step 4 — Build the flow

There are two common patterns depending on how "sync" should behave. Pick
one based on your needs.

### Option A — Event-driven (near real-time), single folder, no subfolders

Best when the source is one flat folder and you want fast propagation.

1. **Create** → **Automated cloud flow**.
2. Trigger: **When a file is created or modified (properties only)**
   (SharePoint connector).
   - **Site Address**: your SharePoint site.
   - **List/Library**: the document library.
   - **Folder**: (optional) restrict to a subfolder, e.g. `/Shared Documents/ExportFolder`.
3. Add action: **Get file content** (SharePoint) — use the trigger's
   `Id`/`ItemId` (or `{Identifier}`) as the file identifier.
4. Add action: **Create file** (File System) — connection from Step 2.
   - **Folder Path**: `/` (root of the connection) or a subfolder matching
     the SharePoint folder structure.
   - **File Name**: use the SharePoint trigger's file name field (`{FilenameWithExtension}`).
   - **File Content**: output of **Get file content**.
5. Because "Create file" fails if the file already exists, add error handling:
   - Set **Create file**'s "Configure run after" is not needed; instead use
     **Update file** as a fallback, or better: check first.
   - Recommended pattern: **Get file metadata using path** (File System) on
     the target path → **Condition**: if it exists, use **Update file**;
     otherwise use **Create file**. Configure the "Get file metadata" action
     to **not** fail the flow on a "file not found" error (Settings →
     Configure run after → include "has failed", then branch on that).
6. Save and test by uploading/editing a file in the SharePoint folder.

> Limitation: this trigger only reports created/changed files, not deletions,
> and it doesn't recurse into subfolders by default.

### Option B — Scheduled full/incremental sync (recommended for folder trees)

Best when you need subfolders, deletions handled, or a periodic reconciliation
job instead of firing on every micro-change.

1. **Create** → **Scheduled cloud flow**. Set recurrence, e.g. every 15/30/60
   minutes.
2. Action: **Get files (properties only)** (SharePoint) or
   **Send an HTTP request to SharePoint** with a REST call to
   `_api/web/GetFolderByServerRelativeUrl('/sites/.../Shared Documents/ExportFolder')/Files`
   if you need recursion through subfolders (the simple "Get files" action
   is not recursive).
   - To handle nested folders, use a **Do until** loop with a queue of
     folder paths, or call a **Child flow** recursively (Power Automate
     supports calling a flow from another flow for recursion).
3. **Apply to each** file returned:
   a. **Get file content** (SharePoint), using the file's `Id`/path.
   b. **Get file metadata using path** (File System) at the mirrored path on
      the file share — configure run-after to catch "not found" as a
      non-terminating branch.
   c. **Condition**: compare SharePoint's `Modified` timestamp to the file
      share's `LastModifiedDateTime` (skip the copy if the file share copy is
      already current — this keeps it an *incremental* sync).
   d. If new or newer: **Create file** or **Update file** (File System) to
      write the content to the mirrored path.
4. (Optional, for true mirroring) Add a step to detect files present on the
   file share but no longer in SharePoint, and delete them via
   **Delete file** (File System). This requires listing the file share
   contents too (**List files in folder** – File System) and diffing the two
   lists in a **Compose**/**Filter array** step.
5. Save and do a manual **Test → Manually** run first before relying on the
   recurrence trigger.

---

## Step 5 — Preserve folder structure (if syncing a tree)

- Compute the relative path once by trimming the SharePoint library root off
  each file's server-relative URL, e.g.:
  `substring(triggerOutputs()?['body/{Path}'], length('/Shared Documents/ExportFolder/'))`
- Use that relative path when calling **Create file/Update file** so the
  file share mirrors the same subfolder layout.
- If a subfolder doesn't exist yet on the file share, add a
  **Create folder** (File System) call guarded by a "does folder exist"
  check (same pattern as Step 4A.5), before writing files into it.

## Step 6 — Error handling & resiliency

1. Wrap the core actions in a **Scope** named `Try`, followed by a second
   **Scope** named `Catch` configured to run after `Try` has **failed,
   timed out, or been skipped**.
2. In `Catch`, add a **Send an email (V2)** or **Post message in Teams**
   action to notify an admin, including `result('Try')` for diagnostics.
3. Turn on flow-level **run-after settings** so one file's failure (e.g., a
   locked file) doesn't stop the whole batch — set "Configure run after" on
   downstream actions inside the loop to continue on failure, and log
   failures to a **SharePoint list** or **Compose**/**Append to array
   variable** for a summary at the end.
4. Set the flow's **Concurrency Control** (on the "Apply to each") if you
   need to throttle how many files are processed in parallel — useful to
   avoid overloading the gateway/file share.

## Step 7 — Test

1. Add/modify a test file in the SharePoint folder.
2. Run the flow manually (or wait for the trigger/recurrence).
3. Check **Flow run history** for success, and confirm the file appears
   correctly at the file share path.
4. Test edge cases: large files (check for size limits/timeouts), special
   characters in filenames, nested folders, and (for Option B) deletions.
5. Test failure handling by pointing the File System connection at an
   invalid path temporarily, or by taking the gateway offline, and confirm
   your Catch/notification logic fires.

## Step 8 — Monitor and maintain

- **Data → Gateways**: periodically check status is **Online**; enable
  auto-update on the gateway machine.
- **Connections**: File System / SharePoint connection credentials can
  expire (password rotation) — update the connection, not the flow, when
  that happens.
- **Flow analytics**: use **Flow run history** and, for critical syncs, an
  **Alert** (Power Automate → Monitor) on run failures.
- Consider moving the recurrence interval and folder paths into
  **Environment Variables** so the flow can be promoted across
  Dev/Test/Prod environments without hardcoding paths.

---

## Common pitfalls

| Issue | Cause | Fix |
|---|---|---|
| "The gateway is offline" | Gateway machine sleeping/rebooted/service stopped | Ensure it's a server-class machine, disable sleep, set gateway service to auto-start |
| "Create file" fails with "file already exists" | No existence check before create | Use Get-metadata → Condition → Create/Update pattern (Step 4) |
| Only top-level files sync, subfolders ignored | `Get files` action isn't recursive | Use recursive HTTP REST call or child-flow recursion (Option B) |
| Deleted SharePoint files remain on file share | Sync only handles create/update | Add a reconciliation/diff step to delete orphaned files (Step 4B.4) |
| Flow times out on large libraries | Loop iterating too many items serially | Enable concurrency control, or paginate with `top`/`skip` in the REST call |
| "Unauthorized" on File System connection | Service account lacks share permissions, or password rotated | Re-enter credentials on the connection; verify NTFS + share permissions |
