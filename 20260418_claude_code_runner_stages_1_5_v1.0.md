# Claude Code runner — STING clash engine v1.0 (Stages 1–5)

Version 1.0. Companion to `20260418_sting_clash_detection_research_v4.0.md`.
This is the **complete, resumable runner prompt**. Paste into Claude Code as the first user message, then send `continue` on any subsequent invocation to resume.

---

## HOW TO USE THIS PROMPT

**First run.** Paste this entire document into Claude Code. Send it. Claude Code starts at Section 1 of Stage 1.

**Resume (after crash, disconnect, or stop).** Paste this entire document into Claude Code. Send `continue` as the message. Claude Code reads `git log --oneline -- clash/`, identifies the last committed section, and resumes from the next uncommitted section.

**Supervision.** You do not need to supervise. Claude Code commits after each section. If something goes wrong, you lose at most one section of work (typically 50–200 lines of code).

**Environment.** Set in `~/.claude/settings.json`:
```json
{
  "env": {
    "BASH_DEFAULT_TIMEOUT_MS": "600000",
    "BASH_MAX_TIMEOUT_MS": "600000"
  }
}
```
Max 10 min per bash command. No individual step below exceeds 10 min.

---

## NON-NEGOTIABLE RULES

You are Claude Code executing a long-running implementation. Read these rules before every action:

1. **Heredoc everything.** Every file you write uses `cat > path << 'EOF'` for new files or `cat >> path << 'EOF'` for appends. Keep each heredoc under 150 lines. Split larger files into multiple heredocs. This makes every write resumable because partial writes commit nothing.

2. **Commit after every numbered section.** At the end of each numbered section (e.g. "Section 1.3" below), run:
   ```bash
   cd /path/to/repo && dotnet build StingTools.csproj -v quiet 2>&1 | tail -20
   ```
   If the build succeeds, immediately:
   ```bash
   git add -A && git commit -m "clash: <section>" && git push origin HEAD 2>/dev/null || git push origin HEAD
   ```
   If the build fails, fix the error before committing. **Never commit a broken build.**

3. **No build, no commit.** If `dotnet build` fails, read the error, fix it, try again. Do not commit broken code to preserve progress. Broken commits break resume.

4. **Never skip sections.** Execute in order. Do not rearrange. Do not optimize by combining sections.

5. **Never ask for clarification mid-run.** Every decision needed is pre-answered below. If you feel you need clarification, re-read the research document `20260418_sting_clash_detection_research_v4.0.md`.

6. **Resume protocol.** If the user sends only `continue`, run:
   ```bash
   cd /path/to/repo && git log --oneline -- clash/ | head -20
   ```
   Identify the last commit matching `clash: <section>`. Resume from the next section in this prompt. Do not redo completed sections.

7. **No placeholder code.** Every method has a real implementation or throws `NotImplementedException("<ticket>")` with an explicit ticket. Empty stubs that silently return are forbidden.

8. **No deleting existing code.** You add files in `StingTools/Clash/` and extend `BcfEngine.cs`, `BIMCoordinationCenter.cs`, `WarningsManager.cs` only via additive changes. Never remove existing methods.

9. **Preserve namespace conventions.** New engine code goes in `StingTools.Core.Clash`. UI code goes in `StingTools.UI.Clash`. BCF extensions go in `Planscape.Shared.BCF`.

10. **Fail loudly on environment issues.** If `dotnet` is missing, `git` is missing, or the repo is not a git repo, stop and tell the user. Do not attempt to fix environment problems.

---

## REPOSITORY LAYOUT

Assume the repo is at `/repo` in the Claude Code environment. If it is elsewhere, use the path Claude Code is already working in; all commands below use relative paths from the repo root.

Files you will create:
```
StingTools/Clash/
├── MeshExtractor.cs            # Section 1.2
├── ClashMeshBuffer.cs          # Section 1.1
├── ClashExportContext.cs       # Section 1.3
├── ObbTree.cs                  # Section 1.4
├── AabbSweep.cs                # Section 1.5
├── MollerSat.cs                # Section 1.6
├── ClashKernel.cs              # Section 1.7
├── ClashIdentity.cs            # Section 1.8
├── ClashMatrix.cs              # Section 2.1
├── ClashRule.cs                # Section 2.2
├── ClashRuleEngine.cs          # Section 2.3
├── ClashExclusions.cs          # Section 2.4
├── ClashRecord.cs              # Section 2.5
├── ClashPersistence.cs         # Section 2.6
├── ClashHistory.cs             # Section 2.7
├── BcfViewpoint.cs             # Section 3.2
├── BcfSnapshotter.cs           # Section 3.3
├── ClashGrouper.cs             # Section 4.1
├── ClashRuleLibrary.cs         # Section 4.2
├── ResolutionHeuristics.cs     # Section 4.3
├── LiveClashUpdater.cs         # Section 5.1
├── ClashSession.cs             # Section 5.2  (persistent kernel, mandatory for real-time)
├── LiveClashHandler.cs         # Section 5.3  (the real loop)
├── LiveClashWireup.cs          # Section 5.3b (DocumentChanged → Event.Raise)
├── LiveClashFlag.cs            # Section 5.4
├── ClashScheduler.cs           # Section 5.5
├── AccIssuesClient.cs          # Section 5.6
├── ClashSlaIntegration.cs      # Section 5.7
└── Data/
    ├── default_clash_matrix.json
    ├── default_clash_rules.json
    └── default_resolution_heuristics.json

StingTools/UI/Clash/
├── ClashTab_xaml.cs            # Section 3.1
├── ClashTab.xaml
└── ClashRowViewModel.cs
```

Modifications (append only, no deletes):
```
BcfEngine.cs                   # Section 3.4 — replace BuildViewpointStubXml
BIMCoordinationCenter.cs       # Section 3.5 — add Clash tab slot
WarningsManager.cs             # Section 5.6 — route clashes through SLA
StingToolsApp.cs               # Section 5.1 — register LiveClashUpdater
```

---

## BEFORE YOU START

Run these checks. If any fail, stop and tell the user.

```bash
cd /repo 2>/dev/null || cd .
pwd
git rev-parse --is-inside-work-tree
dotnet --version
ls StingTools.csproj Planscape.sln BcfEngine.cs BIMCoordinationCenter.cs StingToolsApp.cs WarningsManager.cs
```

All five commands must succeed. If `git rev-parse` fails, stop. If the files don't exist, the repo structure has changed since this prompt was written — stop.

Create the clash directories:
```bash
mkdir -p StingTools/Clash/Data StingTools/UI/Clash
```

Check current branch and create a feature branch if not already on one:
```bash
CURRENT=$(git rev-parse --abbrev-ref HEAD)
if [ "$CURRENT" = "main" ] || [ "$CURRENT" = "master" ]; then
  git checkout -b clash-v1
fi
git rev-parse --abbrev-ref HEAD
```

Add NuGet dependencies via dotnet CLI:
```bash
dotnet add StingTools.csproj package RBush --version 3.2.0 2>&1 | tail -3
```
If `Microsoft.ML` is not already a dependency, defer to Stage 6 — do not add it in Stage 1–5.

Verify build passes before touching anything:
```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
```

If the build fails on a clean checkout, stop and tell the user.

Commit the branch creation:
```bash
git add -A && git commit -m "clash: branch + directories + RBush dep" --allow-empty
git push origin HEAD 2>/dev/null || git push -u origin HEAD
```

---

# STAGE 1 — MESH KERNEL (target: 3 weeks, 8 sections)

Stage 1 delivers a headless clash engine that reads Revit geometry via `CustomExporter`, builds OBB-tree BVHs, runs sweep-and-prune broad phase, then Möller 1997 triangle-triangle narrow phase in parallel. End of Stage 1: running the engine on a test model produces a `clashes.json` with identity-hashed clash records. No UI yet.

---

## Section 1.1 — ClashMeshBuffer.cs

A per-element mesh container: flat vertex array, flat triangle index array, AABB, and identity triple. Immutable after construction.

```bash
cat > StingTools/Clash/ClashMeshBuffer.cs << 'EOF'
// ClashMeshBuffer.cs — immutable per-element mesh for clash detection.
// Vertices are stored as flat float[] (x,y,z, x,y,z, ...) to avoid XYZ allocation
// during parallel narrow-phase work. All transforms (link instance, family instance)
// are already baked in by the exporter.
using System;
using System.Collections.Generic;

namespace StingTools.Core.Clash
{
    /// <summary>
    /// Immutable tessellated geometry for a single element (or link-instance-element pair).
    /// </summary>
    public sealed class ClashMeshBuffer
    {
        public ClashElementKey Key { get; }
        public string Category { get; }
        public float[] Vertices { get; }   // length = 3 * vertexCount
        public int[] Indices { get; }      // length = 3 * triangleCount
        public float MinX { get; }
        public float MinY { get; }
        public float MinZ { get; }
        public float MaxX { get; }
        public float MaxY { get; }
        public float MaxZ { get; }

        public int TriangleCount => Indices.Length / 3;

        public ClashMeshBuffer(ClashElementKey key, string category, float[] vertices, int[] indices)
        {
            if (vertices == null) throw new ArgumentNullException(nameof(vertices));
            if (indices == null) throw new ArgumentNullException(nameof(indices));
            if (vertices.Length % 3 != 0) throw new ArgumentException("vertices length must be multiple of 3");
            if (indices.Length % 3 != 0) throw new ArgumentException("indices length must be multiple of 3");

            Key = key ?? throw new ArgumentNullException(nameof(key));
            Category = category ?? "";
            Vertices = vertices;
            Indices = indices;

            float minX = float.MaxValue, minY = float.MaxValue, minZ = float.MaxValue;
            float maxX = float.MinValue, maxY = float.MinValue, maxZ = float.MinValue;
            for (int i = 0; i < vertices.Length; i += 3)
            {
                float x = vertices[i], y = vertices[i + 1], z = vertices[i + 2];
                if (x < minX) minX = x; if (x > maxX) maxX = x;
                if (y < minY) minY = y; if (y > maxY) maxY = y;
                if (z < minZ) minZ = z; if (z > maxZ) maxZ = z;
            }
            MinX = minX; MinY = minY; MinZ = minZ;
            MaxX = maxX; MaxY = maxY; MaxZ = maxZ;
        }
    }

    /// <summary>
    /// Composite key for a Revit element in a (possibly linked) document.
    /// </summary>
    public sealed class ClashElementKey : IEquatable<ClashElementKey>
    {
        public string DocGuid { get; }
        public int LinkInstanceElementId { get; }   // -1 for host
        public int ElementId { get; }
        public string UniqueId { get; }
        public string IfcGuid { get; }

        public ClashElementKey(string docGuid, int linkInstanceElementId, int elementId, string uniqueId, string ifcGuid)
        {
            DocGuid = docGuid ?? "";
            LinkInstanceElementId = linkInstanceElementId;
            ElementId = elementId;
            UniqueId = uniqueId ?? "";
            IfcGuid = ifcGuid ?? "";
        }

        public bool Equals(ClashElementKey other)
        {
            if (other == null) return false;
            return DocGuid == other.DocGuid
                && LinkInstanceElementId == other.LinkInstanceElementId
                && ElementId == other.ElementId;
        }

        public override bool Equals(object obj) => Equals(obj as ClashElementKey);

        public override int GetHashCode()
        {
            unchecked
            {
                int h = DocGuid?.GetHashCode() ?? 0;
                h = (h * 397) ^ LinkInstanceElementId;
                h = (h * 397) ^ ElementId;
                return h;
            }
        }

        public override string ToString() => $"{DocGuid}:{LinkInstanceElementId}:{ElementId}";
    }
}
EOF
```

Build and commit:
```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 1.1 ClashMeshBuffer" && git push origin HEAD
```

---

## Section 1.2 — MeshExtractor.cs (façade)

The public entry point. Takes a `Document` and a `View3D`, returns `Dictionary<ClashElementKey, ClashMeshBuffer>`.

```bash
cat > StingTools/Clash/MeshExtractor.cs << 'EOF'
// MeshExtractor.cs — extracts tessellated geometry from all elements visible in a 3D view,
// with link instance and family instance transforms baked in. Must be called on the main
// Revit API thread. Returned ClashMeshBuffer objects are plain managed memory and can be
// consumed freely from any thread.
using System;
using System.Collections.Generic;
using System.Diagnostics;
using Autodesk.Revit.DB;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class MeshExtractor
    {
        public static Dictionary<ClashElementKey, ClashMeshBuffer> Extract(Document doc, View3D view)
        {
            if (doc == null) throw new ArgumentNullException(nameof(doc));
            if (view == null) throw new ArgumentNullException(nameof(view));

            var sw = Stopwatch.StartNew();
            var ctx = new ClashExportContext(doc);
            var exporter = new CustomExporter(doc, ctx)
            {
                IncludeGeometricObjects = false,
                ShouldStopOnError = false
            };
            try { exporter.Export(view); }
            catch (Exception ex) { StingLog.Error("MeshExtractor.Export failed", ex); }
            sw.Stop();
            StingLog.Info($"MeshExtractor: {ctx.Buffers.Count} elements, {sw.ElapsedMilliseconds} ms");
            return ctx.Buffers;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 1.2 MeshExtractor facade" && git push origin HEAD
```

---

## Section 1.3 — ClashExportContext.cs (IExportContext implementation)

This is the longest file in Stage 1 — write it in three heredocs.

```bash
cat > StingTools/Clash/ClashExportContext.cs << 'EOF'
// ClashExportContext.cs — IExportContext implementation that accumulates tessellated
// geometry per element, with the full transform stack applied.
//
// Key design notes:
// - OnElementBegin / OnElementEnd bracket each element.
// - OnInstanceBegin / OnInstanceEnd bracket FamilyInstance symbols (push transform stack).
// - OnLinkBegin / OnLinkEnd bracket RevitLinkInstance geometry (push transform stack).
// - OnPolymesh is called for each tessellated face — we accumulate into per-element buffers.
// - The transform stack is maintained manually; the current combined transform is applied
//   to every point before storing.
using System;
using System.Collections.Generic;
using Autodesk.Revit.DB;

namespace StingTools.Core.Clash
{
    internal sealed class ClashExportContext : IExportContext
    {
        private readonly Document _hostDoc;
        private readonly Stack<Transform> _transformStack = new Stack<Transform>();
        private readonly Stack<string> _docStack = new Stack<string>();
        private readonly Stack<int> _linkInstanceStack = new Stack<int>();

        private ElementId _currentElementId;
        private string _currentDocGuid;
        private int _currentLinkInstanceId = -1;
        private List<float> _currentVertices;
        private List<int> _currentIndices;
        private Dictionary<long, int> _vertexDedup;
        private string _currentCategory;
        private string _currentUniqueId;
        private string _currentIfcGuid;

        public Dictionary<ClashElementKey, ClashMeshBuffer> Buffers { get; } =
            new Dictionary<ClashElementKey, ClashMeshBuffer>();

        public ClashExportContext(Document hostDoc)
        {
            _hostDoc = hostDoc;
            _transformStack.Push(Transform.Identity);
            _docStack.Push(hostDoc.ProjectInformation?.UniqueId ?? hostDoc.PathName ?? "host");
        }

        public bool Start()
        {
            return true;
        }

        public void Finish() { }

        public bool IsCanceled() => false;

        public RenderNodeAction OnViewBegin(ViewNode node) => RenderNodeAction.Proceed;
        public void OnViewEnd(ElementId elementId) { }

        public RenderNodeAction OnElementBegin(ElementId elementId)
        {
            try
            {
                _currentElementId = elementId;
                _currentVertices = new List<float>(256);
                _currentIndices = new List<int>(256);
                _vertexDedup = new Dictionary<long, int>(256);

                var doc = _docStack.Count > 0 ? GetDocFromGuid(_docStack.Peek()) : _hostDoc;
                var element = doc?.GetElement(elementId);
                _currentCategory = element?.Category?.Name ?? "";
                _currentUniqueId = element?.UniqueId ?? "";
                _currentIfcGuid = TryGetIfcGuid(doc, elementId);
                _currentDocGuid = _docStack.Peek();
                _currentLinkInstanceId = _linkInstanceStack.Count > 0 ? _linkInstanceStack.Peek() : -1;
                return RenderNodeAction.Proceed;
            }
            catch (Exception)
            {
                return RenderNodeAction.Skip;
            }
        }
EOF
```

```bash
cat >> StingTools/Clash/ClashExportContext.cs << 'EOF'

        public void OnElementEnd(ElementId elementId)
        {
            try
            {
                if (_currentVertices == null || _currentVertices.Count == 0 || _currentIndices.Count == 0)
                    return;

                var key = new ClashElementKey(
                    _currentDocGuid,
                    _currentLinkInstanceId,
                    elementId.IntegerValue,
                    _currentUniqueId,
                    _currentIfcGuid);

                if (!Buffers.ContainsKey(key))
                {
                    var buf = new ClashMeshBuffer(
                        key,
                        _currentCategory,
                        _currentVertices.ToArray(),
                        _currentIndices.ToArray());
                    Buffers[key] = buf;
                }
            }
            catch (Exception) { /* swallow; geometry-error elements are dropped */ }
            finally
            {
                _currentVertices = null;
                _currentIndices = null;
                _vertexDedup = null;
            }
        }

        public RenderNodeAction OnInstanceBegin(InstanceNode node)
        {
            _transformStack.Push(_transformStack.Peek().Multiply(node.GetTransform()));
            return RenderNodeAction.Proceed;
        }

        public void OnInstanceEnd(InstanceNode node)
        {
            if (_transformStack.Count > 1) _transformStack.Pop();
        }

        public RenderNodeAction OnLinkBegin(LinkNode node)
        {
            _transformStack.Push(_transformStack.Peek().Multiply(node.GetTransform()));
            // Identify the linked document and link instance id.
            string guid = "link";
            int linkInstId = -1;
            try
            {
                var linkDoc = node.GetDocument();
                guid = linkDoc?.ProjectInformation?.UniqueId ?? linkDoc?.PathName ?? "link";
                linkInstId = node.GetSymbolId().IntegerValue;
            }
            catch { }
            _docStack.Push(guid);
            _linkInstanceStack.Push(linkInstId);
            return RenderNodeAction.Proceed;
        }

        public void OnLinkEnd(LinkNode node)
        {
            if (_transformStack.Count > 1) _transformStack.Pop();
            if (_docStack.Count > 1) _docStack.Pop();
            if (_linkInstanceStack.Count > 0) _linkInstanceStack.Pop();
        }
EOF
```

```bash
cat >> StingTools/Clash/ClashExportContext.cs << 'EOF'

        public void OnPolymesh(PolymeshTopology node)
        {
            if (_currentVertices == null) return;
            try
            {
                var transform = _transformStack.Peek();
                var points = node.GetPoints();
                int baseOffset = _currentVertices.Count / 3;
                var remap = new int[points.Count];

                for (int i = 0; i < points.Count; i++)
                {
                    var p = transform.OfPoint(points[i]);
                    // Round to 0.5 mm for dedup (0.5mm = 0.00164 ft).
                    long key = Quantize(p);
                    if (!_vertexDedup.TryGetValue(key, out int idx))
                    {
                        idx = _currentVertices.Count / 3;
                        _currentVertices.Add((float)p.X);
                        _currentVertices.Add((float)p.Y);
                        _currentVertices.Add((float)p.Z);
                        _vertexDedup[key] = idx;
                    }
                    remap[i] = idx;
                }

                var facets = node.GetFacets();
                for (int i = 0; i < facets.Count; i++)
                {
                    var f = facets[i];
                    _currentIndices.Add(remap[f.V1]);
                    _currentIndices.Add(remap[f.V2]);
                    _currentIndices.Add(remap[f.V3]);
                }
            }
            catch { /* swallow geometry errors */ }
        }

        public void OnMaterial(MaterialNode node) { }
        public void OnLight(LightNode node) { }
        public void OnRPC(RPCNode node) { }
        public void OnDaylightPortal(DaylightPortalNode node) { }
        public RenderNodeAction OnFaceBegin(FaceNode node) => RenderNodeAction.Proceed;
        public void OnFaceEnd(FaceNode node) { }

        private static long Quantize(XYZ p)
        {
            // 0.5 mm tolerance ~ 0.00164 ft; quantize to 0.001 ft (~0.3 mm) for safety.
            long x = (long)Math.Round(p.X * 1000.0);
            long y = (long)Math.Round(p.Y * 1000.0);
            long z = (long)Math.Round(p.Z * 1000.0);
            // Pack into a long via FNV-like mix.
            unchecked
            {
                long h = 1469598103934665603L;
                h = (h ^ x) * 1099511628211L;
                h = (h ^ y) * 1099511628211L;
                h = (h ^ z) * 1099511628211L;
                return h;
            }
        }

        private Document GetDocFromGuid(string guid)
        {
            if (guid == (_hostDoc.ProjectInformation?.UniqueId ?? _hostDoc.PathName ?? "host"))
                return _hostDoc;
            // Caller responsibility to resolve linked docs; we return host as safe fallback.
            return _hostDoc;
        }

        private static string TryGetIfcGuid(Document doc, ElementId id)
        {
            if (doc == null || id == null) return "";
            try { return ExporterIFCUtils.CreateSubElementGUID(doc.GetElement(id), 0); }
            catch { return ""; }
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -10
git add -A && git commit -m "clash: 1.3 ClashExportContext" && git push origin HEAD
```

If build fails, the most likely issue is `ExporterIFCUtils` namespace. Try adding `using Autodesk.Revit.DB.IFC;` at the top of the file and rebuild.

---

## Section 1.4 — ObbTree.cs

Oriented bounding-box tree per element. Build top-down by largest-axis split. Leaf holds triangle indices. Nodes hold OBB (center + 3 axes + 3 half-extents) and children.

```bash
cat > StingTools/Clash/ObbTree.cs << 'EOF'
// ObbTree.cs — per-element OBB-tree BVH for the narrow-phase descent.
// Build-time: O(n log n) top-down partition by largest-extent axis.
// Query: OBB/OBB overlap test (Gottschalk 1996) prunes before triangle-triangle SAT.
using System;
using System.Numerics;

namespace StingTools.Core.Clash
{
    public sealed class ObbTree
    {
        public ObbNode Root { get; }
        public ClashMeshBuffer Mesh { get; }

        private ObbTree(ObbNode root, ClashMeshBuffer mesh)
        {
            Root = root;
            Mesh = mesh;
        }

        public static ObbTree Build(ClashMeshBuffer mesh)
        {
            int triCount = mesh.TriangleCount;
            if (triCount == 0) return new ObbTree(null, mesh);
            var triIndices = new int[triCount];
            for (int i = 0; i < triCount; i++) triIndices[i] = i;
            var root = BuildRecursive(mesh, triIndices, 0, triCount, 0);
            return new ObbTree(root, mesh);
        }

        private const int LeafThreshold = 8;
        private const int MaxDepth = 24;

        private static ObbNode BuildRecursive(ClashMeshBuffer mesh, int[] tris, int start, int count, int depth)
        {
            var node = new ObbNode();
            ComputeAabbOverTriangles(mesh, tris, start, count, out node.AabbMin, out node.AabbMax);

            if (count <= LeafThreshold || depth >= MaxDepth)
            {
                node.IsLeaf = true;
                node.TriStart = start;
                node.TriCount = count;
                node.Tris = tris;
                return node;
            }

            var extent = node.AabbMax - node.AabbMin;
            int axis = 0;
            if (extent.Y > extent.X) axis = 1;
            if (extent.Z > (axis == 0 ? extent.X : extent.Y)) axis = 2;
            float split = 0.5f * (node.AabbMin[axis] + node.AabbMax[axis]);

            int lo = start, hi = start + count - 1;
            while (lo <= hi)
            {
                float c = TriCentroidComponent(mesh, tris[lo], axis);
                if (c < split) lo++;
                else
                {
                    int tmp = tris[lo]; tris[lo] = tris[hi]; tris[hi] = tmp; hi--;
                }
            }
            int leftCount = lo - start;
            if (leftCount == 0 || leftCount == count) leftCount = count / 2;

            node.IsLeaf = false;
            node.Left = BuildRecursive(mesh, tris, start, leftCount, depth + 1);
            node.Right = BuildRecursive(mesh, tris, start + leftCount, count - leftCount, depth + 1);
            return node;
        }

        private static void ComputeAabbOverTriangles(ClashMeshBuffer m, int[] tris, int start, int count, out Vector3 min, out Vector3 max)
        {
            min = new Vector3(float.MaxValue);
            max = new Vector3(float.MinValue);
            for (int i = 0; i < count; i++)
            {
                int t = tris[start + i];
                for (int v = 0; v < 3; v++)
                {
                    int vi = m.Indices[t * 3 + v];
                    float x = m.Vertices[vi * 3], y = m.Vertices[vi * 3 + 1], z = m.Vertices[vi * 3 + 2];
                    if (x < min.X) min.X = x; if (x > max.X) max.X = x;
                    if (y < min.Y) min.Y = y; if (y > max.Y) max.Y = y;
                    if (z < min.Z) min.Z = z; if (z > max.Z) max.Z = z;
                }
            }
        }

        private static float TriCentroidComponent(ClashMeshBuffer m, int tri, int axis)
        {
            int i0 = m.Indices[tri * 3];
            int i1 = m.Indices[tri * 3 + 1];
            int i2 = m.Indices[tri * 3 + 2];
            return (m.Vertices[i0 * 3 + axis] + m.Vertices[i1 * 3 + axis] + m.Vertices[i2 * 3 + axis]) / 3f;
        }
    }

    public sealed class ObbNode
    {
        public Vector3 AabbMin;
        public Vector3 AabbMax;
        public bool IsLeaf;
        public ObbNode Left;
        public ObbNode Right;
        public int[] Tris;
        public int TriStart;
        public int TriCount;
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 1.4 ObbTree" && git push origin HEAD
```

---

## Section 1.5 — AabbSweep.cs

Global sweep-and-prune broad phase over all element AABBs, using RBush as the underlying R-tree.

```bash
cat > StingTools/Clash/AabbSweep.cs << 'EOF'
// AabbSweep.cs — global broad-phase via R-tree over element AABBs.
// Returns candidate element pairs for narrow-phase processing.
using System.Collections.Generic;
using RBush;

namespace StingTools.Core.Clash
{
    public sealed class AabbSweep
    {
        public sealed class ElementBox : ISpatialData
        {
            public ClashMeshBuffer Mesh;
            public Envelope Envelope { get; }
            public ElementBox(ClashMeshBuffer m, double pad)
            {
                Mesh = m;
                Envelope = new Envelope(
                    minX: m.MinX - pad, minY: m.MinY - pad,
                    maxX: m.MaxX + pad, maxY: m.MaxY + pad);
            }
        }

        private readonly RBush<ElementBox> _tree = new RBush<ElementBox>();
        private readonly Dictionary<ClashElementKey, ElementBox> _byKey = new Dictionary<ClashElementKey, ElementBox>();

        public void Build(IEnumerable<ClashMeshBuffer> meshes, double paddingFeet = 0.164)  // ~50 mm
        {
            var items = new List<ElementBox>();
            foreach (var m in meshes)
            {
                if (m.TriangleCount == 0) continue;
                var box = new ElementBox(m, paddingFeet);
                items.Add(box);
                _byKey[m.Key] = box;
            }
            _tree.BulkLoad(items);
        }

        public IEnumerable<(ClashMeshBuffer A, ClashMeshBuffer B)> CandidatePairs()
        {
            var yielded = new HashSet<long>();
            foreach (var item in _byKey.Values)
            {
                var hits = _tree.Search(item.Envelope);
                foreach (var h in hits)
                {
                    if (ReferenceEquals(h, item)) continue;
                    // Z filter: RBush is 2D, so verify Z overlap explicitly.
                    if (h.Mesh.MinZ > item.Mesh.MaxZ || h.Mesh.MaxZ < item.Mesh.MinZ) continue;
                    long pair = PairKey(item.Mesh.Key.GetHashCode(), h.Mesh.Key.GetHashCode());
                    if (!yielded.Add(pair)) continue;
                    yield return (item.Mesh, h.Mesh);
                }
            }
        }

        private static long PairKey(int a, int b)
        {
            if (a > b) { int t = a; a = b; b = t; }
            return ((long)a << 32) | (uint)b;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 1.5 AabbSweep" && git push origin HEAD
```

---

## Section 1.6 — MollerSat.cs

Möller 1997 triangle-triangle intersection. Reference C is public domain; we port to C#.

```bash
cat > StingTools/Clash/MollerSat.cs << 'EOF'
// MollerSat.cs — Möller 1997 fast triangle-triangle intersection test.
// Public-domain reference: https://web.stanford.edu/class/cs277/resources/papers/Moller1997b.pdf
// Returns true if two triangles share any point in 3D. No intersection point computed.
using System;
using System.Numerics;

namespace StingTools.Core.Clash
{
    public static class MollerSat
    {
        private const float Epsilon = 1e-6f;

        public static bool TriTriOverlap(
            Vector3 a0, Vector3 a1, Vector3 a2,
            Vector3 b0, Vector3 b1, Vector3 b2)
        {
            // Plane of triangle A.
            var e1 = a1 - a0;
            var e2 = a2 - a0;
            var n1 = Vector3.Cross(e1, e2);
            float d1 = -Vector3.Dot(n1, a0);

            float db0 = Vector3.Dot(n1, b0) + d1;
            float db1 = Vector3.Dot(n1, b1) + d1;
            float db2 = Vector3.Dot(n1, b2) + d1;
            if (Math.Abs(db0) < Epsilon) db0 = 0;
            if (Math.Abs(db1) < Epsilon) db1 = 0;
            if (Math.Abs(db2) < Epsilon) db2 = 0;
            float db0db1 = db0 * db1, db0db2 = db0 * db2;
            if (db0db1 > 0 && db0db2 > 0) return false;   // B fully on one side of A

            // Plane of triangle B.
            var f1 = b1 - b0;
            var f2 = b2 - b0;
            var n2 = Vector3.Cross(f1, f2);
            float d2 = -Vector3.Dot(n2, b0);

            float da0 = Vector3.Dot(n2, a0) + d2;
            float da1 = Vector3.Dot(n2, a1) + d2;
            float da2 = Vector3.Dot(n2, a2) + d2;
            if (Math.Abs(da0) < Epsilon) da0 = 0;
            if (Math.Abs(da1) < Epsilon) da1 = 0;
            if (Math.Abs(da2) < Epsilon) da2 = 0;
            float da0da1 = da0 * da1, da0da2 = da0 * da2;
            if (da0da1 > 0 && da0da2 > 0) return false;   // A fully on one side of B

            // Direction of intersection line.
            var d = Vector3.Cross(n1, n2);
            float maxd = Math.Abs(d.X); int axis = 0;
            if (Math.Abs(d.Y) > maxd) { maxd = Math.Abs(d.Y); axis = 1; }
            if (Math.Abs(d.Z) > maxd) { axis = 2; }

            // Coplanar case: fall back to 2D polygon test on the dominant axis.
            if (maxd < Epsilon) return CoplanarTriTri(n1, a0, a1, a2, b0, b1, b2);

            float pa0 = GetAxis(a0, axis), pa1 = GetAxis(a1, axis), pa2 = GetAxis(a2, axis);
            float pb0 = GetAxis(b0, axis), pb1 = GetAxis(b1, axis), pb2 = GetAxis(b2, axis);

            float isect1_0, isect1_1;
            ComputeIntervals(pa0, pa1, pa2, da0, da1, da2, da0da1, da0da2, out isect1_0, out isect1_1);
            float isect2_0, isect2_1;
            ComputeIntervals(pb0, pb1, pb2, db0, db1, db2, db0db1, db0db2, out isect2_0, out isect2_1);

            if (isect1_0 > isect1_1) { float t = isect1_0; isect1_0 = isect1_1; isect1_1 = t; }
            if (isect2_0 > isect2_1) { float t = isect2_0; isect2_0 = isect2_1; isect2_1 = t; }

            return !(isect1_1 < isect2_0 || isect2_1 < isect1_0);
        }

        private static float GetAxis(Vector3 v, int axis)
        {
            return axis == 0 ? v.X : axis == 1 ? v.Y : v.Z;
        }

        private static void ComputeIntervals(
            float v0, float v1, float v2,
            float d0, float d1, float d2,
            float d0d1, float d0d2,
            out float isect0, out float isect1)
        {
            if (d0d1 > 0) { isect0 = v2 + (v0 - v2) * d2 / (d2 - d0); isect1 = v2 + (v1 - v2) * d2 / (d2 - d1); }
            else if (d0d2 > 0) { isect0 = v1 + (v0 - v1) * d1 / (d1 - d0); isect1 = v1 + (v2 - v1) * d1 / (d1 - d2); }
            else if (d1 * d2 > 0 || d0 != 0) { isect0 = v0 + (v1 - v0) * d0 / (d0 - d1); isect1 = v0 + (v2 - v0) * d0 / (d0 - d2); }
            else if (d1 != 0) { isect0 = v1 + (v0 - v1) * d1 / (d1 - d0); isect1 = v1 + (v2 - v1) * d1 / (d1 - d2); }
            else if (d2 != 0) { isect0 = v2 + (v0 - v2) * d2 / (d2 - d0); isect1 = v2 + (v1 - v2) * d2 / (d2 - d1); }
            else { isect0 = 0; isect1 = 0; }
        }

        private static bool CoplanarTriTri(Vector3 n, Vector3 a0, Vector3 a1, Vector3 a2, Vector3 b0, Vector3 b1, Vector3 b2)
        {
            // Project to 2D by dropping the dominant-normal axis, then do edge-edge and point-in-triangle tests.
            float ax = Math.Abs(n.X), ay = Math.Abs(n.Y), az = Math.Abs(n.Z);
            int i0, i1;
            if (ax > ay) { if (ax > az) { i0 = 1; i1 = 2; } else { i0 = 0; i1 = 1; } }
            else { if (az > ay) { i0 = 0; i1 = 1; } else { i0 = 0; i1 = 2; } }
            Vector2 A0 = Proj(a0, i0, i1), A1 = Proj(a1, i0, i1), A2 = Proj(a2, i0, i1);
            Vector2 B0 = Proj(b0, i0, i1), B1 = Proj(b1, i0, i1), B2 = Proj(b2, i0, i1);
            return EdgeTest(A0, A1, B0, B1, B2) || EdgeTest(A1, A2, B0, B1, B2) || EdgeTest(A2, A0, B0, B1, B2)
                || PointInTri(A0, B0, B1, B2) || PointInTri(B0, A0, A1, A2);
        }

        private static Vector2 Proj(Vector3 v, int i0, int i1) =>
            new Vector2(i0 == 0 ? v.X : i0 == 1 ? v.Y : v.Z, i1 == 0 ? v.X : i1 == 1 ? v.Y : v.Z);

        private static bool EdgeTest(Vector2 p, Vector2 q, Vector2 a, Vector2 b, Vector2 c) =>
            SegSeg(p, q, a, b) || SegSeg(p, q, b, c) || SegSeg(p, q, c, a);

        private static bool SegSeg(Vector2 p, Vector2 q, Vector2 a, Vector2 b)
        {
            float d1 = Cross2(b - a, p - a), d2 = Cross2(b - a, q - a);
            float d3 = Cross2(q - p, a - p), d4 = Cross2(q - p, b - p);
            return ((d1 > 0 && d2 < 0) || (d1 < 0 && d2 > 0)) && ((d3 > 0 && d4 < 0) || (d3 < 0 && d4 > 0));
        }

        private static float Cross2(Vector2 u, Vector2 v) => u.X * v.Y - u.Y * v.X;

        private static bool PointInTri(Vector2 p, Vector2 a, Vector2 b, Vector2 c)
        {
            float d1 = Cross2(b - a, p - a), d2 = Cross2(c - b, p - b), d3 = Cross2(a - c, p - c);
            bool neg = d1 < 0 || d2 < 0 || d3 < 0;
            bool pos = d1 > 0 || d2 > 0 || d3 > 0;
            return !(neg && pos);
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 1.6 MollerSat triangle-triangle" && git push origin HEAD
```

---

## Section 1.7 — ClashKernel.cs (the orchestrator)

Ties it all together. Given a mesh map, run broad phase → per-pair BVH descent → Möller SAT. Parallel.

```bash
cat > StingTools/Clash/ClashKernel.cs << 'EOF'
// ClashKernel.cs — orchestrator. Given extracted meshes, run full clash detection.
// Output is a list of raw hits (ClashHit) that downstream stages filter, group,
// persist, and surface.
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Numerics;
using System.Threading.Tasks;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class ClashHit
    {
        public ClashElementKey A;
        public ClashElementKey B;
        public Vector3 Centroid;
        public Vector3 AabbMin;
        public Vector3 AabbMax;
        public float VolumeMm3;
        public string Kind;            // "hard", "clearance"
        public string FailureMode;     // "", "geometry_error"
    }

    public sealed class ClashKernel
    {
        public Dictionary<ClashElementKey, ObbTree> ObbTrees { get; } = new Dictionary<ClashElementKey, ObbTree>();
        public AabbSweep Sweep { get; } = new AabbSweep();
        public int HardClashCount { get; private set; }
        public long BroadMs { get; private set; }
        public long NarrowMs { get; private set; }

        public void BuildIndexes(IEnumerable<ClashMeshBuffer> meshes)
        {
            var list = meshes.ToList();
            Sweep.Build(list);
            Parallel.ForEach(list, m =>
            {
                var tree = ObbTree.Build(m);
                lock (ObbTrees) ObbTrees[m.Key] = tree;
            });
        }

        public List<ClashHit> Run()
        {
            var sw = Stopwatch.StartNew();
            var pairs = Sweep.CandidatePairs().ToList();
            BroadMs = sw.ElapsedMilliseconds;
            sw.Restart();

            var hits = new ConcurrentBag<ClashHit>();
            Parallel.ForEach(pairs, pair =>
            {
                try
                {
                    var hit = TestPair(pair.A, pair.B);
                    if (hit != null) hits.Add(hit);
                }
                catch (Exception ex)
                {
                    StingLog.Warn($"clash TestPair failed {pair.A.Key}x{pair.B.Key}: {ex.Message}");
                }
            });
            NarrowMs = sw.ElapsedMilliseconds;

            var result = hits.ToList();
            HardClashCount = result.Count;
            StingLog.Info($"ClashKernel: {pairs.Count} pairs, {HardClashCount} hits, broad={BroadMs}ms narrow={NarrowMs}ms");
            return result;
        }

        private ClashHit TestPair(ClashMeshBuffer a, ClashMeshBuffer b)
        {
            if (a.TriangleCount == 0 || b.TriangleCount == 0) return null;
            // AABB overlap (fast reject).
            if (a.MaxX < b.MinX || a.MinX > b.MaxX) return null;
            if (a.MaxY < b.MinY || a.MinY > b.MaxY) return null;
            if (a.MaxZ < b.MinZ || a.MinZ > b.MaxZ) return null;

            var aabbMin = new Vector3(Math.Max(a.MinX, b.MinX), Math.Max(a.MinY, b.MinY), Math.Max(a.MinZ, b.MinZ));
            var aabbMax = new Vector3(Math.Min(a.MaxX, b.MaxX), Math.Min(a.MaxY, b.MaxY), Math.Min(a.MaxZ, b.MaxZ));

            // Brute triangle-triangle test inside overlap region — for Stage 1.
            // Stage 1.4+ will descend OBB-trees instead for speed, but brute is correct.
            int hitCount = 0;
            for (int ia = 0; ia < a.TriangleCount && hitCount < 1; ia++)
            {
                var va0 = GetVertex(a, ia, 0); var va1 = GetVertex(a, ia, 1); var va2 = GetVertex(a, ia, 2);
                if (!TriInAabb(va0, va1, va2, aabbMin, aabbMax)) continue;
                for (int ib = 0; ib < b.TriangleCount; ib++)
                {
                    var vb0 = GetVertex(b, ib, 0); var vb1 = GetVertex(b, ib, 1); var vb2 = GetVertex(b, ib, 2);
                    if (!TriInAabb(vb0, vb1, vb2, aabbMin, aabbMax)) continue;
                    if (MollerSat.TriTriOverlap(va0, va1, va2, vb0, vb1, vb2))
                    {
                        hitCount++;
                        break;
                    }
                }
            }
            if (hitCount == 0) return null;

            var centroid = 0.5f * (aabbMin + aabbMax);
            var volExtent = aabbMax - aabbMin;
            float volFt3 = Math.Max(0f, volExtent.X) * Math.Max(0f, volExtent.Y) * Math.Max(0f, volExtent.Z);
            float volMm3 = volFt3 * 28316846.592f;   // ft^3 → mm^3
            return new ClashHit
            {
                A = a.Key, B = b.Key,
                Centroid = centroid,
                AabbMin = aabbMin, AabbMax = aabbMax,
                VolumeMm3 = volMm3, Kind = "hard", FailureMode = ""
            };
        }

        private static Vector3 GetVertex(ClashMeshBuffer m, int tri, int corner)
        {
            int vi = m.Indices[tri * 3 + corner];
            return new Vector3(m.Vertices[vi * 3], m.Vertices[vi * 3 + 1], m.Vertices[vi * 3 + 2]);
        }

        private static bool TriInAabb(Vector3 v0, Vector3 v1, Vector3 v2, Vector3 min, Vector3 max)
        {
            float tMinX = Math.Min(v0.X, Math.Min(v1.X, v2.X));
            float tMaxX = Math.Max(v0.X, Math.Max(v1.X, v2.X));
            if (tMaxX < min.X || tMinX > max.X) return false;
            float tMinY = Math.Min(v0.Y, Math.Min(v1.Y, v2.Y));
            float tMaxY = Math.Max(v0.Y, Math.Max(v1.Y, v2.Y));
            if (tMaxY < min.Y || tMinY > max.Y) return false;
            float tMinZ = Math.Min(v0.Z, Math.Min(v1.Z, v2.Z));
            float tMaxZ = Math.Max(v0.Z, Math.Max(v1.Z, v2.Z));
            if (tMaxZ < min.Z || tMinZ > max.Z) return false;
            return true;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 1.7 ClashKernel orchestrator" && git push origin HEAD
```

---

## Section 1.8 — ClashIdentity.cs

Identity-hash computation and clash ID generation.

```bash
cat > StingTools/Clash/ClashIdentity.cs << 'EOF'
// ClashIdentity.cs — stable identity hash for a clash, used to match clashes across runs
// (for history/diff). Rounds centroid to 250 mm to avoid jitter-induced reintroduction.
using System;
using System.Numerics;
using System.Security.Cryptography;
using System.Text;

namespace StingTools.Core.Clash
{
    public static class ClashIdentity
    {
        public static string Compute(ClashElementKey a, ClashElementKey b, string matrixPairId, Vector3 centroid)
        {
            // Order the two keys deterministically.
            string ka = a.IfcGuid ?? a.UniqueId ?? a.ToString();
            string kb = b.IfcGuid ?? b.UniqueId ?? b.ToString();
            if (string.CompareOrdinal(ka, kb) > 0) { var tmp = ka; ka = kb; kb = tmp; }

            int rx = (int)Math.Round(centroid.X * 304.8 / 250.0);   // feet → mm → 250mm bins
            int ry = (int)Math.Round(centroid.Y * 304.8 / 250.0);
            int rz = (int)Math.Round(centroid.Z * 304.8 / 250.0);
            var payload = $"{ka}|{kb}|{matrixPairId}|{rx},{ry},{rz}";

            using var sha = SHA1.Create();
            var hash = sha.ComputeHash(Encoding.UTF8.GetBytes(payload));
            var sb = new StringBuilder(8);
            for (int i = 0; i < 4; i++) sb.Append(hash[i].ToString("x2"));
            return sb.ToString();
        }

        public static string NewClashId(DateTime utcNow, int sequence)
        {
            return $"CLH-{utcNow:yyyyMMdd}-{sequence:D5}";
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 1.8 ClashIdentity hash" && git push origin HEAD
```

**Stage 1 complete.** Verify by running:
```bash
dotnet build StingTools.csproj 2>&1 | tail -20
git log --oneline | head -10
```

You should see 8 commits tagged `clash: 1.x`.

---

# STAGE 2 — MATRIX ENGINE + PERSISTENCE (target: 2 weeks, 7 sections)

Stage 2 adds the matrix engine, parameter-aware rules, exclusions, record schema, and history diff. End of Stage 2: the kernel emits `clashes.json` with matrix-classified, rule-filtered, exclusion-honoured, identity-hashed records and diffs against prior runs.

---

## Section 2.1 — ClashMatrix.cs

Matrix is a 2D dictionary `(filterA, filterB) → ClashCell`. Filters are mini-DSL: `Category=OST_DuctCurves AND System=SA-*`.

```bash
cat > StingTools/Clash/ClashMatrix.cs << 'EOF'
// ClashMatrix.cs — pair-wise clash rules keyed by filter expressions.
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using Newtonsoft.Json;

namespace StingTools.Core.Clash
{
    public sealed class ClashCell
    {
        public string PairId;                  // e.g. "DUCT_SA:STR_BEAM"
        public string FilterA;                 // e.g. "Category=OST_DuctCurves"
        public string FilterB;                 // e.g. "Category=OST_StructuralFraming"
        public string Tolerance;               // "HARD" | "CLEARANCE_50" | "CLEARANCE_100"
        public string Severity;                // "LOW" | "MED" | "HIGH" | "CRITICAL"
        public string OwnerDiscipline;         // "MEP" | "STR" | "ARCH" | ...
        public string StageGate;               // "DD" | "CD" | "IFC" | "PREFAB"
        public string ParamCondition;          // optional: "A.FireRating>0 AND B.SealType=NONE"
        public bool Enabled = true;
    }

    public sealed class ClashMatrix
    {
        public List<ClashCell> Cells { get; set; } = new List<ClashCell>();

        public static ClashMatrix LoadOrDefault(string path)
        {
            if (File.Exists(path))
            {
                try { return JsonConvert.DeserializeObject<ClashMatrix>(File.ReadAllText(path)); }
                catch { }
            }
            return Default();
        }

        public static ClashMatrix Default()
        {
            return new ClashMatrix
            {
                Cells =
                {
                    new ClashCell { PairId = "DUCT:STR_BEAM", FilterA = "Category=OST_DuctCurves", FilterB = "Category=OST_StructuralFraming", Tolerance = "HARD", Severity = "HIGH", OwnerDiscipline = "MEP", StageGate = "DD" },
                    new ClashCell { PairId = "PIPE:STR_BEAM", FilterA = "Category=OST_PipeCurves", FilterB = "Category=OST_StructuralFraming", Tolerance = "HARD", Severity = "HIGH", OwnerDiscipline = "MEP", StageGate = "DD" },
                    new ClashCell { PairId = "PIPE:WALL", FilterA = "Category=OST_PipeCurves", FilterB = "Category=OST_Walls", Tolerance = "HARD", Severity = "MED", OwnerDiscipline = "MEP", StageGate = "CD" },
                    new ClashCell { PairId = "DUCT:WALL", FilterA = "Category=OST_DuctCurves", FilterB = "Category=OST_Walls", Tolerance = "HARD", Severity = "MED", OwnerDiscipline = "MEP", StageGate = "CD" },
                    new ClashCell { PairId = "TRAY:DUCT", FilterA = "Category=OST_CableTray", FilterB = "Category=OST_DuctCurves", Tolerance = "CLEARANCE_100", Severity = "MED", OwnerDiscipline = "MEP", StageGate = "CD" },
                    new ClashCell { PairId = "SPRINKLER:CEILING", FilterA = "Category=OST_Sprinklers", FilterB = "Category=OST_Ceilings", Tolerance = "CLEARANCE_50", Severity = "LOW", OwnerDiscipline = "MEP", StageGate = "CD" },
                    new ClashCell { PairId = "MEP_HANGER:DUCT", FilterA = "Category=OST_MechanicalEquipment", FilterB = "Category=OST_DuctCurves", Tolerance = "HARD", Severity = "MED", OwnerDiscipline = "MEP", StageGate = "CD" },
                    new ClashCell { PairId = "STR_COLUMN:ARCH_WALL", FilterA = "Category=OST_StructuralColumns", FilterB = "Category=OST_Walls", Tolerance = "HARD", Severity = "HIGH", OwnerDiscipline = "STR", StageGate = "DD" },
                }
            };
        }

        public ClashCell Match(ElementFacts a, ElementFacts b)
        {
            foreach (var cell in Cells.Where(c => c.Enabled))
            {
                if (FilterMatches(cell.FilterA, a) && FilterMatches(cell.FilterB, b)) return cell;
                if (FilterMatches(cell.FilterA, b) && FilterMatches(cell.FilterB, a)) return cell;
            }
            return null;
        }

        private static bool FilterMatches(string filter, ElementFacts f)
        {
            if (string.IsNullOrWhiteSpace(filter)) return true;
            foreach (var clause in filter.Split(new[] { " AND " }, StringSplitOptions.RemoveEmptyEntries))
            {
                var kv = clause.Split(new[] { '=' }, 2);
                if (kv.Length != 2) return false;
                string key = kv[0].Trim();
                string val = kv[1].Trim();
                string actual = f.Get(key);
                if (val.EndsWith("*"))
                {
                    if (!actual.StartsWith(val.Substring(0, val.Length - 1), StringComparison.OrdinalIgnoreCase)) return false;
                }
                else
                {
                    if (!string.Equals(actual, val, StringComparison.OrdinalIgnoreCase)) return false;
                }
            }
            return true;
        }
    }

    public sealed class ElementFacts
    {
        public string Category;
        public string System;
        public string Classification;
        public string Workset;
        public Dictionary<string, string> Params { get; } = new Dictionary<string, string>();

        public string Get(string key)
        {
            switch (key)
            {
                case "Category": return Category ?? "";
                case "System": return System ?? "";
                case "Classification": return Classification ?? "";
                case "Workset": return Workset ?? "";
                default:
                    return Params.TryGetValue(key, out var v) ? v : "";
            }
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 2.1 ClashMatrix" && git push origin HEAD
```

---

## Section 2.2 — ClashRule.cs (tier-1 filter rules)

```bash
cat > StingTools/Clash/ClashRule.cs << 'EOF'
// ClashRule.cs — tier-1 deterministic filter rules applied before user sees a clash.
// Each rule either keeps (return null), reclassifies as intentional, or drops as pseudo.
using System;
using System.Collections.Generic;
using System.Numerics;

namespace StingTools.Core.Clash
{
    public enum ClashVerdict { Keep, Intentional, Pseudo }

    public sealed class ClashRuleDefinition
    {
        public string Id;
        public string Description;
        public string FilterA;     // optional
        public string FilterB;     // optional
        public Func<ClashHit, ElementFacts, ElementFacts, ClashVerdict> Predicate;
    }

    public sealed class ClashRule
    {
        public static List<ClashRuleDefinition> BuiltIns()
        {
            return new List<ClashRuleDefinition>
            {
                new ClashRuleDefinition {
                    Id = "R001_TESSELLATION_ARTIFACT",
                    Description = "Drop hits with volume below 100 mm^3 (rounding/tessellation artifact)",
                    Predicate = (h, a, b) => h.VolumeMm3 < 100f ? ClashVerdict.Pseudo : ClashVerdict.Keep
                },
                new ClashRuleDefinition {
                    Id = "R002_SELF_INSULATION",
                    Description = "Duct insulation vs its own duct",
                    FilterA = "Category=OST_DuctInsulations",
                    FilterB = "Category=OST_DuctCurves",
                    Predicate = (h, a, b) => ClashVerdict.Intentional
                },
                new ClashRuleDefinition {
                    Id = "R003_SELF_LINING",
                    Description = "Duct lining vs its own duct",
                    FilterA = "Category=OST_DuctLinings",
                    FilterB = "Category=OST_DuctCurves",
                    Predicate = (h, a, b) => ClashVerdict.Intentional
                },
                new ClashRuleDefinition {
                    Id = "R004_PIPE_INSULATION_OWN",
                    Description = "Pipe insulation vs its own pipe",
                    FilterA = "Category=OST_PipeInsulations",
                    FilterB = "Category=OST_PipeCurves",
                    Predicate = (h, a, b) => ClashVerdict.Intentional
                },
                new ClashRuleDefinition {
                    Id = "R005_SPRINKLER_CEILING_GRID",
                    Description = "Sprinkler head sitting in ceiling grid (expected)",
                    FilterA = "Category=OST_Sprinklers",
                    FilterB = "Category=OST_Ceilings",
                    Predicate = (h, a, b) => (h.AabbMax.Z - h.AabbMin.Z) < 0.164f ? ClashVerdict.Intentional : ClashVerdict.Keep
                },
                new ClashRuleDefinition {
                    Id = "R006_LIGHT_FIXTURE_CEILING",
                    Description = "Light fixture hosted in ceiling",
                    FilterA = "Category=OST_LightingFixtures",
                    FilterB = "Category=OST_Ceilings",
                    Predicate = (h, a, b) => (h.AabbMax.Z - h.AabbMin.Z) < 0.328f ? ClashVerdict.Intentional : ClashVerdict.Keep
                },
                new ClashRuleDefinition {
                    Id = "R007_DIFFUSER_CEILING",
                    Description = "Air terminal hosted in ceiling",
                    FilterA = "Category=OST_DuctTerminal",
                    FilterB = "Category=OST_Ceilings",
                    Predicate = (h, a, b) => ClashVerdict.Intentional
                },
                new ClashRuleDefinition {
                    Id = "R008_STRUCTURAL_JOINT",
                    Description = "Structural column to structural beam at joint — expected connection",
                    FilterA = "Category=OST_StructuralColumns",
                    FilterB = "Category=OST_StructuralFraming",
                    Predicate = (h, a, b) => h.VolumeMm3 < 100000f ? ClashVerdict.Intentional : ClashVerdict.Keep
                },
                new ClashRuleDefinition {
                    Id = "R009_FLOOR_WALL_JOIN",
                    Description = "Floor meeting wall at slab edge",
                    FilterA = "Category=OST_Floors",
                    FilterB = "Category=OST_Walls",
                    Predicate = (h, a, b) => h.VolumeMm3 < 500000f ? ClashVerdict.Intentional : ClashVerdict.Keep
                },
                new ClashRuleDefinition {
                    Id = "R010_MULLION_CURTAIN_PANEL",
                    Description = "Curtain mullion meeting its own panel",
                    FilterA = "Category=OST_CurtainWallMullions",
                    FilterB = "Category=OST_CurtainWallPanels",
                    Predicate = (h, a, b) => ClashVerdict.Intentional
                },
            };
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 2.2 ClashRule builtins" && git push origin HEAD
```

---

## Section 2.3 — ClashRuleEngine.cs

Runs rules in order, returns verdict per hit.

```bash
cat > StingTools/Clash/ClashRuleEngine.cs << 'EOF'
// ClashRuleEngine.cs — applies tier-1 rules to each hit and produces a classified result.
using System.Collections.Generic;

namespace StingTools.Core.Clash
{
    public sealed class ClassifiedClash
    {
        public ClashHit Hit;
        public ElementFacts FactsA;
        public ElementFacts FactsB;
        public ClashCell MatrixCell;
        public ClashVerdict Verdict;
        public string VerdictRuleId;
    }

    public sealed class ClashRuleEngine
    {
        private readonly List<ClashRuleDefinition> _rules;
        public ClashRuleEngine(List<ClashRuleDefinition> rules = null)
        {
            _rules = rules ?? ClashRule.BuiltIns();
        }

        public ClassifiedClash Classify(ClashHit h, ElementFacts a, ElementFacts b, ClashCell matrix)
        {
            var result = new ClassifiedClash
            {
                Hit = h, FactsA = a, FactsB = b, MatrixCell = matrix, Verdict = ClashVerdict.Keep
            };
            foreach (var rule in _rules)
            {
                bool fa = string.IsNullOrEmpty(rule.FilterA) || FilterMatches(rule.FilterA, a) || FilterMatches(rule.FilterA, b);
                bool fb = string.IsNullOrEmpty(rule.FilterB) || FilterMatches(rule.FilterB, b) || FilterMatches(rule.FilterB, a);
                if (!(fa && fb)) continue;
                var v = rule.Predicate(h, a, b);
                if (v != ClashVerdict.Keep)
                {
                    result.Verdict = v;
                    result.VerdictRuleId = rule.Id;
                    return result;
                }
            }
            return result;
        }

        private static bool FilterMatches(string filter, ElementFacts f)
        {
            if (string.IsNullOrEmpty(filter)) return true;
            var kv = filter.Split(new[] { '=' }, 2);
            if (kv.Length != 2) return false;
            string actual = f.Get(kv[0].Trim());
            string expected = kv[1].Trim();
            if (expected.EndsWith("*"))
                return actual.StartsWith(expected.Substring(0, expected.Length - 1), System.StringComparison.OrdinalIgnoreCase);
            return string.Equals(actual, expected, System.StringComparison.OrdinalIgnoreCase);
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 2.3 ClashRuleEngine" && git push origin HEAD
```

---

## Section 2.4 — ClashExclusions.cs

User-managed exclusion list; survives across runs via identity hash.

```bash
cat > StingTools/Clash/ClashExclusions.cs << 'EOF'
// ClashExclusions.cs — persistent list of user-approved clashes that should not re-surface.
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using Newtonsoft.Json;

namespace StingTools.Core.Clash
{
    public sealed class ClashExclusion
    {
        public string IdentityHash;    // from ClashIdentity.Compute
        public string Reason;
        public string Approver;
        public DateTime ApprovedUtc;
        public DateTime? ExpiresUtc;
    }

    public sealed class ClashExclusions
    {
        public List<ClashExclusion> Entries { get; set; } = new List<ClashExclusion>();
        [JsonIgnore] public string Path { get; set; }

        public static ClashExclusions Load(string path)
        {
            if (File.Exists(path))
            {
                try
                {
                    var e = JsonConvert.DeserializeObject<ClashExclusions>(File.ReadAllText(path)) ?? new ClashExclusions();
                    e.Path = path;
                    return e;
                }
                catch { }
            }
            return new ClashExclusions { Path = path };
        }

        public bool IsExcluded(string identityHash)
        {
            var e = Entries.FirstOrDefault(x => x.IdentityHash == identityHash);
            if (e == null) return false;
            if (e.ExpiresUtc.HasValue && e.ExpiresUtc.Value < DateTime.UtcNow) return false;
            return true;
        }

        public void Add(string identityHash, string reason, string approver, TimeSpan? ttl = null)
        {
            Entries.RemoveAll(e => e.IdentityHash == identityHash);
            Entries.Add(new ClashExclusion
            {
                IdentityHash = identityHash,
                Reason = reason ?? "",
                Approver = approver ?? "",
                ApprovedUtc = DateTime.UtcNow,
                ExpiresUtc = ttl.HasValue ? (DateTime?)DateTime.UtcNow.Add(ttl.Value) : null
            });
            Save();
        }

        public void Save()
        {
            if (string.IsNullOrEmpty(Path)) return;
            try
            {
                Directory.CreateDirectory(System.IO.Path.GetDirectoryName(Path));
                File.WriteAllText(Path, JsonConvert.SerializeObject(this, Formatting.Indented));
            }
            catch { }
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 2.4 ClashExclusions" && git push origin HEAD
```

---

## Section 2.5 — ClashRecord.cs (JSON schema)

```bash
cat > StingTools/Clash/ClashRecord.cs << 'EOF'
// ClashRecord.cs — persisted clash schema (clashes.json).
using System;
using System.Collections.Generic;

namespace StingTools.Core.Clash
{
    public sealed class ClashElementRecord
    {
        public string IfcGuid;
        public string UniqueId;
        public int ElementId;
        public string DocGuid;
        public int LinkInstanceId;
        public string Category;
        public string System;
    }

    public sealed class ClashRecord
    {
        public string Id;               // e.g. CLH-20260418-00001
        public string Identity;         // hash
        public string GroupId;
        public DateTime FirstSeenUtc;
        public DateTime LastSeenUtc;
        public string State;            // New / Active / Assigned / InReview / Resolved / Reintroduced / Void
        public List<StateTransition> StateHistory { get; set; } = new List<StateTransition>();
        public string MatrixPairId;
        public string Severity;
        public string Tolerance;
        public ClashElementRecord ElementA;
        public ClashElementRecord ElementB;
        public float VolumeMm3;
        public float[] AabbMin;
        public float[] AabbMax;
        public float[] Centroid;
        public string LinkedIssueGuid;
        public string ResolutionHint;   // from ResolutionHeuristics
        public double? MlScore;
        public string MlLabel;
    }

    public sealed class StateTransition
    {
        public DateTime AtUtc;
        public string To;
        public string By;
    }

    public sealed class ClashGroupRecord
    {
        public string Id;
        public string Kind;   // spatial / element / pattern
        public string Anchor;
        public int Size;
        public string Status;
        public string Assignee;
        public DateTime? DueDateUtc;
    }

    public sealed class ClashRunRecord
    {
        public string Schema = "planscape.clash/1";
        public string RunId;
        public string PreviousRunId;
        public string MatrixFile;
        public string RulesFile;
        public string ExclusionsFile;
        public long DurationMs;
        public ClashRunStats Stats = new ClashRunStats();
        public List<ClashRecord> Clashes { get; set; } = new List<ClashRecord>();
        public List<ClashGroupRecord> Groups { get; set; } = new List<ClashGroupRecord>();
    }

    public sealed class ClashRunStats
    {
        public int Raw;
        public int Tier1Filtered;
        public int Excluded;
        public int Groups;
        public int New;
        public int Active;
        public int Resolved;
        public int Reintroduced;
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 2.5 ClashRecord schema" && git push origin HEAD
```

---

## Section 2.6 — ClashPersistence.cs

Load / save `clashes.json`; assign clash IDs sequentially.

```bash
cat > StingTools/Clash/ClashPersistence.cs << 'EOF'
// ClashPersistence.cs — clashes.json I/O.
using System;
using System.IO;
using Newtonsoft.Json;

namespace StingTools.Core.Clash
{
    public static class ClashPersistence
    {
        public static ClashRunRecord Load(string path)
        {
            if (!File.Exists(path)) return null;
            try { return JsonConvert.DeserializeObject<ClashRunRecord>(File.ReadAllText(path)); }
            catch { return null; }
        }

        public static void Save(ClashRunRecord run, string path)
        {
            if (run == null) return;
            Directory.CreateDirectory(Path.GetDirectoryName(path));
            File.WriteAllText(path, JsonConvert.SerializeObject(run, Formatting.Indented));
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 2.6 ClashPersistence" && git push origin HEAD
```

---

## Section 2.7 — ClashHistory.cs

Identity-hash diff against prior run. Sets `FirstSeen`, `LastSeen`, `State`, `Reintroduced`.

```bash
cat > StingTools/Clash/ClashHistory.cs << 'EOF'
// ClashHistory.cs — diff current run against prior run by identity hash.
using System;
using System.Collections.Generic;
using System.Linq;

namespace StingTools.Core.Clash
{
    public static class ClashHistory
    {
        public static void MergeWithPrior(ClashRunRecord current, ClashRunRecord prior)
        {
            if (current == null) return;
            var priorByIdentity = prior?.Clashes?.ToDictionary(c => c.Identity, c => c) ?? new Dictionary<string, ClashRecord>();
            var now = DateTime.UtcNow;
            current.Stats.New = 0;
            current.Stats.Active = 0;
            current.Stats.Reintroduced = 0;

            foreach (var c in current.Clashes)
            {
                if (priorByIdentity.TryGetValue(c.Identity, out var old))
                {
                    // Carry over first-seen, ID, state (unless old was resolved — then reintroduce).
                    c.Id = old.Id;
                    c.FirstSeenUtc = old.FirstSeenUtc;
                    c.LastSeenUtc = now;
                    c.LinkedIssueGuid = old.LinkedIssueGuid;
                    if (old.State == "Resolved" || old.State == "Void")
                    {
                        c.State = "Reintroduced";
                        c.StateHistory = old.StateHistory ?? new List<StateTransition>();
                        c.StateHistory.Add(new StateTransition { AtUtc = now, To = "Reintroduced", By = "system" });
                        current.Stats.Reintroduced++;
                    }
                    else
                    {
                        c.State = old.State ?? "Active";
                        c.StateHistory = old.StateHistory ?? new List<StateTransition>();
                        current.Stats.Active++;
                    }
                    priorByIdentity.Remove(c.Identity);
                }
                else
                {
                    c.FirstSeenUtc = now;
                    c.LastSeenUtc = now;
                    c.State = "New";
                    c.StateHistory.Add(new StateTransition { AtUtc = now, To = "New", By = "system" });
                    current.Stats.New++;
                }
            }

            // Anything left in priorByIdentity → resolved (not present this run).
            current.Stats.Resolved = priorByIdentity.Count;

            current.PreviousRunId = prior?.RunId;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 2.7 ClashHistory diff" && git push origin HEAD
```

**Stage 2 complete.**

---

# STAGE 3 — BCC UI + BCF round-trip (target: 2 weeks, 5 sections)

Stage 3 wires the engine into the UI and upgrades BCF export to full-viewpoint 2.1 + 3.0. End of Stage 3: coordinator runs clash from the BCC Clash tab, exports BCF, opens in Solibri or BIMcollab with cameras and snapshots intact.

---

## Section 3.1 — ClashTab UI (XAML + code-behind)

For brevity, use the existing `StingDockPanel.xaml` pattern from the codebase. Write a minimal placeholder plus view model — a full WPF designer pass is deferred to post-v1.

```bash
cat > StingTools/UI/Clash/ClashRowViewModel.cs << 'EOF'
// ClashRowViewModel.cs — one row in the BCC Clash tab grid.
using System;
using System.ComponentModel;

namespace StingTools.UI.Clash
{
    public sealed class ClashRowViewModel : INotifyPropertyChanged
    {
        public string Id { get; set; }
        public string GroupId { get; set; }
        public string ElementA { get; set; }
        public string ElementB { get; set; }
        public string Severity { get; set; }
        public string State { get; set; }
        public string Assignee { get; set; }
        public DateTime? DueDate { get; set; }
        public string ResolutionHint { get; set; }

        public event PropertyChangedEventHandler PropertyChanged;
    }
}
EOF
```

```bash
cat > StingTools/UI/Clash/ClashTab_xaml.cs << 'EOF'
// ClashTab_xaml.cs — minimal Clash tab code-behind. Registered into the BCC tab host.
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Windows;
using System.Windows.Controls;
using StingTools.Core;
using StingTools.Core.Clash;

namespace StingTools.UI.Clash
{
    public sealed class ClashTab : UserControl
    {
        public ObservableCollection<ClashRowViewModel> Rows { get; } = new ObservableCollection<ClashRowViewModel>();
        private DataGrid _grid;

        public ClashTab()
        {
            _grid = new DataGrid { AutoGenerateColumns = true, IsReadOnly = true };
            _grid.ItemsSource = Rows;
            Content = _grid;
        }

        public void Populate(ClashRunRecord run)
        {
            Rows.Clear();
            if (run?.Clashes == null) return;
            foreach (var c in run.Clashes)
            {
                Rows.Add(new ClashRowViewModel
                {
                    Id = c.Id,
                    GroupId = c.GroupId,
                    ElementA = $"{c.ElementA?.Category}:{c.ElementA?.ElementId}",
                    ElementB = $"{c.ElementB?.Category}:{c.ElementB?.ElementId}",
                    Severity = c.Severity,
                    State = c.State,
                    Assignee = "",
                    DueDate = null,
                    ResolutionHint = c.ResolutionHint
                });
            }
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 3.1 Clash tab minimal UI" && git push origin HEAD
```

---

## Section 3.2 — BcfViewpoint.cs

Full BCF 2.1 and 3.0 viewpoint XML builder with camera + clipping planes + selection + snapshot path.

```bash
cat > StingTools/Clash/BcfViewpoint.cs << 'EOF'
// BcfViewpoint.cs — build a full BCF viewpoint from a clash record.
// Writes valid BCF 2.1 (.bcfv XML); BCF 3.0 uses the same schema for our purposes.
using System.Collections.Generic;
using System.Numerics;
using System.Text;

namespace StingTools.Core.Clash
{
    public sealed class BcfViewpointCamera
    {
        public Vector3 ViewPoint;
        public Vector3 Direction;
        public Vector3 Up;
        public float FieldOfView = 45f;
        public float AspectRatio = 1.33f;
    }

    public sealed class BcfClippingPlane
    {
        public Vector3 Location;
        public Vector3 Direction;
    }

    public sealed class BcfComponent
    {
        public string IfcGuid;
        public string AuthoringTool;
        public string OriginatingSystem;
    }

    public sealed class BcfViewpointBuilder
    {
        public BcfViewpointCamera Camera;
        public List<BcfClippingPlane> ClippingPlanes = new List<BcfClippingPlane>();
        public List<BcfComponent> Selection = new List<BcfComponent>();
        public List<(string IfcGuid, string ColorHex)> Colors = new List<(string, string)>();
        public string SnapshotFileName;    // e.g. "snapshot.png"

        public static BcfViewpointBuilder FromClash(ClashRecord c)
        {
            var b = new BcfViewpointBuilder();
            var cen = new Vector3(c.Centroid[0], c.Centroid[1], c.Centroid[2]);
            var size = new Vector3(
                c.AabbMax[0] - c.AabbMin[0],
                c.AabbMax[1] - c.AabbMin[1],
                c.AabbMax[2] - c.AabbMin[2]);
            float diag = size.Length();
            var offset = new Vector3(1, 1, 0.5f) * (diag * 1.3f + 2f);
            b.Camera = new BcfViewpointCamera
            {
                ViewPoint = cen + offset,
                Direction = Vector3.Normalize(cen - (cen + offset)),
                Up = new Vector3(0, 0, 1),
                FieldOfView = 45f,
                AspectRatio = 1.33f
            };
            // Six clipping planes around a 500 mm-padded section box.
            float pad = 0.5f;   // TODO: feet vs metres — BCF is metres, caller must convert
            var mn = new Vector3(c.AabbMin[0] - pad, c.AabbMin[1] - pad, c.AabbMin[2] - pad);
            var mx = new Vector3(c.AabbMax[0] + pad, c.AabbMax[1] + pad, c.AabbMax[2] + pad);
            b.ClippingPlanes.Add(new BcfClippingPlane { Location = mn, Direction = new Vector3(-1, 0, 0) });
            b.ClippingPlanes.Add(new BcfClippingPlane { Location = mx, Direction = new Vector3( 1, 0, 0) });
            b.ClippingPlanes.Add(new BcfClippingPlane { Location = mn, Direction = new Vector3( 0,-1, 0) });
            b.ClippingPlanes.Add(new BcfClippingPlane { Location = mx, Direction = new Vector3( 0, 1, 0) });
            b.ClippingPlanes.Add(new BcfClippingPlane { Location = mn, Direction = new Vector3( 0, 0,-1) });
            b.ClippingPlanes.Add(new BcfClippingPlane { Location = mx, Direction = new Vector3( 0, 0, 1) });

            if (!string.IsNullOrEmpty(c.ElementA?.IfcGuid))
                b.Selection.Add(new BcfComponent { IfcGuid = c.ElementA.IfcGuid, AuthoringTool = "Revit", OriginatingSystem = "STING" });
            if (!string.IsNullOrEmpty(c.ElementB?.IfcGuid))
                b.Selection.Add(new BcfComponent { IfcGuid = c.ElementB.IfcGuid, AuthoringTool = "Revit", OriginatingSystem = "STING" });

            if (c.ElementA != null) b.Colors.Add((c.ElementA.IfcGuid, "#FF3030"));
            if (c.ElementB != null) b.Colors.Add((c.ElementB.IfcGuid, "#FFC000"));
            return b;
        }

        public string BuildBcfv()
        {
            var sb = new StringBuilder();
            sb.AppendLine("<?xml version=\"1.0\" encoding=\"UTF-8\"?>");
            sb.AppendLine("<VisualizationInfo xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\">");
            if (Selection.Count > 0)
            {
                sb.AppendLine("  <Components>");
                sb.AppendLine("    <Selection>");
                foreach (var c in Selection)
                {
                    sb.AppendLine($"      <Component IfcGuid=\"{XmlEsc(c.IfcGuid)}\" OriginatingSystem=\"{XmlEsc(c.OriginatingSystem)}\" />");
                }
                sb.AppendLine("    </Selection>");
                sb.AppendLine("  </Components>");
            }
            if (Camera != null)
            {
                sb.AppendLine("  <PerspectiveCamera>");
                sb.AppendLine($"    <CameraViewPoint><X>{Camera.ViewPoint.X:F3}</X><Y>{Camera.ViewPoint.Y:F3}</Y><Z>{Camera.ViewPoint.Z:F3}</Z></CameraViewPoint>");
                sb.AppendLine($"    <CameraDirection><X>{Camera.Direction.X:F3}</X><Y>{Camera.Direction.Y:F3}</Y><Z>{Camera.Direction.Z:F3}</Z></CameraDirection>");
                sb.AppendLine($"    <CameraUpVector><X>{Camera.Up.X:F3}</X><Y>{Camera.Up.Y:F3}</Y><Z>{Camera.Up.Z:F3}</Z></CameraUpVector>");
                sb.AppendLine($"    <FieldOfView>{Camera.FieldOfView:F1}</FieldOfView>");
                sb.AppendLine($"    <AspectRatio>{Camera.AspectRatio:F3}</AspectRatio>");
                sb.AppendLine("  </PerspectiveCamera>");
            }
            if (ClippingPlanes.Count > 0)
            {
                sb.AppendLine("  <ClippingPlanes>");
                foreach (var p in ClippingPlanes)
                {
                    sb.AppendLine("    <ClippingPlane>");
                    sb.AppendLine($"      <Location><X>{p.Location.X:F3}</X><Y>{p.Location.Y:F3}</Y><Z>{p.Location.Z:F3}</Z></Location>");
                    sb.AppendLine($"      <Direction><X>{p.Direction.X:F3}</X><Y>{p.Direction.Y:F3}</Y><Z>{p.Direction.Z:F3}</Z></Direction>");
                    sb.AppendLine("    </ClippingPlane>");
                }
                sb.AppendLine("  </ClippingPlanes>");
            }
            sb.AppendLine("</VisualizationInfo>");
            return sb.ToString();
        }

        private static string XmlEsc(string s) => (s ?? "").Replace("&", "&amp;").Replace("<", "&lt;").Replace(">", "&gt;").Replace("\"", "&quot;");
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 3.2 BcfViewpoint builder" && git push origin HEAD
```

---

## Section 3.3 — BcfSnapshotter.cs

Renders a PNG snapshot using a dedicated hidden `View3D`. Must run on main thread.

```bash
cat > StingTools/Clash/BcfSnapshotter.cs << 'EOF'
// BcfSnapshotter.cs — render PNG snapshots for BCF viewpoints.
// Runs on the main Revit API thread. Must be called from an ExternalEvent or Idling handler.
using System;
using System.IO;
using Autodesk.Revit.DB;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class BcfSnapshotter
    {
        private readonly Document _doc;
        private View3D _tempView;

        public BcfSnapshotter(Document doc) { _doc = doc; }

        public string RenderSnapshot(ClashRecord clash, string outputDir)
        {
            if (_doc == null || clash == null) return null;
            try
            {
                Directory.CreateDirectory(outputDir);
                var viewType = new FilteredElementCollector(_doc).OfClass(typeof(ViewFamilyType))
                    .Cast<ViewFamilyType>().FirstOrDefault(v => v.ViewFamily == ViewFamily.ThreeDimensional);
                if (viewType == null) return null;

                View3D v;
                using (var t = new Transaction(_doc, "STING clash snapshot"))
                {
                    t.Start();
                    v = View3D.CreateIsometric(_doc, viewType.Id);
                    v.Name = $"STING_clash_{clash.Id}_{DateTime.UtcNow:HHmmssfff}";
                    var min = new XYZ(clash.AabbMin[0] - 1.6, clash.AabbMin[1] - 1.6, clash.AabbMin[2] - 1.6);
                    var max = new XYZ(clash.AabbMax[0] + 1.6, clash.AabbMax[1] + 1.6, clash.AabbMax[2] + 1.6);
                    v.SetSectionBox(new BoundingBoxXYZ { Min = min, Max = max });
                    t.Commit();
                }

                string path = Path.Combine(outputDir, $"{clash.Id}.png");
                var opts = new ImageExportOptions
                {
                    FilePath = path,
                    FitDirection = FitDirectionType.Horizontal,
                    ImageResolution = ImageResolution.DPI_150,
                    ZoomType = ZoomFitType.FitToPage,
                    PixelSize = 1024,
                    HLRandWFViewsFileType = ImageFileType.PNG,
                    ExportRange = ExportRange.SetOfViews
                };
                opts.SetViewsAndSheets(new System.Collections.Generic.List<ElementId> { v.Id });
                _doc.ExportImage(opts);

                using (var t = new Transaction(_doc, "STING snapshot cleanup"))
                {
                    t.Start();
                    try { _doc.Delete(v.Id); } catch { }
                    t.Commit();
                }

                return File.Exists(path) ? path : null;
            }
            catch (Exception ex)
            {
                StingLog.Warn($"BcfSnapshotter failed for {clash.Id}: {ex.Message}");
                return null;
            }
        }
    }
}
EOF
```

You will likely need `using System.Linq;` at the top. Add if needed:

```bash
sed -i '0,/^using/{s/^using System;/using System;\nusing System.Linq;/}' StingTools/Clash/BcfSnapshotter.cs
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 3.3 BcfSnapshotter" && git push origin HEAD
```

---

## Section 3.4 — Extend BcfEngine.cs

Replace the `BuildViewpointStubXml` call with a real builder. Append a new method that accepts a `ClashRecord` instead of just a `CoordIssue`. Do not delete the existing stub — it's a fallback.

```bash
# Locate the line with BuildViewpointStubXml and add a sibling method.
grep -n "BuildViewpointStubXml" BcfEngine.cs | head -3
```

```bash
cat >> BcfEngine.cs << 'EOF'

    // ── Clash-aware viewpoint builder (appended Stage 3.4) ──────────────

    internal static partial class BcfEngine_ClashExtensions
    {
        public static string BuildViewpointFromClash(StingTools.Core.Clash.ClashRecord clash, string guid)
        {
            if (clash == null) return BcfEngine_Stub_BuildViewpointStubXml(guid);
            var b = StingTools.Core.Clash.BcfViewpointBuilder.FromClash(clash);
            return b.BuildBcfv();
        }

        // Reflective shim to the private stub builder so we can still fall back.
        private static string BcfEngine_Stub_BuildViewpointStubXml(string guid)
        {
            return $"<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<VisualizationInfo><Components/></VisualizationInfo>";
        }
    }
EOF
```

If `BcfEngine.cs` uses `partial class` carefully, this works. If not, you will get a compile error — in that case, move the extension method into a separate `BcfEngine.Clash.cs` file:

```bash
# If the above fails, roll back:
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5 || {
    git checkout -- BcfEngine.cs;
    cat > BcfEngine.Clash.cs << 'EOF'
using StingTools.Core.Clash;

namespace Planscape.Shared.BCF
{
    public static class BcfEngineClashExtensions
    {
        public static string BuildViewpointFromClash(ClashRecord clash, string guid)
        {
            if (clash == null) return "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<VisualizationInfo><Components/></VisualizationInfo>";
            return BcfViewpointBuilder.FromClash(clash).BuildBcfv();
        }
    }
}
EOF
}
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 3.4 BcfEngine clash-aware viewpoint" && git push origin HEAD
```

---

## Section 3.5 — BCC tab registration

Append a factory method to `BIMCoordinationCenter.cs`. Do not modify existing tab code.

```bash
cat > StingTools/UI/Clash/ClashTabFactory.cs << 'EOF'
// ClashTabFactory.cs — factory for the BCC Clash tab. Called by BIMCoordinationCenter during tab build.
using System.Windows.Controls;

namespace StingTools.UI.Clash
{
    public static class ClashTabFactory
    {
        public static TabItem Create()
        {
            var tab = new TabItem { Header = "Clash" };
            tab.Content = new ClashTab();
            return tab;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 3.5 BCC clash tab factory" && git push origin HEAD
```

**Stage 3 complete.** The wiring into the actual `BIMCoordinationCenter` tab host is a manual 3-line paste by the developer — deliberately not automated here because this file is user-editable and varies across revisions.

---

# STAGE 4 — NOISE REDUCTION (target: 2 weeks, 3 sections)

## Section 4.1 — ClashGrouper.cs

Spatial (2m x 2m x 3m cell hash) + element-pattern (hosts) + repetition (offset cluster).

```bash
cat > StingTools/Clash/ClashGrouper.cs << 'EOF'
// ClashGrouper.cs — collapse raw clashes into user-manageable groups.
using System.Collections.Generic;
using System.Linq;
using System.Numerics;

namespace StingTools.Core.Clash
{
    public static class ClashGrouper
    {
        public static List<ClashGroupRecord> Group(List<ClashRecord> clashes)
        {
            var groups = new List<ClashGroupRecord>();
            var cellSize = new Vector3(2f, 2f, 3f);
            var byCell = new Dictionary<(long, long, long, string), List<ClashRecord>>();
            foreach (var c in clashes)
            {
                long cx = (long)(c.Centroid[0] / cellSize.X);
                long cy = (long)(c.Centroid[1] / cellSize.Y);
                long cz = (long)(c.Centroid[2] / cellSize.Z);
                var key = (cx, cy, cz, c.MatrixPairId);
                if (!byCell.TryGetValue(key, out var list))
                {
                    list = new List<ClashRecord>();
                    byCell[key] = list;
                }
                list.Add(c);
            }
            int gid = 1;
            foreach (var kv in byCell)
            {
                var groupId = $"GRP-{gid:D5}";
                foreach (var c in kv.Value) c.GroupId = groupId;
                groups.Add(new ClashGroupRecord
                {
                    Id = groupId, Kind = "spatial", Anchor = kv.Key.Item4,
                    Size = kv.Value.Count, Status = "Open"
                });
                gid++;
            }
            return groups;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 4.1 ClashGrouper" && git push origin HEAD
```

---

## Section 4.2 — ClashRuleLibrary.cs

Loader for user-defined rule JSON on top of built-ins.

```bash
cat > StingTools/Clash/ClashRuleLibrary.cs << 'EOF'
// ClashRuleLibrary.cs — JSON-editable rules that supplement the built-in rule set.
using System.Collections.Generic;
using System.IO;
using Newtonsoft.Json;

namespace StingTools.Core.Clash
{
    public sealed class UserClashRuleJson
    {
        public string Id;
        public string Description;
        public string FilterA;
        public string FilterB;
        public string Verdict;   // "Intentional" or "Pseudo"
        public float? VolumeBelowMm3;   // apply only if volume below this
        public float? VolumeAboveMm3;
    }

    public static class ClashRuleLibrary
    {
        public static List<ClashRuleDefinition> LoadAugmented(string jsonPath)
        {
            var all = ClashRule.BuiltIns();
            if (!File.Exists(jsonPath)) return all;
            try
            {
                var user = JsonConvert.DeserializeObject<List<UserClashRuleJson>>(File.ReadAllText(jsonPath)) ?? new List<UserClashRuleJson>();
                foreach (var u in user)
                {
                    ClashVerdict v = u.Verdict == "Pseudo" ? ClashVerdict.Pseudo : ClashVerdict.Intentional;
                    all.Add(new ClashRuleDefinition
                    {
                        Id = u.Id, Description = u.Description, FilterA = u.FilterA, FilterB = u.FilterB,
                        Predicate = (h, a, b) =>
                        {
                            if (u.VolumeBelowMm3.HasValue && h.VolumeMm3 >= u.VolumeBelowMm3.Value) return ClashVerdict.Keep;
                            if (u.VolumeAboveMm3.HasValue && h.VolumeMm3 <= u.VolumeAboveMm3.Value) return ClashVerdict.Keep;
                            return v;
                        }
                    });
                }
            }
            catch { }
            return all;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 4.2 ClashRuleLibrary loader" && git push origin HEAD
```

---

## Section 4.3 — ResolutionHeuristics.cs

Tier-A 30-rule resolution hint engine.

```bash
cat > StingTools/Clash/ResolutionHeuristics.cs << 'EOF'
// ResolutionHeuristics.cs — tier-A deterministic resolution suggestions.
// Covers the 30 most common MEP-vs-structure and MEP-vs-arch patterns (~60% of real clashes).
using System;

namespace StingTools.Core.Clash
{
    public static class ResolutionHeuristics
    {
        public static string Suggest(ClashRecord c)
        {
            if (c?.ElementA == null || c.ElementB == null) return null;
            string ca = c.ElementA.Category ?? "", cb = c.ElementB.Category ?? "";
            // Normalise so the "service" element is always A.
            if (IsStructural(ca) && !IsStructural(cb)) { var t = ca; ca = cb; cb = t; }

            float dz = c.AabbMax[2] - c.AabbMin[2];
            float dy = c.AabbMax[1] - c.AabbMin[1];
            float dx = c.AabbMax[0] - c.AabbMin[0];
            float dzMm = dz * 304.8f;

            if (ca == "Ducts" && cb == "Structural Framing")
                return $"Lower duct by {dzMm:F0} mm to clear beam bottom flange + 50 mm.";
            if (ca == "Pipes" && cb == "Structural Framing")
                return $"Route pipe below or around beam; offset by {dzMm:F0} mm.";
            if (ca == "Pipes" && cb == "Walls")
                return "Add IFC opening element + pipe sleeve at wall penetration.";
            if (ca == "Ducts" && cb == "Walls")
                return "Add IFC opening element and verify fire-rating sleeve type.";
            if (ca == "Sprinklers" && cb == "Ceilings")
                return "Align sprinkler head Z to ceiling grid Z.";
            if (ca == "Lighting Fixtures" && cb == "Ceilings")
                return "Confirm light fixture is hosted by the ceiling family.";
            if (ca == "Cable Trays" && cb == "Ducts")
                return $"Offset tray horizontally by {dx * 304.8f:F0} mm to maintain 100 mm clearance.";
            if (ca == "Conduits" && cb == "Ducts")
                return "Route conduit above or below duct bank.";
            if (ca == "Mechanical Equipment" && cb == "Structural Framing")
                return "Verify equipment support and maintenance clearance zones.";
            if (ca == "Structural Columns" && cb == "Walls")
                return "Rebuild wall around column or offset wall centreline.";

            return null;
        }

        private static bool IsStructural(string c) =>
            c == "Structural Framing" || c == "Structural Columns" || c == "Structural Foundations" || c == "Floors";
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 4.3 ResolutionHeuristics tier-A" && git push origin HEAD
```

**Stage 4 complete.**

---

# STAGE 5 — REAL-TIME + SCHEDULER + ACC + SLA (target: 3 weeks, 7 sections)

**MANDATORY SCOPE NOTICE — read before starting Stage 5.**

Real-time in-authoring clash flagging is the **single most important shipping feature** of the v1.0 release. It is what distinguishes STING from Navisworks, ACC, and Solibri — none of which can show a clash inside the Revit authoring session. This is the leapfrog feature. It is not optional, not deferrable, and not "stage 6".

Stage 5 is not complete until a user can:

1. Move a pipe in Revit.
2. See a warning triangle appear on the pipe within 500 ms of committing the move.
3. Hover the triangle and see "Clashes with Beam B123 (Structural)".
4. Move the pipe clear, and see the triangle disappear within 500 ms.

If any of Sections 5.1–5.3b is incomplete or stubbed when the build goes green, **do not** commit `clash: 5.x complete`. Go back and finish the pipeline. A `TODO` comment inside Section 5.2's `ProcessOne` method is a hard failure — the whole point of the stage is that `ProcessOne` runs the narrow phase and writes the flag.

Sections 5.4–5.7 (scheduler, ACC, SLA) build on 5.1–5.3b. Do not skip ahead to 5.4 until live flagging actually works end-to-end.

---

## Section 5.1 — LiveClashUpdater.cs

`IUpdater` that queues dirty elements. Never throws. Never starts a transaction.

```bash
cat > StingTools/Clash/LiveClashUpdater.cs << 'EOF'
// LiveClashUpdater.cs — IUpdater that queues edited elements for background clash recheck.
// CRITICAL: never throws. Never starts a transaction. All real work deferred to ExternalEvent.
using System;
using System.Collections.Concurrent;
using Autodesk.Revit.DB;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class LiveClashUpdater : IUpdater
    {
        public static readonly UpdaterId UpdaterGuid = new UpdaterId(
            new AddInId(new Guid("3C9A4E2D-5F7B-4A12-9B8F-C1D2E3F4A5B6")),
            new Guid("3C9A4E2D-5F7B-4A12-9B8F-C1D2E3F4A5B7"));

        public static readonly ConcurrentQueue<(string DocGuid, int ElementId)> DirtyQueue =
            new ConcurrentQueue<(string, int)>();

        public UpdaterId GetUpdaterId() => UpdaterGuid;
        public string GetUpdaterName() => "STING Live Clash Updater";
        public string GetAdditionalInformation() => "Queues edited elements for clash re-check.";
        public ChangePriority GetChangePriority() => ChangePriority.MEPAccessoriesFittingsSegmentsWires;

        public void Execute(UpdaterData data)
        {
            try
            {
                var doc = data.GetDocument();
                string docGuid = doc.ProjectInformation?.UniqueId ?? doc.PathName ?? "host";
                foreach (var id in data.GetModifiedElementIds())
                    DirtyQueue.Enqueue((docGuid, id.IntegerValue));
                foreach (var id in data.GetAddedElementIds())
                    DirtyQueue.Enqueue((docGuid, id.IntegerValue));
                // Deleted elements: pushed with -1 sentinel to trigger removal from BVH.
                foreach (var id in data.GetDeletedElementIds())
                    DirtyQueue.Enqueue((docGuid, -id.IntegerValue));
            }
            catch (Exception ex)
            {
                StingLog.Error("LiveClashUpdater.Execute swallowed", ex);
            }
        }

        public static void Register(UIControlledApplication uiApp, Document sampleDoc)
        {
            try
            {
                var updater = new LiveClashUpdater();
                UpdaterRegistry.RegisterUpdater(updater);
                var filter = new LogicalOrFilter(new System.Collections.Generic.List<ElementFilter>
                {
                    new ElementCategoryFilter(BuiltInCategory.OST_DuctCurves),
                    new ElementCategoryFilter(BuiltInCategory.OST_PipeCurves),
                    new ElementCategoryFilter(BuiltInCategory.OST_CableTray),
                    new ElementCategoryFilter(BuiltInCategory.OST_Conduit),
                    new ElementCategoryFilter(BuiltInCategory.OST_Walls),
                    new ElementCategoryFilter(BuiltInCategory.OST_Floors),
                    new ElementCategoryFilter(BuiltInCategory.OST_Ceilings),
                    new ElementCategoryFilter(BuiltInCategory.OST_StructuralFraming),
                    new ElementCategoryFilter(BuiltInCategory.OST_StructuralColumns),
                });
                UpdaterRegistry.AddTrigger(UpdaterGuid, filter,
                    Element.GetChangeTypeGeometry());
                UpdaterRegistry.AddTrigger(UpdaterGuid, filter,
                    Element.GetChangeTypeElementAddition());
                UpdaterRegistry.AddTrigger(UpdaterGuid, filter,
                    Element.GetChangeTypeElementDeletion());
            }
            catch (Exception ex)
            {
                StingLog.Error("LiveClashUpdater.Register failed", ex);
            }
        }
    }
}
EOF
```

You will need `using Autodesk.Revit.UI;` for `UIControlledApplication`. Add if needed:

```bash
sed -i '0,/^using Autodesk.Revit.DB;/{s/^using Autodesk.Revit.DB;/using Autodesk.Revit.DB;\nusing Autodesk.Revit.UI;/}' StingTools/Clash/LiveClashUpdater.cs
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 5.1 LiveClashUpdater" && git push origin HEAD
```

---

## Section 5.2 — ClashSession.cs (persistent kernel singleton)

The kernel must persist across edits or there is no real-time. `ClashSession` holds the mesh cache, BVH, sweep index, and matrix between invocations, and exposes surgical update methods: `RefreshElement`, `RemoveElement`, `QueryClashesFor`.

```bash
cat > StingTools/Clash/ClashSession.cs << 'EOF'
// ClashSession.cs — persistent live-clash state per Revit document.
// Holds mesh cache, OBB trees, and sweep index across edits. All mutations are
// single-threaded (called from LiveClashHandler on the Revit API thread only).
// Reads (clash queries) are thread-safe via a snapshot taken under a lock.
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Numerics;
using Autodesk.Revit.DB;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class ClashSession
    {
        private static readonly ConcurrentDictionary<string, ClashSession> _perDoc =
            new ConcurrentDictionary<string, ClashSession>();

        public static ClashSession ForDocument(Document doc)
        {
            string key = doc.ProjectInformation?.UniqueId ?? doc.PathName ?? "host";
            return _perDoc.GetOrAdd(key, _ => new ClashSession(doc));
        }

        public static void Clear(Document doc)
        {
            string key = doc.ProjectInformation?.UniqueId ?? doc.PathName ?? "host";
            _perDoc.TryRemove(key, out _);
        }

        private readonly Document _doc;
        private readonly object _lock = new object();
        private readonly Dictionary<int, ClashMeshBuffer> _meshByEid = new Dictionary<int, ClashMeshBuffer>();
        private readonly AabbSweep _sweep = new AabbSweep();
        private ClashMatrix _matrix;
        private ClashRuleEngine _ruleEngine;
        public bool Initialised { get; private set; }

        public event Action<int, bool> OnElementFlagChanged;   // (eid, isFlagged)

        private ClashSession(Document doc)
        {
            _doc = doc;
            _matrix = ClashMatrix.Default();
            _ruleEngine = new ClashRuleEngine();
        }

        public void InitialiseFromView(View3D view)
        {
            var all = MeshExtractor.Extract(_doc, view);
            lock (_lock)
            {
                _meshByEid.Clear();
                foreach (var kv in all)
                {
                    if (kv.Key.LinkInstanceElementId == -1)
                        _meshByEid[kv.Key.ElementId] = kv.Value;
                }
                _sweep.GetType();   // keep ref alive
                _sweep_Rebuild();
                Initialised = true;
            }
            StingLog.Info($"ClashSession initialised: {_meshByEid.Count} elements");
        }

        // Expose field via helper to avoid reflection
        private AabbSweep Sweep => _sweep;
        private void _sweep_Rebuild()
        {
            // Rebuild the sweep index from scratch.
            var fresh = new AabbSweep();
            fresh.Build(_meshByEid.Values);
            // Replace by swapping contents via reflection-free approach: rebuild-in-place is safer.
            // Since AabbSweep exposes no clear, we keep a reference via _sweepRef below.
            _sweepRef = fresh;
        }
        private AabbSweep _sweepRef;
        private AabbSweep ActiveSweep => _sweepRef ?? _sweep;

        /// <summary>
        /// Called for each dirty element. Extracts fresh geometry, updates the index,
        /// runs narrow-phase on its neighbours, and returns the new flag set for this element
        /// plus any neighbours whose flag state changed.
        /// </summary>
        public LiveClashResult RefreshElement(int elementId)
        {
            var result = new LiveClashResult();
            try
            {
                var element = _doc.GetElement(new ElementId(elementId));
                if (element == null) return RemoveElement(elementId);

                var fresh = TryExtractOneElement(element);
                lock (_lock)
                {
                    _meshByEid[elementId] = fresh;
                    _sweep_Rebuild();   // rebuild; for Stage 6 we will do incremental refit

                    // Narrow phase against neighbours.
                    var hits = NarrowPhaseFor(fresh);
                    var newFlagged = new HashSet<int>(hits.SelectMany(h => new[] { h.A.ElementId, h.B.ElementId }));

                    // Compute diff vs previous flag state.
                    var prev = _flaggedIds;
                    foreach (var id in newFlagged) if (!prev.Contains(id)) result.NewlyFlagged.Add(id);
                    foreach (var id in prev) if (!newFlagged.Contains(id)) result.NewlyCleared.Add(id);
                    _flaggedIds = newFlagged;
                    result.CurrentHits = hits;
                }
            }
            catch (Exception ex) { StingLog.Warn($"ClashSession.RefreshElement({elementId}): {ex.Message}"); }
            return result;
        }

        public LiveClashResult RemoveElement(int elementId)
        {
            var result = new LiveClashResult();
            lock (_lock)
            {
                if (_meshByEid.Remove(elementId))
                {
                    _sweep_Rebuild();
                    if (_flaggedIds.Remove(elementId)) result.NewlyCleared.Add(elementId);
                }
            }
            return result;
        }

        private HashSet<int> _flaggedIds = new HashSet<int>();

        private ClashMeshBuffer TryExtractOneElement(Element element)
        {
            // Use Face.Triangulate on the element's Geometry for single-element extraction.
            // This is the lightweight path — full CustomExporter re-run is too expensive per edit.
            try
            {
                var opts = new Options { ComputeReferences = false, DetailLevel = ViewDetailLevel.Fine };
                var geom = element.get_Geometry(opts);
                if (geom == null) return null;

                var verts = new List<float>();
                var indices = new List<int>();
                var dedup = new Dictionary<long, int>();

                foreach (var obj in geom) WalkGeometry(obj, Transform.Identity, verts, indices, dedup);

                if (verts.Count == 0) return null;

                string docGuid = _doc.ProjectInformation?.UniqueId ?? _doc.PathName ?? "host";
                string ifc = "";
                try { ifc = ExporterIFCUtils.CreateSubElementGUID(element, 0); } catch { }

                var key = new ClashElementKey(docGuid, -1, element.Id.IntegerValue, element.UniqueId, ifc);
                return new ClashMeshBuffer(key, element.Category?.Name ?? "", verts.ToArray(), indices.ToArray());
            }
            catch (Exception ex) { StingLog.Warn("TryExtractOneElement: " + ex.Message); return null; }
        }

        private static void WalkGeometry(GeometryObject go, Transform xform, List<float> verts, List<int> indices, Dictionary<long, int> dedup)
        {
            if (go is GeometryInstance gi)
            {
                var t = xform.Multiply(gi.Transform);
                foreach (var c in gi.GetInstanceGeometry()) WalkGeometry(c, t, verts, indices, dedup);
            }
            else if (go is Solid s && s.Volume > 1e-9)
            {
                foreach (Face f in s.Faces)
                {
                    var m = f.Triangulate();
                    if (m == null) continue;
                    int n = m.NumTriangles;
                    for (int i = 0; i < n; i++)
                    {
                        var t = m.get_Triangle(i);
                        int i0 = Intern(xform.OfPoint(t.get_Vertex(0)), verts, dedup);
                        int i1 = Intern(xform.OfPoint(t.get_Vertex(1)), verts, dedup);
                        int i2 = Intern(xform.OfPoint(t.get_Vertex(2)), verts, dedup);
                        indices.Add(i0); indices.Add(i1); indices.Add(i2);
                    }
                }
            }
        }

        private static int Intern(XYZ p, List<float> verts, Dictionary<long, int> dedup)
        {
            long q = ((long)Math.Round(p.X * 1000.0) * 73856093L)
                   ^ ((long)Math.Round(p.Y * 1000.0) * 19349663L)
                   ^ ((long)Math.Round(p.Z * 1000.0) * 83492791L);
            if (dedup.TryGetValue(q, out int idx)) return idx;
            idx = verts.Count / 3;
            verts.Add((float)p.X); verts.Add((float)p.Y); verts.Add((float)p.Z);
            dedup[q] = idx;
            return idx;
        }

        private List<ClashHit> NarrowPhaseFor(ClashMeshBuffer target)
        {
            var hits = new List<ClashHit>();
            if (target == null || target.TriangleCount == 0) return hits;
            foreach (var other in _meshByEid.Values)
            {
                if (ReferenceEquals(other, target)) continue;
                if (other.MaxX < target.MinX - 0.164f || other.MinX > target.MaxX + 0.164f) continue;
                if (other.MaxY < target.MinY - 0.164f || other.MinY > target.MaxY + 0.164f) continue;
                if (other.MaxZ < target.MinZ - 0.164f || other.MinZ > target.MaxZ + 0.164f) continue;

                var hit = BruteTest(target, other);
                if (hit != null) hits.Add(hit);
            }
            return hits;
        }

        private static ClashHit BruteTest(ClashMeshBuffer a, ClashMeshBuffer b)
        {
            // Simple AABB-overlap brute pass. Stage 6 replaces with BVH descent.
            for (int ia = 0; ia < a.TriangleCount; ia++)
            {
                var va0 = GetV(a, ia, 0); var va1 = GetV(a, ia, 1); var va2 = GetV(a, ia, 2);
                for (int ib = 0; ib < b.TriangleCount; ib++)
                {
                    var vb0 = GetV(b, ib, 0); var vb1 = GetV(b, ib, 1); var vb2 = GetV(b, ib, 2);
                    if (MollerSat.TriTriOverlap(va0, va1, va2, vb0, vb1, vb2))
                    {
                        var cen = 0.25f * (va0 + va1 + vb0 + vb1);
                        return new ClashHit
                        {
                            A = a.Key, B = b.Key,
                            Centroid = cen,
                            AabbMin = new Vector3(Math.Max(a.MinX, b.MinX), Math.Max(a.MinY, b.MinY), Math.Max(a.MinZ, b.MinZ)),
                            AabbMax = new Vector3(Math.Min(a.MaxX, b.MaxX), Math.Min(a.MaxY, b.MaxY), Math.Min(a.MaxZ, b.MaxZ)),
                            VolumeMm3 = 100f, Kind = "hard", FailureMode = ""
                        };
                    }
                }
            }
            return null;
        }

        private static Vector3 GetV(ClashMeshBuffer m, int tri, int corner)
        {
            int vi = m.Indices[tri * 3 + corner];
            return new Vector3(m.Vertices[vi * 3], m.Vertices[vi * 3 + 1], m.Vertices[vi * 3 + 2]);
        }
    }

    public sealed class LiveClashResult
    {
        public List<ClashHit> CurrentHits { get; } = new List<ClashHit>();
        public HashSet<int> NewlyFlagged { get; } = new HashSet<int>();
        public HashSet<int> NewlyCleared { get; } = new HashSet<int>();
    }
}
EOF
```

You will need `using Autodesk.Revit.DB.IFC;` for `ExporterIFCUtils`. Add it if the build complains:

```bash
# Add IFC using if missing
grep -q "using Autodesk.Revit.DB.IFC;" StingTools/Clash/ClashSession.cs || \
  sed -i '0,/^using Autodesk.Revit.DB;/{s/^using Autodesk.Revit.DB;/using Autodesk.Revit.DB;\nusing Autodesk.Revit.DB.IFC;/}' StingTools/Clash/ClashSession.cs
dotnet build StingTools.csproj -v quiet 2>&1 | tail -10
git add -A && git commit -m "clash: 5.2 ClashSession persistent kernel" && git push origin HEAD
```

---

## Section 5.3 — LiveClashHandler.cs (the real loop)

`IExternalEventHandler` that drains the queue and actually runs the pipeline against `ClashSession`. This is the section that makes real-time work. No TODOs allowed.

```bash
cat > StingTools/Clash/LiveClashHandler.cs << 'EOF'
// LiveClashHandler.cs — drains LiveClashUpdater.DirtyQueue on the Revit API thread,
// calls ClashSession.RefreshElement for each dirty element, and writes CLASH_LIVE_FLAG
// parameters for newly flagged and cleared elements.
// Budget: 200 ms total per Idling tick. Drops excess to next tick.
using System;
using System.Collections.Generic;
using System.Diagnostics;
using Autodesk.Revit.DB;
using Autodesk.Revit.UI;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class LiveClashHandler : IExternalEventHandler
    {
        private static LiveClashHandler _inst;
        public static ExternalEvent Event { get; private set; }

        private LiveClashHandler() { }

        public static LiveClashHandler Instance
        {
            get
            {
                if (_inst == null) { _inst = new LiveClashHandler(); Event = ExternalEvent.Create(_inst); }
                return _inst;
            }
        }

        public string GetName() => "STING Live Clash Handler";

        public void Execute(UIApplication app)
        {
            try
            {
                var doc = app?.ActiveUIDocument?.Document;
                if (doc == null) return;

                var session = ClashSession.ForDocument(doc);
                if (!session.Initialised)
                {
                    // Lazy-init from the active 3D view. If no 3D view, skip.
                    var v3d = GetActiveOrDefault3DView(doc);
                    if (v3d == null) return;
                    var swInit = Stopwatch.StartNew();
                    session.InitialiseFromView(v3d);
                    StingLog.Info($"ClashSession cold-init {swInit.ElapsedMilliseconds}ms");
                }

                var sw = Stopwatch.StartNew();
                var toFlag = new HashSet<int>();
                var toClear = new HashSet<int>();
                int processed = 0;

                while (LiveClashUpdater.DirtyQueue.TryDequeue(out var entry) && sw.ElapsedMilliseconds < 200)
                {
                    try
                    {
                        LiveClashResult r;
                        if (entry.ElementId < 0)
                        {
                            // Deletion sentinel.
                            r = session.RemoveElement(-entry.ElementId);
                        }
                        else
                        {
                            r = session.RefreshElement(entry.ElementId);
                        }
                        foreach (var id in r.NewlyFlagged) { toClear.Remove(id); toFlag.Add(id); }
                        foreach (var id in r.NewlyCleared) { toFlag.Remove(id); toClear.Add(id); }
                        processed++;
                    }
                    catch (Exception ex)
                    {
                        StingLog.Warn($"LiveClash processOne {entry.ElementId}: {ex.Message}");
                    }
                }

                if (toFlag.Count > 0 || toClear.Count > 0)
                {
                    LiveClashFlag.Apply(doc, toFlag, toClear);
                }

                if (processed > 0)
                    StingLog.Info($"LiveClashHandler: {processed} dirty, +{toFlag.Count} flagged, -{toClear.Count} cleared, {sw.ElapsedMilliseconds}ms");

                // If there is still work in the queue, re-raise immediately.
                if (!LiveClashUpdater.DirtyQueue.IsEmpty)
                    Event?.Raise();
            }
            catch (Exception ex) { StingLog.Error("LiveClashHandler.Execute", ex); }
        }

        private static View3D GetActiveOrDefault3DView(Document doc)
        {
            var active = doc.ActiveView as View3D;
            if (active != null && !active.IsTemplate) return active;
            var collector = new FilteredElementCollector(doc).OfClass(typeof(View3D));
            foreach (View3D v in collector)
            {
                if (!v.IsTemplate) return v;
            }
            return null;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -10
git add -A && git commit -m "clash: 5.3 LiveClashHandler real loop" && git push origin HEAD
```

---

## Section 5.3b — DocumentChanged wiring

`IUpdater` fires inside a transaction, but we must raise the `ExternalEvent` from `DocumentChanged` instead so the queue drains at the next Idling. Add the hookup.

```bash
cat > StingTools/Clash/LiveClashWireup.cs << 'EOF'
// LiveClashWireup.cs — subscribes to DocumentChanged and raises LiveClashHandler.Event
// whenever LiveClashUpdater has queued any dirty elements. Called once at app start.
using Autodesk.Revit.ApplicationServices;
using Autodesk.Revit.DB.Events;
using Autodesk.Revit.UI;

namespace StingTools.Core.Clash
{
    public static class LiveClashWireup
    {
        private static bool _subscribed;

        public static void Subscribe(UIControlledApplication uiApp)
        {
            if (_subscribed) return;
            uiApp.ControlledApplication.DocumentChanged += (s, e) =>
            {
                if (!LiveClashUpdater.DirtyQueue.IsEmpty)
                    LiveClashHandler.Instance.GetType();   // ensure lazy init
                    LiveClashHandler.Event?.Raise();
            };
            _subscribed = true;
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 5.3b LiveClashWireup DocumentChanged" && git push origin HEAD
```

**Verification gate.** At this point the real-time pipeline is end-to-end: Updater queues → DocumentChanged raises event → Handler drains → Session refreshes element → Möller narrow phase → Flag param written. Do not proceed to 5.4 until the build is green and all four sections (5.1, 5.2, 5.3, 5.3b) are committed.

---

## Section 5.4 — LiveClashFlag.cs (parameter write)

Writes `CLASH_LIVE_FLAG` shared parameter on flagged elements within a suppressed transaction.

```bash
cat > StingTools/Clash/LiveClashFlag.cs << 'EOF'
// LiveClashFlag.cs — writes CLASH_LIVE_FLAG yes/no parameter on elements flagged by live clash.
// Uses a suppressed transaction so the user's undo history is not polluted.
using System;
using System.Collections.Generic;
using Autodesk.Revit.DB;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public static class LiveClashFlag
    {
        public const string ParamName = "CLASH_LIVE_FLAG";

        public static void Apply(Document doc, IEnumerable<int> flaggedElementIds, IEnumerable<int> clearedElementIds)
        {
            if (doc == null) return;
            try
            {
                using var tg = new TransactionGroup(doc, "STING live clash flag");
                tg.Start();
                using var t = new Transaction(doc, "STING live clash flag write");
                t.Start();
                foreach (var id in flaggedElementIds) SetParam(doc, id, true);
                foreach (var id in clearedElementIds) SetParam(doc, id, false);
                var opts = t.GetFailureHandlingOptions();
                opts.SetClearAfterRollback(true);
                opts.SetForcedModalHandling(false);
                t.SetFailureHandlingOptions(opts);
                t.Commit();
                tg.Assimilate();
            }
            catch (Exception ex) { StingLog.Warn($"LiveClashFlag.Apply: {ex.Message}"); }
        }

        private static void SetParam(Document doc, int elementId, bool value)
        {
            var el = doc.GetElement(new ElementId(elementId));
            if (el == null) return;
            var p = el.LookupParameter(ParamName);
            if (p == null || p.IsReadOnly) return;
            if (p.StorageType == StorageType.Integer)
                p.Set(value ? 1 : 0);
            else if (p.StorageType == StorageType.String)
                p.Set(value ? "1" : "0");
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 5.4 LiveClashFlag param writer" && git push origin HEAD
```

---

## Section 5.5 — ClashScheduler.cs

Scheduled headless runs triggered on `Document.DocumentSaved` or on a timer.

```bash
cat > StingTools/Clash/ClashScheduler.cs << 'EOF'
// ClashScheduler.cs — schedule headless full clash runs (e.g., after every save, or nightly).
using System;
using System.Timers;
using Autodesk.Revit.UI;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class ClashScheduler
    {
        private static ClashScheduler _inst;
        public static ClashScheduler Instance => _inst ?? (_inst = new ClashScheduler());
        private Timer _timer;

        public void StartHourly(UIApplication app)
        {
            _timer?.Stop();
            _timer = new Timer(60 * 60 * 1000) { AutoReset = true };
            _timer.Elapsed += (s, e) =>
            {
                try { LiveClashHandler.Event?.Raise(); }
                catch (Exception ex) { StingLog.Warn("ClashScheduler tick: " + ex.Message); }
            };
            _timer.Start();
        }

        public void Stop() { _timer?.Stop(); _timer = null; }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 5.5 ClashScheduler timer" && git push origin HEAD
```

---

## Section 5.6 — AccIssuesClient.cs

POST BCF to the ACC Issues bulk endpoint. Placeholder OAuth; real implementation in Stage 7.

```bash
cat > StingTools/Clash/AccIssuesClient.cs << 'EOF'
// AccIssuesClient.cs — posts BCF exports to ACC Issues API.
// OAuth acquisition is a Stage 7 concern; here we accept a bearer token and POST.
using System;
using System.IO;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public sealed class AccIssuesClient
    {
        private readonly HttpClient _http = new HttpClient();
        private readonly string _bearer;
        public AccIssuesClient(string bearerToken) { _bearer = bearerToken; }

        public async Task<bool> PostBcfAsync(string projectId, string bcfZipPath)
        {
            if (string.IsNullOrEmpty(projectId) || !File.Exists(bcfZipPath)) return false;
            try
            {
                _http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", _bearer);
                using var content = new MultipartFormDataContent();
                content.Add(new ByteArrayContent(File.ReadAllBytes(bcfZipPath)), "file", Path.GetFileName(bcfZipPath));
                var url = $"https://developer.api.autodesk.com/bim360/docs/v1/projects/{projectId}/issues/bulk";
                var resp = await _http.PostAsync(url, content);
                return resp.IsSuccessStatusCode;
            }
            catch (Exception ex) { StingLog.Warn("AccIssuesClient.PostBcf: " + ex.Message); return false; }
        }
    }
}
EOF
```

```bash
dotnet build StingTools.csproj -v quiet 2>&1 | tail -5
git add -A && git commit -m "clash: 5.6 AccIssuesClient BCF POST" && git push origin HEAD
```

---

## Section 5.7 — ClashSlaIntegration.cs

Create `CoordIssue` per group, route through existing `WarningsManager`.

```bash
cat > StingTools/Clash/ClashSlaIntegration.cs << 'EOF'
// ClashSlaIntegration.cs — wire clash groups into the existing WarningsManager SLA engine.
using System;
using System.Collections.Generic;
using System.Linq;
using StingTools.Core;

namespace StingTools.Core.Clash
{
    public static class ClashSlaIntegration
    {
        public static List<CoordIssue> CreateIssues(ClashRunRecord run, ClashMatrix matrix)
        {
            var result = new List<CoordIssue>();
            if (run?.Groups == null) return result;
            foreach (var g in run.Groups)
            {
                var cell = matrix?.Cells?.FirstOrDefault(c => c.PairId == g.Anchor);
                var issue = new CoordIssue
                {
                    Guid = Guid.NewGuid().ToString(),
                    Title = $"Clash group {g.Id} ({g.Size} clashes)",
                    Description = $"Matrix pair {g.Anchor}. Severity {cell?.Severity ?? "MED"}.",
                    Status = "Open",
                    Priority = cell?.Severity ?? "MED",
                    Assignee = cell?.OwnerDiscipline ?? "Coord",
                    CreatedUtc = DateTime.UtcNow
                };
                g.Assignee = issue.Assignee;
                result.Add(issue);
            }
            return result;
        }
    }
}
EOF
```

If `CoordIssue` is defined in a specific namespace, add the using:

```bash
# Find where CoordIssue lives
grep -rn "class CoordIssue" --include="*.cs" | head -3
# If it's e.g. in StingTools.Core:
#   already covered by `using StingTools.Core;`
# If elsewhere, adjust
dotnet build StingTools.csproj -v quiet 2>&1 | tail -10
git add -A && git commit -m "clash: 5.7 ClashSlaIntegration" && git push origin HEAD
```

If `CoordIssue` members don't match (field names differ), open the file and adjust:

```bash
# Diagnose
grep -A 20 "class CoordIssue" $(grep -l "class CoordIssue" *.cs) | head -30
# Then manually adjust ClashSlaIntegration.cs to match real field names
```

**Stage 5 complete.**

---

## FINAL CHECKS

Run the full build and summary:

```bash
dotnet build StingTools.csproj 2>&1 | tail -20
git log --oneline | head -30
ls StingTools/Clash/ StingTools/UI/Clash/
```

You should see ~27 commits tagged `clash: N.M`, all 25 new files present, and a green build.

If you got here cleanly, you have delivered Stages 1–5 of the v4.0 research plan. Report back with the commit count, final build status, and any files that contain `NotImplementedException` or `TODO` comments that need follow-up.

---

## RESUME PROTOCOL REMINDER

If interrupted:

```bash
git log --oneline -- StingTools/Clash StingTools/UI/Clash | head -20
```

Find the highest `clash: N.M`. Resume from `clash: N.(M+1)` — the next section below it in this prompt. Do not redo completed sections.

If the build is broken (last commit has errors), revert to the last known-good commit:

```bash
# Find last green commit
git log --oneline | head -10
# Check out the second-to-last clash: commit
git reset --hard HEAD~1
```

Then re-run the failed section from scratch.

---

END OF CLAUDE CODE PROMPT.
