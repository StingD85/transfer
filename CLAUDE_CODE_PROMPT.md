# Planscape — Claude Code Implementation Prompt

Paste this entire file into Claude Code. Work through every phase in order.
Commit after every numbered step. Do not skip any step.

---

## Context you must read first

Before writing any code, read the following files in full:

```
CLAUDE.md
StingTools/StingTools.csproj
```

Also read these to understand current data shape:

```
StingTools/Data/PARAMETER_REGISTRY.json     (partial — first 100 lines)
StingTools/Data/TAG_CONFIG_v5_0_CONTAINERS.csv
issues.json
project_bep.json
schedule_4d.json
```

The project is at **Phase 91, item 889**. The Revit plugin is production-complete
(193 C# files, 755+ commands). A full .NET 8 REST API server (`Planscape.API/`)
and plugin-side sync client (`Planscape.PluginSync/`) already exist.

Do NOT re-implement anything already in CLAUDE.md as "Completed".
Do NOT add stubs, TODOs, or NotImplementedException entries.
Every step must compile and pass all existing tests before you commit.

---

## Phase 1 — Resolve documented critical gaps in the plugin

Work through the CRITICAL and HIGH gap table in CLAUDE.md under
"Remaining Future Enhancement Gaps (Phase 78 Triage)".

Check each gap ID below. If it is NOT already marked as done in a later
phase entry, implement it. If it is already done, write a one-line comment
in the commit message confirming it and move on.

### Step 1 — BIM-CDE-APPROVAL-01

File: `StingTools/BIMManager/GapFixCommands.cs`

The `CdeApprovalWorkflowCommand` class must enforce ISO 19650-2 §5.6.
Load `approvals.json` from the BIM directory. Block any SHARED→PUBLISHED
CDE transition where `status == "PENDING"` approvals exist for the document.
Show the pending approver names and requested dates in the TaskDialog.
Write the blocked transition attempt to `STING_WORKFLOW_LOG.json` with
`action = "PUBLISH_BLOCKED"`, `reason = "pending_approvals"`, and the count.

Commit message: `fix(cde): enforce ISO 19650-2 §5.6 approval gate [BIM-CDE-APPROVAL-01]`

---

### Step 2 — BIM-CROSS-LINK-01

File: `StingTools/BIMManager/GapFixCommands.cs`

The `CrossSystemLinkCommand` class must build and persist a live
issue↔revision↔transmittal dependency graph to
`cross_system_links.json` in the BIM directory.

Structure each entry as:
```json
{
  "issue_id": "NCR-0012",
  "linked_revisions": ["REV-003"],
  "linked_transmittals": ["T-0021"],
  "link_type": "causal",
  "linked_at": "2026-04-08T10:00:00Z"
}
```

The command must read `issues.json`, `revisions.json`, and `transmittals.json`,
match on existing `linked_transmittals` / `linked_revisions` / `linked_issues`
arrays already written by the tagging pipeline, and upsert into
`cross_system_links.json` atomically (temp file + replace).

Commit message: `feat(bim): issue↔revision↔transmittal JSON cross-linking [BIM-CROSS-LINK-01]`

---

### Step 3 — BIM-EXCEL-STREAM-01

File: `StingTools/BIMManager/ExcelLinkCommands.cs`

The `ImportFromExcel` command currently loads the entire workbook into memory.
Rows beyond approximately 10 000 cause an `OutOfMemoryException` on machines
with 8 GB RAM and a 200 MB Revit model already loaded.

Refactor the import to use `ClosedXML`'s `IXLRow` enumerator so rows are
processed one at a time without materialising the full sheet.
Add a `StingProgressDialog` that updates every 500 rows and supports
Escape-key cancellation via `EscapeChecker`.
Write failed rows (index + reason) to a sidecar
`excel_import_errors_{timestamp}.csv` at the output path.

Commit message: `perf(excel): streaming row-by-row import for 10K+ worksheets [BIM-EXCEL-STREAM-01]`

---

### Step 4 — BIM-4D-HANDOVER-01

File: `StingTools/BIMManager/SchedulingCommands.cs` and
`StingTools/Data/schedule_4d.json`

The `Scheduling4DEngine` calculates trade sequences but does not link
milestone dates to the DD1–DD4 data drop dates in `project_bep.json`.

Add a method `LinkHandoverMilestones(Document doc)` to `Scheduling4DEngine`
that reads `DD1_DATE`, `DD2_DATE`, `DD3_DATE`, `DD4_DATE` from `project_bep.json`
and writes them as phase milestone records into `schedule_4d.json` under a
`"handover_milestones"` array. Each record must include `code`, `date`,
`days_to_completion` (from today), and `rag` (`green`/`amber`/`red`
based on 0/>0/>–14 days respectively).

Expose a new command `BIM.LinkHandoverMilestonesCommand` in the BIM panel.

Commit message: `feat(4d): link DD1-DD4 handover dates into 4D schedule [BIM-4D-HANDOVER-01]`

---

### Step 5 — TAG-SORT-LEVEL-01

File: `StingTools/Tags/TagOperationCommands.cs`

The `SmartSort` command rebuilds the level elevation map
(`FilteredElementCollector` on levels) on every invocation.
On a 300-level hospital model this takes 800 ms per sort call.

Add a static `ConcurrentDictionary<string, SortedList<double, string>> _levelElevCache`
keyed on `doc.PathName ?? doc.Title`. Populate once and re-use.
Invalidate the entry in `StingToolsApp.OnDocumentClosed` by calling
a new `TagOperationCommands.ClearLevelElevationCache(string docKey)` method.

Commit message: `perf(tags): cache level elevation map per document in SmartSort [TAG-SORT-LEVEL-01]`

---

### Step 6 — BIM-BCF-SYNC-01

File: `StingTools/BIMManager/PlatformLinkCommands.cs`

The existing `BCFImport` command reads a BCF 2.1 ZIP and creates issues
in `issues.json` but does not write changes back when issues are resolved
in STING and re-exported.

Implement bidirectional sync:
1. `BCFImport` — on re-import of a BCF file whose `Guid` already exists in
   `issues.json`, update `status`, `modified_date`, and `resolution_note`
   rather than creating a duplicate entry.
2. `BCFExport` — when exporting, set `<TopicStatus>` from the STING `status`
   field, and include a `<Comment>` element for each `resolution_note`.
3. Write a `bcf_sync_log.json` entry per sync with `direction`, `guid`,
   `previous_status`, `new_status`, and `timestamp`.

Commit message: `feat(bcf): bidirectional BCF 2.1 sync with external tools [BIM-BCF-SYNC-01]`

---

### Step 7 — MEDIUM gaps: sidecar versioning and stale-element warnings

These two are small but block a clean handover.

**BIM-SIDECAR-VER-01** — Add a `"schema_version": 2` field to the top of every
JSON sidecar written by the plugin (`issues.json`, `transmittals.json`,
`revisions.json`, `cross_system_links.json`, `approvals.json`).
Add a `SidecarVersion` constant to a new `StingTools/Core/SidecarSchema.cs`
file. Readers that encounter a missing or lower version must log a
`StingLog.Warn` and continue rather than throwing.

**TAG-STALE-WARN-01** — In `StingStaleMarker.Execute()`, after marking an
element stale, check `issues.json` for an existing OPEN issue whose
`element_ids` array contains the element's integer ID. If none exists,
call `BIMManagerEngine.CreateIssue()` with
`type = "DATA_ISSUE"`, `priority = "LOW"`,
`title = $"Stale element: {el.Category?.Name} {el.Id.Value}"`.
Throttle to one issue per element per 24 hours using a static
`ConcurrentDictionary<long, DateTime> _staleIssueThrottle`.

Commit message: `feat(core): sidecar versioning + stale-element auto-issues [BIM-SIDECAR-VER-01, TAG-STALE-WARN-01]`

---

## Phase 2 — Wire the three unused data files into live commands

These files exist in `StingTools/Data/` but are not used by any command.
Implement each as a standalone command that can be run independently
and also called from `MasterSetupCommand` (step 10 of the existing
15-step setup sequence).

### Step 8 — CATEGORY_BINDINGS.csv → CSV-driven parameter binding

File: new command in `StingTools/Tags/LoadSharedParamsCommand.cs`
(append to existing file, do not replace it).

Add `LoadSharedParamsCsvDrivenCommand` (class name, register in ribbon
and dock panel under Tags > Setup, tag `"Tags.LoadSharedParamsCsvDriven"`).

The command must:
1. Load `CATEGORY_BINDINGS.csv` via `StingToolsApp.FindDataFile()`.
2. Parse columns: `ParameterName`, `CategoryName`, `BindingType`
   (`Instance` or `Type`), `GroupName`.
3. Bind each parameter to the listed category using the shared param file
   path from `project_config.json["shared_param_file"]`.
4. Skip parameters already bound (check `doc.ParameterBindings`).
5. Report bound / skipped / failed counts in a `StingResultPanel`.

Replace the hardcoded category list in the existing `LoadSharedParamsCommand`
with a call to this new method so the two remain in sync.

Commit message: `feat(tags): CSV-driven shared parameter binding from CATEGORY_BINDINGS.csv`

---

### Step 9 — FAMILY_PARAMETER_BINDINGS.csv → family parameter injection

File: `StingTools/Tags/FamilyParamCreatorCommand.cs`

Add `FamilyParamCsvBatchCommand`. Register under Tags > Setup,
tag `"Tags.FamilyParamCsvBatch"`.

The command must:
1. Load `FAMILY_PARAMETER_BINDINGS.csv`. Columns: `FamilyName`,
   `ParameterName`, `ParameterType`, `IsInstance`, `GroupName`.
2. For each family in the document whose `Family.Name` matches a row,
   open the family document (via `doc.EditFamily()`), inject the parameter
   using the existing `FamilyParamEngine.InjectSharedParams()` method,
   load the family back, and close the family editor.
3. Show a `StingProgressDialog` updated per family.
4. Write a `family_param_injection_{timestamp}.csv` audit file.

Commit message: `feat(tags): CSV-driven family parameter injection from FAMILY_PARAMETER_BINDINGS.csv`

---

### Step 10 — SCHEDULE_FIELD_REMAP.csv → replace hardcoded remaps

File: `StingTools/Temp/ScheduleCommands.cs`

The `ScheduleHelper` class contains a hardcoded `Dictionary<string, string>`
of deprecated field names mapped to current ones. This dictionary duplicates
`SCHEDULE_FIELD_REMAP.csv` and has drifted from it.

1. Add `ScheduleHelper.LoadFieldRemaps(string dataDir)` that reads
   `SCHEDULE_FIELD_REMAP.csv` (columns: `OldFieldName`, `NewFieldName`,
   `ScheduleType`) and returns a `Dictionary<string, string>`.
2. Replace the hardcoded dictionary in `ScheduleHelper` with a lazy
   static field populated by `LoadFieldRemaps` on first use.
3. Add a `ScheduleFieldRemapAuditCommand` that prints a table of all
   remaps and flags any `OldFieldName` still present in the model's
   schedules. Register under Temp > Schedule Mgr, tag `"Temp.ScheduleFieldRemapAudit"`.

Commit message: `refactor(schedules): replace hardcoded field remaps with CSV-driven SCHEDULE_FIELD_REMAP.csv`

---

## Phase 3 — Mobile app scaffold (React Native)

### Step 11 — Initialise the mobile app project

From the repository root:

```bash
npx create-expo-app StingMobile --template blank-typescript
cd StingMobile
npx expo install expo-router expo-secure-store expo-file-system \
  expo-notifications @tanstack/react-query axios zustand
```

Create `StingMobile/README.md` describing the app architecture:
- Expo Router (file-based routing)
- TanStack Query (server state)
- Zustand (local/offline state)
- Base URL read from `EXPO_PUBLIC_API_URL` environment variable

Commit message: `feat(mobile): initialise React Native / Expo project scaffold`

---

### Step 12 — Authentication screen

File: `StingMobile/app/(auth)/login.tsx`

Implement a login screen that:
1. Accepts `email` and `password`.
2. Calls `POST /api/auth/login` on the Planscape server.
3. Stores the returned JWT and refresh token using `expo-secure-store`
   under keys `planscape_token` and `planscape_refresh`.
4. On success, navigates to `/(tabs)/dashboard`.
5. Shows an inline error message on 401.

Create `StingMobile/lib/api.ts` — an Axios instance that reads
`EXPO_PUBLIC_API_URL`, attaches the JWT from SecureStore as
`Authorization: Bearer {token}`, and on 401 attempts one token refresh
before failing.

Commit message: `feat(mobile): login screen + JWT auth with secure token storage`

---

### Step 13 — Project dashboard tab

File: `StingMobile/app/(tabs)/dashboard.tsx`

Fetch `GET /api/projects` and render a flat list of projects.
Each card shows: project name, compliance percentage (from
`GET /api/projects/{id}/dashboard`), count of OPEN issues,
and a RAG dot derived from compliance (green ≥ 80%, amber ≥ 60%, red < 60%).

Tapping a card navigates to `/(tabs)/project/[id]`.

Use TanStack Query with a 60-second stale time. Show a skeleton loader
while data is loading.

Commit message: `feat(mobile): project dashboard tab with compliance RAG`

---

### Step 14 — Element browser

File: `StingMobile/app/(tabs)/project/[id]/elements.tsx`

Fetch `GET /api/tagsync/elements?projectId={id}` with optional
`?level=`, `?discipline=`, and `?search=` query params.

Render a searchable `FlatList`. Each row shows: element TAG1 value,
category, level, and a coloured discipline badge.

Tapping a row navigates to `/(tabs)/project/[id]/element/[elementId]`
which shows all tag token values (DISC, LOC, ZONE, LVL, SYS, FUNC,
PROD, SEQ) and all container values (TAG1–TAG6) in a read-only detail view.

Commit message: `feat(mobile): element browser with tag detail view`

---

### Step 15 — Issue creation screen

File: `StingMobile/app/(tabs)/project/[id]/issues/new.tsx`

Form fields: Title, Type (picker from the 19 BCF/NEC/JCT types),
Priority (CRITICAL / HIGH / MEDIUM / LOW), Description,
Location (free text), Assignee (picker from
`GET /api/projects/{id}/members`).

Photo capture: `expo-image-picker` — up to 3 photos.
Each photo is uploaded via `POST /api/projects/{id}/issues/{iid}/attachments`
immediately after issue creation.

On submit: `POST /api/projects/{id}/issues`.
On success: navigate back to the issue list and show a toast.

Commit message: `feat(mobile): issue creation with photo attachments`

---

### Step 16 — Completeness dashboard (mobile mirror)

File: `StingMobile/app/(tabs)/project/[id]/compliance.tsx`

Fetch `GET /api/projects/{id}/compliance` (latest result).

Render:
- Overall compliance percentage in a large circular progress indicator.
- Per-discipline breakdown as a horizontal bar chart
  (use `react-native-svg` or `victory-native`).
- Trend sparkline from `GET /api/projects/{id}/compliance?history=10`.

Commit message: `feat(mobile): compliance dashboard with discipline breakdown and trend`

---

### Step 17 — Offline queue

File: `StingMobile/lib/offlineQueue.ts`

Implement a persistent queue backed by `expo-file-system`
(write to `FileSystem.documentDirectory + 'offline_queue.json'`).

Exported functions:
- `enqueue(item: QueueItem)` — append to queue file atomically.
- `flush(api: AxiosInstance)` — read queue, attempt each item,
  remove successfully sent items, leave failed ones.
- `getPendingCount()` — return queue length.

Call `flush()` from the TanStack Query `onlineManager` callback
when the device comes back online (use `@react-native-community/netinfo`).

Show a persistent banner at the top of every tab when
`getPendingCount() > 0` showing the pending count and a Sync button.

Commit message: `feat(mobile): offline queue with auto-flush on reconnect`

---

### Step 18 — Push notification registration

File: `StingMobile/lib/notifications.ts`

On app startup (in `StingMobile/app/_layout.tsx`):
1. Request notification permissions via `expo-notifications`.
2. Get the Expo push token.
3. Call `POST /api/notifications/subscribe` with
   `{ token, platform: "fcm" | "apns" }`.
4. Listen for incoming notifications and navigate to the relevant
   issue or project screen.

Commit message: `feat(mobile): push notification registration and routing`

---

## Phase 4 — Close remaining server gaps

These items are in the server (`Planscape.API/`) which already exists.
Read `src/Planscape.API/` before modifying anything.

### Step 19 — Full-sync endpoint performance

The `POST /api/tagsync/sync` endpoint accepts a `PluginSyncPayload` and
processes elements sequentially. On a 50 000-element model the endpoint
times out at the default 30-second Kestrel limit.

1. Add `"RequestTimeout": "00:05:00"` to `appsettings.json`.
2. Refactor the element upsert loop in `TagSyncController` to use
   `Parallel.ForEachAsync` with a degree of parallelism capped at 4
   (to avoid SQLite write-lock contention on the default SQLite dev DB).
3. Wrap the entire sync in a single `DbContext` `SaveChangesAsync` call
   at the end rather than one per element.
4. Return `{ "synced": N, "skipped": M, "errors": [] }` in the response body.

Commit message: `perf(server): batch element upsert in tagsync endpoint`

---

### Step 20 — Compliance trend endpoint

File: `src/Planscape.API/Controllers/ComplianceController.cs`

Add `GET /api/projects/{id}/compliance?history={n}` where `n` defaults to 10.
Return the last `n` compliance scan results as an array ordered oldest-first:
```json
[
  { "scanned_at": "...", "overall_pct": 78.4, "by_discipline": { "M": 82, "E": 74 } }
]
```

The endpoint must be tenant-isolated (check project ownership before returning).

Commit message: `feat(server): compliance history endpoint for trend sparkline`

---

### Step 21 — Global search type filter

The existing `GET /api/search?q=...` endpoint returns mixed results.
The mobile app needs type-filtered responses.

Add an optional `?type=elements|issues|documents` query parameter.
When `type` is provided, filter results to that entity type only.
When absent, return all types interleaved sorted by relevance score.

Commit message: `feat(server): type-filtered global search endpoint`

---

### Step 22 — Project members endpoint for mobile picker

The mobile issue creation screen needs assignee names.

Add `GET /api/projects/{id}/members` returning:
```json
[{ "id": "uuid", "name": "Alice Lee", "role": "BIM Coordinator", "discipline": "M" }]
```

Read from the `ProjectMember` table. Tenant-isolated.

Commit message: `feat(server): project members list endpoint for mobile assignee picker`

---

## Phase 5 — Tests

### Step 23 — Unit tests for Phase 1 gap fixes

File: `tests/Planscape.Tests/GapFixTests.cs` (create if absent)

Write tests for:
- `BIM-CDE-APPROVAL-01`: a mock approval list with one PENDING entry blocks
  the publish transition; zero pending entries allows it.
- `BIM-CROSS-LINK-01`: given seed `issues.json` + `transmittals.json`,
  the cross-link builder produces the expected JSON structure.
- `BIM-EXCEL-STREAM-01`: streaming import of a 15 000-row workbook
  completes without `OutOfMemoryException` and reports the correct
  row count.
- `TAG-STALE-WARN-01`: the throttle dictionary prevents a second
  issue being created for the same element within 24 hours.

Commit message: `test: unit tests for Phase 1 gap fix implementations`

---

### Step 24 — Unit tests for data-file commands (Phase 2)

File: `tests/Planscape.Tests/DataFileCommandTests.cs`

Write tests for:
- `LoadSharedParamsCsvDrivenCommand`: parsing a 5-row
  `CATEGORY_BINDINGS.csv` fragment produces the correct binding map.
- `ScheduleHelper.LoadFieldRemaps`: parsing a 3-row
  `SCHEDULE_FIELD_REMAP.csv` returns a dictionary with the right keys.
- `FamilyParamCsvBatchCommand`: a CSV with two families produces
  two `InjectSharedParams` calls (mock `FamilyParamEngine`).

Commit message: `test: unit tests for CSV-driven data file commands`

---

### Step 25 — Integration smoke test for the mobile API

File: `tests/Planscape.Tests/MobileApiSmokeTests.cs`

Using `Microsoft.AspNetCore.Mvc.Testing.WebApplicationFactory`:
- `POST /api/auth/login` with demo credentials returns 200 and a token.
- `GET /api/projects` with the token returns a non-empty list.
- `GET /api/projects/{id}/compliance?history=5` returns an array of ≤5 items.
- `GET /api/projects/{id}/members` returns a list with at least one entry.
- `GET /api/search?q=test&type=issues` returns only issue results.

Commit message: `test: integration smoke tests for mobile API endpoints`

---

## General rules for all steps

- Read `CLAUDE.md` conventions before touching any C# file.
- Use `StingToolsApp.FindDataFile(fileName)` for all data file lookups.
- Use `OutputLocationHelper.GetOutputPath(doc, ...)` for all file writes.
- Match the existing transaction pattern:
  `[Transaction(TransactionMode.Manual)]` for writes,
  `[Transaction(TransactionMode.ReadOnly)]` for reads.
- New commands must be registered in `StingCommandHandler.cs` and in
  the dock panel XAML. Follow the pattern of the nearest existing command.
- Never force-push. Never use `git reset --hard` or `git clean -f`
  without confirmation.
- Run `dotnet build StingTools/StingTools.csproj` after every step
  and fix all errors before committing.
- Commit message format: `type(scope): description [GAP-ID if applicable]`
