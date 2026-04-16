<!-- v1.0 | 20260411 -->

# Planscape Claude Code implementation prompts

All prompts below target the remaining gaps identified in `20260411_planscape_gap_analysis_roadmap_v1.0.docx`. Sections marked 100% (S07, S08, S09, S10, S12) are excluded.

Usage: `/clear` before each prompt. Paste the prompt. Wait for commit. `/clear` again before the next one.

---

## S01 part 1 of 2: Migration file and new entity classes

```
# S01 part 1 — Migration file and new entities

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 1 file per subtask. State briefly what you did, then continue immediately.

## 01a-01g: Create migration file
Find the migration folder:
Run: find . -name "001_initial_schema.sql" -not -path "*/obj/*"

Create 002_gps_versioning_sync.sql in that same folder with this exact content:

ALTER TABLE bim_issues ADD COLUMN IF NOT EXISTS latitude DOUBLE PRECISION;
ALTER TABLE bim_issues ADD COLUMN IF NOT EXISTS longitude DOUBLE PRECISION;
ALTER TABLE bim_issues ADD COLUMN IF NOT EXISTS location_accuracy DOUBLE PRECISION;
ALTER TABLE bim_issues ADD COLUMN IF NOT EXISTS device_id TEXT;

ALTER TABLE document_records ADD COLUMN IF NOT EXISTS latitude DOUBLE PRECISION;
ALTER TABLE document_records ADD COLUMN IF NOT EXISTS longitude DOUBLE PRECISION;

ALTER TABLE tagged_elements ADD COLUMN IF NOT EXISTS version INTEGER NOT NULL DEFAULT 1;
ALTER TABLE tagged_elements ADD COLUMN IF NOT EXISTS last_modified_utc TIMESTAMPTZ;

CREATE TABLE IF NOT EXISTS sync_watermarks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    device_id TEXT NOT NULL,
    last_sync_utc TIMESTAMPTZ NOT NULL,
    element_count INTEGER NOT NULL DEFAULT 0,
    UNIQUE (project_id, device_id)
);

CREATE TABLE IF NOT EXISTS document_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_record_id UUID NOT NULL REFERENCES document_records(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    file_path TEXT NOT NULL,
    file_hash TEXT,
    file_size BIGINT,
    uploaded_by TEXT,
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    change_description TEXT
);

CREATE TABLE IF NOT EXISTS sync_conflicts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    element_id TEXT NOT NULL,
    conflict_type TEXT NOT NULL,
    server_value JSONB,
    client_value JSONB,
    resolution TEXT NOT NULL,
    resolved_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_by TEXT
);

INSERT INTO schema_versions (version, description)
VALUES (2, 'GPS coordinates, element versioning, sync watermarks, document versions, conflict log')
ON CONFLICT DO NOTHING;

Do NOT read any files for this step. Just create the file.

## 01h part 1: Create new entity classes
Find the entity folder:
Run: find . -name "BimIssue.cs" -not -path "*/obj/*"

Use the same directory and namespace as BimIssue.cs but do NOT read BimIssue.cs yet.

Create SyncWatermark.cs:

using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace Planscape.API.Models;

public class SyncWatermark
{
    [Key] public Guid Id { get; set; }
    public Guid TenantId { get; set; }
    public Guid ProjectId { get; set; }
    [Required] public string DeviceId { get; set; } = string.Empty;
    public DateTime LastSyncUtc { get; set; }
    public int ElementCount { get; set; }
    [ForeignKey("TenantId")] public Tenant? Tenant { get; set; }
    [ForeignKey("ProjectId")] public Project? Project { get; set; }
}

Create SyncConflict.cs:

using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace Planscape.API.Models;

public class SyncConflict
{
    [Key] public Guid Id { get; set; }
    public Guid TenantId { get; set; }
    public Guid ProjectId { get; set; }
    [Required] public string ElementId { get; set; } = string.Empty;
    [Required] public string ConflictType { get; set; } = string.Empty;
    [Column(TypeName = "jsonb")] public string? ServerValue { get; set; }
    [Column(TypeName = "jsonb")] public string? ClientValue { get; set; }
    [Required] public string Resolution { get; set; } = string.Empty;
    public DateTime ResolvedAt { get; set; }
    public string? ResolvedBy { get; set; }
}

Check if DocumentVersion.cs already exists:
Run: find . -name "DocumentVersion.cs" -not -path "*/obj/*"

If it does NOT exist, create DocumentVersion.cs:

using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace Planscape.API.Models;

public class DocumentVersion
{
    [Key] public Guid Id { get; set; }
    public Guid DocumentRecordId { get; set; }
    public int VersionNumber { get; set; }
    [Required] public string FilePath { get; set; } = string.Empty;
    public string? FileHash { get; set; }
    public long? FileSize { get; set; }
    public string? UploadedBy { get; set; }
    public DateTime UploadedAt { get; set; }
    public string? ChangeDescription { get; set; }
    [ForeignKey("DocumentRecordId")] public DocumentRecord? DocumentRecord { get; set; }
}

Run `dotnet build` to check for namespace issues. Fix any errors but do NOT read unrelated files. Do not commit yet. Start now.
```

---

## S01 part 2 of 2: Update existing entities and DbContext

```
# S01 part 2 — Update existing entities and DbContext

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 2 files total. Use grep + surgical str_replace edits.

## Add GPS properties to BimIssue
Run: find . -name "BimIssue.cs" -not -path "*/obj/*"
Read ONLY that file. Find the last property before the closing brace. Add after it using str_replace:

    public double? Latitude { get; set; }
    public double? Longitude { get; set; }
    public double? LocationAccuracy { get; set; }
    public string? DeviceId { get; set; }

## Add GPS properties to DocumentRecord
Run: find . -name "DocumentRecord.cs" -not -path "*/obj/*"
Read ONLY that file. Add after the last property:

    public double? Latitude { get; set; }
    public double? Longitude { get; set; }

## Add Version/LastModifiedUtc to TaggedElement
Run: find . -name "TaggedElement.cs" -not -path "*/obj/*"
Read ONLY that file. Add after the last property:

    public int Version { get; set; } = 1;
    public DateTime? LastModifiedUtc { get; set; }

## Register DbSets in PlanscapeDbContext
Run: grep -rn "class.*DbContext" --include="*.cs" -l | grep -v obj
Read ONLY that file. Find existing DbSet declarations. Add if not already present:

    public DbSet<SyncWatermark> SyncWatermarks { get; set; }
    public DbSet<SyncConflict> SyncConflicts { get; set; }
    public DbSet<DocumentVersion> DocumentVersions { get; set; }

## Build and commit
Run: dotnet build
Fix any errors.
Run: git add -A
Run: git commit -m "fix(planscape): S01 — add GPS, versioning, sync watermark, conflict log schema"

Do not push. Start now.
```

---

## S02: TagSync conflict resolution

```
# S02 — TagSync conflict resolution

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 3 files total. State briefly what you did after each subtask, then continue immediately.

## 02a: Add timestamp comparison
Run: find . -name "TagSyncController.cs" -not -path "*/obj/*"
Read ONLY that file. Find the MapDtoToEntity call inside the sync/upsert loop. Replace the blind overwrite with:

1. Before overwriting, compare dto.LastModifiedUtc against existing.LastModifiedUtc
2. If dto.LastModifiedUtc > existing.LastModifiedUtc (or existing has no timestamp): apply the update, increment existing.Version, set existing.LastModifiedUtc = dto.LastModifiedUtc
3. If dto.LastModifiedUtc <= existing.LastModifiedUtc: add to a conflicts list, do NOT overwrite. Create a SyncConflict record in the database with ConflictType = "STALE_UPDATE", Resolution = "SERVER_WINS"

Use surgical str_replace edits. Do NOT rewrite the entire file.

## 02b: Return conflicts in response
Find the sync response DTO:
Run: grep -rn "class.*SyncResponse\|class.*TagSync.*Response\|class.*TagSync.*Result" --include="*.cs" -l | grep -v obj

If a response DTO exists, add a List<SyncConflictDto> property. If none exists, create a simple response class.

SyncConflictDto should contain: ElementId (string), ServerTimestamp (DateTime?), ClientTimestamp (DateTime?), Resolution (string).

Update the sync endpoint return to include the conflicts list.

## 02c: Add unit test
Find the test project:
Run: find . -name "*Tests.csproj" -not -path "*/obj/*"
Run: find . -name "TagSyncController*" -path "*/Test*" -not -path "*/obj/*" 2>/dev/null || echo "no existing test file"

Create a new test file TagSyncConflictTests.cs in the test project folder. Write ONE test:
- Arrange: create a TaggedElement with LastModifiedUtc = 2026-01-01 and Version = 1
- Act: call the sync method with the same element but LastModifiedUtc = 2025-12-01 (older)
- Assert: the element's data is unchanged, a SyncConflict record exists, the response includes the conflict

Do NOT read existing test files for patterns. Use xUnit with [Fact] attribute. Mock the DbContext with an in-memory list or use the test project's existing pattern if obvious from the csproj.

## Build and commit
Run: dotnet build
Fix any errors.
Run: dotnet test 2>&1 | tail -20
Run: git add -A
Run: git commit -m "fix(planscape): S02 — TagSync conflict resolution with timestamp comparison"

Do not push. Start now.
```

---

## S03 part 1 of 2: Wire SyncScheduler and fix OnDocumentSaved

```
# S03 part 1 — Wire SyncScheduler and fix OnDocumentSaved

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 2 files per subtask. State briefly what you did, then continue immediately.

## 03a: Verify dead code exists
Run: find . -name "SyncScheduler.cs" -not -path "*/obj/*"
Run: find . -name "SyncClient.cs" -not -path "*/obj/*"
Run: find . -name "OfflineQueue.cs" -not -path "*/obj/*"
Run: grep -rn "SyncScheduler\|new SyncScheduler\|SyncScheduler.Start" --include="*.cs" | grep -v "obj/" | grep -v "\.cs:.*\/\/"
Report which files exist and whether SyncScheduler is instantiated anywhere. Do NOT read the full files.

## 03b: Wire SyncScheduler.Start()
Run: find . -name "StingToolsApp.cs" -not -path "*/obj/*"
Read ONLY that file. Find the OnStartup() method (or equivalent startup). After the ribbon build completes, add:

var serverUrl = StingBIMServerClient.Instance?.ServerUrl;
var authToken = StingBIMServerClient.Instance?.AuthToken;
if (!string.IsNullOrEmpty(serverUrl) && !string.IsNullOrEmpty(authToken))
{
    SyncScheduler.Start(serverUrl, authToken);
}

Add the using for the PluginSync namespace if missing. Use str_replace for a surgical edit.

## 03c: Fix OnDocumentSaved
In the same StingToolsApp.cs (already open, do NOT re-read), find the OnDocumentSaved handler. Find where it sets _pendingSyncDoc / _pendingSyncTime. Replace those lines with a call to OfflineQueue.Enqueue() passing the compliance data. If OfflineQueue.Enqueue requires specific parameters, read ONLY OfflineQueue.cs to check its method signature.

## 03d: Remove dead static fields
In the same file, remove the _pendingSyncDoc and _pendingSyncTime static field declarations.

Do NOT commit yet. Run `dotnet build` and fix errors. Start now.
```

---

## S03 part 2 of 2: Redirect PlatformSyncCommand and commit

```
# S03 part 2 — Redirect PlatformSyncCommand and commit

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 2 files total. State briefly what you did, then continue immediately.

## 03e: Redirect PlatformSyncCommand
Run: find . -name "PlatformLinkCommands.cs" -not -path "*/obj/*"
Read ONLY that file. Find the PlatformSyncCommand handler method. Replace the body that calls StingBIMServerClient sync methods with:

SyncScheduler.SyncNow();

Keep any UI feedback (TaskDialog, status bar updates) around the call. Just replace the sync mechanism. Add the using for the PluginSync namespace if missing.

## 03f: Verify dispatch
Run: grep -n "PlatformSyncCommand" --include="*.cs" -r | grep -v obj
Confirm the dispatch in StingCommandHandler.cs still routes to the handler you just modified. If the routing is indirect (via a dictionary or switch), just verify the mapping exists. Do NOT read the entire StingCommandHandler file — just grep for the relevant lines.

## 03g: Mark old sync methods obsolete
In PlatformLinkCommands.cs (already open), find the StingBIMServerClient class definition or its sync methods. If StingBIMServerClient is in this file, add [Obsolete("Use SyncScheduler for sync operations")] to its sync methods (PushElements, PushCompliance, PullIssues or equivalent). If it is in a separate file:
Run: find . -name "StingBIMServerClient.cs" -not -path "*/obj/*"
Read ONLY that file and add the [Obsolete] attributes to sync methods.

## Build and commit
Run: dotnet build
Fix any errors.
Run: git add -A
Run: git commit -m "fix(planscape): S03 — wire PluginSync into StingToolsApp, retire manual sync path"

Do not push. Start now.
```

---

## S04 part 1 of 2: Thumbnail service and EXIF extraction

```
# S04 part 1 — Thumbnail service and EXIF GPS extraction

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 1 file per subtask. State briefly what you did, then continue immediately.

## 04a: Add ImageSharp dependency
Check if already present:
Run: grep -i "ImageSharp" *.csproj 2>/dev/null || find . -name "*.csproj" -not -path "*/obj/*" -exec grep -l "ImageSharp" {} \;

If NOT present, find the server csproj:
Run: find . -name "Planscape_API.csproj" -not -path "*/obj/*" -o -name "*.csproj" -not -path "*/obj/*" -not -name "*Test*" | head -1
Run: dotnet add <that_csproj> package SixLabors.ImageSharp

## 04b: Create thumbnail service interface and implementation
Create IThumbnailService.cs in a Services folder (find it or create it):
Run: find . -type d -name "Services" -not -path "*/obj/*" | head -1

Create IThumbnailService.cs:

using System.IO;
using System.Threading.Tasks;

namespace Planscape.API.Services;

public interface IThumbnailService
{
    Task<Dictionary<int, byte[]>> GenerateThumbnailsAsync(Stream imageStream);
    (double? Latitude, double? Longitude) ExtractGpsFromExif(Stream imageStream);
}

Create ImageSharpThumbnailService.cs in the same folder. Do NOT read any existing files:

using SixLabors.ImageSharp;
using SixLabors.ImageSharp.Processing;
using SixLabors.ImageSharp.Formats.Jpeg;
using SixLabors.ImageSharp.Metadata.Profiles.Exif;
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;

namespace Planscape.API.Services;

public class ImageSharpThumbnailService : IThumbnailService
{
    private static readonly int[] Sizes = { 150, 300, 600 };

    public async Task<Dictionary<int, byte[]>> GenerateThumbnailsAsync(Stream imageStream)
    {
        var results = new Dictionary<int, byte[]>();
        imageStream.Position = 0;
        using var image = await Image.LoadAsync(imageStream);

        foreach (var size in Sizes)
        {
            using var clone = image.Clone(ctx =>
            {
                var ratio = (double)size / Math.Max(image.Width, image.Height);
                var newW = (int)(image.Width * ratio);
                var newH = (int)(image.Height * ratio);
                ctx.Resize(newW, newH);
            });
            using var ms = new MemoryStream();
            await clone.SaveAsJpegAsync(ms, new JpegEncoder { Quality = 80 });
            results[size] = ms.ToArray();
        }
        return results;
    }

    public (double? Latitude, double? Longitude) ExtractGpsFromExif(Stream imageStream)
    {
        try
        {
            imageStream.Position = 0;
            using var image = Image.Load(imageStream);
            var exif = image.Metadata.ExifProfile;
            if (exif == null) return (null, null);

            var latValue = exif.GetValue(ExifTag.GPSLatitude);
            var lonValue = exif.GetValue(ExifTag.GPSLongitude);
            var latRef = exif.GetValue(ExifTag.GPSLatitudeRef);
            var lonRef = exif.GetValue(ExifTag.GPSLongitudeRef);

            if (latValue == null || lonValue == null) return (null, null);

            var lat = ConvertDmsToDecimal(latValue.Value);
            var lon = ConvertDmsToDecimal(lonValue.Value);

            if (latRef?.Value == "S") lat = -lat;
            if (lonRef?.Value == "W") lon = -lon;

            return (lat, lon);
        }
        catch { return (null, null); }
    }

    private static double ConvertDmsToDecimal(Rational[] dms)
    {
        if (dms == null || dms.Length < 3) return 0;
        return dms[0].ToDouble() + dms[1].ToDouble() / 60.0 + dms[2].ToDouble() / 3600.0;
    }
}

Run `dotnet build`. Fix any errors (the Rational type may need adjustment for the ImageSharp version installed). Do NOT commit yet. Start now.
```

---

## S04 part 2 of 2: Wire into controllers, add endpoint, register DI

```
# S04 part 2 — Wire thumbnails into uploads, add endpoint, register DI

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 2 files per subtask. State briefly what you did, then continue immediately.

## 04c: Wire into IssuesController attachment upload
Run: find . -name "IssuesController.cs" -not -path "*/obj/*"
Read ONLY that file. Find the attachment upload action. After the file is saved to disk, add:

var imageTypes = new[] { "image/jpeg", "image/png", "image/webp" };
if (imageTypes.Contains(file.ContentType.ToLower()))
{
    using var thumbStream = file.OpenReadStream();
    var thumbnails = await _thumbnailService.GenerateThumbnailsAsync(thumbStream);
    var basePath = Path.GetDirectoryName(savedFilePath);
    var baseName = Path.GetFileNameWithoutExtension(savedFilePath);
    var ext = ".jpg";
    foreach (var (size, bytes) in thumbnails)
    {
        var thumbPath = Path.Combine(basePath!, $"{baseName}_{size}{ext}");
        await System.IO.File.WriteAllBytesAsync(thumbPath, bytes);
    }

    using var exifStream = file.OpenReadStream();
    var (lat, lng) = _thumbnailService.ExtractGpsFromExif(exifStream);
    // If parent issue has no GPS, populate from EXIF
}

Inject IThumbnailService via constructor. Use str_replace for surgical edits.

## 04d: Wire into DocumentsController
Run: find . -name "DocumentsController.cs" -not -path "*/obj/*"
Read ONLY that file. Add the same thumbnail generation after file save, same pattern as above. Inject IThumbnailService via constructor.

## 04e: Add thumbnail endpoint
In IssuesController (already open), add:

[HttpGet("{id}/attachments/{attachmentId}/thumbnail")]
public async Task<IActionResult> GetThumbnail(Guid id, Guid attachmentId, [FromQuery] int size = 300)
{
    var validSizes = new[] { 150, 300, 600 };
    if (!validSizes.Contains(size)) size = 300;
    // Find the attachment file path, construct thumbnail path with _{size}.jpg suffix
    // Return File(bytes, "image/jpeg") or NotFound()
}

Adapt to the existing attachment storage pattern you see in the file.

## 04f: Register in Program.cs
Run: grep -n "AddScoped\|AddTransient\|AddSingleton" Program.cs | head -5
Read ONLY Program.cs. Add near other service registrations:

builder.Services.AddScoped<IThumbnailService, ImageSharpThumbnailService>();

Add the using for Planscape.API.Services if missing.

## Build and commit
Run: dotnet build
Fix any errors.
Run: git add -A
Run: git commit -m "fix(planscape): S04 — image thumbnailing with ImageSharp and EXIF GPS extraction"

Do not push. Start now.
```

---

## S05: Storage abstraction (complete remaining controller refactoring)

```
# S05 — Complete storage abstraction

The IFileStorageService interface and LocalFileStorageService already exist. The gap is that controllers still use direct filesystem calls. Do NOT stop to ask anything. Do NOT read more than 2 files per subtask.

## 05a: Check current state
Run: grep -rn "IFileStorageService" --include="*.cs" -l | grep -v obj
Run: grep -rn "File.WriteAllBytes\|File.ReadAllBytes\|File.Create\|FileStream\|System.IO.File" --include="*.cs" -l | grep -v obj | grep -v "LocalFileStorage\|FileStorage"

Report which controllers still have direct filesystem calls.

## 05b: Refactor DocumentsController
Run: find . -name "DocumentsController.cs" -not -path "*/obj/*"
Read ONLY that file. Find any direct File.WriteAllBytes, File.ReadAllBytes, FileStream, or similar calls. Replace them with _fileStorageService.SaveAsync() / _fileStorageService.GetAsync() / _fileStorageService.DeleteAsync(). If IFileStorageService is not already injected, add it to the constructor.

## 05c: Refactor IssuesController
Run: find . -name "IssuesController.cs" -not -path "*/obj/*"
Read ONLY that file. Same refactoring as above — replace direct filesystem calls with IFileStorageService methods.

## 05d: Verify no direct calls remain
Run: grep -rn "File.WriteAllBytes\|File.ReadAllBytes\|System.IO.File.Create" --include="*.cs" Controllers/ 2>/dev/null || grep -rn "File.WriteAllBytes\|File.ReadAllBytes" --include="*.cs" -l | grep -v obj | grep -v "LocalFileStorage\|FileStorage\|Thumbnail"

If any remain in controllers, fix them.

## Build and commit
Run: dotnet build
Fix any errors.
Run: git add -A
Run: git commit -m "fix(planscape): S05 — complete IFileStorageService abstraction in all controllers"

Do not push. Start now.
```

---

## S06: Delta sync with watermarks and batch operations

```
# S06 — Delta sync and batch operations

Do all subtasks in sequence. Do NOT stop to ask anything. Do NOT read more than 2 files per subtask. State briefly what you did, then continue immediately.

## 06a: Add delta sync to TagSyncController
Run: find . -name "TagSyncController.cs" -not -path "*/obj/*"
Read ONLY that file. Find the GET endpoint that returns elements for a project. Add an optional [FromQuery] DateTime? lastSyncUtc parameter. When provided:
1. Filter results to only elements where LastModifiedUtc > lastSyncUtc
2. After returning results, upsert a SyncWatermark record for the requesting device (get device ID from X-Device-Id header or default to "desktop")

Use str_replace for surgical edits.

## 06b: Create batch operations endpoint
Create a new file BatchController.cs in the Controllers folder. Do NOT read existing controllers for reference:

[ApiController]
[Route("api/batch")]
[Authorize]
public class BatchController : ControllerBase
{
    private readonly PlanscapeDbContext _db;
    public BatchController(PlanscapeDbContext db) => _db = db;

    [HttpPost]
    public async Task<IActionResult> Execute([FromBody] BatchRequest req)
    {
        if (req.Operations.Count > 100)
            return BadRequest("Maximum 100 operations per batch");

        var tenantId = /* extract from JWT tenant_id claim */;
        var results = new List<BatchOperationResult>();

        foreach (var op in req.Operations)
        {
            try
            {
                switch (op.Type)
                {
                    case "CREATE_ISSUE": /* create issue from op.Payload */ break;
                    case "UPDATE_ISSUE": /* update issue from op.Payload */ break;
                    case "TRANSITION_CDE": /* transition document from op.Payload */ break;
                    default: results.Add(new BatchOperationResult { Index = results.Count, Success = false, Error = $"Unknown type: {op.Type}" }); continue;
                }
                results.Add(new BatchOperationResult { Index = results.Count, Success = true });
            }
            catch (Exception ex)
            {
                results.Add(new BatchOperationResult { Index = results.Count, Success = false, Error = ex.Message });
            }
        }
        await _db.SaveChangesAsync();
        return Ok(new BatchResponse { Results = results });
    }
}

Adapt the tenant extraction and issue/document creation to match existing patterns in the codebase. Find the tenant claim name:
Run: grep -rn "tenant" --include="*.cs" Controllers/ | grep -i "claim\|FindFirst" | head -5

Check if BatchRequest/BatchResponse DTOs already exist:
Run: find . -name "*Batch*" -name "*.cs" -not -path "*/obj/*"

If they already exist from S06 work, reuse them. If not, create them in a Dtos folder.

## Build and commit
Run: dotnet build
Fix any errors.
Run: git add -A
Run: git commit -m "fix(planscape): S06 — delta sync with watermarks and batch replay endpoint"

Do not push. Start now.
```

---

## S07: Document versioning (already 100%, verify only)

```
# S07 — Verify document versioning

This section was marked 100% complete. Run verification only. Do NOT read any source files.

Run: grep -rn "versions" --include="*.cs" Controllers/ | grep -i "HttpGet\|endpoint\|route" | head -5
Run: dotnet build 2>&1 | tail -5

If build passes and version endpoints exist, report "S07 verified, no changes needed" and stop. Do NOT commit.

Start now.
```

---

## S08: QR code generation (already 100%, verify only)

```
# S08 — Verify QR code generation

This section was marked 100% complete. Run verification only. Do NOT read any source files.

Run: find . -name "QrController.cs" -not -path "*/obj/*"
Run: dotnet build 2>&1 | tail -5

If the file exists and build passes, report "S08 verified, no changes needed" and stop. Do NOT commit.

Start now.
```

---

## S09: WebSocket auth (already 100%, verify only)

```
# S09 — Verify WebSocket auth

This section was marked 100% complete. Run verification only. Do NOT read any source files.

Run: grep -rn "OnMessageReceived\|access_token" --include="*.cs" | grep -v obj | head -5
Run: grep -rn "JoinProject" --include="*.cs" | grep -v obj | head -5
Run: dotnet build 2>&1 | tail -5

If JWT query string support and JoinProject methods exist and build passes, report "S09 verified, no changes needed" and stop. Do NOT commit.

Start now.
```

---

## S10: Rate limiting (already 100%, verify only)

```
# S10 — Verify rate limiting

This section was marked 100% complete. Run verification only. Do NOT read any source files.

Run: grep -rn "mobile\|EnableRateLimiting\|AddPolicy" --include="*.cs" Program.cs | head -10
Run: dotnet build 2>&1 | tail -5

If mobile rate limiter policy exists and build passes, report "S10 verified, no changes needed" and stop. Do NOT commit.

Start now.
```

---

## S11: Audit trail wiring

```
# S11 — Wire audit log insertion into controllers

Infrastructure exists (AuditLog entity, MobileContextMiddleware, AdminController query). The gap is that controllers do not INSERT audit records on write operations. Do NOT stop to ask anything. Do NOT read more than 3 files total.

## 11a: Create AuditService
Find the Services folder:
Run: find . -type d -name "Services" -not -path "*/obj/*" | head -1

Create AuditService.cs. Do NOT read existing services:

using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;

namespace Planscape.API.Services;

public interface IAuditService
{
    Task LogAsync(string action, string entityType, string entityId, string? details = null);
}

public class AuditService : IAuditService
{
    private readonly PlanscapeDbContext _db;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AuditService(PlanscapeDbContext db, IHttpContextAccessor httpContextAccessor)
    {
        _db = db;
        _httpContextAccessor = httpContextAccessor;
    }

    public async Task LogAsync(string action, string entityType, string entityId, string? details = null)
    {
        var ctx = _httpContextAccessor.HttpContext;
        var userId = ctx?.User?.FindFirst("sub")?.Value ?? ctx?.User?.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value;
        var tenantId = ctx?.User?.FindFirst("tenant_id")?.Value;

        var log = new AuditLog
        {
            Id = Guid.NewGuid(),
            Action = action,
            EntityType = entityType,
            EntityId = entityId,
            Details = details,
            UserId = userId,
            Timestamp = DateTime.UtcNow,
            DeviceId = ctx?.Items["DeviceId"] as string,
            Latitude = ctx?.Items["Latitude"] as double?,
            Longitude = ctx?.Items["Longitude"] as double?,
            Source = ctx?.Items["Source"] as string ?? "desktop"
        };

        _db.AuditLogs.Add(log);
        await _db.SaveChangesAsync();
    }
}

Adjust property names to match the actual AuditLog entity. Check its shape:
Run: find . -name "AuditLog.cs" -not -path "*/obj/*"
Read ONLY that file to verify property names. Adapt AuditService accordingly.

## 11b: Register and inject
Read ONLY Program.cs. Add:

builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<IAuditService, AuditService>();

## 11c: Wire into IssuesController
Run: find . -name "IssuesController.cs" -not -path "*/obj/*"
Read ONLY that file. Inject IAuditService via constructor. Add audit calls after each create/update/delete operation:

await _auditService.LogAsync("CREATE", "Issue", issue.Id.ToString());
await _auditService.LogAsync("UPDATE", "Issue", id.ToString());
await _auditService.LogAsync("DELETE", "Issue", id.ToString());

Use str_replace for surgical edits. Add one line after each SaveChangesAsync call.

## 11d: Wire into DocumentsController
Run: find . -name "DocumentsController.cs" -not -path "*/obj/*"
Read ONLY that file. Same pattern: inject IAuditService, add LogAsync after creates/updates/deletes.

## Build and commit
Run: dotnet build
Fix any errors.
Run: git add -A
Run: git commit -m "fix(planscape): S11 — audit log wired into issue and document controllers"

Do not push. Start now.
```

---

## S12: Geofence validation (already 100%, verify only)

```
# S12 — Verify geofence validation

This section was marked 100% complete. Run verification only. Do NOT read any source files.

Run: find . -name "GeofenceValidationService.cs" -not -path "*/obj/*"
Run: grep -rn "IsInsideBoundary\|boundary_polygon\|BoundaryPolygon" --include="*.cs" | grep -v obj | head -5
Run: dotnet build 2>&1 | tail -5

If the service exists and build passes, report "S12 verified, no changes needed" and stop. Do NOT commit.

Start now.
```

---

## S13: Fix PLANSCAPE_GAPS.md

```
# S13 — Fix PLANSCAPE_GAPS.md

Do both subtasks using command-line tools only. Do NOT open or read the full file. Do NOT stop to ask anything.

## 13a: Fix roadmap checkboxes
Run: grep -c "\[x\]" PLANSCAPE_GAPS.md
Run: sed -i 's/\[x\]/[ ]/g' PLANSCAPE_GAPS.md
Run: grep -c "\[x\]" PLANSCAPE_GAPS.md

Second count must be 0.

## 13b: Fix week range arithmetic
Run: grep -n "12-16 weeks\|12–16 weeks" PLANSCAPE_GAPS.md
Run: sed -i 's/12-16 weeks/11-14 weeks/g' PLANSCAPE_GAPS.md
Run: sed -i 's/12–16 weeks/11–14 weeks/g' PLANSCAPE_GAPS.md
Run: grep -n "11-14\|11–14" PLANSCAPE_GAPS.md

## Commit
Run: git add -A
Run: git commit -m "fix(planscape): S13 — fix PLANSCAPE_GAPS.md roadmap checkboxes and week arithmetic"

Do not push. Start now.
```

---

## Final verification (paste after all sections are committed)

```
# Final verification

Do these checks in order. Do NOT read any source files. Report results only.

1. Run: dotnet build 2>&1 | tail -10
2. Run: dotnet test 2>&1 | tail -20
3. Run: git log --oneline -20

Report:
- Does the build pass with zero errors?
- Do all tests pass?
- List every commit with "fix(planscape):" prefix
- Are any S01-S13 sections missing a commit?

Do not push. Do not fix anything yet. Just report.

Start now.
```

---

## Summary: Execution order

| Order | Prompt | `/clear` before? |
|-------|--------|-------------------|
| 1 | S01 part 1 | Yes |
| 2 | S01 part 2 | Yes |
| 3 | S02 | Yes |
| 4 | S03 part 1 | Yes |
| 5 | S03 part 2 | Yes |
| 6 | S04 part 1 | Yes |
| 7 | S04 part 2 | Yes |
| 8 | S05 | Yes |
| 9 | S06 | Yes |
| 10 | S07 verify | Yes |
| 11 | S08 verify | Yes |
| 12 | S09 verify | Yes |
| 13 | S10 verify | Yes |
| 14 | S11 | Yes |
| 15 | S12 verify | Yes |
| 16 | S13 | Yes |
| 17 | Final verification | Yes |

Sections at 100% (S07, S08, S09, S10, S12) are verify-only: quick grep checks and a build, no code changes, no commits. If a verification prompt reveals a problem, stop and handle it manually before continuing.
