# Power Automate: Sync an On-Prem File Share to SharePoint

This guide walks through building a Power Automate flow that keeps a
SharePoint document library/folder in sync with a file share that is only
reachable via an on-premises data gateway.

## Architecture overview

```
On-prem / network file share (\\server\share\path)
        │  (File System connector - routed through On-Premises Data Gateway)
        ▼
   Power Automate flow
        │  (SharePoint connector - cloud, no gateway needed)
        ▼
SharePoint Online (destination folder)
```

- **File share side (source)**: uses the **File System** connector, which
  requires the **On-premises Data Gateway** because the share is not
  internet-reachable.
- **SharePoint side (destination)**: uses the built-in **SharePoint**
  connector (cloud-to-cloud, no gateway required).

---

## Prerequisites

1. **Gateway host machine**: A Windows machine/VM that has network access to the
   source file share (`\\server\sharename\folder`) and stays powered on/online.
2. **On-premises Data Gateway installed** on that machine, registered under the
   same Microsoft 365 / Entra ID tenant as your Power Automate environment.
   - Download: Power Automate portal → **Settings (gear) → On-premises data
     gateway → Install gateway**.
   - During setup you'll create/join a gateway and set a **recovery key** —
     store this somewhere safe (needed to add more admins or recover config).
3. **Service account** with at least **read** NTFS permissions on the source
   share, used to run the gateway service and the File System connection.
4. **Permissions**:
   - You (or the flow owner) must have **Write** access to the SharePoint
     library/folder being synced to.
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
   - **Username / Password**: the service account with read access to that
     share (format `DOMAIN\svc-account`).
4. Save. The connection will validate against the gateway; fix any
   credential/network errors before continuing.

## Step 3 — Create/confirm the SharePoint connection

- The **SharePoint** connector uses your normal Microsoft 365 sign-in — no
  gateway needed. It's created automatically the first time you add a
  SharePoint action in the flow (Step 4).

## Step 4 — Build the flow

There are two common patterns depending on how "sync" should behave. Pick
one based on your needs.

### Option A — Event-driven (near real-time), single folder, no subfolders

Best when the source is one flat folder and you want fast propagation.

1. **Create** → **Automated cloud flow**.
2. Trigger: **When a file is created (properties only)** — and/or add a
   parallel flow with **When a file is modified (properties only)**
   (File System connector, connection from Step 2).
   - **Folder**: the on-prem folder to watch, relative to the connection's
     root folder (e.g. `/ExportFolder`).
   - This trigger **polls** the share through the gateway on an interval you
     set (default every few minutes) — it is not instant push notification.
3. Add action: **Get file content** (File System) — use the trigger's
   `Path` output as the file identifier.
4. Add action: **Create file** (SharePoint) — connection from Step 3.
   - **Site Address**: your SharePoint site.
   - **Folder Path**: the destination library/folder, e.g.
     `/Shared Documents/ExportFolder`.
   - **File Name**: the trigger's file name (e.g. `{DisplayName}`/`{Name}`).
   - **File Content**: output of **Get file content**.
5. Because SharePoint's **Create file** action will happily overwrite an
   existing file by default in most cases, but you may still want
   update-vs-create logic (e.g. to preserve version history intentionally or
   branch behavior):
   - **Get file metadata** (SharePoint) on the target path → **Condition**:
     if it exists, use **Update file**; otherwise use **Create file**.
     Configure "Get file metadata"'s **Configure run after** to continue
     even when it fails (file not found), then branch on that.
6. Save and test by adding/editing a file on the on-prem share.

> Limitation: this trigger only reports created/modified files on a polling
> interval, not deletions, and it doesn't recurse into subfolders by default.

### Option B — Scheduled full/incremental sync (recommended for folder trees)

Best when you need subfolders, deletions handled, or a periodic reconciliation
job instead of relying on the polling trigger for every change.

1. **Create** → **Scheduled cloud flow**. Set recurrence, e.g. every 15/30/60
   minutes.
2. Action: **List files in folder** (File System), pointed at the source
   folder on the share.
   - This action is **not recursive**. To handle nested folders, use a
     **Do until** loop with a queue of folder paths, or call a
     **Child flow** recursively (Power Automate supports calling a flow
     from another flow for recursion) — recurse using
     **List files in folder** on each discovered subfolder.
3. **Apply to each** file returned:
   a. **Get file content** (File System), using the file's `Path`.
   b. **Get file metadata** (SharePoint) at the mirrored destination path —
      configure run-after to catch "not found" as a non-terminating branch.
   c. **Condition**: compare the file share's `LastModified` timestamp to
      SharePoint's `Modified` field (skip the copy if the SharePoint copy is
      already current — this keeps it an *incremental* sync).
   d. If new or newer: **Create file** or **Update file** (SharePoint) to
      write the content to the mirrored path.
4. (Optional, for true mirroring) Add a step to detect files present in
   SharePoint but no longer on the source file share, and delete them via
   **Delete file** (SharePoint). This requires listing the SharePoint
   library contents too (**Get files (properties only)** or a REST call)
   and diffing the two lists in a **Compose**/**Filter array** step.
5. Save and do a manual **Test → Manually** run first before relying on the
   recurrence trigger.

---

## Step 5 — Preserve folder structure (if syncing a tree)

- Compute the relative path once by trimming the File System connection's
  root folder off each file's full path, e.g.:
  `substring(items('Apply_to_each')?['Path'], length('/ExportFolder/'))`
- Use that relative path when calling **Create file/Update file** (SharePoint)
  so the destination library mirrors the same subfolder layout.
- If a subfolder doesn't exist yet in SharePoint, add a **Create new folder**
  (SharePoint) call guarded by a "does folder exist" check (same pattern as
  Step 4A.5), before writing files into it.

## Step 6 — Error handling & resiliency

1. Wrap the core actions in a **Scope** named `Try`, followed by a second
   **Scope** named `Catch` configured to run after `Try` has **failed,
   timed out, or been skipped**.
2. In `Catch`, add a **Send an email (V2)** or **Post message in Teams**
   action to notify an admin, including `result('Try')` for diagnostics.
3. Turn on flow-level **run-after settings** so one file's failure (e.g., a
   locked file on the share) doesn't stop the whole batch — set "Configure
   run after" on downstream actions inside the loop to continue on failure,
   and log failures to a **SharePoint list** or **Compose**/**Append to
   array variable** for a summary at the end.
4. Set the flow's **Concurrency Control** (on the "Apply to each") if you
   need to throttle how many files are processed in parallel — useful to
   avoid overloading the gateway/file share, and to respect SharePoint
   throttling limits.

## Step 7 — Test

1. Add/modify a test file on the on-prem file share.
2. Run the flow manually (or wait for the trigger/recurrence).
3. Check **Flow run history** for success, and confirm the file appears
   correctly in the SharePoint destination folder.
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
| Files don't sync immediately | File System triggers are polling-based, not instant push | Set a shorter polling interval, or fall back to the scheduled-flow pattern (Option B) for tighter control |
| Only top-level files sync, subfolders ignored | `List files in folder` action isn't recursive | Use a `Do until` loop with a folder queue, or child-flow recursion (Option B) |
| Deleted file-share files remain in SharePoint | Sync only handles create/update | Add a reconciliation/diff step to delete orphaned SharePoint files (Step 4B.4) |
| Flow times out on large shares | Loop iterating too many items serially | Enable concurrency control, or paginate the `List files in folder` results |
| "Unauthorized" on File System connection | Service account lacks share permissions, or password rotated | Re-enter credentials on the connection; verify NTFS + share permissions |
| SharePoint throttling (429 errors) | Too many rapid create/update calls | Add concurrency limits and retry policies on the SharePoint actions |
