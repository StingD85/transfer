# StingTools Crash Analysis & Fix Proposal

**Date:** 2026-03-07  
**Scope:** Full codebase audit — 62 source files, 257 IExternalCommand classes, ~58,440 LOC  
**Purpose:** Actionable proposal for Claude Code to systematically eliminate all crash vectors

---

## Executive Summary

The StingTools plugin has **6 critical crash categories** and **4 moderate risk areas**. The single biggest issue is that **252 of 257 commands lack `IPanelCommand` implementation**, meaning they receive `null` for `ExternalCommandData` when invoked from the dockable panel. While `ParameterHelpers.GetApp()` provides a fallback, **189 commands chain `.ActiveUIDocument.Document` without null-checks**, creating a NullReferenceException cascade when no document is open, the fallback fails, or the UIApplication reference is stale.

---

## CRITICAL ISSUES (Crash-Causing)

### C1. Missing IPanelCommand on 252 of 257 Commands

**Root cause:** Only 5 commands implement `IPanelCommand`: `CheckDataCommand`, `CreateParametersCommand`, `JournalParserCommand`, `MasterSetupCommand`, `ProjectSetupCommand`. The remaining 252 commands are dispatched via `RunCommand<T>()` which passes `null` for `ExternalCommandData`.

**How it crashes:** When `RunCommand<T>` encounters a command without `IPanelCommand`, it calls `cmd.Execute(null, ref message, elements)`. Inside those commands, `ParameterHelpers.GetApp(commandData)` falls back to `StingCommandHandler.CurrentApp`. If `CurrentApp` is null (rare but possible during race conditions) or if the command accesses `commandData` directly before calling `GetApp()`, Revit crashes.

**Evidence:** The comment in `StingCommandHandler.cs` states: *"CreateCommandData REMOVED — was the root cause of Revit crashes. ExternalCommandData is a native COM wrapper that cannot be fabricated from user code."*

**Files affected:** ALL command files — `CategorySelectCommands.cs` (15 commands), `TokenWriterCommands.cs` (7), `TagOperationCommands.cs` (39), `ColorCommands.cs` (5), `ScheduleCommands.cs` (6), `TemplateManagerCommands.cs` (17), `TemplateExtCommands.cs` (6), `FamilyCommands.cs` (6), `DocAutomationCommands.cs`, `DocAutomationExtCommands.cs` (11), `DataPipelineCommands.cs` (11), `SmartTagPlacementCommand.cs` (9), `ScheduleEnhancementCommands.cs` (9), `StateSelectCommands.cs` (8), `ViewAutomationCommands.cs` (6), `RichTagDisplayCommands.cs` (6), `ViewportCommands.cs` (4), `LegendBuilderCommands.cs` (31), `HandoverExportCommands.cs` (5), `BatchTagCommand.cs`, `AutoTagCommand.cs`, `CombineParametersCommand.cs`, `FormulaEvaluatorCommand.cs`, `SyncParameterSchemaCommand.cs`, and more.

**Fix:**
```
For each command file:
1. Add `: IPanelCommand` to class declaration (alongside IExternalCommand)
2. Add `public Result Execute(UIApplication app)` method that calls the core logic
3. Ensure the Execute(ExternalCommandData...) method uses ParameterHelpers.GetApp()
```

### C2. Unsafe Null Dereference Chains (189 occurrences)

**Root cause:** 189 commands use the pattern:
```csharp
Document doc = ParameterHelpers.GetApp(commandData).ActiveUIDocument.Document;
```
This is a triple-dereference chain with no null checking. If `ActiveUIDocument` is null (no document open, family editor context, or Revit startup race), this throws `NullReferenceException`.

**Worst offenders:**
- `DataPipelineCommands.cs` — 5 unsafe chains
- `DocAutomationCommands.cs` — 3 unsafe chains  
- `DocAutomationExtCommands.cs` — 3+ unsafe chains
- `ColorCommands.cs` — 5 unsafe chains
- `LegendBuilderCommands.cs` — multiple unsafe chains
- `CategorySelectCommands.cs` — 15 commands, ALL with unsafe `uidoc.Document` then `doc.ActiveView.Id` chains

**Note:** The guard in `StingCommandHandler.Execute()` at line ~68 checks `app.ActiveUIDocument == null` and shows a dialog — but this only protects the dispatch level. If a command is called directly (from ribbon, from another command, from `RunCommand` inside `ProjectSetupCommand`/`MasterSetupCommand`), the guard is bypassed.

**Fix:**
```
Replace all occurrences of:
  Document doc = ParameterHelpers.GetApp(commandData).ActiveUIDocument.Document;
With:
  UIApplication uiApp = ParameterHelpers.GetApp(commandData);
  UIDocument uidoc = uiApp?.ActiveUIDocument;
  if (uidoc == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
  Document doc = uidoc.Document;
```

### C3. Unsafe `doc.ActiveView` Access Without Null Checks

**Root cause:** Many commands access `doc.ActiveView.Id` without checking if `ActiveView` is null. In Revit, `ActiveView` can be null in family editor, during document switching, or when a schedule/legend is the active view.

**Specific crash pattern in `CategorySelectCommands.cs`:**
```csharp
var ids = new FilteredElementCollector(doc, doc.ActiveView.Id)  // CRASH if ActiveView null
```
This affects all 15 category select commands, plus `ColorCommands.cs`, `FamilyStagePopulateCommand.cs`, `LegendBuilderCommands.cs`, and more.

**Fix:**
```
Add ActiveView null check after Document retrieval:
  View activeView = doc.ActiveView;
  if (activeView == null) { TaskDialog.Show("STING", "No active view."); return Result.Failed; }
```

### C4. Transaction Handling in ProjectSetupCommand

**Root cause:** `ProjectSetupCommand` runs standalone transactions BEFORE a `TransactionGroup`, and mixes unit changes (which trigger massive Revit regeneration) with group operations. The code comments explicitly warn: *"Units change triggers massive Revit-internal regeneration... Running this INSIDE a TransactionGroup causes Revit to crash during deferred DOPT."*

**Risk:** If any step within the `TransactionGroup` fails with an unhandled exception, the group may not be properly assimilated or rolled back, leaving Revit in a corrupted transaction state.

**Additional risk in `RunStep`:** The signature `Func<r>` appears to be a typo/issue — it should be `Func<Result>`. If this is a real typo in the source, it would cause a compilation error. If it compiled, the delegate invocation semantics need verification.

**Fix:**
```
1. Wrap TransactionGroup in try-finally with tg.RollBack() in the finally
2. Verify Func<r> vs Func<Result> — fix if typo
3. Add explicit rollback handling for each sub-step failure
```

### C5. Assembly Version Conflicts (ClosedXML Dependency Chain)

**Root cause:** ClosedXML 0.104.2 pulls in `DocumentFormat.OpenXml → System.IO.Packaging → WindowsBase`. The csproj has a `RemoveConflictingAssemblies` target and `OnAssemblyResolve` whitelist, but the whitelist approach only covers 8 specific assemblies. Any new transitive dependency added by a ClosedXML update could slip through.

**Journal evidence:** *"WindowsBase 4.0.0.0 conflicts with preloaded 8.0.0.0"* (documented in csproj comments).

**Risk:** If a ClosedXML path is triggered before the assembly resolver fires (e.g., during static initialization), the load could fail and crash Revit.

**Fix:**
```
1. Add try-catch around first ClosedXML usage in each command (lazy initialization)
2. Consider isolating ClosedXML into a separate AppDomain or AssemblyLoadContext
3. Pin all transitive dependency versions in csproj
4. Add runtime verification: on startup, try loading ClosedXML and log success/failure
```

### C6. IUpdater Exception Propagation (StingAutoTagger)

**Root cause:** Exceptions thrown inside `IUpdater.Execute()` crash Revit immediately — there is no recovery. While `StingAutoTagger` has defensive guards (element count limit, consecutive failure auto-disable), the `TokenAutoPopulator.PopulateAll()` and `TagConfig.BuildAndWriteTag()` calls inside the loop could throw if: the element is deleted between `GetElement` and `PopulateAll`, parameters are not yet bound, or the shared parameter file changes mid-operation.

**Risk:** The auto-tagger processes elements during Revit's internal update cycle. Any exception = instant crash.

**Fix:**
```
1. Wrap EACH element processing in individual try-catch (already partially done)
2. Add IsValidObject check AFTER PopulateAll and before BuildAndWriteTag
3. Add parameter-exists check before writing
4. Consider using PostableCommand or enqueuing work to IExternalEventHandler instead
```

---

## MODERATE ISSUES (Error/Instability Causing)

### M1. Zero Exception Handling in CategorySelectCommands (15 commands)

**Stats:** 15 `Execute` methods, 0 `try-catch` blocks anywhere in the file. The shared `CategorySelector.SelectByCategory()` helper has no exception handling.

**Impact:** Any Revit API error (e.g., view doesn't support category filtering, corrupted element) propagates unhandled through `RunCommand<T>`, which catches it but shows a generic error. The user sees a confusing error message with no context.

**Fix:**
```
Add try-catch to CategorySelector.SelectByCategory() wrapping the 
FilteredElementCollector and SetElementIds calls.
```

### M2. Empty Catch Blocks (240 occurrences)

**Stats:** 240 `catch { }` or `catch { /* comment */ }` blocks across the codebase that silently swallow exceptions.

**Impact:** Bugs are hidden. When something goes wrong, there's no log entry, no user feedback, and no way to diagnose the issue. This is especially problematic in `DataPipelineCommands.cs` (6 empty catches), `DocAutomationExtCommands.cs` (2), `LegendBuilderCommands.cs` (6+), and `FormulaEvaluatorCommand.cs` (2).

**Fix:**
```
Replace catch { } with catch (Exception ex) { StingLog.Warn($"Context: {ex.Message}"); }
Priority: focus on catch blocks inside transactions first (data corruption risk),
then command-level catches, then utility catches.
```

### M3. Memory Pressure from Large ToList() Materializations

**Stats:** 354 `.ToList()` calls, 6 full-model `WhereElementIsNotElementType().ToList()` calls.

**Risk:** On large models (50K+ elements), materializing the entire element list into memory causes GC pressure and potential `OutOfMemoryException`. The `ResolveAllIssuesCommand` and `AnomalyAutoFixCommand` both collect ALL taggable elements into a single list.

**Fix:**
```
1. Use lazy enumeration (IEnumerable) instead of ToList() where possible
2. For large batches, process in chunks of 1000 elements
3. Add element count warning before processing >10K elements
```

### M4. Stale ParameterCache After Parameter Changes

**Root cause:** `ParameterHelpers._paramCache` is a `ConcurrentDictionary` that caches parameter lookups by `(TypeId, paramName)`. After `LoadSharedParamsCommand` binds new parameters, the cache may contain stale "miss" entries (null Definition). `ClearParamCache()` exists but may not be called after every parameter-modifying operation.

**Impact:** Commands that run after parameter binding may fail to find newly-bound parameters, causing incomplete tagging.

**Fix:**
```
1. Call ParameterHelpers.ClearParamCache() at the end of LoadSharedParamsCommand
2. Call ClearParamCache() after MasterSetupCommand and ProjectSetupCommand
3. Add document-change event hook to auto-clear cache
```

---

## LOW RISK BUT WORTH FIXING

### L1. Dockable Panel Floating State Crash

**Evidence from `StingDockPanelProvider.cs`:** *"DockPosition.Right caused 'Only floating document is support!' on every Show(), and floating panes crash Revit on WPF tab switches."* Currently mitigated by using `DockPosition.Tabbed`, but if Revit's UIState.dat caches a floating state, the crash can recur.

**Fix:** Add error handling in the panel Show/Hide toggle.

### L2. TaskDialog Text Length Overflow

**Evidence from `CheckDataCommand.cs`:** *"Revit TaskDialog.MainContent can crash if text exceeds ~2000 chars."* Several commands build long reports that could exceed this limit. `CheckDataCommand` and `MasterSetupCommand` have truncation logic, but other commands (e.g., `AnomalyAutoFixCommand`, `ResolveAllIssuesCommand`) may not.

**Fix:** Create a shared `SafeTaskDialog.Show()` helper that auto-truncates.

### L3. FilteredElementCollector Not Disposed

**Stats:** 382 `new FilteredElementCollector` calls, none wrapped in `using`. While `FilteredElementCollector` implements `IDisposable` in some Revit versions, not disposing it can leak native resources.

**Fix:** Wrap critical collectors in `using` blocks, especially in loops.

---

## PROPOSED FIX EXECUTION PLAN (For Claude Code)

### Phase 1: Critical Null Safety (Highest Impact, ~2 hours)

**Task 1.1:** Add `IPanelCommand` to the top 20 most-used commands (those dispatched by `RunCommand<T>` in `StingCommandHandler.cs`). Priority order:
- `CategorySelectCommands.cs` (15 commands — single helper change covers all)
- `BatchTagCommand.cs`, `AutoTagCommand.cs` 
- `ColorCommands.cs` (5 commands)
- `CombineParametersCommand.cs`

**Task 1.2:** Fix unsafe null chains. Create a shared entry-point helper:
```csharp
// Add to ParameterHelpers.cs
public static (UIApplication app, UIDocument uidoc, Document doc) GetContext(
    ExternalCommandData commandData, string commandName = null)
{
    var app = GetApp(commandData);
    var uidoc = app?.ActiveUIDocument;
    if (uidoc == null)
        throw new InvalidOperationException("No document open.");
    return (app, uidoc, uidoc.Document);
}
```
Then replace all `ParameterHelpers.GetApp(commandData).ActiveUIDocument.Document` chains.

**Task 1.3:** Add `ActiveView` null guards to all commands using `doc.ActiveView.Id` in `FilteredElementCollector`.

### Phase 2: Transaction Safety (~1 hour)

**Task 2.1:** Audit `ProjectSetupCommand` TransactionGroup — ensure rollback on failure.

**Task 2.2:** Verify `Func<r>` return type in `RunStep` helper (both `ProjectSetupCommand.cs` and `MasterSetupCommand.cs`).

**Task 2.3:** Add try-catch around each step in `MasterSetupCommand` and `ProjectSetupCommand` TransactionGroups.

### Phase 3: Exception Handling Uplift (~2 hours)

**Task 3.1:** Add try-catch to `CategorySelector.SelectByCategory()`.

**Task 3.2:** Replace top-priority empty catch blocks (those inside transactions) with logged catches. Target files: `DataPipelineCommands.cs`, `DocAutomationExtCommands.cs`, `LegendBuilderCommands.cs`, `FormulaEvaluatorCommand.cs`.

**Task 3.3:** Create `SafeTaskDialog` helper with auto-truncation. Replace manual truncation in `CheckDataCommand`, `MasterSetupCommand`, and add to all commands with long report strings.

### Phase 4: Remaining IPanelCommand Coverage (~3 hours)

**Task 4.1:** Add `IPanelCommand` to remaining 230+ commands. This can be done systematically per file:
- For simple delegator commands (like `CategorySelectCommands.cs`), add IPanelCommand to the helper class
- For complex commands, add the `Execute(UIApplication app)` overload

### Phase 5: Assembly & Memory (~1 hour)

**Task 5.1:** Add startup assembly verification — try-load ClosedXML on startup, log result.

**Task 5.2:** Add `ParameterHelpers.ClearParamCache()` calls after parameter-modifying operations.

**Task 5.3:** Add element count warnings before large batch operations.

---

## Quick Reference: Files Needing Changes

| File | Issues | Priority |
|------|--------|----------|
| `CategorySelectCommands.cs` | C1, C2, C3, M1 | **P0** |
| `ParameterHelpers.cs` | C2 (add GetContext helper) | **P0** |
| `StingCommandHandler.cs` | C1 (RunCommand fallback) | **P0** |
| `ColorCommands.cs` | C1, C2, C3 | **P0** |
| `TagOperationCommands.cs` | C1, C2 | **P1** |
| `DataPipelineCommands.cs` | C1, C2, M2 | **P1** |
| `LegendBuilderCommands.cs` | C1, C2, M2 | **P1** |
| `DocAutomationCommands.cs` | C1, C2 | **P1** |
| `DocAutomationExtCommands.cs` | C1, C2, M2 | **P1** |
| `TemplateManagerCommands.cs` | C1, C2 | **P1** |
| `BatchTagCommand.cs` | C1, C2 | **P1** |
| `AutoTagCommand.cs` | C1, C2, C3 | **P1** |
| `ProjectSetupCommand.cs` | C4 | **P1** |
| `MasterSetupCommand.cs` | C4 | **P1** |
| `StingAutoTagger.cs` | C6 | **P1** |
| `StingToolsApp.cs` | C5 | **P2** |
| `ScheduleCommands.cs` | C1, C2 | **P2** |
| `ScheduleEnhancementCommands.cs` | C1, C2 | **P2** |
| `FamilyCommands.cs` | C1, C2 | **P2** |
| `SmartTagPlacementCommand.cs` | C1, C2 | **P2** |
| `StateSelectCommands.cs` | C1, C2 | **P2** |
| `ViewAutomationCommands.cs` | C1, C2 | **P2** |
| `ViewportCommands.cs` | C1, C2 | **P2** |
| `TemplateExtCommands.cs` | C1, C2 | **P2** |
| `HandoverExportCommands.cs` | C1, C2 | **P2** |
| `RichTagDisplayCommands.cs` | C1, C2 | **P2** |
| `TokenWriterCommands.cs` | C1, C2 | **P2** |
| All remaining command files | C1 | **P3** |
