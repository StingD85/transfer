# Planscape gap fix: implementation prompt

Paste this entire file into Claude Code as your standing instruction. Follow it section by section. Do not skip ahead. Commit after every section before moving to the next. Re-read the relevant section heading and checklist before starting each block of work.

---

## Ground rules

1. Work through sections in order (1, 2, 3 ...). Each section ends with a COMMIT GATE. Do not start the next section until the current one compiles and is committed.
2. At each COMMIT GATE, run `dotnet build` (for C# changes) or the equivalent build/lint step. Fix any errors before committing.
3. Commit messages use the format: `fix(planscape): SECTION_ID — short description`. Example: `fix(planscape): S01 — add GPS and version columns to schema`.
4. If a section says "migration", create an SQL file named `002_<description>.sql` (incrementing from the existing `001_initial_schema.sql`). Also update the EF Core entity classes and DbContext to match.
5. Never delete working code without a replacement wired in. When retiring `StingBIMServerClient`, redirect its callers first.
6. If you are unsure about a step, re-read this prompt from the relevant section heading. Do not guess.
7. After finishing all sections, re-read the full prompt one final time and verify every checklist item is done.

---

## Current state summary

Read `PLANSCAPE_GAPS.md` for full details. The critical facts:

- **Planscape.PluginSync** (`SyncClient.cs`, `OfflineQueue.cs`, `SyncScheduler.cs`) is dead code: never instantiated, never called.
- **StingBIMServerClient** in `PlatformLinkCommands.cs` is the only active sync path: manual trigger, no retry, no offline queue.
- **StingToolsApp.cs** `OnDocumentSaved` queues data into static fields that nothing consumes.
- **TagSyncController** does blind last-write-wins overwrite with no timestamp or version check.
- **Mobile app** is ~40% scaffold: no camera, no GPS, no offline drain, no state management, no SignalR client, QR scanner is a stub.
- **Server** has no image thumbnailing, no GPS fields, no delta sync, no batch replay endpoint, no file versioning, local filesystem storage only.
- **Database** is missing: GPS columns on `bim_issues` and `document_records`, version/timestamp columns on `tagged_elements`, `sync_watermarks` table, `document_versions` table, `sync_conflicts` table.

---

## S01 — Database schema additions

Add migration file `002_gps_versioning_sync.sql`. Add these changes to the existing schema:

### 01a: GPS on issues
```sql
ALTER TABLE bim_issues ADD COLUMN latitude DOUBLE PRECISION;
ALTER TABLE bim_issues ADD COLUMN longitude DOUBLE PRECISION;
ALTER TABLE bim_issues ADD COLUMN location_accuracy DOUBLE PRECISION;
ALTER TABLE bim_issues ADD COLUMN device_id TEXT;
```

### 01b: GPS on documents
```sql
ALTER TABLE document_records ADD COLUMN latitude DOUBLE PRECISION;
ALTER TABLE document_records ADD COLUMN longitude DOUBLE PRECISION;
```

### 01c: Version tracking on elements
```sql
ALTER TABLE tagged_elements ADD COLUMN version INTEGER NOT NULL DEFAULT 1;
ALTER TABLE tagged_elements ADD COLUMN last_modified_utc TIMESTAMPTZ;
```

### 01d: Sync watermarks
```sql
CREATE TABLE sync_watermarks (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    device_id   TEXT NOT NULL,
    last_sync_utc TIMESTAMPTZ NOT NULL,
    element_count INTEGER NOT NULL DEFAULT 0,
    UNIQUE (project_id, device_id)
);
```

### 01e: Document versions
```sql
CREATE TABLE document_versions (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    document_record_id  UUID NOT NULL REFERENCES document_records(id) ON DELETE CASCADE,
    version_number      INTEGER NOT NULL,
    file_path           TEXT NOT NULL,
    file_hash           TEXT,
    file_size           BIGINT,
    uploaded_by         TEXT,
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    change_description  TEXT
);
```

### 01f: Sync conflict log
```sql
CREATE TABLE sync_conflicts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    element_id      TEXT NOT NULL,
    conflict_type   TEXT NOT NULL,
    server_value    JSONB,
    client_value    JSONB,
    resolution      TEXT NOT NULL,
    resolved_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_by     TEXT
);
```

### 01g: Schema version
```sql
INSERT INTO schema_versions (version, description)
VALUES (2, 'GPS coordinates, element versioning, sync watermarks, document versions, conflict log');
```

### 01h: EF Core entities
Update (or create) entity classes to match: `SyncWatermark`, `DocumentVersion`, `SyncConflict`. Add `Latitude`, `Longitude`, `LocationAccuracy`, `DeviceId` properties to the `BimIssue` entity. Add `Latitude`, `Longitude` to `DocumentRecord`. Add `Version` (int, default 1) and `LastModifiedUtc` to `TaggedElement`. Register new DbSets in `PlanscapeDbContext`.

**COMMIT GATE S01**: `002_gps_versioning_sync.sql` exists. All new entity classes compile. `dotnet build` passes. Commit: `fix(planscape): S01 — add GPS, versioning, sync watermark, conflict log schema`.

---

## S02 — TagSync conflict resolution

Replace the blind overwrite in `TagSyncController`. The current code at the sync endpoint does `MapDtoToEntity(dto, existing, request.UserName)` with no timestamp check.

### 02a: Add timestamp comparison
Before overwriting an existing element, compare `dto.LastModifiedUtc` against `existing.LastModifiedUtc`:
- If `dto.LastModifiedUtc > existing.LastModifiedUtc` (or existing has no timestamp): apply the update, increment `existing.Version`, set `existing.LastModifiedUtc = dto.LastModifiedUtc`.
- If `dto.LastModifiedUtc <= existing.LastModifiedUtc`: add to a conflicts list, do not overwrite. Log the conflict to the `sync_conflicts` table.

### 02b: Return conflicts in the response
The sync response DTO should include a `List<SyncConflictDto>` so the caller knows which elements were rejected. Each conflict DTO: `ElementId`, `ServerTimestamp`, `ClientTimestamp`, `Resolution` ("SERVER_WINS").

### 02c: Add unit test
Write a test that syncs an element, then syncs an older version of the same element, and verifies the older version is rejected and a conflict record is created.

**COMMIT GATE S02**: TagSync no longer does blind overwrite. Conflicts are logged. Test passes. Commit: `fix(planscape): S02 — TagSync conflict resolution with timestamp comparison`.

---

## S03 — Wire PluginSync into StingToolsApp

### 03a: Locate the dead code
Find `SyncScheduler.cs`, `SyncClient.cs`, `OfflineQueue.cs` in the `Planscape.PluginSync` namespace. Verify they are never instantiated anywhere.

### 03b: Wire SyncScheduler.Start()
In `StingToolsApp.cs`, in the `OnStartup()` method (or equivalent application startup), add a call to `SyncScheduler.Start(serverUrl, authToken)` after the ribbon is built. The server URL and auth token should come from `StingBIMServerClient.Instance` (which already stores them after login).

### 03c: Fix OnDocumentSaved
The `OnDocumentSaved` handler currently queues data into `_pendingSyncDoc` / `_pendingSyncTime` static fields. Redirect this to call `OfflineQueue.Enqueue()` instead, so the SyncScheduler's 5-minute timer will pick it up.

### 03d: Remove dead static fields
Remove the `_pendingSyncDoc` and `_pendingSyncTime` static fields from `StingToolsApp.cs`. They are now replaced by the OfflineQueue.

### 03e: Redirect PlatformSyncCommand
In `PlatformLinkCommands.cs`, find the `PlatformSyncCommand` handler. Change it to call `SyncScheduler.SyncNow()` (immediate sync) instead of using `StingBIMServerClient` directly. This makes the manual sync button trigger the same code path as the automatic sync.

### 03f: Update StingCommandHandler dispatch
At `StingCommandHandler.cs` lines ~1873-1876, verify the dispatch for `PlatformSyncCommand` still routes correctly after the redirect.

### 03g: Do NOT delete StingBIMServerClient yet
Keep `StingBIMServerClient` in the codebase for now. It still handles authentication (login, token refresh). The sync methods on it are no longer the primary path, but do not delete the class until all callers are verified redirected. Add a `[Obsolete("Use SyncScheduler for sync operations")]` attribute to its sync methods.

**COMMIT GATE S03**: `SyncScheduler.Start()` is called on startup. `OnDocumentSaved` enqueues to `OfflineQueue`. Manual sync button calls `SyncScheduler.SyncNow()`. Build passes. Commit: `fix(planscape): S03 — wire PluginSync into StingToolsApp, retire manual sync path`.

---

## S04 — Server image thumbnailing

### 04a: Add ImageSharp dependency
Add `SixLabors.ImageSharp` NuGet package to the server project (`Planscape_API.csproj`).

### 04b: Create thumbnail service
Create `IThumbnailService` interface and `ImageSharpThumbnailService` implementation. Method: `Task<(byte[] thumb150, byte[] thumb300, byte[] thumb600)> GenerateThumbnails(Stream imageStream)`. Produces 3 sizes: 150px, 300px, 600px (longest edge). Output format: JPEG at 80% quality.

### 04c: Wire into document/issue upload
In `IssuesController` attachment upload and `DocumentsController` file upload, after saving the original file, check if the MIME type is an image (`image/jpeg`, `image/png`, `image/webp`). If so, call the thumbnail service and save thumbnails alongside the original with suffix `_150`, `_300`, `_600`.

### 04d: EXIF GPS extraction
When processing uploaded images, extract GPS coordinates from EXIF metadata using ImageSharp's `ExifProfile`. If GPS data is present and the parent entity (issue or document) has no coordinates set, auto-populate `Latitude`/`Longitude` from the image EXIF.

### 04e: Add thumbnail endpoint
Add `GET /api/issues/{id}/attachments/{attachmentId}/thumbnail?size=300` endpoint that returns the pre-generated thumbnail.

### 04f: Register service in DI
Register `IThumbnailService` as scoped in `Program.cs`.

**COMMIT GATE S04**: Image uploads generate 3 thumbnail sizes. EXIF GPS is extracted. Thumbnail endpoint works. Build passes. Commit: `fix(planscape): S04 — image thumbnailing with ImageSharp and EXIF GPS extraction`.

---

## S05 — Server storage abstraction

### 05a: Create IFileStorageService interface
```csharp
public interface IFileStorageService
{
    Task<string> SaveAsync(string tenantSlug, string projectCode, string fileName, Stream content);
    Task<Stream?> GetAsync(string path);
    Task<bool> DeleteAsync(string path);
    Task<bool> ExistsAsync(string path);
}
```

### 05b: Implement LocalFileStorage
Extract the current file save logic from `DocumentsController` into a `LocalFileStorageService` class that implements the interface. Path pattern: `{StoragePath}/{tenantSlug}/{projectCode}/{fileName}`.

### 05c: Register in DI
Register `IFileStorageService` as `LocalFileStorageService` (the default). This is Phase 1 of the storage migration. MinIO and Azure Blob implementations come later but the interface is ready.

### 05d: Refactor controllers
Replace all direct `File.WriteAllBytes` / `File.ReadAllBytes` calls in `DocumentsController` and `IssuesController` with `IFileStorageService` calls.

**COMMIT GATE S05**: All file I/O goes through `IFileStorageService`. No direct filesystem calls remain in controllers. Build passes. Commit: `fix(planscape): S05 — IFileStorageService abstraction with LocalFileStorage`.

---

## S06 — Server delta sync and batch operations

### 06a: Delta sync on TagSync
Add `lastSyncUtc` query parameter to `GET /api/tagsync/{projectId}`. If provided, return only elements where `LastModifiedUtc > lastSyncUtc`. Update the `SyncWatermark` for the requesting device after each successful sync.

### 06b: Batch operations endpoint
Add `POST /api/batch` endpoint that accepts a JSON array of operations:
```json
[
  { "type": "CREATE_ISSUE", "payload": { ... } },
  { "type": "UPDATE_ISSUE", "payload": { ... } },
  { "type": "TRANSITION_CDE", "payload": { ... } }
]
```
Process each operation, collect results, return a summary with per-operation success/failure status. This is the server-side counterpart to the mobile offline queue.

### 06c: Batch size limits
Limit batch requests to 100 operations per call. Return 400 if exceeded.

**COMMIT GATE S06**: Delta sync returns only changed elements. Batch endpoint processes queued operations. Build passes. Commit: `fix(planscape): S06 — delta sync with watermarks and batch replay endpoint`.

---

## S07 — Server file versioning

### 07a: Version chain on document upload
When uploading a file to an existing `DocumentRecord` (same project + same file name pattern), create a `DocumentVersion` row linking to the `DocumentRecord`, increment the version number, and update the `DocumentRecord.Revision` field.

### 07b: Version list endpoint
Add `GET /api/documents/{id}/versions` to return the version history for a document.

### 07c: Version download endpoint
Add `GET /api/documents/{id}/versions/{versionNumber}/download` to retrieve a specific historical version.

**COMMIT GATE S07**: Document uploads create version records. Version history endpoint works. Build passes. Commit: `fix(planscape): S07 — document version chain with history and download`.

---

## S08 — Server QR code generation

### 08a: Add ZXing.Net dependency
If not already present on the server project, add the `ZXing.Net` NuGet package.

### 08b: QR generation endpoint
Add `GET /api/projects/{projectId}/elements/{elementId}/qr?size=200` that generates a QR code PNG containing the element's TAG7 identifier. Return as `image/png`.

### 08c: Batch QR generation
Add `POST /api/projects/{projectId}/qr/batch` that accepts a list of element IDs and returns a ZIP file of QR code PNGs, one per element, named by TAG1.

**COMMIT GATE S08**: QR endpoints generate valid PNGs. Build passes. Commit: `fix(planscape): S08 — server-side QR code generation for element tags`.

---

## S09 — Server WebSocket auth for mobile

### 09a: Verify JWT query string support
Check `Program.cs` — the `OnMessageReceived` handler already reads `access_token` from the query string for SignalR hubs. Verify this covers all three hubs: `/hubs/compliance`, `/hubs/tagsync`, `/hubs/notifications`.

### 09b: Add mobile-specific SignalR group
In each SignalR hub, add a method `JoinProject(string projectId)` that adds the connection to a project-specific group. This lets the server broadcast to all mobile clients watching a specific project.

### 09c: Add device tracking
When a mobile client connects to a hub, record the connection ID, device ID, and project ID in a `ConcurrentDictionary` on the hub for diagnostics (no database, in-memory only).

**COMMIT GATE S09**: Mobile clients can authenticate via query string JWT and join project groups. Build passes. Commit: `fix(planscape): S09 — SignalR mobile auth and project group subscriptions`.

---

## S10 — Server rate limiting per device

### 10a: Add device token partitioning
In `Program.cs`, add a new rate limiter policy `"mobile"` that partitions by device token (from a custom `X-Device-Id` header) instead of IP. This prevents shared site WiFi from exhausting limits across all devices. Limit: 120 req/min per device.

### 10b: Apply to mobile-facing endpoints
Apply the `"mobile"` rate limiter to endpoints that mobile clients call: issues CRUD, documents CRUD, tagsync, compliance, batch.

**COMMIT GATE S10**: Per-device rate limiting configured. Build passes. Commit: `fix(planscape): S10 — per-device rate limiting for mobile clients`.

---

## S11 — Server audit log enhancement

### 11a: Add mobile action fields
Add `device_id` and `latitude`/`longitude` columns to the `audit_logs` table (in the migration file or a new `003_audit_mobile.sql`).

### 11b: Populate from mobile requests
Create middleware or a helper that extracts `X-Device-Id`, `X-Latitude`, `X-Longitude` headers from mobile requests and makes them available to the audit log writer. Distinguish mobile vs desktop actions in audit records.

**COMMIT GATE S11**: Audit log records device origin. Migration applies cleanly. Build passes. Commit: `fix(planscape): S11 — audit log mobile device tracking`.

---

## S12 — Server geofence validation

### 12a: Add project boundary
Add `boundary_polygon JSONB` column to `projects` table. Format: GeoJSON polygon.

### 12b: Add geofence middleware
Create `GeofenceValidationService` that checks if a given lat/lng is inside the project boundary polygon (point-in-polygon test). Make it optional: if no boundary is set, skip validation.

### 12c: Apply to sensitive endpoints
On document download and issue creation endpoints, if the project has a boundary polygon and the request includes GPS coordinates, validate that the coordinates fall within the boundary. Return 403 with a clear message if outside.

**COMMIT GATE S12**: Geofence validation works when boundary is set, is a no-op when not set. Build passes. Commit: `fix(planscape): S12 — project geofence boundary validation`.

---

## S13 — Fix PLANSCAPE_GAPS.md issues

### 13a: Fix roadmap checkboxes
In `PLANSCAPE_GAPS.md` section 7, change all `[x]` to `[ ]`. These are planned tasks, not completed ones.

### 13b: Fix week range arithmetic
Section 6.2 says "12-16 weeks" but the phase totals are 21+21+13+10 = 65 days = 13 weeks. Change to "11-14 weeks" (allowing for the stated per-phase week ranges of 4-5 + 3-4 + 2-3 + 2).

**COMMIT GATE S13**: Markdown is corrected. Commit: `fix(planscape): S13 — fix PLANSCAPE_GAPS.md roadmap checkboxes and week arithmetic`.

---

## Final verification

After all 13 sections are committed:

1. Run `dotnet build` on the full solution. Zero errors.
2. Run any existing tests: `dotnet test`. All pass.
3. Review git log: 13 commits, one per section, all with the `fix(planscape):` prefix.
4. Re-read this prompt from top to bottom. Verify every checklist item in every COMMIT GATE is satisfied.
5. If anything was skipped or is incomplete, go back and fix it now.

---

## Quick-reference: gap ID to section mapping

| Gap IDs | Section | Summary |
|---------|---------|---------|
| SRV-03, SRV-04 | S01, S02 | GPS fields, version tracking, conflict resolution |
| INT-01 to INT-06 | S03 | Wire PluginSync, retire manual sync |
| SRV-01, SRV-06 | S04 | Image thumbnailing, EXIF GPS |
| SRV-02 | S05 | Storage abstraction |
| SRV-05, SRV-07, SRV-08 | S06 | Delta sync, batch operations |
| SRV-13 | S07 | Document versioning |
| SRV-10 | S08 | QR code generation |
| SRV-11 | S09 | WebSocket mobile auth |
| SRV-12 | S10 | Per-device rate limiting |
| SRV-14 | S11 | Audit log mobile fields |
| SRV-15 | S12 | Geofence validation |
| — | S13 | Document corrections |
| MOB-01 to MOB-12 | Not in scope | Mobile app changes require React Native environment; handle separately |

The mobile app gaps (MOB-01 through MOB-12) are excluded from this prompt because they require a React Native/Expo development environment, not the C#/.NET server and plugin codebase. Create a separate implementation prompt for mobile once the server-side foundations from S01-S12 are in place.
