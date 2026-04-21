# STING Tools — BOQ & Cost Manager: Full Claude Code Implementation Prompt
# Generated: 2026-04-21 | Project: MBALWA ASBUILT 13-11-25_stingd85

## HOW TO USE THIS PROMPT IN CLAUDE CODE
# 1. Open Claude Code in the StingTools solution root
# 2. Paste this entire document as your initial prompt
# 3. Claude Code will read all referenced files before writing any code
# 4. Implementation is broken into phases — Claude Code should complete
#    each phase, compile-check, then proceed to the next
# 5. Never skip the "Read first" instruction at the top of each phase

---

## MANDATORY PRE-READ (before ANY code is written)

Claude Code MUST read every file listed below before writing a single line.
Use the Read tool on each. Do not rely on training memory for their content.

FILES TO READ:
  src/Commands/Scheduling/SchedulingCommands.cs
  src/Commands/Scheduling/SchedulingCostDashboard.cs
  src/Commands/Carbon/CarbonTrackingCommands.cs
  src/Temp/BOQTemplateLibrary.cs
  src/UI/BIMCoordinationCenter.cs
  src/UI/StingCommandHandler.cs
  src/UI/StingDockPanel.xaml.cs
  src/UI/StingResultPanel.cs
  src/Commands/Excel/ExcelLinkCommands.cs
  src/Commands/Revision/RevisionManagementCommands.cs
  src/Core/ParameterHelpers.cs
  src/Core/TagConfig.cs
  src/Core/SharedParamGuids.cs
  src/Core/ParamRegistry.cs
  data/MR_PARAMETERS.txt
  data/BOQ_DESCRIPTIONS.json
  data/cost_rates_5d.csv
  data/COBIE_TYPE_MAP.csv
  data/MATERIAL_LOOKUP.csv
  data/MATERIAL_SCHEMA.json
  data/CATEGORY_BINDINGS.csv

CODING STANDARDS — read and follow exactly:
- PascalCase methods, _camelCase private fields matching existing codebase style
- StingLog.Info/Warn/Error for all logging — never Console.WriteLine
- All Revit API writes inside [Transaction(TransactionMode.Manual)] commands
- All reads in [Transaction(TransactionMode.ReadOnly)]
- No TaskDialog for multi-line output — always use StingResultPanel
- No new XAML files — all WPF built in C# code-behind only
- ThemeManager colour constants throughout — no hardcoded hex values in WPF
- All file I/O in try/catch; wrap with StingLog.Warn on failure
- CultureInfo.InvariantCulture for all file/CSV numeric parsing
- ExternalEvent dispatch via ActionDispatcher pattern (see BIMCoordinationCenter.cs)
- Use ClosedXML for all Excel output — never OpenXML directly
- CollectionViewSource + ObservableCollection for all WPF DataGrid bindings
- StingCommandHandler.SetExtraParam/GetExtraParam for UI→command parameter passing

ARCHITECTURE RULE:
  New commands go in src/Commands/BOQ/ (create this folder).
  New WPF panels go in src/UI/.
  Engine logic goes in src/Commands/BOQ/BOQCostManager.cs.
  Never modify SchedulingCommands.cs — extend via BOQCostManager.

---

## PHASE 1 — New shared parameters (MR_PARAMETERS.txt + LoadSharedParamsCommand.cs)

### 1A. Add to MR_PARAMETERS.txt

Append the following PARAM lines to MR_PARAMETERS.txt.
Generate real GUIDs (Guid.NewGuid() pattern — unique, not placeholder text).
Group 3 = cost parameters. Group 12 = material parameters. Group 1 = asset parameters.
Group 13 = project-level (ProjectInfo binding).

COST PARAMETERS — group 3, bind to AllCategoryEnums (coreCats):
  CST_UNIT_RATE_UGX        TEXT   "Unit rate UGX applied to this element [ISO 19650-3:2020]"
  CST_UNIT_RATE_USD        TEXT   "Unit rate USD applied to this element [ISO 19650-3:2020]"
  CST_QTY_MEASURED         TEXT   "Measured quantity (area/length/volume/count) [ISO 19650-3:2020]"
  CST_MODELED_TOTAL_UGX    NUMBER "Computed element cost total in UGX [ISO 19650-3:2020]"
  CST_RATE_SOURCE          TEXT   "Rate source: CSV | COBie | Manual | Override [ISO 19650-3:2020]"
  CST_BOQ_SNAPSHOT_REF     TEXT   "Snapshot label when this element was last costed [ISO 19650-3:2020]"
  CST_PROVISIONAL_SUM      YESNO  "True if this element represents a provisional sum [ISO 19650-3:2020]"
  CST_PS_DESCRIPTION       TEXT   "Provisional sum scope description [ISO 19650-3:2020]"
  CST_EMBODIED_CARBON_KG   NUMBER "Embodied carbon in kgCO2e for this element [ISO 19650-3:2020]"
  CST_LIFECYCLE_COST_UGX   NUMBER "Whole-life cost estimate in UGX (capital + maintenance) [ISO 19650-3:2020]"

ASSET PARAMETERS — group 1, bind to AllCategoryEnums (coreCats):
  ASS_NRM2_PARA_TXT        TEXT   "Resolved NRM2 BOQ paragraph, all tokens substituted [ISO 19650-3:2020]"
  ASS_BOQ_LINE_REF         TEXT   "BOQ line reference e.g. 14.3.2 — Excel roundtrip join key [ISO 19650-3:2020]"
  ASS_BOQ_SECTION_NAME     TEXT   "BOQ section name this element belongs to [ISO 19650-3:2020]"

MATERIAL PARAMETERS — group 12, bind to BLE element categories (walls/floors/ceilings/roofs):
  MAT_UNIT_RATE_UGX        TEXT   "Material unit rate in UGX per measured unit [ISO 19650-1:2018]"
  MAT_UNIT_RATE_USD        TEXT   "Material unit rate in USD per measured unit [ISO 19650-1:2018]"
  MAT_MEASURED_AREA_M2     NUMBER "Authoritative measured area m² — single source for BOQ + COBie [ISO 19650-1:2018]"
  MAT_CARBON_FACTOR        NUMBER "Embodied carbon factor kgCO2e per kg for this material [ISO 19650-1:2018]"

PROJECT PARAMETERS — bind to ProjectInfo category only (not AllCategoryEnums):
  PROJECT_BUDGET_UGX       NUMBER "Project budget set by cost consultant in UGX [ISO 19650-3:2020]"
  CST_BUDGET_VARIANCE_UGX  NUMBER "Budget variance: budget minus grand total UGX [ISO 19650-3:2020]"
  CST_BOQ_COVERAGE_PCT     NUMBER "Percentage of elements with resolved BOQ line items [ISO 19650-3:2020]"
  CST_LAST_COSTED_DATE     TEXT   "Date of last BOQ cost calculation [ISO 19650-3:2020]"

### 1B. Register in ParamRegistry / SharedParamGuids

For each new parameter, add an entry to ParamRegistry.AllParamGuids using the GUID
assigned in MR_PARAMETERS.txt. Follow the exact format of existing entries.

For the PROJECT_BUDGET_UGX and CST_* project-level parameters, bind to a new
CategorySet containing only the ProjectInfo category. Create a helper method
SharedParamGuids.BuildProjectInfoCategorySet(Document doc) following the same
pattern as BuildCategorySet.

### 1C. LoadSharedParamsCommand.cs modifications

In BuildGroupCategoryOverrides() add a case for the new "CST_PROJECT" group name
that returns the ProjectInfo-only CategorySet.

Also add the material parameters (MAT_UNIT_RATE_UGX etc.) to the same category
override as existing BLE_MAT_* parameters (walls, floors, ceilings, roofs, structural).

After implementing, verify: run Load Shared Params → check zero GUID conflicts.
Do NOT use placeholder GUIDs — generate real ones.

---

## PHASE 2 — Data model: BOQLineItem, BOQSection, BOQDocument

Create src/Commands/BOQ/BOQModels.cs

```csharp
namespace StingTools.BOQ
{
    public enum BOQRowSource { Model, Manual, ProvisionalSum }
    public enum BOQChangeType { NoChange, RateRevised, QtyChanged, NewItem, ItemRemoved, PSAdded, SourcePromoted }

    public class BOQLineItem
    {
        public string Id = Guid.NewGuid().ToString("N");
        public string NRM2Section;          // "14"
        public string Category;             // Revit category name
        public string Discipline;           // A/S/M/E/P/FP/PS
        public string ItemName;
        public string FamilyName;
        public string TypeName;
        public double Quantity;
        public string Unit;                 // m², m³, m, each, item, kg, tonne
        public double RateUGX;
        public double RateUSD;
        public double TotalUGX => Math.Round(Quantity * RateUGX, 0);
        public double TotalUSD => Math.Round(Quantity * RateUSD, 2);
        public double EmbodiedCarbonKg;     // kgCO2e — populated if MATERIAL_LOOKUP has data
        public double LifecycleCostUGX;     // capital + maintenance NPV 25yr
        public string ResolvedNRM2Paragraph; // complete sentence, zero tokens
        public string BOQLineRef;           // "14.3.2"
        public string Note;
        public BOQRowSource Source;
        public string SnapshotRef;
        public long RevitElementId = -1;    // -1 for manual/PS rows
        public string UniqueId;             // Revit UniqueId for cross-doc reference
        public string Level;
        public string Location;             // room name or spatial code
        public DateTime LastCosted = DateTime.UtcNow;
        public string RateSource;           // "CSV" | "COBie" | "Manual" | "Override" | "Carbon"
        public int SortOrder;               // for stable ordering within section
    }

    public class BOQSection
    {
        public string Id = Guid.NewGuid().ToString("N");
        public string NRM2Section;
        public string Name;
        public string Discipline;
        public BOQRowSource DefaultSource = BOQRowSource.Model;
        public List<BOQLineItem> Items = new List<BOQLineItem>();
        public double TotalUGX => Items.Sum(i => i.TotalUGX);
        public double TotalUSD => Items.Sum(i => i.TotalUSD);
        public double ModeledUGX => Items.Where(i => i.Source == BOQRowSource.Model).Sum(i => i.TotalUGX);
        public double ProvUGX => Items.Where(i => i.Source != BOQRowSource.Model).Sum(i => i.TotalUGX);
        public double TotalCarbonKg => Items.Sum(i => i.EmbodiedCarbonKg);
    }

    public class BOQDocument
    {
        public string ProjectName;
        public string DocumentTitle;
        public string SnapshotLabel;
        public DateTime SnapshotDate = DateTime.UtcNow;
        public string SnapshotType;         // "DD" | "Stage" | "Weekly" | "Handover" | "Manual"
        public double ProjectBudgetUGX;
        public double PrelimPct = 12.0;
        public double ContingencyPct = 10.0;
        public double OverheadPct = 8.0;
        public string Currency = "UGX";     // display currency
        public List<BOQSection> Sections = new List<BOQSection>();
        public List<BOQLineItem> AllItems => Sections.SelectMany(s => s.Items).ToList();
        public double ModeledTotalUGX => AllItems.Where(i => i.Source == BOQRowSource.Model).Sum(i => i.TotalUGX);
        public double ProvTotalUGX    => AllItems.Where(i => i.Source != BOQRowSource.Model).Sum(i => i.TotalUGX);
        public double SubtotalUGX     => AllItems.Sum(i => i.TotalUGX);
        public double GrandTotalUGX   => Math.Round(SubtotalUGX * (1 + PrelimPct/100 + ContingencyPct/100 + OverheadPct/100), 0);
        public double BudgetVarianceUGX => ProjectBudgetUGX > 0 ? ProjectBudgetUGX - GrandTotalUGX : 0;
        public double BudgetCoveragePct => ProjectBudgetUGX > 0 ? SubtotalUGX / ProjectBudgetUGX * 100 : 0;
        public double TotalCarbonKg => AllItems.Sum(i => i.EmbodiedCarbonKg);
        public int ResolvedParagraphCount => AllItems.Count(i => !string.IsNullOrEmpty(i.ResolvedNRM2Paragraph));
        public double ParagraphCoveragePct => AllItems.Count > 0 ? 100.0 * ResolvedParagraphCount / AllItems.Count : 0;
    }

    public class BOQSnapshotDiff
    {
        public string LabelA, LabelB, TypeA, TypeB;
        public DateTime DateA, DateB;
        public double TotalA, TotalB;
        public double NetChange => TotalB - TotalA;
        public double NetChangePct => TotalA > 0 ? (TotalB - TotalA) / TotalA * 100 : 0;
        public List<SectionDiff> SectionDiffs = new List<SectionDiff>();
        public List<CategoryDiff> CategoryDiffs = new List<CategoryDiff>();
        public double ModeledA, ModeledB, ProvA, ProvB;
        public string PlainSummary;         // auto-generated narrative paragraph
    }

    public class SectionDiff
    {
        public string NRM2Section, Name, Discipline;
        public double TotalA, TotalB;
        public double Delta => TotalB - TotalA;
        public double DeltaPct => TotalA > 0 ? Delta / TotalA * 100 : 0;
    }

    public class CategoryDiff : SectionDiff
    {
        public double QtyA, QtyB, RateA, RateB;
        public BOQChangeType ChangeType;
        public string ChangeReason; // human-readable sentence
    }

    public class BOQManualStore
    {
        public string SchemaVersion = "1.1";
        public double ProjectBudgetUGX;
        public DateTime LastSaved = DateTime.UtcNow;
        public string LastSavedBy;
        public List<BOQLineItem> ManualRows = new List<BOQLineItem>();
    }
}
```

---

## PHASE 3 — BOQCostManager engine (src/Commands/BOQ/BOQCostManager.cs)

This is the central engine. Read SchedulingCommands.cs completely first —
BuildBOQDocument must reuse and extend (not duplicate) its DeriveQuantity and
LoadCostRates logic by calling those internal methods via a static helper or by
copying the logic with a comment marking the source.

### 3A. BuildBOQDocument(Document doc, IEnumerable<BOQLineItem> extraManualRows = null)

Returns BOQDocument. Steps in strict order:

STEP 1 — Load config
  - Load PROJECT_BUDGET_UGX, COST_PRELIMINARIES_PCT, COST_CONTINGENCY_PCT,
    COST_OVERHEAD_PROFIT_PCT from project_config.json via TagConfig.GetConfigDouble.
  - Load exchange rate UGX_PER_USD from project_config.json (default 3700).

STEP 2 — Load rates (3-source merge, highest priority first)
  a) project-level cost_rates_5d.csv (per-element override if present)
  b) COBIE_TYPE_MAP.csv: read CostRateCode column → look up in cost_rates_5d
  c) Default rates from SchedulingCommands.DefaultCostRates dictionary

STEP 3 — Load embodied carbon factors
  - Load MATERIAL_LOOKUP.csv using CarbonTrackingEngine.EnsureLoaded() pattern.
  - Map category → carbon factor for use in lifecycle calculation.

STEP 4 — Collect and filter elements
  - FilteredElementCollector over AllCategoryEnums (WhereElementIsNotElementType).
  - Skip: demolished phase, temporary phase, in-place families flagged in config.
  - Skip Rooms and Spaces (no direct cost, add as informational only if config flag set).

STEP 5 — Per-element costing
  For each element:
  a) Derive quantity (call DeriveQuantity with unit from rate lookup).
  b) Look up rate: CSV by category name → CSV by PROD code → COBie type map → default.
  c) Compute TotalUGX. TotalUSD = TotalUGX / exchangeRate (round 2dp).
  d) Resolve NRM2 paragraph:
       - Read ASS_NRM2_PARA_TXT first. If non-empty and has no [token] patterns → use it.
       - Else: call BOQTemplateLibrary.SelectBestTemplate(category, element).
         If template found → call ResolveForElement(template, element, doc).
         Store result. Mark dirty for write-back.
       - If no template → use category name as fallback: "Supply and fix {category}."
  e) Assign BOQLineRef: "{nrm2section}.{sectionIndex}.{rowIndex}" (1-based).
  f) Compute embodied carbon: qty × carbon factor from step 3.
  g) Compute lifecycle cost: TotalUGX × (1 + maintenanceFactor × 25yr NPV at 3.5%).
     MaintenanceFactor loaded from COBIE_TYPE_MAP.csv MaintenanceFreqMonths column
     (annual maintenance ≈ replacementCost / expectedLife / 12 × maintenanceFreq).

STEP 6 — Merge manual rows
  - Load from project_boq_manual.json (call LoadManualRows).
  - Merge extraManualRows parameter.
  - Assign BOQLineRefs continuing from modeled rows within their NRM2 section.

STEP 7 — Group into sections
  Group items by (NRM2Section, Discipline). Sort sections by NRM2 section number
  (parse as integer for sort, preserve string for display).
  Within each section sort by: Source (Model first), then Category, then ItemName.

STEP 8 — Return complete BOQDocument.

### 3B. WriteElementParameters(Document doc, IEnumerable<BOQLineItem> items, Transaction tx)

For each item where RevitElementId != -1:
  Write: CST_UNIT_RATE_UGX, CST_UNIT_RATE_USD, CST_QTY_MEASURED,
         CST_MODELED_TOTAL_UGX, CST_RATE_SOURCE, ASS_NRM2_PARA_TXT,
         ASS_BOQ_LINE_REF, ASS_BOQ_SECTION_NAME, CST_EMBODIED_CARBON_KG,
         CST_LIFECYCLE_COST_UGX.
  Only write if value changed (dirty check — read current value first).
  Batch into the provided transaction (caller manages tx.Start/Commit).

### 3C. WriteProjectParameters(Document doc, BOQDocument boq, Transaction tx)

Write to ProjectInfo element:
  PROJECT_BUDGET_UGX, CST_BUDGET_VARIANCE_UGX, CST_BOQ_COVERAGE_PCT,
  CST_LAST_COSTED_DATE (DateTime.Now.ToString("yyyy-MM-dd HH:mm")).

### 3D. SaveSnapshot(Document doc, BOQDocument boq, string label, string snapshotType)

- Serialise boq to JSON (Newtonsoft) → file:
  {bimDir}/boq_snapshot_{snapshotType}_{SafeLabel}_{yyyyMMdd_HHmmss}.json
- Update CST_BOQ_SNAPSHOT_REF on all modeled elements in a new transaction.
- Write project parameters via WriteProjectParameters.
- Prune old snapshots: keep latest 20 total; delete oldest if > 20.
- Return the file path.

### 3E. LoadSnapshot(string path) → BOQDocument

Deserialise from JSON. Return null on error (log warning).

### 3F. ListSnapshots(Document doc) → List<BOQSnapshotMeta>

Scan bimDir for boq_snapshot_*.json. Parse filename for type + label + date.
Return list sorted descending by date.

BOQSnapshotMeta: { string Path, Label, Type; DateTime Date; double GrandTotalUGX; }

### 3G. CompareSnapshots(string pathA, string pathB) → BOQSnapshotDiff

- Load both documents via LoadSnapshot.
- Match items by BOQLineRef first; fall back to CategoryName + ItemName composite key.
- Compute per-section and per-category deltas.
- Set ChangeType:
    RATE_REVISED: same item, qty unchanged, rate changed > 1%
    QTY_CHANGED: same item, rate unchanged, qty changed > 0.1%
    NEW_ITEM: in B not in A (modeled source)
    ITEM_REMOVED: in A not in B
    PS_ADDED: in B not in A (PS source)
    SOURCE_PROMOTED: was Manual in A, is Model in B
    NO_CHANGE: delta < 0.1%
- Generate PlainSummary: auto-build a paragraph from the diffs:
    "The net movement between {LabelA} and {LabelB} is {sign}{delta:N0} UGX
    ({pct:F1}%). [Top 3 changes listed]. [Carbon change if any]. [PS changes if any]."

### 3H. LoadManualRows(Document doc) → List<BOQLineItem>

Read project_boq_manual.json → deserialise BOQManualStore → return ManualRows.
Return empty list if file not found.

### 3I. SaveManualRows(Document doc, List<BOQLineItem> rows)

Build BOQManualStore. Serialise to project_boq_manual.json.
Also update PROJECT_BUDGET_UGX in project_config.json if it changed.

### 3J. ReconcileProvisionals(Document doc, BOQDocument boq) → List<ReconcileMatch>

For each PS row:
  - Find modeled elements in the same Category where TotalUGX is within ±30% of PS TotalUGX.
  - If match found: add to ReconcileMatch list with confidence score.
Return list. Caller presents to user for confirmation.

ReconcileMatch: { BOQLineItem PSRow; BOQLineItem ModeledRow; double ConfidencePct; string Reason; }

### 3K. GenerateCashFlowWithBOQ(Document doc, BOQDocument boq) → JObject

Extend the existing Scheduling4DEngine.GenerateCashFlow logic:
  - Use BOQDocument totals instead of the old estimate JSON.
  - Distribute modeled costs according to 4D schedule phase dates.
  - Distribute PS costs as lump sums at end of project (or phase if note specifies).
  - Return monthly cash flow JObject in same format as existing cash_flow_5d.json.

---

## PHASE 4 — NRM2 paragraph automation (BOQTemplateLibrary.cs enhancements)

Read BOQTemplateLibrary.cs completely first. All additions must be additive —
do NOT change any existing method signatures or modify existing public APIs.

### 4A. ResolveForElement(BOQTemplate template, Element el, Document doc) → string

Add this NEW public static method. Returns the fully resolved paragraph with
all [token] strings substituted and prose cleaned.

TOKEN RESOLUTION ORDER for each [token]:
  1. Shared parameter: el.LookupParameter(token) then ParameterHelpers.GetString(el, token).
  2. Built-in parameter mapping (case-insensitive token match):
       material          → first non-null of: ALL_MODEL_MATERIAL_ASSET_NAME,
                           first compound layer material name (via el.GetMaterialIds)
       location          → ASS_LOC_TXT → room name from ParameterHelpers spatial lookup
       level             → SCHEDULE_LEVEL_PARAM display value
       length            → CURVE_ELEM_LENGTH × 0.3048 (ft→m) formatted as "X.Xm"
       width             → family type param "Width" → FAMILY_WIDTH_PARAM
       height            → family type param "Height" → FAMILY_HEIGHT_PARAM
       thickness         → family type param "Thickness" → first wall layer thickness
       depth             → family type param "Depth"
       diameter          → RBS_PIPE_DIAMETER_PARAM → family param "Diameter"
       nominal_size      → RBS_PIPE_DIAMETER_PARAM formatted as integer mm
       dimension         → width × height formatted from duct/pipe params
       manufacturer      → ALL_MODEL_MANUFACTURER
       model             → ALL_MODEL_MODEL
       model_ref         → ALL_MODEL_MODEL → fallback to TypeName
       fire_rating       → DOOR_FIRE_RATING → family param "Fire Rating"
       finish            → ASS_FINISH_TXT → family param "Finish"
       insulation        → family param "Insulation" → "as specification"
       substrate         → first inner layer material of compound wall
       spacing           → family param "Spacing" formatted as integer mm
       standard          → discipline-based default:
                             A → "BS EN 1996" | S → "BS EN 1992" | M → "CIBSE Guide B"
                             E → "BS 7671:2018+A1:2020" | P → "BS EN 1057"
                             FP → "BS EN 12845" | default → "as specification"
       concrete_spec     → family param "Concrete Grade" → "C25/30"
       reinforcement     → family param "Reinforcement" → "as structural drawings"
       section_size      → family param "Section Size" → structural section symbol param
       airflow           → RBS_DUCT_FLOW_PARAM formatted as integer l/s
       rating            → RBS_ELEC_CIRCUIT_RATING_PARAM formatted as integer A
       voltage           → family param "Voltage"
       fixture_type      → family name cleaned of manufacturer prefix
       equipment_type    → family name
       capacity          → family param "Capacity" → flow param
       wattage           → family param "Wattage" → RBS_ELEC_APPARENT_LOAD_PARAM
       lux_target        → family param "Lux Level" → "300 lux"
       service           → RBS_SYSTEM_NAME_PARAM → MEP system type name
  3. If still unresolved after step 2: leave blank (do NOT keep the [token] literal).
  4. After all substitutions: run the existing tidy logic (regex strip remaining tokens,
     collapse multiple spaces, fix ", ," → ",", ensure single trailing period).

IMPORTANT: The method must NEVER throw. Wrap the entire body in try/catch and return
the template paragraph (with tokens stripped) on any exception.

### 4B. SelectBestTemplate(string category, Element el) overload

Add an overload that accepts an optional Element parameter:
  public static BOQTemplate SelectBestTemplate(string category, Element el = null)

Scoring (additive, highest score wins):
  +1  category matches template.Category (base requirement)
  +2  disc code matches template.DiscContains
  +3  element family name contains template.FamilyContains (case-insensitive)
  +2  element type name contains template.TypeContains (case-insensitive)
  +1  element MEP system name contains template.SystemContains
  +2  template.Version is higher than competing same-score template

Tie-break: project source beats company beats builtin.
Return null if no template found for category.

### 4C. BuildResolvabilityReport(Document doc) → ResolvabilityReport

Already has the class definition. Implement the actual calculation:
  - Load all templates (builtin + company + project).
  - For each template's category: collect matching elements via FilteredElementCollector.
  - For each element: call ResolveForElement. Count resolved vs unresolved per token.
  - Populate ResolvabilityReport.
  - Log summary via StingLog.Info.

### 4D. BOQNarrativeEnhancer (inner class or separate file)

Add a static class BOQNarrativeEnhancer in the same namespace with:

  EnhanceWithMEPContext(BOQLineItem item, Element el, Document doc)
    For MEP elements: append system name, served area, and flow/capacity to the
    resolved paragraph as a supplementary clause.
    Pattern: "... serving {systemName} at {location}; rated at {capacity}."
    Only adds if ResolvedNRM2Paragraph does not already contain the system name.

  EnhanceWithStructuralContext(BOQLineItem item, Element el, Document doc)
    For structural elements: append grid reference and level span.
    Pattern: "... at grid reference {gridRef}, spanning {levelFrom} to {levelTo}."
    Read grid intersection from analytical model or nearest grid elements.

  GenerateDiffAnnotation(CategoryDiff diff) → string
    Returns the italic annotation sentence used in snapshot comparison reports.
    Pattern varies by ChangeType:
      RATE_REVISED: "Rate revised {cat} UGX {rateA:N0} → {rateB:N0}/unit following {reason}."
      QTY_CHANGED: "{noun} {direction} {qtyA} → {qtyB} {unit} per {reason}."
      NEW_ITEM: "{qty} no. newly modeled; previously {psOrManual}."
      ITEM_REMOVED: "Removed from model — {reason}."
      PS_ADDED: "PC sum registered: {description}."
      SOURCE_PROMOTED: "Promoted from manual row to modeled element."

---

## PHASE 5 — BOQ & Cost Manager dock panel (src/UI/BOQCostManagerPanel.cs)

Create a WPF UserControl. No XAML file. All layout in C# code-behind.
Use ThemeManager colours. Follow StingResultPanel.cs as the style reference.
This panel is hosted in BIMCoordinationCenter._4dPanelArea (ContentControl).

### 5A. ViewModels (inner classes or file BOQViewModels.cs)

  BOQSectionViewModel : INotifyPropertyChanged
    Properties: Id, Name, Discipline, NRM2Section, IsExpanded (bool),
    Items (ObservableCollection<BOQItemViewModel>), TotalDisplay (string),
    DeltaDisplay (string), DeltaIsPositive (bool)

  BOQItemViewModel : INotifyPropertyChanged
    Properties: Id, ItemName, FamilyName, Quantity, Unit, RateDisplay,
    TotalDisplay, Note, Source (BOQRowSource), SourceColor (SolidColorBrush),
    SnapshotDelta (string), RevitElementId, IsEditing (bool)
    Commands: EditNameCommand, EditRateCommand, EditQtyCommand, EditNoteCommand

### 5B. Panel layout (BuildUI() method called from constructor)

LAYOUT: DockPanel with sections docked:
  TOP: Header strip (project name, currency toggle UGX/USD, refresh button)
  TOP: Budget strip (4 metric cards + progress bar + "Set budget" button)
  TOP: Snapshot row (ComboBox + badge + diff label + "Save ▾" + "Compare" buttons)
  TOP: Tab strip (BOQ | Materials | NRM2 Templates | Carbon)
  TOP: Toolbar (search TextBox + discipline ToggleButtons + match hint TextBlock)
  FILL: Content area (TabControl with 4 tabs)
  BOTTOM: Footer bar (grand total + action buttons)

HEADER STRIP:
  - TextBlock: project name from doc.Title
  - ToggleButton pair "UGX" / "USD": on toggle, call RefreshDisplayCurrency()
  - Button "Refresh": raises BOQRefresh ExternalEvent
  - Button "Set budget": InputDialog → save to project_config.json → UpdateBudgetStrip()

BUDGET STRIP:
  UniformGrid { Columns = 5 } containing metric cards:
  Card 1: "Project budget" | PROJECT_BUDGET_UGX from ProjectInfo parameter
  Card 2: "Modeled cost" | SUM(CST_MODELED_TOTAL_UGX) via live FilteredElementCollector
  Card 3: "Provisional / manual" | sum from LoadManualRows()
  Card 4: "Variance" | budget − total; TextBlock colour: green/amber/red
  Card 5: "Coverage %" | modeled / budget × 100
  Below cards: two-segment progress bar (green = modeled, purple = provisional)
  Update method UpdateBudgetStrip() recalculates without calling BuildBOQDocument.

SNAPSHOT ROW:
  - ComboBox (snap-sel): populated by BOQCostManager.ListSnapshots(doc).
    Item template: "{Type badge} {Label} — {Date:dd MMM yyyy} — UGX {Total:N0}"
    On selection change: show diff label ("{+/−} UGX X vs snapshot") → refresh sections.
  - Button "Save snapshot ▾": ContextMenu with 5 items (DD/Stage/Weekly/Handover/Manual).
    Each calls BOQCostManager.SaveSnapshot then refreshes ComboBox.
  - Button "Compare": dispatches BOQSnapshotCompare action.

TOOLBAR (hidden on NRM2 tab):
  - TextBox (search): oninput → filter CollectionViewSource predicate on ALL fields
    including numeric values: ItemName, FamilyName, Unit, Note, RateUGX.ToString(),
    RateUSD.ToString(), TotalUGX.ToString(), NRM2Section, Category, Location, Level.
  - ToggleButton row for disciplines: ALL / A / S / M / E / P / FP / PS
    Multi-select: holding Ctrl keeps previous selections.
    Clicking a disc button without Ctrl selects only that discipline.
    Clicking ALL resets all.
  - TextBlock match hint: "Showing N of M sections | N items matching 'X'"

BOQ TAB:
  ItemsControl bound to filtered ObservableCollection<BOQSectionViewModel>.
  Each section rendered as Expander:
    Header: NRM2 section badge + name + disc badge + item count + total + snapshot delta pill
    Content: DataGrid with columns as specified in previous phase document.
  DataGrid columns:
    Col 0 (30px): source dot (Ellipse, green=model/amber=manual/purple=PS)
    Col 1 (⋯ btn, 24px): ContextMenu trigger
    Col 2 (Item name, *): editable on double-click; shows FamilyName in secondary style
    Col 3 (Qty, 60px): editable double-click
    Col 4 (Unit, 55px): editable double-click → dropdown of known units
    Col 5 (Rate, 110px): editable double-click; formatted per currency
    Col 6 (Total, 110px): read-only computed; formatted per currency
    Col 7 (Carbon kgCO2e, 90px): read-only; grey if zero
    Col 8 (Note, *): editable double-click
  Context menu on row right-click (and ⋯ click):
    Edit name / Edit rate / Edit quantity / Edit unit / Edit note (all 5 open inline edit)
    ── separator ──
    Mark as modeled / Mark as manual / Mark as provisional sum
    ── separator ──
    Duplicate row / Delete row
    ── separator ──
    Select in Revit (raises SelectInRevit action; disabled for manual/PS)
    Open NRM2 template (switches to NRM2 tab and selects this item's category)
  Add-row strip at bottom of each section:
    "＋ Modeled row" | "＋ Manual / unmodeled" | "＋ Provisional sum"
    Each opens a quick-add dialog (name + qty + unit + rate).
  Footer: "＋ Section" button → InputDialog for name + discipline

MATERIALS TAB:
  Same toolbar filter applies. Sections per material class.
  DataGrid columns: source dot, material name, area m², volume m³, unit, rate, total, carbon, note.

NRM2 TEMPLATES TAB:
  Two-column Grid { ColumnDefinitions: 200px * }.
  Left pane: ScrollViewer > ListView of templates sorted by §NRM2 section.
    Item template: "§{section}" badge + category name + disc badge.
    TextBox above for filter (searches category + section + paragraph content).
    "＋ New template" button at bottom.
  Right pane: on selection show:
    a) RichTextBox (edit mode on double-click): paragraph with [token] spans coloured blue.
       On text change: re-parse tokens → refresh fields grid.
    b) UniformGrid of TextBox per token (2 columns). Label = [token name].
       On each TextBox change: update live preview.
    c) "Auto-fill from model" button: calls ResolveForElement using current Revit selection.
    d) TextBlock preview: resolved paragraph (wrapping, italic).
    e) Actions row: "Save template", "Add row to BOQ", "Suggest rewording ↗"
       "Suggest rewording" dispatches to NLP command processor for alternative phrasing.

CARBON TAB (new tab, not in previous version):
  Summary cards:
    Total embodied carbon (tCO2e), Carbon intensity (kgCO2e/m² GIA),
    RIBA 2030 benchmark comparison, Top 3 categories by carbon.
  Bar chart using Canvas + Rectangle elements (no external chart lib):
    Discipline breakdown as horizontal bars, labelled with kgCO2e value.
  "Export Carbon Report" button → dispatches CarbonBOQExport action.
  "Compare to baseline" button → compares current to stored carbon baseline.

FOOTER BAR:
  Left: "Grand total (incl. provisional)" label + large TextBlock value
  Right: action buttons: "＋ Section" | "Reset rates" | "Export ↗" | "Save" (primary style)

### 5C. Inline editing implementation

For each editable DataGrid cell:
  MouseDoubleClick handler on the DataGridCell:
    1. Set IsEditing = true on the row ViewModel.
    2. Replace the cell's ContentTemplate with an edit template containing a TextBox.
    3. TextBox PreviewKeyDown: Enter → commit, Escape → cancel.
    4. TextBox LostFocus: commit.
  On commit:
    - Validate the new value (numeric check for rate/qty, non-empty for name).
    - Update the ViewModel property.
    - Call BOQCostManager.SaveManualRows if Source != Model.
    - If Source == Model, set CST_* parameters via ExternalEvent dispatch.
    - Refresh totals in section header and footer.

### 5D. Dispatch and refresh

All Revit API calls go through ExternalEvent dispatch:
  "BOQRefresh"           → BuildBOQDocument + WriteElementParameters + UpdateBudgetStrip
  "BOQSave"              → SaveManualRows + SaveSnapshot if auto-snapshot enabled
  "BOQSnapshotCompare"   → CompareSnapshots + show StingResultPanel
  "BOQCarbonExport"      → Carbon report generation
  "SelectInRevit"        → uidoc.Selection.SetElementIds with element from ExtraParam
  "ReconcileProvisionals"→ ReconcileProvisionals + show confirmation dialog
  "BOQImport"            → Excel roundtrip import

Register all tags in StingCommandHandler.cs switch block using RunCommand<T> pattern.
Add "BOQ Cost Manager" button to StingDockPanel.xaml in the Schedules section:
  <Button Style="{StaticResource GreenBtn}" Content="★ BOQ Cost Manager"
          Tag="BOQCostManager" Click="Cmd_Click" FontWeight="Bold"
          ToolTip="Full BOQ editor: NRM2 paragraphs, rates, provisionals, snapshots, carbon"/>

---

## PHASE 6 — BOQ Excel export (src/Commands/BOQ/BOQExportCommand.cs)

[Transaction(TransactionMode.Manual)] command, dispatched via "BOQExport" tag.
Uses ClosedXML. Creates a zero-edit-required XLSX — no post-processing needed by QS.

### 6A. Pre-export pipeline (in Execute())

  1. Call BOQCostManager.BuildBOQDocument(doc) → boq.
  2. Run BOQTemplateLibrary.BuildResolvabilityReport(doc) → report.
     If report.ResolvedPercent < 80:
       Show WPF dialog (NOT TaskDialog) listing unresolved tokens by category.
       Three buttons: "Continue anyway", "Open NRM2 Templates", "Cancel".
       On "Open NRM2 Templates": dispatch "BOQCostManager" and return Cancelled.
       On "Cancel": return Cancelled.
  3. Write element parameters in a transaction: BOQCostManager.WriteElementParameters.
  4. Write project parameters: BOQCostManager.WriteProjectParameters.
  5. Build XLWorkbook.

### 6B. Workbook structure (8 sheets)

Create sheets in this order. Each method BuildSheet_X(wb, boq) returns IXLWorksheet.

SHEET 1 — "BOQ Summary"
  A4 paper, landscape orientation set on page setup.
  Print area set to used range. Freeze pane at row 7 (after column headers).
  Repeat rows 1:6 on print (PrintTitlesRows).

  ROWS 1–2: Project header block
    Row 1: merge A1:M1. Text: "{doc.Title} — Bill of Quantities". 14pt bold.
            Fill: RGB(26, 58, 92) navy. Font white.
    Row 2: merge A2:M2. Text: "Prepared by STING Tools | {DateTime.Now:dd MMMM yyyy} | {currency}"
            11pt. Same fill. Font white.

  ROW 3: blank separator.

  ROW 4: Summary metrics (inline, not a table):
    A4: "Project budget:" B4: {budget:N0} C4: "Modeled:" D4: {modeled:N0}
    E4: "Provisional:" F4: {prov:N0} G4: "Grand total:" H4: {grand:N0}
    I4: "Coverage:" J4: {pct:F1}% K4: "Carbon:" L4: {carbon:F0} kgCO2e
    Bold labels, normal values. Light fill row.

  ROW 5: blank.

  ROW 6: Column headers — 13 columns:
    A: NRM2 §  B: Line ref  C: Description (NRM2 narrative)  D: Unit
    E: Quantity  F: Rate UGX  G: Total UGX  H: Rate USD  I: Total USD
    J: Source  K: Discipline  L: Level / Location  M: Note / Annotation
    Fill: RGB(46, 94, 142). Font white. 10pt bold. Row height 18px.

  Column widths (in characters): A=8 B=10 C=58 D=6 E=9 F=14 G=14 H=12 I=12 J=9 K=9 L=18 M=22

  DATA ROWS starting at row 7:
    Before each section's first item: insert one section header row.
      Merge A:M. Text: "§{section} — {sectionName}" 11pt semi-bold.
      Fill by discipline:
        A → RGB(214,228,240) blue-grey | S → RGB(235,230,250) purple-grey
        M → RGB(255,243,224) amber-light | E → RGB(252,235,235) red-light
        P → RGB(225,245,238) green-light | FP → RGB(252,235,235) same as E
        PS → RGB(237,231,246) purple-light
      Set row height to 16px.

    For each BOQLineItem:
      Col A: NRM2 section number (no §)
      Col B: BOQLineRef (e.g. "14.3.2")
      Col C: item.ResolvedNRM2Paragraph — WrapText=true, row height AdjustToContents
             (min 20px, max 80px — cap to prevent oversized rows)
      Col D: item.Unit
      Col E: item.Quantity — format "#,##0.000" for area/volume, "#,##0" for count
      Col F: item.RateUGX — format "#,##0" (no decimals for UGX)
      Col G: item.TotalUGX — format "#,##0" — FORMULA: =E{row}*F{row}
      Col H: item.RateUSD — format "#,##0.00"
      Col I: item.TotalUSD — format "#,##0.00" — FORMULA: =E{row}*H{row}
      Col J: item.Source.ToString() — "Model" / "Manual" / "Provisional Sum"
      Col K: item.Discipline
      Col L: item.Level + (item.Location != "" ? " / "+item.Location : "")
      Col M: item.Note
      Row fill by source:
        Model → none (white) | Manual → RGB(255,251,230) cream | PS → RGB(243,240,252) lavender
      Apply thin borders on all cells (RGB(200,200,200)).

  SUMMARY BLOCK (3 blank rows after last data row, then):
    Subtotal row: Col F "Subtotal" merged A:F bold. Col G: =SUM(G7:G{lastRow}) formula.
    Prelims row: Col F "Preliminaries ({pct}%)" merged. Col G: =G{subtotal}*{pct/100} formula.
    Contingency row: same pattern.
    Overhead row: same pattern.
    Grand total row: Col F "GRAND TOTAL" 12pt bold merged A:F. Fill navy. Font white.
      Col G: SUM formula. Col I: USD equivalent.
    If budget set: Budget variance row: show "Budget: {budget:N0} | Variance: {variance:N0}"
      Fill: green if variance ≥ 0, red if variance < 0.

  DISCIPLINE SUMMARY block (5 rows after grand total):
    Compact 3-column table: Discipline | Total UGX | % of total
    Sorted descending by total. Last row: "Total modeled" + "Total PS" sub-totals.

SHEET 2 — "Item Schedule"
  Flat table, one row per item. Same columns as Sheet 1 plus:
    Col N: RevitElementId (for re-import matching)
    Col O: EmbodiedCarbonKg
    Col P: LifecycleCostUGX
  Apply AutoFilter to row 1 (headers). Freeze row 1.
  This is the import-roundtrip sheet. BOQLineRef (col B) is the join key.
  Include ALL items — model + manual + PS — so QS can edit all in one sheet.

SHEET 3 — "Material Schedule"
  Header: same navy style as Sheet 1.
  Columns: NRM2 § | Category | Material name | Area m² | Volume m³ | Unit |
           Rate UGX | Rate USD | Total UGX | Source | Level | Carbon kgCO2e
  Items from matData equivalent (elements where MAT_MEASURED_AREA_M2 is set).
  AutoFilter, freeze row 1.

SHEET 4 — "Provisional Sums"
  Items where Source == BOQRowSource.ProvisionalSum.
  Columns: PS Ref | Description | NRM2 § | Scope / note | Rate UGX | Rate USD |
           Status (Open/Instructed/Closed) | Instructed by | Instruction date
  Status read from item.Note parsing (look for "Status:" prefix).
  PS total row at bottom.
  If no PS rows: write "No provisional sums registered on this project." in A2.

SHEET 5 — "Snapshot Comparison"
  Only written if BOQCostManager.ListSnapshots returns ≥ 2 snapshots.
  Compares latest two snapshots via CompareSnapshots.
  Columns: NRM2 § | Category | Disc | Snap A total | Snap B total |
           Delta UGX | Delta % | Change type | Auto-generated reason
  Section header rows per discipline (same fill colours as Sheet 1).
  Conditional formatting: green fill for positive delta rows, red for negative.
  KPI row at top: Snapshot A label + date + total, Snapshot B label + date + total, net change.
  Plain summary paragraph in merged cells below KPI row.
  NRM2 narrative section: for each changed item, full resolved paragraph + GenerateDiffAnnotation.

SHEET 6 — "NRM2 Reference"
  One row per unique NRM2 section used in this BOQ.
  Columns: Section § | Section name | Discipline | Template paragraph (with [tokens] shown) |
           Resolved coverage % | Source (builtin/company/project)
  Footer note in last used row + 2:
    "Paragraphs are STING-authored construction descriptions following NRM2 work section
     structure. Not reproduced from RICS NRM2. © STING Tools {year}."

SHEET 7 — "Carbon & Lifecycle"
  Per-element carbon and lifecycle data.
  Columns: Line ref | Category | Discipline | Quantity | Unit | Material |
           Carbon factor kgCO2e/kg | Total kgCO2e | Lifecycle cost UGX 25yr |
           RIBA benchmark (Excellent/Good/Average/High/Very High)
  Discipline subtotals. Project total row. kgCO2e/m² GIA metric.
  RIBA 2030 target reference row: target 300 kgCO2e/m².

SHEET 8 — "Audit Trail"
  One row per item. Columns: Line ref | Element ID | UniqueId | Category | Family |
           Rate source | Quantity basis (area/length/volume/count) |
           Rate UGX | Rate USD | Last costed | Snapshot ref | Paragraph source
  This sheet makes every rate and paragraph fully auditable.

### 6C. Post-export

  - Open output folder via Process.Start.
  - Show StingResultPanel with:
      File path and name
      "Sheets: 8 | Items: {N} | Modeled: UGX {X:N0} | Provisional: UGX {Y:N0}"
      "Grand total: UGX {Z:N0} | Carbon: {C:F0} kgCO2e"
      "Paragraph coverage: {pct:F0}% ({resolved}/{total} items with NRM2 descriptions)"
      Colour-coded coverage badge: green ≥90%, amber 70-89%, red <70%.

---

## PHASE 7 — BCC 4D/5D tab integration (BIMCoordinationCenter.cs)

Read Build4D5DTab() and the existing KPI card section completely before editing.
Add to the EXISTING kpiGrid — do not replace it.

### 7A. New KPI cards (add after existing 4 cards)

BUDGET card:
  Label: "BUDGET"
  Value: Read PROJECT_BUDGET_UGX from ProjectInfo parameter. If zero: "Not set".
  Format: "UGX {N:N0}" or "$ {N:N2}" depending on project currency setting.
  Colour: Blue (neutral — this is a target, not a status).
  Tooltip: "Project budget | Double-click to open BOQ Cost Manager"
  Double-click: DispatchAction("BOQCostManager")

COVERAGE card:
  Label: "BOQ COVERAGE"
  Value: CST_BOQ_COVERAGE_PCT from ProjectInfo. If null: "N/A".
  Colour: Green ≥90%, Amber 70–89%, Red <70%.
  Tooltip: "Percentage of modeled cost against project budget\n\nDouble-click for BOQ"
  Double-click: DispatchAction("BOQCostManager")

VARIANCE card:
  Label: "VARIANCE"
  Value: CST_BUDGET_VARIANCE_UGX from ProjectInfo. Prefix + if positive, − if negative.
  Colour: Green if positive (under budget), Red if negative (over budget), Amber if within 5%.
  Tooltip: "Budget variance (budget − modeled+provisional total)\n\nDouble-click for BOQ"
  Double-click: DispatchAction("BOQCostManager")

CARBON card:
  Label: "EMBODIED CO2"
  Value: Sum of CST_EMBODIED_CARBON_KG via FilteredElementCollector. Format as "{N:F0} t".
         (divide kg sum by 1000 → tonnes)
  Colour: Green < 300 kgCO2e/m², Amber 300–800, Red > 800 (per RIBA benchmark).
  Tooltip: "Total embodied carbon (kgCO2e) | RIBA 2030 benchmark: 300 kgCO2e/m²\n\nDouble-click for Carbon tab"
  Double-click: DispatchAction("BOQCostManager") with ExtraParam "BOQTab"="Carbon"

### 7B. BOQ inline panel area

Add a new ContentControl _boqPanelArea below the 4D KPI grid.
When "BOQCostManager" is dispatched: instantiate BOQCostManagerPanel and set as
_boqPanelArea.Content. Pattern identical to _4dPanelArea used for cost rate editor.

### 7C. StingCommandHandler.cs additions

In the Execute() switch block, add:
  case "BOQCostManager":
    // Open BOQ panel in BCC inline area
    BIMCoordinationCenter.CurrentInstance?.ShowBOQPanel();
    break;
  case "BOQExport":
    RunCommand<BOQ.BOQExportCommand>(app); break;
  case "BOQImport":
    RunCommand<BOQ.BOQImportCommand>(app); break;
  case "BOQSnapshotCompare":
    RunCommand<BOQ.BOQSnapshotCompareCommand>(app); break;
  case "ReconcileProvisionals":
    RunCommand<BOQ.ReconcileProvisionalsCommand>(app); break;
  case "BOQCarbonExport":
    RunCommand<BOQ.BOQCarbonExportCommand>(app); break;

Add ShowBOQPanel() method to BIMCoordinationCenter:
  Sets _boqPanelArea.Content to a new BOQCostManagerPanel(doc).
  Activates the 4D/5D tab if not already active.

---

## PHASE 8 — Excel roundtrip import (src/Commands/BOQ/BOQImportCommand.cs)

[Transaction(TransactionMode.Manual)]. Dispatch tag: "BOQImport".

PIPELINE:
  1. Show OpenFileDialog filtered to *.xlsx. User selects previously exported BOQ.
  2. Load workbook. Find "Item Schedule" sheet. If not found → error dialog + return.
  3. Read headers from row 1. Locate columns: "Line ref" (col B), "Rate UGX", "Rate USD",
     "Description (NRM2 narrative)", "Note", "Source".
  4. For each data row (skip empty, skip section headers by checking merge):
     a. Read BOQLineRef from col B.
     b. Find matching element: query FilteredElementCollector where
        ASS_BOQ_LINE_REF matches. Use a pre-built Dictionary<string, ElementId>
        for O(1) lookup (build once before loop).
     c. For manual/PS rows (Source != "Model"): update project_boq_manual.json.
     d. For modeled elements: collect (elementId, newRate, newParagraph, newNote) into batch.
  5. Validate: rate > 0 required for modeled elements. Warn on blank paragraph.
     Show pre-import summary: N matched, N manual, N unmatched, N validation errors.
     User confirms: "Import {matched} rows?" Yes/No.
  6. On confirm: write parameters to elements in single transaction.
     Write CST_UNIT_RATE_UGX, CST_UNIT_RATE_USD, ASS_NRM2_PARA_TXT (if non-empty).
     Write CST_RATE_SOURCE = "Override".
  7. Update manual rows via BOQCostManager.SaveManualRows.
  8. Call BOQCostManager.BuildBOQDocument to refresh totals.
  9. Show StingResultPanel: matched, updated, skipped, errors. Rate change summary.

---

## PHASE 9 — Snapshot comparison report (src/Commands/BOQ/BOQSnapshotCompareCommand.cs)

[Transaction(TransactionMode.ReadOnly)]. Dispatch tag: "BOQSnapshotCompare".

  1. Load snapshot list. If < 2 snapshots: show "Need at least 2 snapshots to compare.
     Save a snapshot now via the BOQ panel." dialog + return.
  2. Show snapshot picker dialog: two ComboBoxes (Snapshot A / Snapshot B)
     pre-populated with latest two. User can change selection.
  3. Call BOQCostManager.CompareSnapshots(pathA, pathB) → diff.
  4. Build StingResultPanel with the following sections:
     Section "COMPARISON OVERVIEW" (ResultType.Metric):
       Items: LabelA, LabelB, Net change (UGX), Net change %, Changed categories count,
              Modeled change, Provisional change, Carbon change
     Section "BY DISCIPLINE" (ResultType.Table):
       Columns: Discipline, Snap A, Snap B, Delta, Movement
     Section "BY CATEGORY — changed items only" (ResultType.Table):
       Columns: NRM2 §, Category, Disc, Snap A, Snap B, Delta, Change type, Reason
     Section "NRM2 NARRATIVES — changed items" (ResultType.Text per changed item):
       For each CategoryDiff where ChangeType != NoChange:
         Header: "{§section} {category}" + change type badge
         Paragraph: item.ResolvedNRM2Paragraph from snapshot B
         Annotation: BOQNarrativeEnhancer.GenerateDiffAnnotation(diff) — italic note
     Section "SUMMARY" (ResultType.Text):
       diff.PlainSummary (auto-generated narrative)
  5. Add action buttons to result panel:
     "Export comparison to Excel" → saves the diff as a new sheet in a new XLSX
     "Update budget" → dispatches budget set dialog
     "View in BOQ panel" → dispatches BOQCostManager

---

## PHASE 10 — Reconcile provisionals (src/Commands/BOQ/ReconcileProvisionalsCommand.cs)

[Transaction(TransactionMode.Manual)]. Dispatch tag: "ReconcileProvisionals".

  1. Call BOQCostManager.BuildBOQDocument → boq.
  2. Call BOQCostManager.ReconcileProvisionals(doc, boq) → matches.
  3. If no matches: "No provisional sums could be automatically matched to modeled elements."
  4. Show a WPF dialog listing each ReconcileMatch:
     "PS: {psRow.ItemName} (UGX {ps.TotalUGX:N0}) ↔ Modeled: {modRow.ItemName}
      ({confidence:F0}% confidence — {reason})"
     CheckBox per match row. "Promote selected" button.
  5. On "Promote selected":
     - For each confirmed match: set PSRow.Source = Model, set RevitElementId.
     - Save to project_boq_manual.json (removing the PS row, it becomes model-linked).
     - Write CST_PROVISIONAL_SUM = false on the Revit element.
     - Log "Promoted {N} provisional sums to modeled elements."
  6. Refresh BOQ panel.

---

## PHASE 11 — Additional automation enhancements (research-based additions)

### 11A. Rate intelligence engine (BOQRateEngine.cs)

Create src/Commands/BOQ/BOQRateEngine.cs.

RATE CONFIDENCE SCORING:
  Each BOQLineItem gets a RateConfidence score (0–100):
    100 = rate manually verified by QS this session
    90  = rate from cost_rates_5d.csv with recent date (< 6 months)
    75  = rate from COBie type map
    60  = rate from DefaultCostRates in SchedulingCommands
    40  = rate interpolated from similar category (see below)
    20  = single default estimate (no matching rate found)
  Store confidence as CST_RATE_CONFIDENCE integer parameter (add to MR_PARAMETERS.txt).
  Show confidence as colour-coded column in BOQ panel (green ≥75, amber 40–74, red <40).

RATE INTERPOLATION:
  For categories with no exact rate match, attempt interpolation:
    - Find categories in the same NRM2 section with known rates.
    - Weight by material similarity (e.g. "Concrete Columns" uses Structural Columns rate × 0.9).
    - Fallback: use discipline average rate × 1.0.
  Log all interpolated rates: "Rate interpolated for {category}: {interpolated} UGX/unit
  (based on {sourceCategory}, confidence 40)."

RATE HISTORY TRACKING:
  On each save, if a rate changed, append to rate_history_{category}.json in bimDir:
    { date, old_rate_ugx, new_rate_ugx, changed_by (Environment.UserName), source }
  Cap to last 50 entries per category.
  "Rate history" button in context menu → StingResultPanel showing history for that category.

### 11B. Automatic provisional sum promotion triggers

In BOQCostManager.BuildBOQDocument, after collecting all modeled elements:
  For each open PS row: check if a matching modeled element now exists
  (same Category + TotalUGX within ±30%). If found, add a flag PS_MATCH_FOUND to the
  PS row. The BCC budget strip shows a badge: "3 PS rows have matching modeled elements —
  click Reconcile to review." This badge links to ReconcileProvisionals command.

### 11C. BOQ health score

Add BOQCostManager.ComputeBOQHealth(BOQDocument boq) → BOQHealthScore

BOQHealthScore:
  double OverallScore (0–100)
  string Grade ("Excellent" / "Good" / "Fair" / "Poor")
  List<string> Issues
  List<string> Recommendations

Scoring factors:
  +25 pts: paragraph coverage ≥ 90%
  +20 pts: rate confidence average ≥ 75
  +15 pts: no [token] strings remaining in any paragraph
  +15 pts: all items have BOQLineRef assigned
  +10 pts: budget is set and coverage is between 80–110%
  +10 pts: all provisional sums have a note / scope description
  +5 pts: carbon data available for ≥ 50% of items

Show BOQ Health score as a KPI card in the BOQ panel header strip.
Surface in BCC Overview tab alongside the other health KPI cards.

### 11D. NRM2 paragraph versioning and audit

When ASS_NRM2_PARA_TXT is overwritten by a new resolution:
  1. Read the existing value.
  2. If different: write the old value to ASS_NRM2_PARA_PREV_TXT (new parameter, group 1).
  3. Write the change timestamp to ASS_NRM2_PARA_DATE_TXT (new parameter, group 1).
  4. Write the change author to ASS_NRM2_PARA_AUTHOR_TXT (new parameter, group 1).
  These form an audit trail of paragraph changes per element.
  The BOQ panel context menu "View paragraph history" shows all 3 values in a tooltip.

Add to MR_PARAMETERS.txt:
  ASS_NRM2_PARA_PREV_TXT   TEXT   group 1   "Previous NRM2 paragraph (for audit) [ISO 19650-3:2020]"
  ASS_NRM2_PARA_DATE_TXT   TEXT   group 1   "Date NRM2 paragraph was last updated [ISO 19650-3:2020]"
  ASS_NRM2_PARA_AUTHOR_TXT TEXT   group 1   "Author who last updated the NRM2 paragraph [ISO 19650-3:2020]"

### 11E. Multi-currency support

Add to project_config.json:
  "EXCHANGE_RATES": { "UGX_PER_USD": 3700, "UGX_PER_GBP": 4700, "UGX_PER_EUR": 4050 }
  "BOQ_DISPLAY_CURRENCIES": ["UGX", "USD"]  // which currencies to show in the BOQ panel
  "BOQ_PRIMARY_CURRENCY": "UGX"

In BOQ panel currency toggle: change from binary UGX/USD to a ComboBox listing
all currencies in BOQ_DISPLAY_CURRENCIES. On change: recalculate all rate/total
displays using the exchange rates from config.

In Excel export: if BOQ_DISPLAY_CURRENCIES has 2+ currencies, write ALL currency
columns to the export (Rate UGX, Total UGX, Rate USD, Total USD, Rate GBP, Total GBP etc.).
Dynamically generate columns based on config — no hardcoded USD-only assumption.

### 11F. COBie Resource integration for provisional sums

After PS rows are saved in project_boq_manual.json, call:
  BOQCostManager.SyncPSToCobie(doc)

This creates/updates a COBie Resource record for each PS row:
  Name: "PS-{boqLineRef}-{itemName}"
  Category: "Provisional Sum"
  ExternalSystem: "STING BOQ"
  Description: item.ResolvedNRM2Paragraph
  Cost: item.TotalUGX.ToString()

The existing BIMManagerCommands.cs COBie export already reads from the Resource list.
This makes PS rows visible in the COBie handover package automatically.

### 11G. Revit Schedule integration

Add BOQScheduleCommand (dispatch tag: "BOQCreateSchedule"):
  Creates Revit schedules directly from BOQ data:
    Schedule 1: "STING — BOQ Summary" — one row per category.
      Fields: Category, BOQLineRef, NRM2Section, Quantity, Unit, RateUGX, TotalUGX.
      Reads from CST_* shared parameters.
    Schedule 2: "STING — NRM2 Descriptions" — one row per tagged element.
      Fields: ASS_BOQ_LINE_REF, Category, Level, ASS_NRM2_PARA_TXT.
    Schedule 3: "STING — Carbon Register" — one row per element with carbon data.
      Fields: Category, Discipline, Level, CST_EMBODIED_CARBON_KG, CST_LIFECYCLE_COST_UGX.
  Uses ViewSchedule API. Applies existing ComplianceScan schedule template style if available.

### 11H. Intelligent BOQ update detection (change monitoring)

On every Build4DTab refresh in BCC:
  Compare current element count and total area per category against
  the last BOQ snapshot's values (stored in boq_snapshot_latest.json if present).
  If significant change detected (any category ±5% qty or ±10% area):
    Show a banner in the BCC budget strip: "Model changed since last BOQ — N elements
    added/removed. Click Refresh to update costs."
  This avoids the user having to manually track whether the BOQ is stale.

### 11I. BOQ workflow preset integration

Add "BOQ Package" to the BCC workflow presets in BIMCoordinationCenter.cs:
  Steps: ["Pre-Tag Audit", "Validate Tags", "BOQ Refresh", "BOQ Export",
          "Carbon Report", "BOQ Snapshot Save", "COBie Export"]
  Label: "BOQ Package"
  Description: "Complete BOQ + carbon + COBie data package for QS handover"
  Colour: green (same as Handover preset)

### 11J. project_boq_manual.json schema v1.1 (final)

{
  "schema_version": "1.1",
  "project_budget_ugx": 250000000,
  "boq_display_currencies": ["UGX", "USD"],
  "last_saved": "2026-04-20T17:06:00Z",
  "last_saved_by": "sting-user",
  "boq_health_score": 82,
  "manual_rows": [
    {
      "id": "GUID",
      "nrm2_section": "4",
      "category": "Structural Foundations",
      "discipline": "S",
      "item_name": "Foundation pad C30/37",
      "family_name": "—",
      "quantity": 8,
      "unit": "each",
      "rate_ugx": 2100000,
      "rate_usd": 568,
      "rate_confidence": 75,
      "rate_source": "CSV",
      "resolved_nrm2_paragraph": "Excavate, form and construct pad foundation at grid intersections...",
      "boq_line_ref": "4.1.2",
      "note": "Not yet modeled",
      "source": "Manual",
      "snapshot_ref": "",
      "level": "Ground",
      "location": "Grid A–C / 1–4",
      "last_costed": "2026-04-20T17:06:00Z",
      "embodied_carbon_kg": 4200.0,
      "lifecycle_cost_ugx": 3780000,
      "ps_match_found": false
    }
  ]
}

---

## PHASE 12 — Automation gap closure checklist

Implement every item below in the same implementation pass as the phases above.
Each is a small targeted change — none requires a new command file.

| # | Gap | File to change | Specific fix |
|---|-----|----------------|--------------|
| G1 | DeriveQuantity result not persisted | SchedulingCommands.cs → ElementCostTraceCommand.Execute() | After computing qty, write to CST_QTY_MEASURED before computing cost |
| G2 | Rate source not recorded per element | Same command | After rate lookup, write "CSV"/"COBie"/"Default" to CST_RATE_SOURCE |
| G3 | USD rates evaporate on model close | BOQCostManager.WriteElementParameters | Always write both CST_UNIT_RATE_UGX and CST_UNIT_RATE_USD in same transaction |
| G4 | Budget variance not in model | BOQCostManager.WriteProjectParameters | Write CST_BUDGET_VARIANCE_UGX to ProjectInfo after every BuildBOQDocument |
| G5 | BOQ Health score not surfaced | BIMCoordinationCenter.Build4D5DTab | Add BOQ Health KPI card reading CST_BOQ_COVERAGE_PCT |
| G6 | NRM2 paragraph blank in export | BOQCostManager.BuildBOQDocument step 5d | Resolve paragraph for every item before returning; fallback = "Supply and fix {category}." |
| G7 | Unresolved tokens in exported BOQ | BOQExportCommand pre-export pipeline | Run BuildResolvabilityReport; strip remaining [token] patterns from col C before writing cell |
| G8 | BOQLineRef not round-trip safe | BOQCostManager.WriteElementParameters | Only write ASS_BOQ_LINE_REF if current value is empty; never overwrite an existing ref unless explicitly renumbering |
| G9 | Carbon data missing from BOQ export | BOQCostManager.BuildBOQDocument step 5f | Compute EmbodiedCarbonKg = qty × carbon_factor; write to CST_EMBODIED_CARBON_KG |
| G10 | PS rows not in COBie | BOQCostManager.SaveManualRows | After saving, call SyncPSToCobie(doc) |
| G11 | Material rates not in schedules | BOQCostManager.WriteElementParameters | Write MAT_UNIT_RATE_UGX and MAT_MEASURED_AREA_M2 for elements with area-based costing |
| G12 | No rate confidence in export | BOQExportCommand Sheet 2 | Add "Rate confidence" column reading CST_RATE_CONFIDENCE; conditional format green/amber/red |
| G13 | BOQ stale after model change | BIMCoordinationCenter.BuildCoordData | After building coord data, compare category element counts against last snapshot; set flag for budget strip banner |
| G14 | Carbon tab missing from panel | BOQCostManagerPanel | Add Carbon tab as specified in Phase 5 |
| G15 | No Revit schedules from BOQ | StingCommandHandler | Add BOQCreateSchedule dispatch tag routing to BOQScheduleCommand |

---

## PHASE 13 — Files to create and files to modify

### New files (create these):

  src/Commands/BOQ/BOQModels.cs          (Phase 2 — data model)
  src/Commands/BOQ/BOQCostManager.cs     (Phase 3 — engine)
  src/Commands/BOQ/BOQRateEngine.cs      (Phase 11A — rate confidence)
  src/Commands/BOQ/BOQExportCommand.cs   (Phase 6 — Excel export)
  src/Commands/BOQ/BOQImportCommand.cs   (Phase 8 — roundtrip import)
  src/Commands/BOQ/BOQSnapshotCompareCommand.cs  (Phase 9)
  src/Commands/BOQ/ReconcileProvisionalsCommand.cs (Phase 10)
  src/Commands/BOQ/BOQScheduleCommand.cs (Phase 11G)
  src/Commands/BOQ/BOQCarbonExportCommand.cs (Phase 11 carbon)
  src/UI/BOQCostManagerPanel.cs          (Phase 5 — WPF panel)

### Files to modify (targeted changes only):

  data/MR_PARAMETERS.txt
    → Add 20+ new parameters (Phases 1, 11D)

  src/Core/ParamRegistry.cs
    → Register new GUIDs

  src/Core/SharedParamGuids.cs
    → Add BuildProjectInfoCategorySet(); add MAT_ params to BLE override

  src/Commands/SharedParams/LoadSharedParamsCommand.cs
    → Add ProjectInfo category set binding; add MAT_ material category override

  src/Temp/BOQTemplateLibrary.cs
    → Add ResolveForElement(); upgrade SelectBestTemplate(); add BuildResolvabilityReport()
    → Add BOQNarrativeEnhancer class

  src/Commands/Scheduling/SchedulingCommands.cs
    → ElementCostTraceCommand: write CST_QTY_MEASURED, CST_RATE_SOURCE, CST_UNIT_RATE_USD

  src/UI/BIMCoordinationCenter.cs
    → Add 4 new KPI cards; add _boqPanelArea ContentControl; add ShowBOQPanel() method
    → Add "BOQ Package" workflow preset
    → Add BOQ stale detection in BuildCoordData

  src/UI/StingCommandHandler.cs
    → Add 8 new dispatch tags in Execute() switch block

  src/UI/StingDockPanel.xaml
    → Add "★ BOQ Cost Manager" button in Schedules section

---

## IMPLEMENTATION ORDER

Claude Code should implement in this exact sequence to avoid circular dependencies:
  1. MR_PARAMETERS.txt additions + ParamRegistry/SharedParamGuids updates
  2. LoadSharedParamsCommand.cs binding additions
  3. BOQModels.cs (no dependencies)
  4. BOQCostManager.cs (depends on BOQModels, BOQTemplateLibrary, existing engines)
  5. BOQTemplateLibrary.cs enhancements (no new dependencies)
  6. BOQRateEngine.cs (depends on BOQModels, BOQCostManager)
  7. BOQExportCommand.cs (depends on BOQCostManager, BOQRateEngine)
  8. BOQImportCommand.cs (depends on BOQCostManager)
  9. BOQSnapshotCompareCommand.cs (depends on BOQCostManager)
  10. ReconcileProvisionalsCommand.cs (depends on BOQCostManager)
  11. BOQScheduleCommand.cs (depends on BOQCostManager)
  12. BOQCarbonExportCommand.cs (depends on BOQCostManager, CarbonTrackingEngine)
  13. BOQCostManagerPanel.cs (depends on all commands above)
  14. BIMCoordinationCenter.cs modifications (depends on BOQCostManagerPanel)
  15. StingCommandHandler.cs additions (depends on all commands)
  16. StingDockPanel.xaml button addition (no dependencies)
  17. SchedulingCommands.cs targeted fix (standalone)

After each file: compile and confirm zero errors before proceeding.
If a compile error references a missing type from a later phase,
add a stub/TODO and continue — resolve in that phase.

