# STING TOOLS — COMPLETE MR PARAMETER REPAIR + ALL DATA FOLDER ALIGNMENT
# Final Prompt for Claude Code | 2026-03-17
# All 12 "review" items resolved: ADD each as a new MR parameter.
# Covers every file in the data folder that references parameter names.

---

## DECISIONS FOR THE 12 PREVIOUSLY-FLAGGED PARAMS

All 12 are confirmed: **add as new parameters to MR_PARAMETERS.txt**.
Do NOT remap these to existing params — they are semantically distinct.

| New Param to Add | GROUP | DATATYPE | Rationale |
|---|---|---|---|
| `BLE_DOOR_TYPE_TXT` | 2 | TEXT | Door type (Solid/Glazed/Fire/Acoustic) ≠ operation type (Swing/Slide) |
| `BLE_ROOF_AREA_SQ_M` | 2 | AREA | No roof-specific area param exists; `BLE_FLR_AREA_SQ_M` is floor only |
| `BLE_ROOF_THICKNESS_MM` | 2 | LENGTH | Roof build-up thickness ≠ floor thickness |
| `BLE_WALL_CORE_MATERIAL_TXT` | 2 | TEXT | Core material (Concrete/Masonry/Steel) ≠ primary finish material |
| `BLE_WALL_INSULATION_TXT` | 2 | TEXT | Insulation type (PIR/Mineral Wool/XPS) ≠ moisture barrier |
| `BLE_WALL_WATERPROOF_TXT` | 2 | TEXT | Waterproofing system (Tanking/Membrane/etc.) ≠ moisture barrier bool |
| `ELC_CONN_TYPE_TXT` | 4 | TEXT | General connector type ≠ lightning protection connection type |
| `ELC_VOLTAGE_V` | 4 | NUMBER | Device-level voltage ≠ panel voltage (`ELC_PNL_VLT_V`) |
| `FAB_HANGER_LOAD_KN` | 16 | NUMBER | Numeric load data param; `WARN_FAB_HANGER_LOAD_KN` is TEXT warning only |
| `FAB_HANGER_TYPE_TXT` | 16 | TEXT | MEP fabrication hanger type ≠ HVAC duct hanger type |
| `HVC_DCT_SZ_TXT` | 5 | TEXT | Overall duct size designation ≠ terminal size (`HVC_DCT_TERMINAL_SZ_TXT`) |
| `INS_MATERIAL_TXT` | 5 | TEXT | Generic insulation material for both duct and pipe insulation categories |

---

## COMPLETE FILE LIST — WHAT GETS UPDATED

```
INPUT  (read from /mnt/project/ — project knowledge, read-only)
  MR_PARAMETERS.txt
  PARAMETER_REGISTRY.json
  LABEL_DEFINITIONS.json
  TAGGING_REFERENCE.md

INPUT  (read from /mnt/user-data/uploads/)
  STING_TAG_CONFIG_v5_0_COMBINED__1_.xlsx   ← source of truth for tag config names
  MR_PARAMETERS.txt                          ← same as project version, use for editing

OUTPUT (write to /mnt/user-data/outputs/)
  MR_PARAMETERS_v5_6.txt
  PARAMETER_REGISTRY_v5_6.json
  LABEL_DEFINITIONS_v5_6.json
  SCHEDULE_FIELD_REMAP_v5_6.csv
  STING_TAG_CONFIG_v5_0_ARCH_v5_6.csv
  STING_TAG_CONFIG_v5_0_MEP_v5_6.csv
  STING_TAG_CONFIG_v5_0_STR_v5_6.csv
  STING_TAG_CONFIG_v5_0_GEN_v5_6.csv
  TAGGING_REFERENCE_v5_6.md
  MR_CHANGE_LOG.md
```

**Note on source CSVs:** The four `STING_TAG_CONFIG_v5_0_*.csv` files exist in the
project data folder but are not in /mnt/project/. Claude Code must reconstruct their
content from the combined Excel file `/mnt/user-data/uploads/STING_TAG_CONFIG_v5_0_COMBINED__1_.xlsx`
by splitting sheets/sections per discipline. Use pandas to read the Excel.

---

## PHASE 1 — FIX MR_PARAMETERS.TXT: ALL DATATYPE ERRORS

Read `/mnt/project/MR_PARAMETERS.txt`. Find each param by exact NAME and change
only the DATATYPE field. Never alter GUIDs or parameter names.

### 1A — MEP Physical Type Fixes (cause "Inconsistent Units" in Revit tag labels)

| Parameter Name | Current → Correct | Reason |
|---|---|---|
| `HVC_DCT_FLW_CFM` | NUMBER → HVAC_AIRFLOW | Duct airflow — Revit needs HVAC_AIRFLOW |
| `HVC_DUCT_FLOWRATE_M3H` | VOLUME → HVAC_AIRFLOW | Same, incorrectly set to VOLUME |
| `HVC_VEL_MPS` | TEXT → NUMBER | Velocity stored as TEXT breaks tag formulas |
| `PLM_PPE_FLW_LPS` | VOLUME → FLOW | Pipe flow must be FLOW not VOLUME |
| `FLS_SFTY_FLW_RATE_LPM_NR` | NUMBER → FLOW | Sprinkler flow requires FLOW type |
| `ELC_CDT_SZ_MM` | TEXT → LENGTH | Conduit size in mm |
| `ELC_CTR_WIDTH_MM` | TEXT → LENGTH | Cable tray width in mm |
| `LTG_CLR_TEMP_K` | TEXT → NUMBER | Colour temperature in Kelvin |
| `WARN_INS_THICKNESS_MM` | TEXT → LENGTH | Insulation thickness |

> FLOW = pipe/drainage flow. HVAC_AIRFLOW = duct air flow. Distinct Revit types.

### 1B — Structural FORCE/MOMENT → NUMBER (tag label display compatibility)

| Parameter Name | Current → Correct |
|---|---|
| `STRUCT_COL_AXIAL_LOAD_KN` | FORCE → NUMBER |
| `STRUCT_FRM_AXIAL_LOAD_KN` | FORCE → NUMBER |
| `STRUCT_COL_MOMENT_KNM` | MOMENT → NUMBER |
| `STRUCT_FRM_MOMENT_KNM` | MOMENT → NUMBER |
| `STRUCT_COL_SHEAR_KN` | FORCE → NUMBER |
| `STRUCT_FRM_SHEAR_KN` | FORCE → NUMBER |
| `BLE_FLR_LD_CAP_KPA` | VOLUME → NUMBER |

### 1C — INTEGER → NUMBER (tag arithmetic)

| Parameter Name | Current → Correct |
|---|---|
| `ELC_CKT_PHASE_COUNT_NR` | INTEGER → NUMBER |
| `ELC_PNL_PHS_COUNT_NR` | INTEGER → NUMBER |
| `ASS_DESIGN_OCCUPANCY_INT` | INTEGER → NUMBER |

### 1D — Tag Label Type Corrections (fix mismatch between MR type and tag config usage)

| MR Param Name | Current → Correct | Reason |
|---|---|---|
| `BLE_CEILING_GRID_SZ_MM` | LENGTH → TEXT | Used as grid description in tags, not a dimension |
| `BLE_DOOR_GLAZING_PCT` | NUMBER → YESNO | Tag config references as boolean |
| `BLE_FLR_MOISTURE_BARRIER_REQUIRED_BOOL` | YESNO → TEXT | Tag config expects system name text |
| `BLE_FLR_SLIP_RESISTANCE_RATING_NR` | NUMBER → TEXT | Rating is text (R9, R11, etc.) |
| `BLE_RAMP_HANDRAIL_BOTH_SIDES_TXT` | TEXT → YESNO | Tag config uses as boolean |
| `PER_THERM_CONDENSATION_RISK_TXT` | TEXT → YESNO | Tag config uses as boolean |
| `PROP_FIRE_RATING` | TEXT → NUMBER | Fire rating in hours must be numeric |

> Keep as-is: `RGL_KCCA_CERT_NR`, `RGL_NEMA_CERT_NR`, `RGL_NWSC_CERT_NR`,
> `RGL_UMEME_CERT_NR` — cert refs may contain letters, remain TEXT.

---

## PHASE 2 — FIX MR_PARAMETERS.TXT: WARN_ PARAMS IN WRONG GROUP

Find each by NAME, change GROUP to 18, confirm VISIBLE=1:

| Parameter Name | Current GROUP → 18 |
|---|---|
| `WARN_BLE_RAMP_SLOPE_PCT_RAMPS` | 15 → 18 |
| `WARN_ELC_VLT_DROP_PCT_ELECTRICAL_EQUI` | 4 → 18 |
| `WARN_FLS_SFTY_COVERAGE_AREA_SQ_M_SPRINKLERS__FIR` | 8 → 18 |
| `WARN_HVC_DCT_SOUNDLVL_DB` | 5 → 18 |
| `WARN_HVC_EFF_RATIO_NR_MECHANICAL_EQUI` | 5 → 18 |
| `WARN_HVC_VEL_MPS_FLEX_DUCTS` | 5 → 18 |
| `WARN_PER_SUST_CARBON_FOOTPRINT_KG_WALLS__FLOORS__` | 9 → 18 |
| `WARN_PER_THERM_U_VALUE_W_M2K_NR_FLOORS` | 9 → 18 |
| `WARN_PER_THERM_U_VALUE_W_M2K_NR_ROOFS` | 9 → 18 |
| `WARN_PER_THERM_U_VALUE_W_M2K_NR_WALLS` | 9 → 18 |
| `WARN_PLM_PPE_FLW_LPS_PIPES__HOT_WATE` | 6 → 18 |
| `WARN_RGL_ACCESS_CLEAR_WIDTH_MM_DOORS__RAMPS__C` | 15 → 18 |

---

## PHASE 3 — ADD ALL NEW PARAMETERS TO MR_PARAMETERS.TXT

Add **73 parameters total**: 61 from the previous analysis + 12 resolved review items.
Format: `PARAM\t<UUID4>\t<NAME>\t<DATATYPE>\t\t<GROUP>\t<VISIBLE>\t<DESCRIPTION>\t<USERMODIFIABLE>`
Generate all GUIDs with `uuid.uuid4()`. All: VISIBLE=1, USERMODIFIABLE=1.
Insert alphabetically within each group section.

### The 12 Resolved Review Items (NEW — previously flagged for review):

```
GROUP 2 (CST_PROC):
BLE_DOOR_TYPE_TXT          TEXT    Door type classification (Solid/Glazed/Fire/Acoustic) [ISO 19650-1:2018]
BLE_ROOF_AREA_SQ_M         AREA    Roof plan area in sq.m [ISO 19650-1:2018]
BLE_ROOF_THICKNESS_MM      LENGTH  Roof construction build-up thickness in mm [ISO 19650-1:2018]
BLE_WALL_CORE_MATERIAL_TXT TEXT    Wall core structural material (Concrete/Masonry/Steel/Timber) [ISO 19650-1:2018]
BLE_WALL_INSULATION_TXT    TEXT    Wall insulation type and specification [ISO 19650-1:2018]
BLE_WALL_WATERPROOF_TXT    TEXT    Wall waterproofing system type [ISO 19650-1:2018]

GROUP 4 (ELC_PWR):
ELC_CONN_TYPE_TXT          TEXT    Electrical connector type (Plug/Terminal/Bus/Lug) [ISO 19650-1:2018]
ELC_VOLTAGE_V              NUMBER  Device operating voltage in V [ISO 19650-1:2018]

GROUP 5 (HVC_SYSTEMS):
HVC_DCT_SZ_TXT             TEXT    Duct size designation (e.g. 400x200, DN300) [ISO 19650-1:2018]
INS_MATERIAL_TXT           TEXT    Insulation material type (PIR/Mineral Wool/Foam/XPS) [ISO 19650-1:2018]

GROUP 16 (BLE_STRUCTURE):
FAB_HANGER_LOAD_KN         NUMBER  MEP fabrication hanger design load in kN [SMACNA/BS 7671]
FAB_HANGER_TYPE_TXT        TEXT    MEP fabrication hanger type and configuration [SMACNA]
```

### The 61 From Previous Analysis (all groups):

```
GROUP 1 (ASS_MNG):
ASS_TIEIN_BY_TXT           TEXT    Tie-in installed/connected by (contractor/trade) [ISO 19650-3:2020]
ASS_TIEIN_CONNECTED_BOOL   YESNO   Tie-in connection status flag [ISO 19650-3:2020]
ASS_TIEIN_ELEV_TXT         TEXT    Tie-in point elevation reference [ISO 19650-3:2020]
ASS_TIEIN_FLOW_DIR_TXT     TEXT    Tie-in flow direction (IN/OUT/BIDIRECTIONAL) [ISO 19650-3:2020]
ASS_TIEIN_IFC_REF_TXT      TEXT    Tie-in IFC GUID reference [ISO 19650-3:2020]
ASS_TIEIN_PHASE_TXT        TEXT    Tie-in construction phase [ISO 19650-3:2020]
ASS_TIEIN_STATUS_TXT       TEXT    Tie-in status: OPEN / CONNECTED / DEFERRED [ISO 19650-3:2020]
ASS_TIEIN_TAG_1_TXT        TEXT    Tie-in connected-to asset primary tag [ISO 19650-3:2020]

GROUP 2 (CST_PROC):
BLE_CASEWORK_ACCESSIBLE_BOOL  YESNO   Accessible casework compliance [ISO 19650-1:2018]
BLE_CASEWORK_COUNTERTOP_TXT   TEXT    Casework countertop material [ISO 19650-1:2018]
BLE_CASEWORK_HARDWARE_TXT     TEXT    Casework hardware specification [ISO 19650-1:2018]
BLE_CASEWORK_MATERIAL_TXT     TEXT    Casework primary material [ISO 19650-1:2018]
BLE_CASEWORK_TYPE_TXT         TEXT    Casework type classification [ISO 19650-1:2018]
BLE_DOOR_THRESHOLD_TXT        TEXT    Door threshold type and specification [ISO 19650-1:2018]
BLE_FURNITURE_FINISH_TXT      TEXT    Furniture surface finish specification [ISO 19650-1:2018]
BLE_HEADROOM_MM               LENGTH  General headroom clearance in mm [ISO 19650-1:2018]
BLE_MULLION_DEPTH_MM          LENGTH  Curtain wall mullion depth in mm [ISO 19650-1:2018]
BLE_MULLION_FINISH_TXT        TEXT    Curtain wall mullion finish [ISO 19650-1:2018]
BLE_MULLION_MATERIAL_TXT      TEXT    Curtain wall mullion material [ISO 19650-1:2018]
BLE_PANEL_GLASS_TYPE_TXT      TEXT    Curtain wall panel glass type [ISO 19650-1:2018]
BLE_PANEL_HEIGHT_MM           LENGTH  Curtain wall panel height in mm [ISO 19650-1:2018]
BLE_PANEL_WIDTH_MM            LENGTH  Curtain wall panel width in mm [ISO 19650-1:2018]
BLE_PARK_ACCESSIBLE_BOOL      YESNO   Parking bay meets accessibility standards [ISO 19650-1:2018]
BLE_PARK_BAY_LENGTH_MM        LENGTH  Parking bay length in mm [ISO 19650-1:2018]
BLE_PARK_BAY_WIDTH_MM         LENGTH  Parking bay width in mm [ISO 19650-1:2018]
BLE_PARK_EV_CHARGING_BOOL     YESNO   EV charging point provided [ISO 19650-1:2018]
BLE_PARK_TYPE_TXT             TEXT    Parking bay type classification [ISO 19650-1:2018]
BLE_RAILING_BALUSTER_SPACING_MM LENGTH Railing baluster spacing in mm [ISO 19650-1:2018]
BLE_RAILING_HEIGHT_MM         LENGTH  Railing height in mm [ISO 19650-1:2018]
BLE_RAILING_INFILL_TXT        TEXT    Railing infill type (Glass/Mesh/Baluster) [ISO 19650-1:2018]
BLE_RAILING_MATERIAL_TXT      TEXT    Railing primary material [ISO 19650-1:2018]
BLE_RAILING_TYPE_TXT          TEXT    Railing type classification [ISO 19650-1:2018]
BLE_RAMP_LANDING_TXT          TEXT    Ramp landing specification [ISO 19650-1:2018]
BLE_ROOM_FIRE_ESCAPE_CAPACITY_NR NUMBER Room fire escape capacity in persons [ISO 19650-1:2018]
BLE_ROOM_VENTILATION_RATE_LPS NUMBER  Room design ventilation rate in l/s [ISO 19650-1:2018]
BLE_SIGN_ILLUMINATED_BOOL     YESNO   Signage is internally illuminated [ISO 19650-1:2018]
BLE_SIGN_TYPE_TXT             TEXT    Signage type classification [ISO 19650-1:2018]
BLE_STAIR_HEADROOM_MM         LENGTH  Stair headroom clearance in mm [ISO 19650-1:2018]

GROUP 3 (COM_DAT):
GEN_HEIGHT_AT_MATURITY_M      LENGTH  Plant height at maturity in metres [ISO 19650-1:2018]
GEN_ROAD_TYPE_TXT             TEXT    Road type classification [ISO 19650-1:2018]
GEN_SPECIES_TXT               TEXT    Plant species name [ISO 19650-1:2018]
GEN_TAG_7_PARA_MISC_TXT       TEXT    Miscellaneous element paragraph container [ISO 19650-3:2020]

GROUP 4 (ELC_PWR):
ELC_CTR_SZ_TXT                TEXT    Cable tray size designation [ISO 19650-1:2018]
ELC_TAG_7_PARA_COMM_TXT       TEXT    Communication device electrical paragraph container [ISO 19650-3:2020]
ELC_TAG_7_PARA_TIEIN_TXT      TEXT    Electrical tie-in point paragraph container [ISO 19650-3:2020]
ELC_WIRE_SIZE_TXT             TEXT    Wire size designation (2.5mm², 4mm², 10mm²) [ISO 19650-1:2018]

GROUP 5 (HVC_SYSTEMS):
HVC_TAG_7_PARA_TIEIN_TXT      TEXT    HVAC tie-in point paragraph container [ISO 19650-3:2020]
MEC_CTRL_FUNCTION_TXT         TEXT    Mechanical control device function [ISO 19650-1:2018]
MEC_CTRL_SIGNAL_TXT           TEXT    Mechanical control signal type (Pneumatic/DDC/Electric) [ISO 19650-1:2018]
MEC_SET_CAPACITY_KW           NUMBER  Mechanical equipment set capacity in kW [ISO 19650-1:2018]

GROUP 6 (PLM_DRN):
PLM_EQP_CAPACITY_L            NUMBER  Plumbing equipment storage capacity in litres [ISO 19650-1:2018]
PLM_EQP_PRESSURE_KPA          NUMBER  Plumbing equipment design pressure in kPa [ISO 19650-1:2018]
PLM_TAG_7_PARA_TIEIN_TXT      TEXT    Plumbing tie-in point paragraph container [ISO 19650-3:2020]

GROUP 8 (FLS_LIFE_SFTY):
FLS_PROT_PRODUCT_TXT          TEXT    Fire protection product specification reference [ISO 19650-1:2018]
FLS_TAG_7_PARA_TIEIN_TXT      TEXT    Fire protection tie-in paragraph container [ISO 19650-3:2020]

GROUP 16 (BLE_STRUCTURE):
FAB_CONTAINMENT_D_MM          LENGTH  MEP fabrication containment depth in mm [SMACNA]
FAB_CONTAINMENT_W_MM          LENGTH  MEP fabrication containment width in mm [SMACNA]
FAB_DCT_H_MM                  LENGTH  MEP fabrication ductwork height in mm [SMACNA]
FAB_DCT_W_MM                  LENGTH  MEP fabrication ductwork width in mm [SMACNA]
FAB_PIPE_DN_MM                LENGTH  MEP fabrication pipework nominal diameter in mm [SMACNA]
FAB_STIFF_TYPE_TXT            TEXT    Ductwork stiffener type [SMACNA]
```

---

## PHASE 4 — UPDATE PARAMETER_REGISTRY.json

Read `/mnt/project/PARAMETER_REGISTRY.json`. Make these surgical updates only.
Preserve ALL existing content exactly — increment version and add new entries.

### 4A — Bump version
Change `"version": "5.5"` → `"version": "5.6"`

### 4B — Fix datatype entries in extended_params sections
In `extended_params.hvac`, find entries for `HVC_DCT_FLW_CFM` and `HVC_DUCT_FLOWRATE_M3H`
and update their descriptions to note HVAC_AIRFLOW type.

### 4C — Add new params to extended_params
Add the 73 new parameters to appropriate `extended_params` sub-sections:

- `extended_params.ble_dimensional` — add all new BLE_ dimensional params (LENGTH/AREA)
- `extended_params.hvac` — add `HVC_DCT_SZ_TXT`, `MEC_CTRL_*`, `MEC_SET_CAPACITY_KW`
- `extended_params.plumbing` — add `PLM_EQP_CAPACITY_L`, `PLM_EQP_PRESSURE_KPA`
- `extended_params.electrical` — add `ELC_CONN_TYPE_TXT`, `ELC_VOLTAGE_V`, `ELC_WIRE_SIZE_TXT`, `ELC_CTR_SZ_TXT`
- `extended_params.paragraph_containers` — add all `*_TAG_7_PARA_TIEIN_TXT` params with their category arrays
- Create new sub-section `extended_params.fabrication` — add all `FAB_*` params
- Create new sub-section `extended_params.tiein` — add all `ASS_TIEIN_*` params
- Create new sub-section `extended_params.building_elements` — add remaining BLE_ TEXT params

Entry format (match existing style exactly):
```json
{
  "key": "ROOF_AREA",
  "param_name": "BLE_ROOF_AREA_SQ_M",
  "guid": "<generated-uuid>",
  "description": "Roof plan area in sq.m",
  "categories": ["Roofs"]
}
```

### 4D — Update warning_thresholds
Add entries for the new warning params in Phase 3 that have thresholds, following the
existing format: `param_name`, `guid`, `description`, `threshold`, `unit`, `severity`, `param_type`, `binding`.

---

## PHASE 5 — UPDATE LABEL_DEFINITIONS.json

Read `/mnt/project/LABEL_DEFINITIONS.json`. This file drives
`TagConfig.LoadCategoryWarningsFromLabels()`. Make targeted additions only.

### 5A — Add new params to tier_2 / tier_3 of relevant categories

The new params belong in these category definitions (use `"param"` key format):

**Doors** — add to `tier_2`:
```json
{"param": "BLE_DOOR_TYPE_TXT", "spaces": 1, "break": false, "note": "Door type classification"}
```

**Roofs** — add to `tier_2`:
```json
{"param": "BLE_ROOF_AREA_SQ_M", "spaces": 1, "suffix": "m²", "break": false, "note": "Roof area"},
{"param": "BLE_ROOF_THICKNESS_MM", "spaces": 1, "suffix": "mm", "break": false, "note": "Roof build-up thickness"}
```

**Walls** — add to `tier_2`:
```json
{"param": "BLE_WALL_CORE_MATERIAL_TXT", "spaces": 1, "break": false, "note": "Wall core material"},
{"param": "BLE_WALL_INSULATION_TXT", "spaces": 1, "break": false, "note": "Wall insulation type"},
{"param": "BLE_WALL_WATERPROOF_TXT", "spaces": 1, "break": false, "note": "Wall waterproofing system"}
```

**Electrical Connectors** — add to `tier_2`:
```json
{"param": "ELC_CONN_TYPE_TXT", "spaces": 1, "break": false, "note": "Connector type"}
```

**Electrical Equipment** and **Electrical Fixtures** — add to `tier_2`:
```json
{"param": "ELC_VOLTAGE_V", "spaces": 1, "suffix": "V", "break": false, "note": "Device voltage"}
```

**Ducts** — add to `tier_2`:
```json
{"param": "HVC_DCT_SZ_TXT", "spaces": 1, "break": false, "note": "Duct size designation"}
```

**Duct Insulation** and **Pipe Insulation** — add to `tier_2`:
```json
{"param": "INS_MATERIAL_TXT", "spaces": 1, "break": false, "note": "Insulation material"}
```

**MEP Fabrication Hangers** — add to `tier_2`:
```json
{"param": "FAB_HANGER_TYPE_TXT", "spaces": 1, "break": false, "note": "Hanger type"},
{"param": "FAB_HANGER_LOAD_KN", "spaces": 1, "suffix": "kN", "break": false, "note": "Design load"}
```

### 5B — Fix abbreviated param names in warnings arrays
In every category's `"warnings"` array, apply the 66-entry name replacement table
(col 2 only — the `"param"` field values). Replace old abbreviated names with actual MR names:

```python
NAME_MAP = {
    "BLE_CEILING_MATERIAL_TXT": "BLE_CEILING_MAT_TXT",
    "BLE_CEILING_TYPE_TXT": "BLE_CEILING_TYPE_CLASSIFICATION_TXT",
    "BLE_DOOR_CLEAR_WIDTH_MM": "BLE_DOOR_WIDTH_MM",
    "BLE_DOOR_HARDWARE_TXT": "BLE_DOOR_HARDWARE_SPECIFICATION_TXT",
    "BLE_DOOR_MATERIAL_TXT": "BLE_DOOR_MAT_TXT",
    "BLE_DOOR_OPERATION_TXT": "BLE_DOOR_OPERATION_TYPE_TXT",
    "BLE_FINISH_EXT_TXT": "BLE_WALL_EXTERIOR_FINISH_TXT",
    "BLE_FINISH_TOP_TXT": "BLE_WALL_INTERIOR_FINISH_TXT",
    "BLE_FLOOR_AREA_SQ_M": "BLE_FLR_AREA_SQ_M",
    "BLE_FLOOR_CONSTRUCTION_METHOD_TXT": "BLE_FLR_STRUCTURAL_SYSTEM_TXT",
    "BLE_FLOOR_MATERIAL_TXT": "BLE_FLR_FINISH_MAT_TXT",
    "BLE_FLOOR_THICKNESS_MM": "BLE_FLR_THICKNESS_MM",
    "BLE_FLOOR_TYPE_TXT": "BLE_FLR_TYPE_TXT",
    "BLE_MATERIAL_TYPE_TXT": "BLE_MAT_ELEM_TYPE_TXT",
    "BLE_ROOF_DRAINAGE_TXT": "BLE_ROOF_PLM_DRN_SYSTEM_TYPE_TXT",
    "BLE_ROOF_MATERIAL_TXT": "BLE_ROOF_COVERING_MAT_TXT",
    "BLE_ROOF_WATERPROOF_TXT": "BLE_ROOF_WATERPROOFING_SYSTEM_TXT",
    "BLE_ROOM_AREA_SQ_M": "ASS_ROOM_AREA_SQ_M",
    "BLE_ROOM_CEILING_FINISH_TXT": "BLE_CEILING_FINISH_TXT",
    "BLE_ROOM_DEPARTMENT_TXT": "ASS_DEPARTMENT_ASSIGNMENT_TXT",
    "BLE_ROOM_FLOOR_FINISH_TXT": "BLE_FLR_FINISH_TXT",
    "BLE_ROOM_LIGHTING_LUX": "ASS_DESIGN_LTG_LVL_LUX_NR",
    "BLE_ROOM_WALL_FINISH_TXT": "BLE_WALL_FINISH_TXT",
    "BLE_STAIR_HANDRAIL_TXT": "BLE_STAIR_HANDRAIL_MAT_TXT",
    "BLE_STAIR_NOSING_TXT": "BLE_STAIR_NOSING_TYPE_TXT",
    "BLE_STAIR_RISER_MM": "BLE_STAIR_RISE_MM",
    "BLE_STAIR_TREAD_MATERIAL_TXT": "BLE_STAIR_TREAD_FINISH_TXT",
    "BLE_WALL_AREA_SQ_M": "BLE_ELE_AREA_SQ_M",
    "BLE_WALL_CONSTRUCTION_TXT": "BLE_WALL_CST_METHOD_TXT",
    "BLE_WALL_MATERIAL_TXT": "BLE_WALL_PRIMARY_WALL_MAT_TXT",
    "BLE_WALL_TYPE_TXT": "BLE_WALL_TYPE_CLASSIFICATION_TXT",
    "BLE_WIN_FRAME_MATERIAL_TXT": "BLE_WINDOW_FRAME_MAT_TXT",
    "BLE_WIN_GLAZING_TYPE_TXT": "BLE_WINDOW_GLAZING_TYPE_SINGLE_DOUBLE_TRIPLE_TXT",
    "BLE_WIN_HEIGHT_MM": "BLE_WINDOW_HEIGHT_MM",
    "BLE_WIN_OPENING_TYPE_TXT": "BLE_WINDOW_OPERATION_TYPE_TXT",
    "BLE_WIN_TYPE_TXT": "BLE_WINDOW_TYPE_CLASSIFICATION_TXT",
    "BLE_WIN_VENT_AREA_SQ_M": "BLE_WINDOW_AREA_SQ_M",
    "BLE_WIN_WIDTH_MM": "BLE_WINDOW_WIDTH_MM",
    "ELC_CDT_FILL_RATIO": "ELC_CDT_CBL_FILL_PCT",
    "ELC_CONN_RATING_A": "ELC_CKT_BRK_RATING_A",
    "ELC_TAG_7_PARA_CIRC_TXT": "ELC_TAG_7_PARA_CIRCUIT_TXT",
    "ELC_WIRE_TYPE_TXT": "ELC_CBL_INS_TYPE_TXT",
    "GEN_PROPERTY_AREA_SQM": "RGL_PLOT_AREA_SQ_M",
    "GEN_TAG_7_PARA_EQUIP_TXT": "ASS_TAG_7_PARA_EQUIPMENT_TXT",
    "GEN_TAG_7_PARA_SITE_TXT": "ARCH_TAG_7_PARA_SITE_TXT",
    "INS_THICKNESS_MM": "PLM_INS_THICKNESS_MM",
    "MAT_CLASS_TXT": "MAT_CATEGORY",
    "PER_ACOUSTIC_NRC_RATING": "BLE_CEILING_NOISE_REDCTION_COEFFICIENT_NRC_NR",
    "PER_ACOUSTIC_STC_RATING": "BLE_WALL_SOUND_TRANSMISSION_CLASS_RATING_NR",
    "PER_EMBODIED_CARBON_KG_CO2_M2": "PER_SUST_EMBODIED_CARBON_KG",
    "PER_SOLAR_G_VALUE": "BLE_WINDOW_SOLAR_HEAT_GAIN_COEFFICIENT_NR",
    "STR_GRADE_TXT": "BLE_STRUCT_STEEL_GRADE_TXT",
    "STR_REBAR_COVER_MM": "CST_S_REI_COVER_MM",
    "STR_REBAR_GRADE_TXT": "CST_S_REI_GRADE_TXT",
    "STR_REBAR_LAP_LENGTH_MM": "CST_S_REI_OVERLAP_LENGTH_MM",
    "STR_REBAR_SPACING_MM": "CST_S_REI_SPACING_MM",
    "STR_SLAB_THICKNESS_MM": "BLE_STRUCT_SLAB_THICKNESS_MM",
    "STR_WALL_THICKNESS_MM": "BLE_WALL_THICKNESS_MM",
    "STR_WEIGHT_KG": "CST_S_REI_WEIGHT_KG",
    "WARN_LTG_DESIGN_ILLUMINANCE_LUX": "WARN_LTG_ILLUMINANCE",
    "WARN_PLM_PPE_FLW_LPS_PIPES": "WARN_PLM_PPE_FLW_LPS_PIPES__HOT_WATE",
    "WARN_STR_BEAM_DEFLECTION_BEAMS": "WARN_STR_BEAM_DEFLECTION",
    "WARN_STR_BEAM_SPAN_DEPTH_BEAMS": "WARN_STR_BEAM_SPAN_DEPTH",
    "WARN_STR_REBAR_COVER_MM": "WARN_STR_RBR_COVER_MM",
    "WARN_STR_REBAR_LAP_LENGTH_MM": "WARN_STR_RBR_LAP_LENGTH_MM",
    "WARN_STR_SLAB_DEFLECTION": "WARN_STR_BEAM_DEFLECTION",
}
```

Apply to `"param"` values inside `"warnings"`, `"tier_2"`, `"tier_3"` arrays across
all 126 categories. Match by exact string equality. Do not touch JSON keys or
any other fields.

### 5C — Update paragraph_template strings
In categories where new params were added (Doors, Roofs, Walls, Ducts, etc.),
append a brief reference to the new params in the `paragraph_template` text
where contextually appropriate. Use a conservative approach — only add if the
existing template already references similar params from the same domain.

---

## PHASE 6 — UPDATE SCHEDULE_FIELD_REMAP.csv

Read the existing file from uploads. Preserve all existing rows.
Append the 66 name remap entries under comment header `# v5.6 2026-03-17`.
Check for duplicates before appending — skip any already present.

The 66 entries to append are the NAME_MAP from Phase 5B, formatted as:
```
Old_Name,New_Name,REMAPPED
```

---

## PHASE 7 — UPDATE STING_TAG_CONFIG_v5_0_*.csv FILES

**Source:** Read the combined Excel `STING_TAG_CONFIG_v5_0_COMBINED__1_.xlsx` using
pandas. Split into four discipline files based on the tag family categories:
- ARCH: Architectural categories (Walls, Doors, Floors, Ceilings, Roofs, Rooms, etc.)
- MEP: MEP categories (Air Terminals, Ducts, Pipes, Electrical, etc.)
- STR: Structural categories (Structural Framing, Columns, Foundations, Rebar, etc.)
- GEN: All remaining (Generic Models, Site, Toposolid, RVT Links, etc.)

**For each output CSV:**
1. Preserve the exact row structure from the Excel (tag family headers, tier rows,
   warning parameter rows, section markers)
2. Apply the NAME_MAP from Phase 5B to column index 2 (Parameter Name column)
   wherever a cell value exactly matches an old name — replace with new MR name
3. Add new params for the 73 new additions: where a tag family's category now has
   new data params (e.g. Roofs gets `BLE_ROOF_AREA_SQ_M` and `BLE_ROOF_THICKNESS_MM`),
   insert new T2 or T3 rows following the existing row format of that family
4. The new WARN_ params added via the 73 new params in Phase 3 should appear in the
   WARNING PARAMETERS section of their relevant tag families

---

## PHASE 8 — UPDATE TAGGING_REFERENCE.md

Read `/mnt/project/TAGGING_REFERENCE.md`. Add a new section at the end:

```markdown
---

## 11. v5.6 Parameter Changes (2026-03-17)

### Parameters Added to MR_PARAMETERS.txt
**Total: 73 new parameters** (1,311 → 1,384)

| Group | Count | Key Additions |
|---|---|---|
| GROUP 1 ASS_MNG | 8 | ASS_TIEIN_* tie-in point feature set |
| GROUP 2 CST_PROC | 42 | Railings, parking, casework, curtain wall, door type, roof area/thickness, wall insulation/waterproof/core |
| GROUP 3 COM_DAT | 4 | GEN site/planting params |
| GROUP 4 ELC_PWR | 6 | Wire size, connector type, device voltage, tie-in paragraphs |
| GROUP 5 HVC_SYSTEMS | 6 | Duct size, MEC controls, insulation material, tie-in |
| GROUP 6 PLM_DRN | 3 | Equipment capacity/pressure, tie-in |
| GROUP 8 FLS_LIFE_SFTY | 2 | Fire protection product, tie-in |
| GROUP 16 BLE_STRUCTURE | 8 | FAB containment/duct/pipe dimensions, hanger load/type |

### Datatype Fixes Applied
28 parameters corrected (see MR_CHANGE_LOG.md for full list)

### Files Updated
- MR_PARAMETERS.txt → v5.6
- PARAMETER_REGISTRY.json → v5.6
- LABEL_DEFINITIONS.json → param names corrected, new params added
- SCHEDULE_FIELD_REMAP.csv → 66 name remap entries added
- STING_TAG_CONFIG_v5_0_*.csv → param names corrected throughout
```

---

## PHASE 9 — VALIDATION

Run this script and fix all errors before writing final output files:

```python
import uuid, json

errors = []

# ── MR_PARAMETERS_v5_6.txt ──
mr_params = {}
with open('/home/claude/MR_PARAMETERS_v5_6.txt', 'r') as f:
    lines = f.readlines()

for i, raw in enumerate(lines, 1):
    line = raw.rstrip()
    if not line.startswith('PARAM\t'):
        continue
    parts = line.split('\t')
    if len(parts) < 9:
        errors.append(f"MR line {i}: wrong column count ({len(parts)})")
        continue
    guid, name, dtype, _, grp, vis = parts[1], parts[2], parts[3], parts[4], parts[5], parts[6]

    try: uuid.UUID(guid)
    except: errors.append(f"MR line {i}: invalid GUID for '{name}'")

    if guid in mr_params:
        errors.append(f"MR line {i}: DUPLICATE GUID '{guid}' ('{name}' vs '{mr_params[guid]}')")
    mr_params[guid] = name

    if name in [v for v in mr_params.values() if v != name]:
        errors.append(f"MR line {i}: DUPLICATE NAME '{name}'")

    if name.startswith('WARN_'):
        if grp != '18': errors.append(f"MR: WARN_ '{name}' in GROUP {grp} (must be 18)")
        if dtype != 'TEXT': errors.append(f"MR: WARN_ '{name}' is {dtype} (must be TEXT)")
        if vis != '1': errors.append(f"MR: WARN_ '{name}' VISIBLE={vis} (must be 1)")

    if name.endswith('_BOOL') and dtype != 'YESNO':
        errors.append(f"MR: BOOL param '{name}' is {dtype} (must be YESNO)")

    if name.endswith('_MM') and not name.startswith('WARN_') and dtype not in ('LENGTH','TEXT'):
        errors.append(f"MR: _MM param '{name}' is {dtype} (expected LENGTH)")

# Check all Phase 1 fixes
phase1 = {
    'HVC_DCT_FLW_CFM': 'HVAC_AIRFLOW', 'HVC_DUCT_FLOWRATE_M3H': 'HVAC_AIRFLOW',
    'HVC_VEL_MPS': 'NUMBER', 'PLM_PPE_FLW_LPS': 'FLOW',
    'FLS_SFTY_FLW_RATE_LPM_NR': 'FLOW', 'ELC_CDT_SZ_MM': 'LENGTH',
    'ELC_CTR_WIDTH_MM': 'LENGTH', 'LTG_CLR_TEMP_K': 'NUMBER',
}
name_to_line = {}
for i, raw in enumerate(lines, 1):
    if raw.startswith('PARAM\t'):
        parts = raw.split('\t')
        if len(parts) >= 4:
            name_to_line[parts[2].strip()] = (i, parts[3].strip())

for pname, expected in phase1.items():
    if pname not in name_to_line:
        errors.append(f"Phase 1: '{pname}' NOT FOUND in MR")
    elif name_to_line[pname][1] != expected:
        errors.append(f"Phase 1: '{pname}' still {name_to_line[pname][1]}, expected {expected}")

# Check all 73 new params were added
new_params = [
    'ASS_TIEIN_BY_TXT','ASS_TIEIN_CONNECTED_BOOL','ASS_TIEIN_ELEV_TXT',
    'ASS_TIEIN_FLOW_DIR_TXT','ASS_TIEIN_IFC_REF_TXT','ASS_TIEIN_PHASE_TXT',
    'ASS_TIEIN_STATUS_TXT','ASS_TIEIN_TAG_1_TXT',
    'BLE_DOOR_TYPE_TXT','BLE_ROOF_AREA_SQ_M','BLE_ROOF_THICKNESS_MM',
    'BLE_WALL_CORE_MATERIAL_TXT','BLE_WALL_INSULATION_TXT','BLE_WALL_WATERPROOF_TXT',
    'ELC_CONN_TYPE_TXT','ELC_VOLTAGE_V','HVC_DCT_SZ_TXT','INS_MATERIAL_TXT',
    'FAB_HANGER_LOAD_KN','FAB_HANGER_TYPE_TXT',
    'FAB_CONTAINMENT_D_MM','FAB_CONTAINMENT_W_MM','FAB_DCT_H_MM','FAB_DCT_W_MM',
    'FAB_PIPE_DN_MM','FAB_STIFF_TYPE_TXT',
    'BLE_RAILING_HEIGHT_MM','BLE_RAILING_MATERIAL_TXT','BLE_RAILING_TYPE_TXT',
    'BLE_PARK_BAY_WIDTH_MM','BLE_PARK_BAY_LENGTH_MM',
    'MEC_CTRL_FUNCTION_TXT','MEC_CTRL_SIGNAL_TXT','MEC_SET_CAPACITY_KW',
    'PLM_EQP_CAPACITY_L','PLM_EQP_PRESSURE_KPA',
    'HVC_TAG_7_PARA_TIEIN_TXT','ELC_TAG_7_PARA_TIEIN_TXT',
    'PLM_TAG_7_PARA_TIEIN_TXT','FLS_TAG_7_PARA_TIEIN_TXT',
]
for p in new_params:
    if p not in name_to_line:
        errors.append(f"Phase 3: '{p}' NOT ADDED to MR")

# ── PARAMETER_REGISTRY_v5_6.json ──
with open('/home/claude/PARAMETER_REGISTRY_v5_6.json') as f:
    reg = json.load(f)
if reg.get('version') != '5.6':
    errors.append(f"REGISTRY: version is {reg.get('version')}, expected 5.6")
if 'fabrication' not in reg.get('extended_params', {}):
    errors.append("REGISTRY: missing extended_params.fabrication section")
if 'tiein' not in reg.get('extended_params', {}):
    errors.append("REGISTRY: missing extended_params.tiein section")

# ── Report ──
total_mr_params = len([l for l in lines if l.startswith('PARAM\t')])
if errors:
    print(f"VALIDATION FAILED — {len(errors)} errors:")
    for e in errors: print(f"  ❌ {e}")
else:
    print(f"✅ VALIDATION PASSED")
    print(f"   MR parameters: {total_mr_params} (was 1,311)")
    print(f"   All datatype checks passed")
    print(f"   All WARN_ params in GROUP 18")
    print(f"   All 73 new params present")
    print(f"   PARAMETER_REGISTRY version: 5.6")
```

---

## PHASE 10 — WRITE MR_CHANGE_LOG.md

```markdown
# MR_PARAMETERS Change Log — v5.6
**Date:** 2026-03-17

## Statistics
| Metric | Before | After |
|---|---|---|
| Total parameters | 1,311 | 1,384 |
| Parameters added | — | +73 |
| Datatype fixes | — | 28 |
| Group/visibility fixes | — | 12 |

## Datatype Fixes (28)
[List all Phase 1A/1B/1C/1D changes with old→new]

## Group Fixes (12)
[List all Phase 2 changes — WARN_ params moved to GROUP 18]

## New Parameters (73)
[List all Phase 3 params grouped by discipline group]

## Name Alignment (66 remaps — no new params, existing files updated)
These params exist in MR under different names.
The following files were updated to use the correct MR names:
- SCHEDULE_FIELD_REMAP.csv (66 entries appended)
- STING_TAG_CONFIG_v5_0_*.csv (param names corrected in column 2)
- LABEL_DEFINITIONS.json (param names corrected in warnings/tier arrays)

[Full 66-entry mapping table]
```

---

## CRITICAL CONSTRAINTS

1. **Never change GUIDs** — breaks all Revit project bindings irreversibly
2. **Never rename existing params** — breaks `LookupParameter()` calls in C# code
3. **Tab-delimited only** for MR_PARAMETERS.txt — use `\t`, never spaces as separators
4. **Preserve file headers** — META and GROUP lines in MR file unchanged
5. **FLOW ≠ HVAC_AIRFLOW** — different Revit internal types, cannot be swapped
6. **Column 2 only** in tag config CSVs — only the Parameter Name column, not formulas
7. **Exact string match** for name replacements — case-sensitive, whole-cell only
8. **Run Phase 9 validation** before writing any output — fix all errors first
9. **All GUIDs unique** — run duplicate check across all 1,384 params in final output
10. **Be conservative with LABEL_DEFINITIONS** — only add to `tier_2`, `tier_3`, and
    `warnings` arrays; never restructure or remove existing entries
