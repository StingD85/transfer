# STINGTOOLS — COMPLETE CLAUDE CODE IMPLEMENTATION PROMPT
# Version: FINAL (Sessions 1–12 consolidated)
# Date: 2026-03-17
# Purpose: Apply ALL confirmed gaps and features to /mnt/project/ files
#
# HOW TO USE THIS PROMPT:
# 1. Feed this entire document to Claude Code
# 2. Claude Code reads each section, locates the exact lines, applies the fix
# 3. After every section, Claude Code appends results to the COMPLETION REPORT at the end
# 4. Do not stop between fixes — apply all fixes sequentially without pausing
#
# WORKING DIRECTORY: /mnt/project/ (all file paths are relative to this)
#
# OPERATING RULES (apply to every fix):
# - Read the target file before editing — never guess line numbers
# - Use str_replace with EXACT text from the file (copy verbatim)
# - After each fix, verify with grep that the change is present
# - If a str_replace fails, read the file again and retry with exact content
# - Never add using directives that already exist in the file
# - Never duplicate class or method names
# - Record actual line numbers applied in the completion report


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 1 — TAGGING PIPELINE: COMPLETE EXCEL LINK IMPORT (CRITICAL)
# File: ExcelLinkCommands.cs
# Status: LAST remaining pipeline gap — 2 missing steps in both paths
# ══════════════════════════════════════════════════════════════════════

## CONFIRMED STATE (verified from file):
# - NativeParamMapper.MapAll ✅  - Formulas ✅  - BuildAndWriteTag ✅
# - WriteContainers ✅  - WriteTag7All ✅  - SaveSeqSidecar ✅
# - TypeTokenInherit ❌ MISSING  - GetGridRef result DISCARDED ❌

## FIX 1A — Path 1: Add TypeTokenInherit before NativeParamMapper

**Locate in ExcelLinkCommands.cs** (search for this exact text):
```
                                    string catName = ParameterHelpers.GetCategoryName(el);
                                    // FIX-10: Bridge native params before tag rebuild
                                    try { NativeParamMapper.MapAll(doc, el); }
```

**Replace with:**
```
                                    string catName = ParameterHelpers.GetCategoryName(el);
                                    // FIX-1A: Inherit family-type token defaults before rebuild
                                    try { TokenAutoPopulator.TypeTokenInherit(doc, el); }
                                    catch (Exception tiEx) { StingLog.Warn($"ExcelLink TypeTokenInherit {el.Id}: {tiEx.Message}"); }
                                    // FIX-10: Bridge native params before tag rebuild
                                    try { NativeParamMapper.MapAll(doc, el); }
```

## FIX 1B — Path 1: Capture and write GridRef result

**Locate in ExcelLinkCommands.cs** (search for this exact text):
```
                                    if (elGridLines != null && elGridLines.Count > 0)
                                    {
                                        try { SpatialAutoDetect.GetGridRef(el, elGridLines); }
                                        catch (Exception grEx) { StingLog.Warn($"ExcelLink GridRef for {el.Id}: {grEx.Message}"); }
                                    }
```

**Replace with:**
```
                                    if (elGridLines != null && elGridLines.Count > 0)
                                    {
                                        try
                                        {
                                            string gridRef = SpatialAutoDetect.GetGridRef(el, elGridLines);
                                            if (!string.IsNullOrEmpty(gridRef))
                                                ParameterHelpers.SetIfEmpty(el, ParamRegistry.GRID_REF, gridRef);
                                        }
                                        catch (Exception grEx) { StingLog.Warn($"ExcelLink GridRef for {el.Id}: {grEx.Message}"); }
                                    }
```

## FIX 1C — Path 2 (RoundTrip): Add TypeTokenInherit

**Locate in ExcelLinkCommands.cs** (search for this exact text):
```
                                    string catName = ParameterHelpers.GetCategoryName(el);
                                    // Phase2: Bridge native params before tag rebuild
                                    try { NativeParamMapper.MapAll(doc, el); }
```

**Replace with:**
```
                                    string catName = ParameterHelpers.GetCategoryName(el);
                                    // FIX-1C: Inherit family-type token defaults before round-trip rebuild
                                    try { TokenAutoPopulator.TypeTokenInherit(doc, el); }
                                    catch (Exception tiEx) { StingLog.Warn($"ExcelLink RT TypeTokenInherit {el.Id}: {tiEx.Message}"); }
                                    // Phase2: Bridge native params before tag rebuild
                                    try { NativeParamMapper.MapAll(doc, el); }
```

## FIX 1D — Path 2 (RoundTrip): Capture and write GridRef result

**Locate in ExcelLinkCommands.cs** (search for this exact text):
```
                                    if (rtGridLines != null && rtGridLines.Count > 0)
                                    {
                                        try { SpatialAutoDetect.GetGridRef(el, rtGridLines); }
                                        catch (Exception grEx) { StingLog.Warn($"ExcelLink RoundTrip GridRef for {el.Id}: {grEx.Message}"); }
                                    }
```

**Replace with:**
```
                                    if (rtGridLines != null && rtGridLines.Count > 0)
                                    {
                                        try
                                        {
                                            string gridRef = SpatialAutoDetect.GetGridRef(el, rtGridLines);
                                            if (!string.IsNullOrEmpty(gridRef))
                                                ParameterHelpers.SetIfEmpty(el, ParamRegistry.GRID_REF, gridRef);
                                        }
                                        catch (Exception grEx) { StingLog.Warn($"ExcelLink RT GridRef for {el.Id}: {grEx.Message}"); }
                                    }
```

**Verify:** `grep -n "TypeTokenInherit\|string gridRef" ExcelLinkCommands.cs` → must show exactly 4 results.


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 2 — THEME SYSTEM: MAKE THEME SWITCHING ACTUALLY WORK
# Files: ThemeManager.cs, StingDockPanel.xaml, StingDockPanel_xaml.cs
# Root cause: All XAML styles use hardcoded hex colors — zero DynamicResource
# bindings exist. ThemeManager writes to Application.Current.Resources but
# nothing reads it. StaticResource is resolved once at load; DynamicResource
# reacts to runtime changes.
# ══════════════════════════════════════════════════════════════════════

## FIX 2A — ThemeManager.cs: Add InitialiseResources() method

**Locate in ThemeManager.cs** (search for this exact text):
```
        /// <summary>Cycle to the next theme in order: Dark -> Light -> Grey -> Corporate.</summary>
        public static string CycleTheme()
```

**Insert immediately before it:**
```csharp
        /// <summary>
        /// FIX-2A: Seed all theme resource keys into Application.Resources at startup.
        /// Must be called before any DynamicResource binding resolves (i.e. in constructor).
        /// Without this, DynamicResource keys don't exist at first render and bindings silently fail.
        /// </summary>
        public static void InitialiseResources()
        {
            var app = Application.Current;
            if (app == null) return;
            if (!Themes.ContainsKey(CurrentTheme)) CurrentTheme = "Light";
            foreach (var kvp in Themes[CurrentTheme])
            {
                try
                {
                    var color = (Color)ColorConverter.ConvertFromString(kvp.Value);
                    app.Resources[kvp.Key] = new SolidColorBrush(color);
                }
                catch { }
            }
            StingLog.Info($"ThemeManager: resources initialised with '{CurrentTheme}' theme");
        }

```

## FIX 2B — StingDockPanel_xaml.cs: Call InitialiseResources in constructor

**Locate in StingDockPanel_xaml.cs** (search for this exact text):
```
            InitializeComponent();
```

**Replace with:**
```csharp
            InitializeComponent();
            // FIX-2B: Seed theme resource keys before DynamicResource bindings resolve
            ThemeManager.InitialiseResources();
```

## FIX 2C — StingDockPanel.xaml: Replace Page.Resources with DynamicResource styles

**Locate in StingDockPanel.xaml** (search for this exact block — lines 7–88):
```xml
    <Page.Resources>
        <Style x:Key="SectionHeader" TargetType="TextBlock">
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="FontSize" Value="11"/>
            <Setter Property="Margin" Value="0,6,0,3"/>
            <Setter Property="Foreground" Value="#6A1B9A"/>
        </Style>
        <Style x:Key="GroupBorder" TargetType="Border">
            <Setter Property="BorderBrush" Value="#DDD"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="CornerRadius" Value="4"/>
            <Setter Property="Padding" Value="6"/>
            <Setter Property="Margin" Value="0,3"/>
            <Setter Property="Background" Value="White"/>
        </Style>
        <Style x:Key="CatBtn" TargetType="Button">
            <Setter Property="MinWidth" Value="34"/>
            <Setter Property="Height" Value="24"/>
            <Setter Property="Margin" Value="1"/>
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="Padding" Value="4,2"/>
            <Setter Property="Background" Value="#F0F0F0"/>
            <Setter Property="BorderBrush" Value="#CCC"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="Cursor" Value="Hand"/>
        </Style>
        <Style x:Key="ActionBtn" TargetType="Button">
            <Setter Property="MinWidth" Value="46"/>
            <Setter Property="Height" Value="26"/>
            <Setter Property="Margin" Value="1"/>
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="Padding" Value="5,2"/>
            <Setter Property="Cursor" Value="Hand"/>
        </Style>
        <Style x:Key="GreenBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#4CAF50"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#388E3C"/>
        </Style>
        <Style x:Key="OrangeBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#FF9800"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#F57C00"/>
        </Style>
        <Style x:Key="RedBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#F44336"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#D32F2F"/>
        </Style>
        <Style x:Key="BlueBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#2196F3"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#1976D2"/>
        </Style>
        <Style x:Key="PurpleBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#9C27B0"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#7B1FA2"/>
        </Style>
        <Style x:Key="TealBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#009688"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#00796B"/>
        </Style>
        <Style x:Key="SectionLabel" TargetType="TextBlock">
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="Foreground" Value="#6A1B9A"/>
            <Setter Property="Margin" Value="0,5,0,2"/>
            <Setter Property="FontWeight" Value="Bold"/>
        </Style>
        <Style x:Key="SubLabel" TargetType="TextBlock">
            <Setter Property="FontSize" Value="9"/>
            <Setter Property="Foreground" Value="#888"/>
            <Setter Property="Margin" Value="0,3,0,1"/>
        </Style>
    </Page.Resources>
```

**Replace with:**
```xml
    <Page.Resources>
        <!-- FIX-2C: DynamicResource bindings so ThemeManager.ApplyTheme() updates the UI -->
        <Style x:Key="SectionHeader" TargetType="TextBlock">
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="FontSize" Value="11"/>
            <Setter Property="Margin" Value="0,6,0,3"/>
            <Setter Property="Foreground" Value="{DynamicResource AccentBrush}"/>
        </Style>
        <Style x:Key="GroupBorder" TargetType="Border">
            <Setter Property="BorderBrush" Value="{DynamicResource BorderColor}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="CornerRadius" Value="4"/>
            <Setter Property="Padding" Value="6"/>
            <Setter Property="Margin" Value="0,3"/>
            <Setter Property="Background" Value="{DynamicResource SecondaryBg}"/>
        </Style>
        <Style x:Key="CatBtn" TargetType="Button">
            <Setter Property="MinWidth" Value="34"/>
            <Setter Property="Height" Value="24"/>
            <Setter Property="Margin" Value="1"/>
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="Padding" Value="4,2"/>
            <Setter Property="Background" Value="{DynamicResource ButtonBg}"/>
            <Setter Property="Foreground" Value="{DynamicResource ButtonFg}"/>
            <Setter Property="BorderBrush" Value="{DynamicResource BorderColor}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="Cursor" Value="Hand"/>
        </Style>
        <Style x:Key="ActionBtn" TargetType="Button">
            <Setter Property="MinWidth" Value="46"/>
            <Setter Property="Height" Value="26"/>
            <Setter Property="Margin" Value="1"/>
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="Padding" Value="5,2"/>
            <Setter Property="Background" Value="{DynamicResource ButtonBg}"/>
            <Setter Property="Foreground" Value="{DynamicResource ButtonFg}"/>
            <Setter Property="BorderBrush" Value="{DynamicResource BorderColor}"/>
            <Setter Property="Cursor" Value="Hand"/>
        </Style>
        <Style x:Key="GreenBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#4CAF50"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#388E3C"/>
        </Style>
        <Style x:Key="OrangeBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#FF9800"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#F57C00"/>
        </Style>
        <Style x:Key="RedBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#F44336"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#D32F2F"/>
        </Style>
        <Style x:Key="BlueBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#2196F3"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#1976D2"/>
        </Style>
        <Style x:Key="PurpleBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#9C27B0"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#7B1FA2"/>
        </Style>
        <Style x:Key="TealBtn" TargetType="Button" BasedOn="{StaticResource ActionBtn}">
            <Setter Property="Background" Value="#009688"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="BorderBrush" Value="#00796B"/>
        </Style>
        <Style x:Key="SectionLabel" TargetType="TextBlock">
            <Setter Property="FontSize" Value="10"/>
            <Setter Property="Foreground" Value="{DynamicResource AccentBrush}"/>
            <Setter Property="Margin" Value="0,5,0,2"/>
            <Setter Property="FontWeight" Value="Bold"/>
        </Style>
        <Style x:Key="SubLabel" TargetType="TextBlock">
            <Setter Property="FontSize" Value="9"/>
            <Setter Property="Foreground" Value="{DynamicResource PanelFg}"/>
            <Setter Property="Margin" Value="0,3,0,1"/>
            <Setter Property="Opacity" Value="0.7"/>
        </Style>
    </Page.Resources>
```

## FIX 2D — StingDockPanel.xaml: Update hardcoded header colors

**Locate** (exact text): `<Page x:Class="StingTools.UI.StingDockPanel"`
On the same element, find `Background="#FAFAFA"` and replace with `Background="{DynamicResource PrimaryBg}"`.

**Locate** (exact text): `<Border Grid.Row="0" Background="#6A1B9A" Padding="8,5">`
**Replace with:** `<Border Grid.Row="0" Background="{DynamicResource HeaderBg}" Padding="8,5">`

**Locate** (exact text): `<TextBlock x:Name="txtStatus" Text="Always on top: ON" FontSize="9" Foreground="#CE93D8"/>`
**Replace with:** `<TextBlock x:Name="txtStatus" Text="Always on top: ON" FontSize="9" Foreground="{DynamicResource HeaderFg}" Opacity="0.8"/>`

**Locate** (exact text): `<TextBlock Grid.Column="1" x:Name="txtSelCount" Text="sel 0" FontSize="9" Foreground="#CE93D8" VerticalAlignment="Center" Margin="0,0,6,0"/>`
**Replace with:** `<TextBlock Grid.Column="1" x:Name="txtSelCount" Text="sel 0" FontSize="9" Foreground="{DynamicResource HeaderFg}" Opacity="0.8" VerticalAlignment="Center" Margin="0,0,6,0"/>`

**Locate** (exact text): `FontSize="9" Background="#7B1FA2" Foreground="White" BorderThickness="0"
                        ToolTip="Pin panel" Click="BtnPin_Click" Margin="0,0,2,0"/>`
**Replace** `Background="#7B1FA2"` with `Background="{DynamicResource AccentBrush}"` on that line.

**Locate** (exact text): `FontSize="11" Background="#7B1FA2" Foreground="White" BorderThickness="0"
                        ToolTip="Toggle theme (Dark/Light/Grey/Corporate)" Tag="CycleTheme" Click="Cmd_Click"/>`
**Replace** `Background="#7B1FA2"` with `Background="{DynamicResource AccentBrush}"` on that line.

**Verify:** `grep -c "DynamicResource" StingDockPanel.xaml` → must be ≥ 20.


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 3 — CONNECT LEADER & ELBOW SLIDERS TO COMMANDS
# Files: StingDockPanel_xaml.cs, SmartTagPlacementCommand.cs
# Root cause: ~15 sliders (sldElbowX/Y/Dist, sldLeaderLen, sldArrowSize etc)
# are purely decorative — no code reads them. AdjustElbowsCommand shows
# its own dialog, completely ignoring all slider values.
# ══════════════════════════════════════════════════════════════════════

## FIX 3A — StingDockPanel_xaml.cs: Add SetLeaderElbowParams() and SetTagStyleParams() helpers

**Locate** in StingDockPanel_xaml.cs (search for this exact text):
```
        /// <summary>UI-05: Read scope radio state and pass to commands.</summary>
        private void SetPlacementScopeParams()
```

**Insert immediately before it:**
```csharp
        /// <summary>
        /// FIX-3A: Read Leader & Elbow tab slider/radio values and pass them as
        /// ExtraParams so AdjustElbowsCommand uses them directly, skipping its dialog.
        /// </summary>
        private void SetLeaderElbowParams()
        {
            try
            {
                // Elbow type mode (0=straight, 1=90°, 2=45°, 3=free)
                string elbowMode = "0";
                if (rbElbow90?.IsChecked == true)   elbowMode = "1";
                else if (rbElbow45?.IsChecked == true)  elbowMode = "2";
                else if (rbElbowFree?.IsChecked == true) elbowMode = "3";
                StingCommandHandler.SetExtraParam("ElbowMode", elbowMode);

                // Elbow position offsets (mm)
                double ex = sldElbowX?.Value ?? 0;
                double ey = sldElbowY?.Value ?? -16;
                double ed = sldElbowDist?.Value ?? 8;
                StingCommandHandler.SetExtraParam("ElbowX",    ex.ToString("F1"));
                StingCommandHandler.SetExtraParam("ElbowY",    ey.ToString("F1"));
                StingCommandHandler.SetExtraParam("ElbowDist", ed.ToString("F1"));

                // Leader mode
                string lMode = "Auto";
                if (rbLeaderAlways?.IsChecked == true) lMode = "Always";
                else if (rbLeaderNever?.IsChecked == true)  lMode = "Never";
                else if (rbLeaderSmart?.IsChecked == true)  lMode = "Smart";
                StingCommandHandler.SetExtraParam("LeaderMode", lMode);

                // Leader length params (mm)
                StingCommandHandler.SetExtraParam("LeaderLen",       (sldLeaderLen?.Value       ?? 14).ToString("F0"));
                StingCommandHandler.SetExtraParam("LeaderMin",       (sldLeaderMin?.Value       ?? 5 ).ToString("F0"));
                StingCommandHandler.SetExtraParam("LeaderMax",       (sldLeaderMax?.Value       ?? 43).ToString("F0"));
                StingCommandHandler.SetExtraParam("LeaderThreshold", (sldLeaderThreshold?.Value ?? 20).ToString("F0"));

                // Arrowhead
                string arrowStyle = (cmbArrowStyle?.SelectedItem as System.Windows.Controls.ComboBoxItem)
                    ?.Content?.ToString() ?? "None";
                string arrowColor = (cmbArrowColor?.SelectedItem as System.Windows.Controls.ComboBoxItem)
                    ?.Content?.ToString() ?? "Match scheme";
                StingCommandHandler.SetExtraParam("ArrowStyle", arrowStyle);
                StingCommandHandler.SetExtraParam("ArrowColor", arrowColor);
                StingCommandHandler.SetExtraParam("ArrowSize",  (sldArrowSize?.Value ?? 4).ToString("F0"));
            }
            catch { /* Controls may not be initialised yet */ }
        }

        /// <summary>
        /// FIX-3A: Read Style & Color tab slider/radio values for tag style commands.
        /// </summary>
        private void SetTagStyleParams()
        {
            try
            {
                StingCommandHandler.SetExtraParam("TagTextSize",
                    (sldTextSize?.Value ?? 2.5).ToString("F2"));
                StingCommandHandler.SetExtraParam("TagLetterSpacing",
                    (sldLetterSpacing?.Value ?? 0).ToString("F0"));

                string weight = "Normal";
                if (rbWeightBold?.IsChecked == true)   weight = "Bold";
                else if (rbWeightItalic?.IsChecked == true) weight = "Italic";
                StingCommandHandler.SetExtraParam("TagTextWeight", weight);

                string color = "Black";
                if (rbColorRed?.IsChecked == true)   color = "Red";
                else if (rbColorBlue?.IsChecked == true) color = "Blue";
                else if (rbColorWhite?.IsChecked == true) color = "White";
                StingCommandHandler.SetExtraParam("TagTextColor", color);
            }
            catch { }
        }

```

## FIX 3B — StingDockPanel_xaml.cs: Call helpers in Cmd_Click

**Locate in StingDockPanel_xaml.cs** (search for this exact text):
```
                // UI-08: Pass preferred compass position to SmartPlace
                if (cmdTag.Contains("SmartPlace") || cmdTag.Contains("Place"))
                {
                    SetPreferredPositionParam();
                }

                _handler?.SetCommand(cmdTag);
```

**Replace with:**
```csharp
                // UI-08: Pass preferred compass position to SmartPlace
                if (cmdTag.Contains("SmartPlace") || cmdTag.Contains("Place"))
                {
                    SetPreferredPositionParam();
                }

                // FIX-3B: Pass Leader & Elbow slider values to elbow/arrow commands
                if (cmdTag == "TagStudio_AdjustElbows" || cmdTag == "TagStudio_SetArrows"
                    || cmdTag == "SnapElbow90" || cmdTag == "SnapElbow45"
                    || cmdTag == "SnapElbowStraight" || cmdTag == "SnapElbowFree")
                {
                    SetLeaderElbowParams();
                }

                // FIX-3B: Pass Style & Color slider values to style commands
                if (cmdTag == "TagStudio_ApplyStyle" || cmdTag == "ApplyTagStyle"
                    || cmdTag == "BatchTagTextSize")
                {
                    SetTagStyleParams();
                }

                _handler?.SetCommand(cmdTag);
```

## FIX 3C — SmartTagPlacementCommand.cs: Use ExtraParams in AdjustElbowsCommand

**Locate in SmartTagPlacementCommand.cs** (search for this exact block):
```csharp
            // Choose elbow style
            TaskDialog dlg = new TaskDialog("STING — Adjust Elbows");
            dlg.MainInstruction = "Select leader elbow style";
            dlg.MainContent = "Adjusts elbows on all tags with leaders in the active view.";
            dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Straight", "Elbow at midpoint of element-to-tag vector");
            dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "90 degrees", "Horizontal-first: elbow at (tagX, elemY)");
            dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "45 degrees", "Angled: elbow offset to create 45-degree bend");
            dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                "Free", "No elbow adjustment — set elbow at leader end");
            dlg.CommonButtons = TaskDialogCommonButtons.Cancel;

            int mode;
            switch (dlg.Show())
            {
                case TaskDialogResult.CommandLink1: mode = 0; break;
                case TaskDialogResult.CommandLink2: mode = 1; break;
                case TaskDialogResult.CommandLink3: mode = 2; break;
                case TaskDialogResult.CommandLink4: mode = 3; break;
                default: return Result.Cancelled;
            }
```

**Replace with:**
```csharp
            // FIX-3C: Use Leader & Elbow panel slider values when set, else show dialog
            string _extraElbowMode = UI.StingCommandHandler.GetExtraParam("ElbowMode");
            int mode;
            if (!string.IsNullOrEmpty(_extraElbowMode) && int.TryParse(_extraElbowMode, out int _parsedMode))
            {
                mode = _parsedMode; // 0=straight, 1=90°, 2=45°, 3=free — from panel radio
                StingLog.Info($"AdjustElbows: panel mode={mode} (skipping dialog)");
                UI.StingCommandHandler.ClearExtraParam("ElbowMode");
            }
            else
            {
                TaskDialog dlg = new TaskDialog("STING — Adjust Elbows");
                dlg.MainInstruction = "Select leader elbow style";
                dlg.MainContent = "Tip: Set radio buttons in the Leader & Elbow panel then click " +
                    "'Adjust elbows' to skip this dialog.";
                dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                    "Straight", "Elbow at midpoint of element-to-tag vector");
                dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                    "90 degrees", "Horizontal-first: elbow at (tagX, elemY)");
                dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                    "45 degrees", "Angled: elbow offset to create 45-degree bend");
                dlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                    "Free", "No elbow adjustment — set elbow at leader end");
                dlg.CommonButtons = TaskDialogCommonButtons.Cancel;
                switch (dlg.Show())
                {
                    case TaskDialogResult.CommandLink1: mode = 0; break;
                    case TaskDialogResult.CommandLink2: mode = 1; break;
                    case TaskDialogResult.CommandLink3: mode = 2; break;
                    case TaskDialogResult.CommandLink4: mode = 3; break;
                    default: return Result.Cancelled;
                }
            }
            // Read optional XYZ offsets from sliders (convert mm → feet)
            double _elbowXFt = 0, _elbowYFt = 0;
            if (double.TryParse(UI.StingCommandHandler.GetExtraParam("ElbowX"),   out double _ex)) _elbowXFt = _ex / 304.8;
            if (double.TryParse(UI.StingCommandHandler.GetExtraParam("ElbowY"),   out double _ey)) _elbowYFt = _ey / 304.8;
            UI.StingCommandHandler.ClearExtraParam("ElbowX");
            UI.StingCommandHandler.ClearExtraParam("ElbowY");
            UI.StingCommandHandler.ClearExtraParam("ElbowDist");
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 4 — PER-EXPORT FOLDER NAVIGATION
# Files: OutputLocationHelper.cs, OperationsCommands.cs,
#        TagOperationCommands.cs
# Root cause: PDFExportCommand, IFCExportCommand, COBieExportCommand
# all hardcode Desktop as output. TagRegisterExport uses OutputLocation
# but gives no folder choice. Users cannot navigate per-export.
# ══════════════════════════════════════════════════════════════════════

## FIX 4A — OutputLocationHelper.cs: Add PromptForExportPath + session memory

**Locate in OutputLocationHelper.cs** (search for this exact text):
```
        private static bool TryEnsureDirectory(string path)
```

**Insert immediately before it:**
```csharp
        // FIX-4A: Session-level folder memory per export type (resets on Revit restart)
        private static readonly System.Collections.Concurrent.ConcurrentDictionary<string, string>
            _sessionFolders = new System.Collections.Concurrent.ConcurrentDictionary<string, string>(
                StringComparer.OrdinalIgnoreCase);

        /// <summary>
        /// FIX-4A: Per-export folder navigation with session memory.
        /// If this export type was used before in this session, offers "use last folder"
        /// as a quick option. Always updates session memory and PreferredDirectory on choice.
        /// Returns null if user cancels.
        /// </summary>
        public static string PromptForExportPath(Document doc, string defaultFileName,
            string filter, string exportTypeKey = null)
        {
            // Check session memory for this export type
            _sessionFolders.TryGetValue(exportTypeKey ?? "", out string lastFolder);

            if (!string.IsNullOrEmpty(lastFolder) && Directory.Exists(lastFolder))
            {
                var quickDlg = new Autodesk.Revit.UI.TaskDialog($"Export — {defaultFileName}");
                quickDlg.MainInstruction = "Choose export location";
                quickDlg.MainContent = $"Last used: {lastFolder}";
                quickDlg.AddCommandLink(Autodesk.Revit.UI.TaskDialogCommandLinkId.CommandLink1,
                    "Use last folder", lastFolder);
                quickDlg.AddCommandLink(Autodesk.Revit.UI.TaskDialogCommandLinkId.CommandLink2,
                    "Navigate to different folder", "Open file browser");
                string projDir2 = Path.GetDirectoryName(doc?.PathName ?? "");
                quickDlg.AddCommandLink(Autodesk.Revit.UI.TaskDialogCommandLinkId.CommandLink3,
                    "Use project folder",
                    string.IsNullOrEmpty(projDir2) ? "Save project first to enable" : projDir2);
                quickDlg.CommonButtons = Autodesk.Revit.UI.TaskDialogCommonButtons.Cancel;

                switch (quickDlg.Show())
                {
                    case Autodesk.Revit.UI.TaskDialogResult.CommandLink1:
                        return Path.Combine(lastFolder, defaultFileName);
                    case Autodesk.Revit.UI.TaskDialogResult.CommandLink3:
                        return string.IsNullOrEmpty(projDir2) ? null
                            : Path.Combine(projDir2, defaultFileName);
                    case Autodesk.Revit.UI.TaskDialogResult.CommandLink2:
                        break; // fall through to SaveFileDialog
                    default:
                        return null;
                }
            }

            var dlg = new Microsoft.Win32.SaveFileDialog
            {
                Title = $"Export — {defaultFileName}",
                FileName = defaultFileName,
                Filter = string.IsNullOrEmpty(filter) ? "All Files|*.*" : filter,
                InitialDirectory = GetOutputDirectory(doc)
            };
            if (dlg.ShowDialog() != true) return null;

            string chosenDir = Path.GetDirectoryName(dlg.FileName);
            if (!string.IsNullOrEmpty(chosenDir))
            {
                PreferredDirectory = chosenDir;
                if (!string.IsNullOrEmpty(exportTypeKey))
                    _sessionFolders[exportTypeKey] = chosenDir;
            }
            return dlg.FileName;
        }

```

## FIX 4B — OperationsCommands.cs: Replace hardcoded Desktop in PDFExportCommand

**Locate in OperationsCommands.cs** (search for this exact text):
```csharp
                string outputDir = Path.Combine(
                    Environment.GetFolderPath(Environment.SpecialFolder.Desktop),
                    $"STING_PDF_{DateTime.Now:yyyyMMdd}");
                Directory.CreateDirectory(outputDir);
```

**Replace with:**
```csharp
                string _pdfPrompt = OutputLocationHelper.PromptForExportPath(
                    doc, $"STING_PDF_{DateTime.Now:yyyyMMdd}.pdf",
                    "PDF Files|*.pdf|All Files|*.*", "PDF");
                if (_pdfPrompt == null) return Result.Cancelled;
                string outputDir = Path.GetDirectoryName(_pdfPrompt)
                    ?? OutputLocationHelper.GetOutputDirectory(doc);
                Directory.CreateDirectory(outputDir);
```

## FIX 4C — OperationsCommands.cs: Replace hardcoded Desktop in IFCExportCommand

**Locate in OperationsCommands.cs** (search for this exact text):
```csharp
                string outputDir = Path.Combine(
                    Environment.GetFolderPath(Environment.SpecialFolder.Desktop),
                    $"STING_IFC_{DateTime.Now:yyyyMMdd}");
                Directory.CreateDirectory(outputDir);
```

**Replace with:**
```csharp
                string _ifcPrompt = OutputLocationHelper.PromptForExportPath(
                    doc, $"STING_IFC_{DateTime.Now:yyyyMMdd}.ifc",
                    "IFC Files|*.ifc|All Files|*.*", "IFC");
                if (_ifcPrompt == null) return Result.Cancelled;
                string outputDir = Path.GetDirectoryName(_ifcPrompt)
                    ?? OutputLocationHelper.GetOutputDirectory(doc);
                Directory.CreateDirectory(outputDir);
```

## FIX 4D — OperationsCommands.cs: Replace hardcoded Desktop in COBieExportCommand

**Locate in OperationsCommands.cs** (search for this exact text):
```csharp
                string outputPath = Path.Combine(
                    Environment.GetFolderPath(Environment.SpecialFolder.Desktop),
                    $"STING_COBie_{DateTime.Now:yyyyMMdd_HHmmss}.xlsx");
```

**Replace with:**
```csharp
                string outputPath = OutputLocationHelper.PromptForExportPath(
                    doc, $"STING_COBie_{DateTime.Now:yyyyMMdd_HHmmss}.xlsx",
                    "Excel Files|*.xlsx|All Files|*.*", "COBie")
                    ?? OutputLocationHelper.GetTimestampedPath(doc, "STING_COBie", ".xlsx");
```

## FIX 4E — TagOperationCommands.cs: Add folder prompt to TagRegisterExport

**Locate in TagOperationCommands.cs** (search for this exact text):
```csharp
            string path = OutputLocationHelper.GetTimestampedPath(doc, "STING_Tag_Register", ".csv");
```

**Replace with:**
```csharp
            string path = OutputLocationHelper.PromptForExportPath(
                doc, $"STING_Tag_Register_{DateTime.Now:yyyyMMdd}.csv",
                "CSV Files|*.csv|All Files|*.*", "TagRegister")
                ?? OutputLocationHelper.GetTimestampedPath(doc, "STING_Tag_Register", ".csv");
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 5 — FIX 24 CRASHING COMMANDS (NULL COMMANDDATA)
# Files: RevisionManagementCommands.cs, IoTMaintenanceCommands.cs
# Root cause: All commands use commandData.Application.ActiveUIDocument directly.
# When called from StingCommandHandler.RunCommand<T>(), commandData IS null.
# Fix: Replace all occurrences with GetContext pattern.
# ══════════════════════════════════════════════════════════════════════

## FIX 5A — RevisionManagementCommands.cs: Fix all 14 crash lines

For each of the following, find the EXACT text and replace it.
There are two patterns — one that needs only doc, one that needs both doc and uidoc.

### Pattern A — doc only (12 occurrences). Find each and replace:

**Find:** `var doc = commandData.Application.ActiveUIDocument.Document;`

**Replace with:**
```csharp
var ctx = ParameterHelpers.GetContext(commandData);
if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
var doc = ctx.Doc;
```

Apply this replacement to ALL 12 occurrences in RevisionManagementCommands.cs at lines:
429, 514, 603, 714, 864, 940, 1046, 1129, 1328, 1465

### Pattern B — doc AND uidoc (2 occurrences).

**Occurrence 1** (around line 776 in TrackElementRevisionsCommand). Find:
```csharp
                var doc = commandData.Application.ActiveUIDocument.Document;
                var uidoc = commandData.Application.ActiveUIDocument;
                string revDir = RevisionEngine.GetRevisionDir(doc);
```
**Replace with:**
```csharp
                var ctx = ParameterHelpers.GetContext(commandData);
                if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
                var doc = ctx.Doc;
                var uidoc = ctx.UIDoc;
                string revDir = RevisionEngine.GetRevisionDir(doc);
```

**Occurrence 2** (around line 1257 in BulkRevisionStampCommand). Find:
```csharp
                var doc = commandData.Application.ActiveUIDocument.Document;
                var uidoc = commandData.Application.ActiveUIDocument;
                var selIds = uidoc.Selection.GetElementIds();
```
**Replace with:**
```csharp
                var ctx = ParameterHelpers.GetContext(commandData);
                if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
                var doc = ctx.Doc;
                var uidoc = ctx.UIDoc;
                var selIds = uidoc.Selection.GetElementIds();
```

**Verify:** `grep -c "commandData.Application.ActiveUIDocument" RevisionManagementCommands.cs` → must be 0.

## FIX 5B — IoTMaintenanceCommands.cs: Fix all 10 crash lines

### Pattern A — uidoc + doc from uidoc (1 occurrence, AssetConditionCommand line 30).
Find:
```csharp
                var uidoc = commandData.Application.ActiveUIDocument;
                var doc = uidoc.Document;
```
**Replace with:**
```csharp
                var ctx = ParameterHelpers.GetContext(commandData);
                if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
                var uidoc = ctx.UIDoc;
                var doc = ctx.Doc;
```

### Pattern B — doc only (9 occurrences at lines 138, 220, 323, 372, 442, 492, 556, 618, 700).
For each, find: `var doc = commandData.Application.ActiveUIDocument.Document;`
**Replace with:**
```csharp
var ctx = ParameterHelpers.GetContext(commandData);
if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
var doc = ctx.Doc;
```

**Verify:** `grep -c "commandData.Application.ActiveUIDocument" IoTMaintenanceCommands.cs` → must be 0.

## FIX 5C — DataPipelineEnhancementCommands.cs: Fix 1 crash line

**Locate** (search for this exact text in DataPipelineEnhancementCommands.cs):
```csharp
            var doc = commandData.Application.ActiveUIDocument.Document;
```
(This appears in ValidateFamilyBindingsCommand around line 299.)

**Replace with:**
```csharp
            var ctx = ParameterHelpers.GetContext(commandData);
            if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
            var doc = ctx.Doc;
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 6 — WIRE IoT MAINTENANCE COMMANDS TO DISPATCH + XAML
# Files: StingCommandHandler.cs, StingDockPanel.xaml
# Root cause: 10 IoT commands fully implemented but zero dispatch entries
# and zero XAML buttons — completely unreachable.
# ══════════════════════════════════════════════════════════════════════

## FIX 6A — StingCommandHandler.cs: Add IoT dispatch entries

**Locate in StingCommandHandler.cs** (search for this exact text):
```csharp
                    case "SetOutputDirectory": RunCommand<BIMManager.SetOutputDirectoryCommand>(app); break;
```

**Insert immediately after it:**
```csharp
                    // FIX-6A: IoT Maintenance commands (were dead code — no dispatch)
                    case "AssetCondition":        RunCommand<Temp.AssetConditionCommand>(app); break;
                    case "MaintenanceSchedule":   RunCommand<Temp.MaintenanceScheduleCommand>(app); break;
                    case "DigitalTwinExport":     RunCommand<Temp.DigitalTwinExportCommand>(app); break;
                    case "EnergyAnalysis":        RunCommand<Temp.EnergyAnalysisCommand>(app); break;
                    case "CommissioningChecklist":RunCommand<Temp.CommissioningChecklistCommand>(app); break;
                    case "SpaceManagement":       RunCommand<Temp.SpaceManagementCommand>(app); break;
                    case "LifecycleCost":         RunCommand<Temp.LifecycleCostCommand>(app); break;
                    case "WarrantyTracker":       RunCommand<Temp.WarrantyTrackerCommand>(app); break;
                    case "HandoverPackage":       RunCommand<Temp.HandoverPackageCommand>(app); break;
                    case "SensorPointMapper":     RunCommand<Temp.SensorPointMapperCommand>(app); break;
```

## FIX 6B — StingCommandHandler.cs: Add DataPipelineEnhancement dispatch entries

**Locate in StingCommandHandler.cs** (search for this exact text):
```csharp
                    // FIX-6A: IoT Maintenance commands (were dead code — no dispatch)
```

**Insert immediately before it:**
```csharp
                    // FIX-6B: DataPipelineEnhancement commands (were dead code — no dispatch)
                    case "CrossValidateRegistry":    RunCommand<Temp.CrossValidateRegistryCommand>(app); break;
                    case "ValidateBindingMatrix":    RunCommand<Temp.ValidateBindingMatrixCommand>(app); break;
                    case "ViewParameterMetadata":    RunCommand<Temp.ViewParameterMetadataCommand>(app); break;
                    case "ValidateFamilyBindings":   RunCommand<Temp.ValidateFamilyBindingsCommand>(app); break;
                    case "DataIntegrityCheck":       RunCommand<Temp.DataIntegrityCheckCommand>(app); break;
                    case "DataReport":               RunCommand<Temp.DataReportCommand>(app); break;
                    case "ExportUnifiedRegistry":    RunCommand<Temp.ExportUnifiedRegistryCommand>(app); break;
```

## FIX 6C — StingDockPanel.xaml: Add IoT Maintenance section to BIM tab

**Locate in StingDockPanel.xaml** (search for this exact text):
```xml
                        <!-- CDE and DOCUMENT CONTROL -->
                        <TextBlock Style="{StaticResource SectionLabel}" Text="CDE — DOCUMENT CONTROL"/>
```

**Insert immediately before it:**
```xml
                        <!-- IoT MAINTENANCE & ASSET HEALTH -->
                        <TextBlock Style="{StaticResource SectionLabel}" Text="IoT — ASSET MAINTENANCE &amp; HEALTH"/>
                        <Border Style="{StaticResource GroupBorder}">
                            <StackPanel>
                                <TextBlock Text="Asset condition, maintenance scheduling, digital twin, sensor mapping" FontSize="9" Foreground="#888" Margin="0,0,0,4"/>
                                <TextBlock Style="{StaticResource SubLabel}" Text="ASSET HEALTH"/>
                                <WrapPanel>
                                    <Button Style="{StaticResource GreenBtn}" Content="Asset Condition" Tag="AssetCondition" Click="Cmd_Click"
                                            ToolTip="Score all equipment on condition (1–5) based on age, maintenance history, warranty status"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Warranty Tracker" Tag="WarrantyTracker" Click="Cmd_Click"
                                            ToolTip="List assets with warranty expiry dates; flag expired/expiring-soon"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Lifecycle Cost" Tag="LifecycleCost" Click="Cmd_Click"
                                            ToolTip="Calculate whole-life cost for all assets using category cost rates"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Energy Analysis" Tag="EnergyAnalysis" Click="Cmd_Click"
                                            ToolTip="Estimate annual energy consumption for mechanical and electrical equipment"/>
                                </WrapPanel>
                                <TextBlock Style="{StaticResource SubLabel}" Text="MAINTENANCE &amp; OPERATIONS"/>
                                <WrapPanel>
                                    <Button Style="{StaticResource BlueBtn}" Content="Maint Schedule" Tag="MaintenanceSchedule" Click="Cmd_Click"
                                            ToolTip="Generate O&amp;M maintenance schedule with intervals and tasks per asset category"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Commissioning" Tag="CommissioningChecklist" Click="Cmd_Click"
                                            ToolTip="Generate commissioning checklist for all mechanical/electrical assets"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Space Mgmt" Tag="SpaceManagement" Click="Cmd_Click"
                                            ToolTip="Room/space usage report: area, occupancy, classification"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Handover Pkg" Tag="HandoverPackage" Click="Cmd_Click"
                                            ToolTip="Generate ISO 19650 handover package: asset register, O&amp;M, warranties"/>
                                </WrapPanel>
                                <TextBlock Style="{StaticResource SubLabel}" Text="DIGITAL TWIN &amp; IoT"/>
                                <WrapPanel>
                                    <Button Style="{StaticResource TealBtn}" Content="Digital Twin" Tag="DigitalTwinExport" Click="Cmd_Click"
                                            ToolTip="Export digital twin dataset: asset data, spatial coordinates, system connections"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Sensor Map" Tag="SensorPointMapper" Click="Cmd_Click"
                                            ToolTip="Map IoT sensor points to BIM elements by proximity and category"/>
                                </WrapPanel>
                            </StackPanel>
                        </Border>

                        <!-- DATA PIPELINE VALIDATION -->
                        <TextBlock Style="{StaticResource SectionLabel}" Text="DATA PIPELINE — VALIDATION &amp; REGISTRY"/>
                        <Border Style="{StaticResource GroupBorder}">
                            <WrapPanel>
                                <Button Style="{StaticResource ActionBtn}" Content="Cross-Validate" Tag="CrossValidateRegistry" Click="Cmd_Click"
                                        ToolTip="Cross-validate PARAMETER_REGISTRY.json vs CATEGORY_BINDINGS.csv vs bound params"/>
                                <Button Style="{StaticResource ActionBtn}" Content="Binding Matrix" Tag="ValidateBindingMatrix" Click="Cmd_Click"
                                        ToolTip="Check all shared param bindings are correct category/type combinations"/>
                                <Button Style="{StaticResource ActionBtn}" Content="Param Metadata" Tag="ViewParameterMetadata" Click="Cmd_Click"
                                        ToolTip="View full metadata for all STING parameters: GUID, group, datatype, categories"/>
                                <Button Style="{StaticResource ActionBtn}" Content="Family Bindings" Tag="ValidateFamilyBindings" Click="Cmd_Click"
                                        ToolTip="Check all loaded families have required STING parameters"/>
                                <Button Style="{StaticResource ActionBtn}" Content="Data Integrity" Tag="DataIntegrityCheck" Click="Cmd_Click"
                                        ToolTip="Full data integrity scan: missing params, invalid GUIDs, duplicate registrations"/>
                                <Button Style="{StaticResource ActionBtn}" Content="Data Report" Tag="DataReport" Click="Cmd_Click"
                                        ToolTip="Generate comprehensive parameter data report"/>
                                <Button Style="{StaticResource ActionBtn}" Content="Export Registry" Tag="ExportUnifiedRegistry" Click="Cmd_Click"
                                        ToolTip="Export unified parameter registry as Excel/CSV for documentation"/>
                            </WrapPanel>
                        </Border>

```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 7 — FIX NLP COMMAND PROCESSOR: MAKE IT ACTUALLY EXECUTE COMMANDS
# File: NLPCommandProcessor.cs
# Root cause: NLPCommandProcessorCommand.Execute() calls ProcessQuery() never —
# it just shows a static dialog. ProcessQuery() works correctly and returns
# CommandTag strings. The fix: collect user input via StingListPicker,
# run ProcessQuery, show top match, execute via WorkflowEngine.RunCommandByTag.
# ══════════════════════════════════════════════════════════════════════

## FIX 7A — NLPCommandProcessor.cs: Replace stub Execute with functional implementation

**Locate in NLPCommandProcessor.cs** (search for this exact text):
```csharp
        public Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
        {
            try
            {
                // Prompt for natural language input
                var inputDlg = new TaskDialog("STING Natural Language Command");
                inputDlg.MainInstruction = "Type what you want to do:";
                inputDlg.MainContent = "Examples:\n• \"Tag all elements in the project\"\n• \"Check ISO 19650 compliance\"\n• \"Export to IFC\"\n• \"Color elements by discipline\"\n• \"Create maintenance schedule\"";
                inputDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1, "Enter Command", "Type a natural language command");
                inputDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2, "Browse Commands", "See all available commands");
                inputDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3, "BIM Knowledge Base", "Search BIM terminology");

                var choice = inputDlg.Show();

                if (choice == TaskDialogResult.CommandLink2)
                {
                    // Show all commands
                    var allCommands = NLPEngine.IntentPatterns
                        .Select(p => (p.CommandTag, p.Description))
                        .Distinct()
                        .OrderBy(c => c.Description)
                        .ToList();

                    var report = new System.Text.StringBuilder();
                    report.AppendLine("═══ ALL STING COMMANDS ═══\n");
                    var grouped = allCommands.GroupBy(c =>
                    {
                        if (c.Description.Contains("tag") || c.Description.Contains("Tag")) return "Tagging";
                        if (c.Description.Contains("Select")) return "Selection";
                        if (c.Description.Contains("export") || c.Description.Contains("Export")) return "Export";
                        if (c.Description.Contains("Create") || c.Description.Contains("create")) return "Creation";
                        if (c.Description.Contains("compliance") || c.Description.Contains("Compliance") || c.Description.Contains("check")) return "Standards";
                        return "Other";
                    });

                    foreach (var group in grouped.OrderBy(g => g.Key))
                    {
                        report.AppendLine($"── {group.Key} ──");
                        foreach (var (tag, desc) in group)
                            report.AppendLine($"  [{tag}] {desc}");
                    }

                    TaskDialog.Show("STING Commands", report.ToString());
                    return Result.Succeeded;
                }

                if (choice == TaskDialogResult.CommandLink3)
                {
                    // Knowledge base
                    var report = new System.Text.StringBuilder();
                    report.AppendLine("═══ BIM KNOWLEDGE BASE ═══\n");
                    foreach (var (term, def) in NLPEngine.BimKnowledge.OrderBy(k => k.Key))
                        report.AppendLine($"  {term}: {def}\n");

                    TaskDialog.Show("BIM Knowledge Base", report.ToString());
                    return Result.Succeeded;
                }

                // For CommandLink1 - show intent processing info
                var resultDlg = new TaskDialog("NLP Command");
                resultDlg.MainInstruction = "Natural Language Processing";
                resultDlg.MainContent = "The NLP engine recognises 90+ intent patterns across tagging, selection, export, standards, MEP, and facility management domains.\n\nUse the STING dockable panel buttons or type command tags directly.";
                resultDlg.Show();

                StingLog.Info("NLP command processor accessed");
                return Result.Succeeded;
            }
            catch (Exception ex)
            {
                StingLog.Error("NLP command failed", ex);
                message = ex.Message;
                return Result.Failed;
            }
        }
```

**Replace the entire method body with:**
```csharp
        public Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
        {
            try
            {
                // FIX-7A: NLP now actually executes matched commands
                var ctx = ParameterHelpers.GetContext(commandData);
                if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }

                // Mode selection
                var inputDlg = new TaskDialog("STING — Natural Language Command");
                inputDlg.MainInstruction = "What do you want to do?";
                inputDlg.MainContent =
                    "Type or pick a command in natural language.\n\n" +
                    "Examples: \"Tag all\", \"Fix duplicates\", \"Export PDF\",\n" +
                    "\"Check compliance\", \"Maintenance schedule\", \"Export IFC\"";
                inputDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                    "Pick from command list", "Browse all 90+ commands by category");
                inputDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                    "Recent / most-used commands", "Quick access to frequently run commands");
                inputDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                    "BIM Knowledge Base", "Search BIM terminology and ISO 19650 definitions");
                inputDlg.CommonButtons = TaskDialogCommonButtons.Cancel;

                var choice = inputDlg.Show();
                if (choice == TaskDialogResult.Cancel) return Result.Cancelled;

                if (choice == TaskDialogResult.CommandLink3)
                {
                    // Knowledge base browse
                    var kbItems = NLPEngine.BimKnowledge
                        .OrderBy(k => k.Key)
                        .Select(k => new UI.StingListPicker.ListItem
                            { Label = k.Key, Detail = k.Value, Tag = k.Key })
                        .ToList();
                    UI.StingListPicker.Show("BIM Knowledge Base",
                        "ISO 19650 and BIM terminology", kbItems, false);
                    return Result.Succeeded;
                }

                // Build command list for picker
                var allCmds = NLPEngine.IntentPatterns
                    .Select(p => (p.CommandTag, p.Description))
                    .Distinct()
                    .OrderBy(c => c.Description)
                    .ToList();

                List<UI.StingListPicker.ListItem> cmdItems;
                if (choice == TaskDialogResult.CommandLink2)
                {
                    // Most-used: tagging + compliance + export
                    var priority = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
                    {
                        "AutoTag", "BatchTag", "TagNewOnly", "ResolveAllIssues",
                        "ValidateTags", "FullAutoPopulate", "SystemParamPush",
                        "IFCExport", "COBieExport", "ProSheetsPDF",
                        "MaintenanceSchedule", "HandoverPackage", "AssetCondition",
                        "CreateRevision", "RevisionDashboard", "DataIntegrityCheck"
                    };
                    cmdItems = allCmds
                        .Where(c => priority.Contains(c.CommandTag))
                        .Select(c => new UI.StingListPicker.ListItem
                            { Label = c.Description, Detail = c.CommandTag, Tag = c.CommandTag })
                        .ToList();
                }
                else
                {
                    // Full list grouped by category
                    cmdItems = allCmds
                        .Select(c => new UI.StingListPicker.ListItem
                        {
                            Label = c.Description,
                            Detail = $"[{c.CommandTag}]",
                            Tag = c.CommandTag
                        })
                        .ToList();
                }

                var picked = UI.StingListPicker.Show(
                    "STING — Natural Language Command",
                    "Select a command to execute:",
                    cmdItems, allowMultiSelect: false);

                if (picked == null || picked.Count == 0) return Result.Cancelled;

                string cmdTag = picked[0].Tag as string;
                if (string.IsNullOrEmpty(cmdTag)) return Result.Cancelled;

                // Confirm before execution
                var confirmDlg = new TaskDialog("Execute Command");
                confirmDlg.MainInstruction = picked[0].Label;
                confirmDlg.MainContent = $"Command tag: {cmdTag}\n\nRun this command now?";
                confirmDlg.CommonButtons = TaskDialogCommonButtons.Ok | TaskDialogCommonButtons.Cancel;
                if (confirmDlg.Show() == TaskDialogResult.Cancel) return Result.Cancelled;

                // Execute via WorkflowEngine resolver (covers all wired commands)
                StingLog.Info($"NLP: executing '{cmdTag}'");
                var cmd = WorkflowEngine.ResolveCommandPublic(cmdTag);
                if (cmd != null)
                {
                    string execMsg = "";
                    var result = cmd.Execute(commandData, ref execMsg, elements);
                    if (result == Result.Failed && !string.IsNullOrEmpty(execMsg))
                        TaskDialog.Show("Command Result", $"{cmdTag} failed: {execMsg}");
                    return result;
                }

                // Fallback: raise via external event handler
                UI.StingCommandHandler.SetExtraParam("NLPCommand", cmdTag);
                TaskDialog.Show("NLP Command",
                    $"'{cmdTag}' is not directly resolvable.\n\n" +
                    $"Click the corresponding button in the STING panel, or use the dispatch tag directly.");
                UI.StingCommandHandler.ClearExtraParam("NLPCommand");
                return Result.Succeeded;
            }
            catch (Exception ex)
            {
                StingLog.Error("NLP command failed", ex);
                message = ex.Message;
                return Result.Failed;
            }
        }
```

## FIX 7B — WorkflowEngine.cs: Make ResolveCommand accessible as public ResolveCommandPublic

**Locate in WorkflowEngine.cs** (search for this exact text):
```csharp
        private static IExternalCommand ResolveCommand(string tag)
```

**Add a public wrapper immediately after the closing `}` of ResolveCommand:**
```csharp
        /// <summary>FIX-7B: Public wrapper for NLPCommandProcessorCommand to call.</summary>
        public static IExternalCommand ResolveCommandPublic(string tag)
            => ResolveCommand(tag);
```

## FIX 7C — WorkflowEngine.cs: Add 18 missing command tags to ResolveCommand

**Locate in WorkflowEngine.cs** (search for this exact text):
```csharp
                default: return null;
            }
        }
```
(The closing of ResolveCommand's switch statement)

**Replace with:**
```csharp
                // FIX-7C: Pipeline commands missing from ResolveCommand
                case "SystemParamPush":     return new Tags.BatchSystemPushCommand();
                case "RepairDuplicateSeq":  return new Tags.RepairDuplicateSeqCommand();
                case "TagSelected":         return new Organise.TagSelectedCommand();
                case "ReTag":               return new Organise.ReTagCommand();
                case "FixDuplicates":       return new Organise.FixDuplicateTagsCommand();
                case "RenumberTags":        return new Organise.RenumberTagsCommand();
                case "CopyTags":            return new Organise.CopyTagsCommand();
                case "Tag3D":               return new Tags.Tag3DCommand();
                case "CheckData":           return new Tags.CheckDataCommand();
                case "SyncParamSchema":     return new Tags.SyncParameterSchemaCommand();
                case "LoadSharedParams":    return new Tags.LoadSharedParamsCommand();
                case "PurgeSharedParams":   return new Tags.PurgeSharedParamsCommand();
                case "GuidedDataEditor":    return new Tags.GuidedDataEditorCommand();
                // IoT Maintenance
                case "AssetCondition":      return new Temp.AssetConditionCommand();
                case "MaintenanceSchedule": return new Temp.MaintenanceScheduleCommand();
                case "WarrantyTracker":     return new Temp.WarrantyTrackerCommand();
                case "HandoverPackage":     return new Temp.HandoverPackageCommand();
                case "DigitalTwinExport":   return new Temp.DigitalTwinExportCommand();
                case "DataIntegrityCheck":  return new Temp.DataIntegrityCheckCommand();
                case "DataReport":          return new Temp.DataReportCommand();

                default: return null;
            }
        }
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 8 — PROSHEETS PDF EXPORTER (Replace stub with full implementation)
# Files: OperationsCommands.cs, StingCommandHandler.cs, StingDockPanel.xaml
# Root cause: PDFExportCommand dumps all sheets to Desktop with no options.
# PdfSelectedSheets/PdfActiveView show a "not available" placeholder.
# ══════════════════════════════════════════════════════════════════════

## FIX 8A — StingCommandHandler.cs: Route PDF buttons to ProSheetsPDFCommand

**Locate in StingCommandHandler.cs** (search for this exact text):
```csharp
                    case "PdfSelectedSheets":
                    case "PdfActiveView":
                        TaskDialog.Show("PDF Export",
                            "PDF export requires Revit's Print/Export API.\n" +
                            "Use File → Export → PDF in Revit for direct PDF output,\n" +
                            "or File → Print with a PDF printer driver.");
                        break;
```

**Replace with:**
```csharp
                    case "PdfSelectedSheets":
                        StingCommandHandler.SetExtraParam("PdfScope", "Selected");
                        RunCommand<Temp.ProSheetsPDFCommand>(app);
                        StingCommandHandler.ClearExtraParam("PdfScope");
                        break;
                    case "PdfActiveView":
                        StingCommandHandler.SetExtraParam("PdfScope", "ActiveView");
                        RunCommand<Temp.ProSheetsPDFCommand>(app);
                        StingCommandHandler.ClearExtraParam("PdfScope");
                        break;
                    case "ProSheetsPDF":
                        RunCommand<Temp.ProSheetsPDFCommand>(app);
                        break;
```

## FIX 8B — StingDockPanel.xaml: Upgrade PDF EXPORT section

**Locate in StingDockPanel.xaml** (search for this exact text):
```xml
                        <!-- PDF EXPORT -->
                        <TextBlock Style="{StaticResource SectionLabel}" Text="PDF EXPORT"/>
                        <Border Style="{StaticResource GroupBorder}">
                            <WrapPanel>
                                <ComboBox x:Name="cmbPaperSize" FontSize="10" Height="24" Width="60">
                                    <ComboBoxItem Content="A1" IsSelected="True"/>
                                    <ComboBoxItem Content="A0"/>
                                    <ComboBoxItem Content="A2"/>
                                    <ComboBoxItem Content="A3"/>
                                    <ComboBoxItem Content="A4"/>
                                </ComboBox>
                                <Button Style="{StaticResource ActionBtn}" Content="Selected Sheets" Tag="PdfSelectedSheets" Click="Cmd_Click"/>
                                <Button Style="{StaticResource ActionBtn}" Content="Active View" Tag="PdfActiveView" Click="Cmd_Click"/>
                            </WrapPanel>
                        </Border>
```

**Replace with:**
```xml
                        <!-- PDF EXPORT — ProSheets style -->
                        <TextBlock Style="{StaticResource SectionLabel}" Text="PDF EXPORT — PROSHEETS"/>
                        <Border Style="{StaticResource GroupBorder}">
                            <StackPanel>
                                <TextBlock Text="Scope, naming scheme, folder navigation, progress dialog" FontSize="9" Foreground="#888" Margin="0,0,0,4"/>
                                <WrapPanel>
                                    <Button Style="{StaticResource GreenBtn}" Content="Export PDF" Tag="ProSheetsPDF"
                                            Click="Cmd_Click" Width="80" Height="28"
                                            ToolTip="ProSheets PDF — choose scope, naming, folder, then export with progress"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Selected" Tag="PdfSelectedSheets"
                                            Click="Cmd_Click" ToolTip="Export selected sheets"/>
                                    <Button Style="{StaticResource ActionBtn}" Content="Active view" Tag="PdfActiveView"
                                            Click="Cmd_Click" ToolTip="Export active sheet/view"/>
                                </WrapPanel>
                            </StackPanel>
                        </Border>
```

## FIX 8C — OperationsCommands.cs: Add ProSheetsPDFCommand class

**Locate in OperationsCommands.cs** (search for this exact text):
```csharp
    // ────────────────────────────────────────────────────────────────────
    //  3. IFCExportCommand — Export to IFC with STING parameter mapping
```

**Insert immediately before it:**
```csharp
    // ────────────────────────────────────────────────────────────────────
    //  2b. ProSheetsPDFCommand — Full ProSheets-equivalent PDF exporter
    // ────────────────────────────────────────────────────────────────────

    /// <summary>
    /// FIX-8C: ProSheets-style PDF exporter — Diroots ProSheets equivalent.
    /// Scope: All / Selected / Filter by discipline / Pick from list.
    /// Naming: Sheet number / Number-Title / Combined / Project-Number-Title.
    /// Per-export folder navigation. Progress dialog with cancel.
    /// Opens output folder on completion.
    /// </summary>
    [Transaction(TransactionMode.ReadOnly)]
    [Regeneration(RegenerationOption.Manual)]
    public class ProSheetsPDFCommand : IExternalCommand
    {
        public Result Execute(ExternalCommandData commandData, ref string msg, ElementSet el)
        {
            var ctx = ParameterHelpers.GetContext(commandData);
            if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
            Document doc = ctx.Doc;
            UIDocument uidoc = ctx.UIDoc;

            var allSheets = new FilteredElementCollector(doc)
                .OfClass(typeof(ViewSheet)).Cast<ViewSheet>()
                .Where(s => !s.IsPlaceholder).OrderBy(s => s.SheetNumber).ToList();

            if (allSheets.Count == 0)
            { TaskDialog.Show("PDF Export", "No sheets found."); return Result.Succeeded; }

            // ── STEP 1: Scope ──────────────────────────────────────────
            string presetScope = UI.StingCommandHandler.GetExtraParam("PdfScope");
            List<ViewSheet> sheets;

            if (presetScope == "ActiveView")
            {
                if (ctx.ActiveView is ViewSheet vs)
                    sheets = new List<ViewSheet> { vs };
                else { TaskDialog.Show("PDF", "Active view is not a sheet."); return Result.Succeeded; }
            }
            else if (presetScope == "Selected")
            {
                sheets = uidoc.Selection.GetElementIds()
                    .Select(id => doc.GetElement(id)).OfType<ViewSheet>().ToList();
                if (sheets.Count == 0)
                { TaskDialog.Show("PDF", "No sheets in selection."); return Result.Cancelled; }
            }
            else
            {
                var scopeDlg = new TaskDialog("ProSheets PDF — Scope");
                scopeDlg.MainInstruction = $"Export to PDF — {allSheets.Count} sheets available";
                scopeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                    "All sheets", $"Export all {allSheets.Count} sheets");
                scopeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                    "Selected sheets", $"{uidoc.Selection.GetElementIds().Count} currently selected");
                scopeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                    "Filter by discipline / STING tag", "Choose sheets by DISC parameter or revision");
                scopeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                    "Pick from list", "Choose individual sheets from scrollable list");
                scopeDlg.CommonButtons = TaskDialogCommonButtons.Cancel;
                switch (scopeDlg.Show())
                {
                    case TaskDialogResult.CommandLink1: sheets = allSheets; break;
                    case TaskDialogResult.CommandLink2:
                        sheets = uidoc.Selection.GetElementIds()
                            .Select(id => doc.GetElement(id)).OfType<ViewSheet>().ToList();
                        if (sheets.Count == 0)
                        { TaskDialog.Show("PDF", "No sheets selected."); return Result.Cancelled; }
                        break;
                    case TaskDialogResult.CommandLink3:
                        sheets = FilterSheetsByDisc(doc, allSheets);
                        if (sheets == null) return Result.Cancelled; break;
                    case TaskDialogResult.CommandLink4:
                        sheets = PickSheets(allSheets);
                        if (sheets == null || sheets.Count == 0) return Result.Cancelled; break;
                    default: return Result.Cancelled;
                }
            }

            // ── STEP 2: Naming ─────────────────────────────────────────
            var nameDlg = new TaskDialog("ProSheets PDF — Naming");
            nameDlg.MainInstruction = "PDF file naming scheme";
            nameDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Sheet Number only", "e.g. A101.pdf");
            nameDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "Sheet Number — Title", "e.g. A101 - Ground Floor Plan.pdf");
            nameDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "One combined PDF", "All sheets merged into single file");
            nameDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                "Project No — Sheet No — Title",
                $"e.g. {(doc.ProjectInformation?.Number ?? "PRJ")}_A101_Title.pdf");
            nameDlg.CommonButtons = TaskDialogCommonButtons.Cancel;
            var nameResult = nameDlg.Show();
            if (nameResult == TaskDialogResult.Cancel) return Result.Cancelled;

            bool combined = nameResult == TaskDialogResult.CommandLink3;
            string projNum = Safe(doc.ProjectInformation?.Number ?? "PRJ");
            Func<ViewSheet, string> getName = nameResult switch
            {
                TaskDialogResult.CommandLink1 => s => Safe(s.SheetNumber),
                TaskDialogResult.CommandLink3 => _ => Safe(doc.Title ?? "STING_Export"),
                TaskDialogResult.CommandLink4 => s => Safe($"{projNum}_{s.SheetNumber}_{s.Name}"),
                _ => s => Safe($"{s.SheetNumber} - {s.Name}")
            };

            // ── STEP 3: Output folder ───────────────────────────────────
            string outputPath = OutputLocationHelper.PromptForExportPath(
                doc, combined ? getName(sheets[0]) + ".pdf" : "PDF_Export_folder",
                "All Files|*.*", "PDF");
            if (outputPath == null) return Result.Cancelled;
            string outputDir = Path.GetDirectoryName(outputPath) ?? outputPath;
            Directory.CreateDirectory(outputDir);

            // ── STEP 4: Export ─────────────────────────────────────────
            var progress = UI.StingProgressDialog.Show("ProSheets PDF", sheets.Count);
            int exported = 0, failed = 0;
            var failedList = new List<string>();

            if (combined)
            {
                try
                {
                    var pdfOpts = MakeOpts(getName(sheets[0]), true);
                    bool ok = doc.Export(outputDir, sheets.Select(s => s.Id).ToList(), pdfOpts);
                    exported = ok ? sheets.Count : 0;
                    if (!ok) { failed = sheets.Count; failedList.Add("combined"); }
                }
                catch (Exception ex) { failed = sheets.Count; StingLog.Error("ProSheetsPDF combined", ex); }
                progress.Increment("Exporting combined PDF...");
            }
            else
            {
                foreach (var sheet in sheets)
                {
                    if (progress.IsCancelled) break;
                    try
                    {
                        var pdfOpts = MakeOpts(getName(sheet), false);
                        bool ok = doc.Export(outputDir, new List<ElementId> { sheet.Id }, pdfOpts);
                        if (ok) exported++; else { failed++; failedList.Add(sheet.SheetNumber); }
                    }
                    catch (Exception ex)
                    {
                        failed++;
                        failedList.Add(sheet.SheetNumber);
                        StingLog.Warn($"ProSheetsPDF {sheet.SheetNumber}: {ex.Message}");
                    }
                    progress.Increment($"Exporting {sheet.SheetNumber}...");
                }
            }
            progress.Close();

            // ── STEP 5: Result dialog ──────────────────────────────────
            var resultDlg = new TaskDialog("PDF Export Complete");
            resultDlg.MainInstruction = $"Exported {exported} of {sheets.Count} sheets";
            resultDlg.MainContent =
                $"Output: {outputDir}\n" +
                (failed > 0 ? $"Failed: {string.Join(", ", failedList.Take(5))}\n" : "") +
                (combined ? $"File: {getName(sheets[0])}.pdf" : "Individual PDFs per sheet.");
            resultDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Open output folder", "Show PDFs in File Explorer");
            resultDlg.CommonButtons = TaskDialogCommonButtons.Close;
            if (resultDlg.Show() == TaskDialogResult.CommandLink1)
                try { System.Diagnostics.Process.Start("explorer.exe", outputDir); } catch { }

            StingLog.Info($"ProSheetsPDF: {exported}/{sheets.Count} → {outputDir}");
            return exported > 0 ? Result.Succeeded : Result.Failed;
        }

        private static PDFExportOptions MakeOpts(string fileName, bool combined) =>
            new PDFExportOptions
            {
                FileName = fileName, Combine = combined,
                ColorDepth = ColorDepthType.Color,
                RasterQuality = RasterQualityType.High,
                PaperPlacement = PaperPlacementType.Center,
                ZoomType = ZoomType.Zoom, ZoomPercentage = 100,
                AlwaysUseRaster = false,
            };

        private static List<ViewSheet> FilterSheetsByDisc(Document doc, List<ViewSheet> all)
        {
            var groups = all.GroupBy(s =>
            {
                string d = ParameterHelpers.GetString(s, ParamRegistry.DISC);
                return string.IsNullOrEmpty(d) ? "General" : d;
            }).ToList();
            var items = groups.Select(g => new UI.StingListPicker.ListItem
                { Label = $"{g.Key} ({g.Count()} sheets)", Tag = g.Key }).ToList();
            var picked = UI.StingListPicker.Show("Filter by Discipline",
                "Select discipline(s)", items, true);
            if (picked == null || picked.Count == 0) return null;
            var discs = new HashSet<string>(picked.Select(p => p.Tag as string),
                StringComparer.OrdinalIgnoreCase);
            return all.Where(s =>
            {
                string d = ParameterHelpers.GetString(s, ParamRegistry.DISC);
                return discs.Contains(string.IsNullOrEmpty(d) ? "General" : d);
            }).ToList();
        }

        private static List<ViewSheet> PickSheets(List<ViewSheet> all)
        {
            var items = all.Select(s => new UI.StingListPicker.ListItem
                { Label = $"{s.SheetNumber} — {s.Name}", Tag = s }).ToList();
            var picked = UI.StingListPicker.Show("Pick Sheets",
                $"Select sheets ({all.Count} available)", items, true);
            return picked?.Select(p => p.Tag as ViewSheet).Where(s => s != null).ToList();
        }

        private static string Safe(string n)
        {
            foreach (char c in Path.GetInvalidFileNameChars()) n = n.Replace(c, '_');
            return n.Length > 200 ? n.Substring(0, 200) : n;
        }
    }

```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 9 — PURGE SHARED PARAMS FROM PROJECT (Harrison-Dean style)
# File: LoadSharedParamsCommand.cs
# ══════════════════════════════════════════════════════════════════════

## FIX 9A — LoadSharedParamsCommand.cs: Add PurgeSharedParamsCommand class

**Locate in LoadSharedParamsCommand.cs** the last line of the last class
(just before the closing namespace `}`). Add as a new class:

```csharp
    /// <summary>
    /// FIX-9A: Purge shared parameters from the project.
    /// Mode 1 — Audit: count bound shared params vs MR_PARAMETERS.txt.
    /// Mode 2 — Purge orphaned: remove params NOT in MR_PARAMETERS.txt.
    /// Mode 3 — Purge all STING: remove all ASS_* / STING_* bindings.
    /// </summary>
    [Transaction(TransactionMode.Manual)]
    [Regeneration(RegenerationOption.Manual)]
    public class PurgeSharedParamsCommand : IExternalCommand
    {
        public Result Execute(ExternalCommandData commandData, ref string msg, ElementSet el)
        {
            var ctx = ParameterHelpers.GetContext(commandData);
            if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
            Document doc = ctx.Doc;

            var modeDlg = new TaskDialog("STING — Purge Shared Parameters");
            modeDlg.MainInstruction = "Shared parameter management";
            modeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Audit only (no changes)",
                "Report bound shared params vs MR_PARAMETERS.txt — safe read-only");
            modeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "Purge orphaned params",
                "Remove params bound in this project that are NOT in MR_PARAMETERS.txt");
            modeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "Purge ALL STING params from project",
                "Remove every ASS_* and STING_* shared param binding from this project");
            modeDlg.CommonButtons = TaskDialogCommonButtons.Cancel;

            switch (modeDlg.Show())
            {
                case TaskDialogResult.CommandLink1: return RunAudit(doc);
                case TaskDialogResult.CommandLink2: return RunPurge(doc, allSting: false);
                case TaskDialogResult.CommandLink3: return RunPurge(doc, allSting: true);
                default: return Result.Cancelled;
            }
        }

        private static HashSet<string> LoadKnown()
        {
            var known = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
            string spFile = StingToolsApp.FindDataFile("MR_PARAMETERS.txt");
            if (string.IsNullOrEmpty(spFile) || !File.Exists(spFile)) return known;
            foreach (string line in File.ReadAllLines(spFile))
            {
                if (!line.StartsWith("PARAM")) continue;
                var parts = line.Split('\t');
                if (parts.Length >= 3) known.Add(parts[2]);
            }
            return known;
        }

        private static Result RunAudit(Document doc)
        {
            var known = LoadKnown();
            var iter = doc.ParameterBindings.ForwardIterator();
            int total = 0, inMR = 0;
            var orphans = new List<string>();
            while (iter.MoveNext())
            {
                if (iter.Key is ExternalDefinition ext)
                {
                    total++;
                    if (known.Contains(ext.Name)) inMR++;
                    else orphans.Add(ext.Name);
                }
            }
            var sb = new StringBuilder();
            sb.AppendLine($"Shared Parameter Audit — {doc.Title}");
            sb.AppendLine(new string('═', 45));
            sb.AppendLine($"  MR_PARAMETERS.txt:   {known.Count} definitions");
            sb.AppendLine($"  Bound in project:    {total}");
            sb.AppendLine($"  Matched to MR file:  {inMR}");
            sb.AppendLine($"  Orphaned:            {orphans.Count}");
            if (orphans.Count > 0)
            {
                sb.AppendLine("\nOrphaned (not in MR_PARAMETERS.txt):");
                foreach (string n in orphans.Take(30)) sb.AppendLine($"  {n}");
                if (orphans.Count > 30) sb.AppendLine($"  ... and {orphans.Count - 30} more");
            }
            TaskDialog.Show("Param Audit", sb.ToString());
            return Result.Succeeded;
        }

        private static Result RunPurge(Document doc, bool allSting)
        {
            var known = LoadKnown();
            var iter = doc.ParameterBindings.ForwardIterator();
            var toRemove = new List<(string name, ExternalDefinition def)>();
            while (iter.MoveNext())
            {
                if (iter.Key is ExternalDefinition ext)
                {
                    bool isSting = ext.Name.StartsWith("ASS_", StringComparison.OrdinalIgnoreCase)
                                || ext.Name.StartsWith("STING_", StringComparison.OrdinalIgnoreCase);
                    bool isOrphan = !known.Contains(ext.Name);
                    if (allSting ? isSting : isOrphan) toRemove.Add((ext.Name, ext));
                }
            }
            if (toRemove.Count == 0)
            {
                TaskDialog.Show("Purge", allSting
                    ? "No STING params found in project."
                    : "No orphaned params found. All bound params are in MR_PARAMETERS.txt.");
                return Result.Succeeded;
            }
            string mode = allSting ? "ALL STING (ASS_*/STING_*)" : "orphaned";
            var confirm = new TaskDialog("Purge — Confirm");
            confirm.MainInstruction = $"Remove {toRemove.Count} {mode} params from project?";
            confirm.MainContent =
                string.Join("\n", toRemove.Take(20).Select(t => $"  {t.name}")) +
                (toRemove.Count > 20 ? $"\n  ... and {toRemove.Count - 20} more" : "") +
                "\n\n⚠ Elements lose parameter values for removed params.\n" +
                "Run 'Load Shared Params' afterward to rebind.";
            confirm.CommonButtons = TaskDialogCommonButtons.Ok | TaskDialogCommonButtons.Cancel;
            if (confirm.Show() == TaskDialogResult.Cancel) return Result.Cancelled;

            int removed = 0;
            using (var tx = new Transaction(doc, "STING Purge Shared Params"))
            {
                tx.Start();
                foreach (var (name, def) in toRemove)
                {
                    try { doc.ParameterBindings.Remove(def); removed++; }
                    catch (Exception ex) { StingLog.Warn($"PurgeParams '{name}': {ex.Message}"); }
                }
                tx.Commit();
            }
            TaskDialog.Show("Purge Complete",
                $"Removed {removed} of {toRemove.Count} parameter bindings.\n\n" +
                "Run 'Load Shared Params' to rebind parameters.");
            return Result.Succeeded;
        }
    }
```

## FIX 9B — StingCommandHandler.cs: Add dispatch entry

**Locate in StingCommandHandler.cs** (search for this exact text):
```csharp
                    case "LoadSharedParams":
```

**Add immediately after the closing `break;` of that case:**
```csharp
                    case "PurgeSharedParams": RunCommand<Tags.PurgeSharedParamsCommand>(app); break;
```

## FIX 9C — StingDockPanel.xaml: Add Purge Params button

**Locate in StingDockPanel.xaml** — find the `LoadSharedParams` button. Add immediately after it:
```xml
<Button Style="{StaticResource RedBtn}" Content="Purge Params" Tag="PurgeSharedParams"
        Click="Cmd_Click"
        ToolTip="Audit or remove orphaned/all STING shared parameter bindings from project"/>
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 10 — FAMILY PARAM CREATOR: FOLDER PICKER + PURGE-BEFORE-CREATE
# File: FamilyParamCreatorCommand.cs
# Root cause: Hardcodes DataPath for .rfa files. No delete-before-inject option.
# ══════════════════════════════════════════════════════════════════════

## FIX 10A — FamilyParamCreatorCommand.cs: Add PurgeFirst to ProcessOptions

**Locate in FamilyParamCreatorCommand.cs** (search for this exact text):
```csharp
        public class ProcessOptions
        {
            public bool InjectTagPos { get; set; } = true;
            public bool InjectFormulas { get; set; } = true;
            public bool CreatePositionTypes { get; set; } = true;
        }
```

**Replace with:**
```csharp
        public class ProcessOptions
        {
            public bool InjectTagPos { get; set; } = true;
            public bool InjectFormulas { get; set; } = true;
            public bool CreatePositionTypes { get; set; } = true;
            public bool PurgeFirst { get; set; } = false; // FIX-10A: purge STING params before injecting
        }
```

## FIX 10B — FamilyParamCreatorCommand.cs: Add purge step in ProcessFamily

**Locate in FamilyParamCreatorCommand.cs** in the ProcessFamily method.
Find the line:
```csharp
                    var (added, skipped) = InjectSharedParams(famDoc, app, paramList);
```

**Insert immediately before it:**
```csharp
                    // FIX-10B: Purge STING params before injection if PurgeFirst requested
                    if (opts.PurgeFirst)
                    {
                        FamilyManager purgeFm = famDoc.FamilyManager;
                        var stingFPs = purgeFm.GetParameters()
                            .Where(p => p.Definition.Name.StartsWith("ASS_", StringComparison.OrdinalIgnoreCase)
                                     || p.Definition.Name.StartsWith("STING_", StringComparison.OrdinalIgnoreCase))
                            .ToList();
                        int purgedCount = 0;
                        foreach (FamilyParameter fp in stingFPs)
                        {
                            try { purgeFm.RemoveParameter(fp); purgedCount++; }
                            catch (Exception pEx)
                            { StingLog.Warn($"FamilyPurge '{fp.Definition.Name}': {pEx.Message}"); }
                        }
                        if (purgedCount > 0)
                            StingLog.Info($"FamilyPurge: removed {purgedCount} params from {Path.GetFileName(rfaPath)}");
                    }
```

## FIX 10C — FamilyParamCreatorCommand.cs: Replace mode dialog + hardcoded paths

**Locate in FamilyParamCreatorCommand.cs** (search for this exact text):
```csharp
            var modeTd = new TaskDialog("STING — Family Param Creator");
            modeTd.MainInstruction = "Family Parameter Injection Mode";
            modeTd.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Single family", "Process one .rfa file");
            modeTd.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "Batch folder", "Process all .rfa files in a folder");
            modeTd.CommonButtons = TaskDialogCommonButtons.Cancel;
            var modeResult = modeTd.Show();

            if (modeResult != TaskDialogResult.CommandLink1 &&
                modeResult != TaskDialogResult.CommandLink2)
                return Result.Cancelled;

            bool isBatch = modeResult == TaskDialogResult.CommandLink2;
```

**Replace with:**
```csharp
            var modeTd = new TaskDialog("STING — Family Param Creator");
            modeTd.MainInstruction = "Family Parameter Injection Mode";
            modeTd.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Single family — add/update params",
                "Pick one .rfa file, skip already-existing params");
            modeTd.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "Batch folder — add/update params",
                "Pick a folder, process all .rfa files inside");
            modeTd.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "Single family — PURGE then re-inject",
                "Remove all STING params from one family, then inject fresh");
            modeTd.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                "Batch folder — PURGE then re-inject",
                "Remove all STING params from all families in a folder, then re-inject");
            modeTd.CommonButtons = TaskDialogCommonButtons.Cancel;
            var modeResult = modeTd.Show();
            if (modeResult == TaskDialogResult.Cancel) return Result.Cancelled;

            bool isBatch = modeResult == TaskDialogResult.CommandLink2
                        || modeResult == TaskDialogResult.CommandLink4;
            bool purgeFirst = modeResult == TaskDialogResult.CommandLink3
                           || modeResult == TaskDialogResult.CommandLink4;
```

**Locate** (search for this exact text):
```csharp
            if (isBatch)
            {
                // Use folder dialog via a simple path prompt
                string searchDir = StingToolsApp.DataPath ?? "";
                if (string.IsNullOrEmpty(searchDir) || !Directory.Exists(searchDir))
                {
                    TaskDialog.Show("STING", "No valid data directory found for batch processing.");
                    return Result.Failed;
                }
                rfaFiles = Directory.GetFiles(searchDir, "*.rfa", SearchOption.AllDirectories).ToList();
                outputDir = searchDir;
            }
            else
            {
                // For single file, look for .rfa in data path
                string searchDir = StingToolsApp.DataPath ?? "";
                rfaFiles = Directory.Exists(searchDir)
                    ? Directory.GetFiles(searchDir, "*.rfa", SearchOption.TopDirectoryOnly).Take(1).ToList()
                    : new List<string>();
                outputDir = searchDir;
            }
```

**Replace with:**
```csharp
            if (isBatch)
            {
                // FIX-10C: Let user navigate to the folder containing .rfa files
                var folderDlg = new Microsoft.Win32.SaveFileDialog
                {
                    Title = "Select folder containing .rfa family files — save dummy to pick folder",
                    FileName = "SELECT_FOLDER",
                    Filter = "All Files|*.*",
                    CheckPathExists = true,
                    OverwritePrompt = false,
                    InitialDirectory = StingToolsApp.DataPath
                        ?? Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments)
                };
                if (folderDlg.ShowDialog() != true) return Result.Cancelled;
                string searchDir = Path.GetDirectoryName(folderDlg.FileName);
                if (string.IsNullOrEmpty(searchDir) || !Directory.Exists(searchDir))
                { TaskDialog.Show("STING", "Invalid folder."); return Result.Failed; }
                rfaFiles = Directory.GetFiles(searchDir, "*.rfa", SearchOption.AllDirectories).ToList();
                outputDir = searchDir;
            }
            else
            {
                // FIX-10C: Single .rfa file picker
                var fileDlg = new Microsoft.Win32.OpenFileDialog
                {
                    Title = "Select Revit Family (.rfa) to process",
                    Filter = "Revit Family|*.rfa|All Files|*.*",
                    InitialDirectory = StingToolsApp.DataPath
                        ?? Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments)
                };
                if (fileDlg.ShowDialog() != true) return Result.Cancelled;
                rfaFiles = new List<string> { fileDlg.FileName };
                outputDir = Path.GetDirectoryName(fileDlg.FileName);
            }
```

**Locate** the `opts` variable initialization:
```csharp
            // Options
            var opts = new FamilyParamEngine.ProcessOptions
            {
                InjectTagPos = true,
                InjectFormulas = true,
                CreatePositionTypes = true
            };
```

**Replace with:**
```csharp
            var opts = new FamilyParamEngine.ProcessOptions
            {
                InjectTagPos = true,
                InjectFormulas = true,
                CreatePositionTypes = true,
                PurgeFirst = purgeFirst  // FIX-10C
            };
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 11 — MAGIC RENAME TOOL (Pangolin-inspired, enhanced)
# Files: ViewAutomationCommands.cs, StingCommandHandler.cs, StingDockPanel.xaml
# ══════════════════════════════════════════════════════════════════════

## FIX 11A — ViewAutomationCommands.cs: Add MagicRenameCommand class

**Locate the last closing `}` of the last class in ViewAutomationCommands.cs.
Add as a new class before the namespace closing `}`:**

```csharp
    /// <summary>
    /// FIX-11A: Magic Rename Tool — unified rename engine.
    /// Targets: Views, Sheets, Family Types, Rooms.
    /// Operations: Find/Replace (plain or regex), Prefix/Suffix, Case, Numbering.
    /// Shows preview before committing. Undo-safe single transaction.
    /// </summary>
    [Transaction(TransactionMode.Manual)]
    [Regeneration(RegenerationOption.Manual)]
    public class MagicRenameCommand : IExternalCommand
    {
        public Result Execute(ExternalCommandData commandData, ref string msg, ElementSet el)
        {
            var ctx = ParameterHelpers.GetContext(commandData);
            if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
            Document doc = ctx.Doc;
            UIDocument uidoc = ctx.UIDoc;

            // ── Step 1: Target type ────────────────────────────────────
            var targetDlg = new TaskDialog("Magic Rename — Target");
            targetDlg.MainInstruction = "What do you want to rename?";
            targetDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Views", "Floor plans, sections, elevations, 3D, details, schedules");
            targetDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "Sheets", "Drawing sheets (number + title)");
            targetDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "Family Types", "Loaded family type names");
            targetDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                "Rooms / Areas / Spaces", "Room names");
            targetDlg.CommonButtons = TaskDialogCommonButtons.Cancel;
            var targetResult = targetDlg.Show();
            if (targetResult == TaskDialogResult.Cancel) return Result.Cancelled;

            // ── Step 2: Operation ──────────────────────────────────────
            var opDlg = new TaskDialog("Magic Rename — Operation");
            opDlg.MainInstruction = "Choose rename operation";
            opDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "Find & Replace (plain text)", "Case-insensitive text substitution");
            opDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "Find & Replace (regex)", "Regular expression with capture groups ($1, $2)");
            opDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "Add Prefix / Suffix", "Prepend or append text");
            opDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                "Case conversion / Numbered sequence", "UPPER, lower, Title Case, or add 001...");
            opDlg.CommonButtons = TaskDialogCommonButtons.Cancel;
            var opResult = opDlg.Show();
            if (opResult == TaskDialogResult.Cancel) return Result.Cancelled;

            // ── Step 3: Collect targets ────────────────────────────────
            var targets = CollectTargets(doc, uidoc, targetResult);
            if (targets.Count == 0)
            { TaskDialog.Show("Magic Rename", "No matching elements found."); return Result.Succeeded; }

            // ── Step 4: Get rename params ──────────────────────────────
            string findStr = "", replaceStr = "", prefix = "", suffix = "";
            bool useRegex = false, addNumber = false, toUpper = false, toLower = false, toTitle = false;

            switch (opResult)
            {
                case TaskDialogResult.CommandLink1:
                case TaskDialogResult.CommandLink2:
                    useRegex = opResult == TaskDialogResult.CommandLink2;
                    var suggestions = targets.Select(t => t.name)
                        .SelectMany(n => new[] { n.Split('-').FirstOrDefault()?.Trim() ?? "" })
                        .Where(s => s.Length > 1).Distinct().Take(8)
                        .Concat(new[] { "Level ", " Copy", "MECH", "ELEC", "PLMB", "ARCH" })
                        .Select(s => new UI.StingListPicker.ListItem { Label = s, Tag = s }).ToList();
                    var findPicked = UI.StingListPicker.Show(
                        useRegex ? "Regex Find Pattern" : "Find Text",
                        $"Text to find in {targets.Count} names (or type a pattern):",
                        suggestions, false);
                    findStr = findPicked?.FirstOrDefault()?.Tag as string ?? "";
                    if (string.IsNullOrEmpty(findStr)) return Result.Cancelled;

                    var replSugg = new[] { "", "L", "M-", "E-", "P-", "GF", "B1", "RF" }
                        .Select(s => new UI.StingListPicker.ListItem
                            { Label = string.IsNullOrEmpty(s) ? "(empty — delete)" : s, Tag = s })
                        .ToList();
                    var replPicked = UI.StingListPicker.Show("Replace With",
                        useRegex ? "Replacement (use $1, $2 for groups):" : "Replace with:",
                        replSugg, false);
                    replaceStr = replPicked?.FirstOrDefault()?.Tag as string ?? "";
                    break;

                case TaskDialogResult.CommandLink3:
                    prefix = PickText("Add Prefix", "Text to prepend (blank to skip):");
                    suffix = PickText("Add Suffix", "Text to append (blank to skip):");
                    break;

                case TaskDialogResult.CommandLink4:
                    var caseDlg = new TaskDialog("Case / Numbering");
                    caseDlg.MainInstruction = "Conversion type";
                    caseDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1, "UPPERCASE", "");
                    caseDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2, "lowercase", "");
                    caseDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3, "Title Case", "");
                    caseDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                        "Add sequence number suffix", "Append _001, _002, _003...");
                    caseDlg.CommonButtons = TaskDialogCommonButtons.Cancel;
                    switch (caseDlg.Show())
                    {
                        case TaskDialogResult.CommandLink1: toUpper  = true; break;
                        case TaskDialogResult.CommandLink2: toLower  = true; break;
                        case TaskDialogResult.CommandLink3: toTitle  = true; break;
                        case TaskDialogResult.CommandLink4: addNumber = true; break;
                        default: return Result.Cancelled;
                    }
                    break;
            }

            // ── Step 5: Build preview ──────────────────────────────────
            int seq = 1;
            var previews = new List<(string oldName, string newName, Element element)>();
            foreach (var (element, name) in targets)
            {
                string n = name;
                try
                {
                    if (!string.IsNullOrEmpty(findStr))
                    {
                        n = useRegex
                            ? System.Text.RegularExpressions.Regex.Replace(n, findStr, replaceStr,
                                System.Text.RegularExpressions.RegexOptions.IgnoreCase)
                            : n.Replace(findStr, replaceStr, StringComparison.OrdinalIgnoreCase);
                    }
                    if (!string.IsNullOrEmpty(prefix)) n = prefix + n;
                    if (!string.IsNullOrEmpty(suffix)) n = n + suffix;
                    if (toUpper)  n = n.ToUpperInvariant();
                    if (toLower)  n = n.ToLowerInvariant();
                    if (toTitle)  n = System.Globalization.CultureInfo.CurrentCulture.TextInfo.ToTitleCase(n.ToLower());
                    if (addNumber) { n = $"{n}_{seq:D3}"; seq++; }
                }
                catch { /* invalid regex keeps original */ }
                previews.Add((name, n, element));
            }

            int changes = previews.Count(p => p.oldName != p.newName);
            var sb = new StringBuilder();
            sb.AppendLine($"Preview — {changes} of {previews.Count} names will change:\n");
            foreach (var (old, nw, _) in previews.Where(p => p.oldName != p.newName).Take(25))
                sb.AppendLine($"  {old}\n    → {nw}");
            if (changes > 25) sb.AppendLine($"  ... and {changes - 25} more");

            var previewDlg = new TaskDialog("Magic Rename — Preview");
            previewDlg.MainInstruction = $"Apply {changes} renames?";
            previewDlg.MainContent = sb.ToString();
            previewDlg.CommonButtons = TaskDialogCommonButtons.Ok | TaskDialogCommonButtons.Cancel;
            if (previewDlg.Show() == TaskDialogResult.Cancel) return Result.Cancelled;

            // ── Step 6: Apply ──────────────────────────────────────────
            int renamed = 0, failed = 0;
            using (var tx = new Transaction(doc, "STING Magic Rename"))
            {
                tx.Start();
                foreach (var (_, newName, element) in previews.Where(p => p.oldName != p.newName))
                {
                    try
                    {
                        if (element is View v) v.Name = newName;
                        else if (element is FamilySymbol fs) fs.Name = newName;
                        else if (element is SpatialElement sp)
                        {
                            var p2 = sp.get_Parameter(BuiltInParameter.ROOM_NAME);
                            if (p2 != null && !p2.IsReadOnly) p2.Set(newName);
                        }
                        else element.Name = newName;
                        renamed++;
                    }
                    catch (Exception ex)
                    {
                        failed++;
                        StingLog.Warn($"MagicRename {element?.Id}: {ex.Message}");
                    }
                }
                tx.Commit();
            }

            TaskDialog.Show("Magic Rename",
                $"Renamed: {renamed}\nFailed: {failed}\nUnchanged: {previews.Count - renamed - failed}");
            return Result.Succeeded;
        }

        private static List<(Element element, string name)> CollectTargets(
            Document doc, UIDocument uidoc, TaskDialogResult targetType)
        {
            var result = new List<(Element, string)>();
            var selIds = uidoc.Selection.GetElementIds();
            switch (targetType)
            {
                case TaskDialogResult.CommandLink1: // Views
                    foreach (View v in (selIds.Count > 0
                        ? selIds.Select(id => doc.GetElement(id)).OfType<View>()
                        : (IEnumerable<View>)new FilteredElementCollector(doc)
                            .OfClass(typeof(View)).Cast<View>()
                            .Where(v2 => !v2.IsTemplate && v2.CanBePrinted)))
                        result.Add((v, v.Name));
                    break;
                case TaskDialogResult.CommandLink2: // Sheets
                    foreach (var s in new FilteredElementCollector(doc)
                        .OfClass(typeof(ViewSheet)).Cast<ViewSheet>()
                        .Where(s2 => !s2.IsPlaceholder))
                        result.Add((s, s.Name));
                    break;
                case TaskDialogResult.CommandLink3: // Family types
                    var syms = selIds.Count > 0
                        ? selIds.Select(id => doc.GetElement(id)).OfType<FamilySymbol>()
                        : new FilteredElementCollector(doc).OfClass(typeof(FamilySymbol)).Cast<FamilySymbol>();
                    foreach (var sym in syms) result.Add((sym, sym.Name));
                    break;
                case TaskDialogResult.CommandLink4: // Rooms
                    foreach (var r in new FilteredElementCollector(doc).OfClass(typeof(SpatialElement)))
                        if (r is Room rm) result.Add((rm, rm.Name));
                    break;
            }
            return result;
        }

        private static string PickText(string title, string prompt)
        {
            var items = new[] { "(empty)", "M-", "E-", "P-", "A-", "S-", "GF", "L01" }
                .Select(s => new UI.StingListPicker.ListItem { Label = s, Tag = s }).ToList();
            var picked = UI.StingListPicker.Show(title, prompt, items, false);
            string val = picked?.FirstOrDefault()?.Tag as string ?? "";
            return val == "(empty)" ? "" : val;
        }
    }
```

## FIX 11B — StingCommandHandler.cs: Add MagicRename dispatch

**Locate in StingCommandHandler.cs:**
```csharp
                    case "BatchRenameViews": RunCommand<Docs.BatchRenameViewsCommand>(app); break;
```

**Add immediately after:**
```csharp
                    case "MagicRename": RunCommand<Docs.MagicRenameCommand>(app); break;
```

## FIX 11C — StingDockPanel.xaml: Add MagicRename button

**Locate in StingDockPanel.xaml** the `BatchRenameViews` button. Add immediately after it:
```xml
<Button Style="{StaticResource ActionBtn}" Content="Magic Rename" Tag="MagicRename"
        Click="Cmd_Click"
        ToolTip="Rename views/sheets/families/rooms — regex, prefix/suffix, case, numbering, with preview"/>
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 12 — VIEW TAB COLOURING (pyRevit + PROISAC VDCStyler combined)
# Files: ViewAutomationCommands.cs, StingCommandHandler.cs, StingDockPanel.xaml
# ══════════════════════════════════════════════════════════════════════

## FIX 12A — ViewAutomationCommands.cs: Add ViewTabColourCommand class

**Add as a new class in ViewAutomationCommands.cs (after MagicRenameCommand):**

```csharp
    /// <summary>
    /// FIX-12A: View Tab Colouring — pyRevit + PROISAC-BIM-VDCStyler combined.
    /// Applies discipline or view-type icons as name prefixes in the Project Browser
    /// for instant visual discipline identification. Fully reversible.
    /// </summary>
    [Transaction(TransactionMode.Manual)]
    [Regeneration(RegenerationOption.Manual)]
    public class ViewTabColourCommand : IExternalCommand
    {
        // Discipline icon prefixes
        private static readonly Dictionary<string, string> DiscIcons =
            new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase)
        {
            { "M",  "⚙ " }, { "E",  "⚡ " }, { "P",  "💧 " },
            { "FP", "🔥 " }, { "LV", "📡 " }, { "S",  "🏗 " },
            { "A",  "🏛 " }, { "G",  "⬜ " },
        };
        // View type icon prefixes
        private static readonly Dictionary<ViewType, string> TypeIcons =
            new Dictionary<ViewType, string>
        {
            { ViewType.FloorPlan,   "📐 " }, { ViewType.CeilingPlan, "⬆ " },
            { ViewType.Section,     "✂ "  }, { ViewType.Elevation,   "📏 " },
            { ViewType.ThreeD,      "🧊 "  }, { ViewType.Schedule,    "📋 " },
            { ViewType.Detail,      "🔍 "  }, { ViewType.DraftingView,"✏ "  },
        };

        public Result Execute(ExternalCommandData commandData, ref string msg, ElementSet el)
        {
            var ctx = ParameterHelpers.GetContext(commandData);
            if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
            Document doc = ctx.Doc;

            var modeDlg = new TaskDialog("STING — View Tab Colouring");
            modeDlg.MainInstruction = "Apply colour codes to view names in Project Browser";
            modeDlg.MainContent =
                "Adds discipline or view-type icons as prefixes to view names.\n" +
                "Visible in the Project Browser for instant visual grouping.\n" +
                "Fully reversible with 'Remove All'.";
            modeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "By discipline (STING DISC param)",
                "⚙ MECH  ⚡ ELEC  💧 PLMB  🔥 FP  📡 LV  🏗 STR  🏛 ARCH");
            modeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "By view type",
                "📐 Plans  ✂ Sections  📏 Elevations  🧊 3D  📋 Schedules");
            modeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "By name keyword (auto-detect discipline)",
                "Detects MECH/ELEC/PLMB/ARCH/STRUCT in view names");
            modeDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                "Remove all STING colour coding",
                "Strip all discipline and type icons applied by this command");
            modeDlg.CommonButtons = TaskDialogCommonButtons.Cancel;

            switch (modeDlg.Show())
            {
                case TaskDialogResult.CommandLink1: return ApplyByParam(doc);
                case TaskDialogResult.CommandLink2: return ApplyByType(doc);
                case TaskDialogResult.CommandLink3: return ApplyByKeyword(doc);
                case TaskDialogResult.CommandLink4: return RemoveAll(doc);
                default: return Result.Cancelled;
            }
        }

        private Result ApplyByParam(Document doc)
        {
            var views = GetViews(doc);
            int applied = 0;
            using (var tx = new Transaction(doc, "STING View Disc Prefix"))
            {
                tx.Start();
                foreach (var v in views)
                {
                    string disc = ParameterHelpers.GetString(v, Core.ParamRegistry.DISC);
                    if (string.IsNullOrEmpty(disc) || !DiscIcons.TryGetValue(disc, out string icon)) continue;
                    if (!StartsWithAnyIcon(v.Name)) { SafeRename(v, icon + v.Name); applied++; }
                }
                tx.Commit();
            }
            TaskDialog.Show("View Tab Colouring",
                $"Applied discipline icons to {applied} views.\nVisible in Project Browser.");
            return Result.Succeeded;
        }

        private Result ApplyByType(Document doc)
        {
            var views = GetViews(doc);
            int applied = 0;
            using (var tx = new Transaction(doc, "STING View Type Prefix"))
            {
                tx.Start();
                foreach (var v in views)
                {
                    if (!TypeIcons.TryGetValue(v.ViewType, out string icon)) continue;
                    if (!StartsWithAnyIcon(v.Name)) { SafeRename(v, icon + v.Name); applied++; }
                }
                tx.Commit();
            }
            TaskDialog.Show("View Tab Colouring",
                $"Applied view-type icons to {applied} views.");
            return Result.Succeeded;
        }

        private Result ApplyByKeyword(Document doc)
        {
            var views = GetViews(doc);
            var keywords = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase)
            {
                { "MECH", "⚙ " }, { "HVAC", "⚙ " }, { "DUCT", "⚙ " }, { "AHU", "⚙ " },
                { "ELEC", "⚡ " }, { "POWER", "⚡ " }, { "LIGHT", "⚡ " }, { "ELV", "⚡ " },
                { "PLMB", "💧 " }, { "DRAIN", "💧 " }, { "WATER", "💧 " },
                { "FIRE", "🔥 " }, { "SPRINK", "🔥 " }, { "FP", "🔥 " },
                { "STRUCT", "🏗 " }, { "STR ", "🏗 " }, { "FOUND", "🏗 " },
                { "ARCH", "🏛 " }, { "FLOOR", "🏛 " }, { "ROOF", "🏛 " },
                { "LOW VOLT", "📡 " }, { "COMMS", "📡 " }, { "DATA", "📡 " },
            };
            int applied = 0;
            using (var tx = new Transaction(doc, "STING View Keyword Prefix"))
            {
                tx.Start();
                foreach (var v in views)
                {
                    if (StartsWithAnyIcon(v.Name)) continue;
                    string nameUpper = v.Name.ToUpperInvariant();
                    foreach (var kv in keywords)
                    {
                        if (nameUpper.Contains(kv.Key.ToUpperInvariant()))
                        { SafeRename(v, kv.Value + v.Name); applied++; break; }
                    }
                }
                tx.Commit();
            }
            TaskDialog.Show("View Tab Colouring",
                $"Applied keyword-based icons to {applied} views.");
            return Result.Succeeded;
        }

        private Result RemoveAll(Document doc)
        {
            var allIcons = DiscIcons.Values.Concat(TypeIcons.Values).ToHashSet();
            var views = new FilteredElementCollector(doc).OfClass(typeof(View)).Cast<View>()
                .Where(v => !v.IsTemplate).ToList();
            int stripped = 0;
            using (var tx = new Transaction(doc, "STING Remove View Icons"))
            {
                tx.Start();
                foreach (var v in views)
                {
                    string icon = allIcons.FirstOrDefault(i => v.Name.StartsWith(i));
                    if (icon != null) { SafeRename(v, v.Name.Substring(icon.Length)); stripped++; }
                }
                tx.Commit();
            }
            TaskDialog.Show("View Tab Colouring", $"Removed icons from {stripped} views.");
            return Result.Succeeded;
        }

        private static IEnumerable<View> GetViews(Document doc) =>
            new FilteredElementCollector(doc).OfClass(typeof(View)).Cast<View>()
                .Where(v => !v.IsTemplate && !(v is ViewSheet));

        private static bool StartsWithAnyIcon(string name) =>
            name.Length > 2 && (name[0] > 127 || name.StartsWith("⚙") || name.StartsWith("⚡")
                || name.StartsWith("💧") || name.StartsWith("🔥") || name.StartsWith("📡")
                || name.StartsWith("🏗") || name.StartsWith("🏛") || name.StartsWith("⬜")
                || name.StartsWith("📐") || name.StartsWith("✂") || name.StartsWith("📏")
                || name.StartsWith("🧊") || name.StartsWith("📋") || name.StartsWith("🔍")
                || name.StartsWith("⬆") || name.StartsWith("✏"));

        private static void SafeRename(View v, string newName)
        {
            try { if (newName.Length < 256) v.Name = newName; }
            catch (Exception ex) { Core.StingLog.Warn($"ViewTabColour rename {v.Id}: {ex.Message}"); }
        }
    }
```

## FIX 12B — StingCommandHandler.cs + StingDockPanel.xaml: Wire ViewTabColour

**In StingCommandHandler.cs**, add after `case "MagicRename":`:
```csharp
                    case "ViewTabColour": RunCommand<Docs.ViewTabColourCommand>(app); break;
```

**In StingDockPanel.xaml**, add after the MagicRename button:
```xml
<Button Style="{StaticResource ActionBtn}" Content="Tab Colours" Tag="ViewTabColour"
        Click="Cmd_Click"
        ToolTip="Colour-code views in Project Browser by discipline or type — pyRevit + VDCStyler style"/>
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 13 — GUIDED DATA FILE EDITOR WITH SYNC
# Files: ConfigEditorCommand.cs, StingCommandHandler.cs, StingDockPanel.xaml
# ══════════════════════════════════════════════════════════════════════

## FIX 13A — ConfigEditorCommand.cs: Add GuidedDataEditorCommand class

**Add as a new class at the end of ConfigEditorCommand.cs (before namespace `}`):**

```csharp
    /// <summary>
    /// FIX-13A: Guided Data File Editor — opens any STING data file in the system
    /// editor with format hints, detects save, and reloads TagConfig on sync.
    /// Covers: project_config.json, MR_PARAMETERS.txt, MATERIAL_SCHEMA.json,
    /// PARAMETER_REGISTRY.json, LABEL_DEFINITIONS.json, TAG_PLACEMENT_PRESETS_DEFAULT.json,
    /// WORKFLOW_*.json, cost_rates_5d.csv
    /// </summary>
    [Transaction(TransactionMode.ReadOnly)]
    [Regeneration(RegenerationOption.Manual)]
    public class GuidedDataEditorCommand : IExternalCommand
    {
        private static readonly (string key, string file, string desc, string fmt)[] Files =
        {
            ("config",    "project_config.json",              "Tag codes, SEQ scheme, separators, compliance gate", "JSON"),
            ("params",    "MR_PARAMETERS.txt",                "Shared parameter definitions (Revit format)",         "TXT"),
            ("registry",  "PARAMETER_REGISTRY.json",          "Parameter registry: GUIDs, groups, categories",       "JSON"),
            ("materials", "MATERIAL_SCHEMA.json",             "Material properties and unit cost schema",             "JSON"),
            ("labels",    "LABEL_DEFINITIONS.json",           "Tag label display definitions",                        "JSON"),
            ("placement", "TAG_PLACEMENT_PRESETS_DEFAULT.json","Tag placement preset coordinates",                     "JSON"),
            ("workflow",  "WORKFLOW_DailyQA_Enhanced.json",   "Workflow step sequences",                              "JSON"),
            ("costs",     "cost_rates_5d.csv",                "5D cost rates per category/system",                    "CSV"),
        };

        public Result Execute(ExternalCommandData commandData, ref string msg, ElementSet el)
        {
            var ctx = ParameterHelpers.GetContext(commandData);
            if (ctx == null) { TaskDialog.Show("STING", "No document open."); return Result.Failed; }
            Document doc = ctx.Doc;

            // Step 1: Choose file
            var td = new TaskDialog("STING — Data File Editor");
            td.MainInstruction = "Choose a data file to edit";
            td.MainContent =
                "Opens the file in your system editor.\n" +
                "After saving, click Sync to reload STING with your changes.";
            td.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                "project_config.json — Tag Configuration",
                "DISC map, SYS map, PROD/FUNC codes, LOC/ZONE, SEQ scheme, compliance gate");
            td.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "MR_PARAMETERS.txt — Shared Parameters",
                "Add/edit shared parameter definitions (Revit .txt format)");
            td.AddCommandLink(TaskDialogCommandLinkId.CommandLink3,
                "MATERIAL_SCHEMA.json — Materials & Costs",
                "Material properties, unit costs for 5D costing");
            td.AddCommandLink(TaskDialogCommandLinkId.CommandLink4,
                "Other files (Registry, Labels, Presets, Workflows, Cost Rates)",
                "PARAMETER_REGISTRY, LABEL_DEFINITIONS, TAG_PLACEMENT_PRESETS, WORKFLOW files");
            td.CommonButtons = TaskDialogCommonButtons.Cancel;

            var choice = td.Show();
            if (choice == TaskDialogResult.Cancel) return Result.Cancelled;

            string fileKey = choice switch
            {
                TaskDialogResult.CommandLink1 => "config",
                TaskDialogResult.CommandLink2 => "params",
                TaskDialogResult.CommandLink3 => "materials",
                TaskDialogResult.CommandLink4 => PickOtherFile(),
                _ => null
            };
            if (fileKey == null) return Result.Cancelled;

            var fileDef = Array.Find(Files, f => f.key == fileKey);
            if (fileDef.file == null) return Result.Cancelled;

            // Step 2: Find file on disk
            string filePath = StingToolsApp.FindDataFile(fileDef.file);
            if (string.IsNullOrEmpty(filePath) || !File.Exists(filePath))
            {
                string projDir = Path.GetDirectoryName(doc.PathName ?? "");
                if (!string.IsNullOrEmpty(projDir))
                {
                    string candidate = Path.Combine(projDir, fileDef.file);
                    if (File.Exists(candidate)) filePath = candidate;
                }
            }
            if (string.IsNullOrEmpty(filePath) || !File.Exists(filePath))
            {
                TaskDialog.Show("Data Editor",
                    $"File not found: {fileDef.file}\n\n" +
                    "Run 'Load Shared Params' or 'Config Editor → Save Config' to create it first.");
                return Result.Succeeded;
            }

            // Step 3: Show format guide + open
            var guide = new TaskDialog($"Editing: {fileDef.file}");
            guide.MainInstruction = $"{fileDef.fmt} — {fileDef.desc}";
            guide.MainContent = GetGuide(fileKey, filePath);
            guide.AddCommandLink(TaskDialogCommandLinkId.CommandLink1,
                $"Open in {fileDef.fmt} editor", $"Opens {fileDef.file} in your default editor");
            guide.AddCommandLink(TaskDialogCommandLinkId.CommandLink2,
                "Copy path to clipboard", filePath);
            guide.CommonButtons = TaskDialogCommonButtons.Cancel;

            var guideChoice = guide.Show();
            if (guideChoice == TaskDialogResult.Cancel) return Result.Cancelled;

            if (guideChoice == TaskDialogResult.CommandLink2)
            {
                try { System.Windows.Clipboard.SetText(filePath); }
                catch { }
                TaskDialog.Show("Path Copied", filePath);
                return Result.Succeeded;
            }

            // Open in system editor
            DateTime modBefore = File.GetLastWriteTime(filePath);
            try
            {
                System.Diagnostics.Process.Start(new System.Diagnostics.ProcessStartInfo
                    { FileName = filePath, UseShellExecute = true });
            }
            catch (Exception ex)
            {
                TaskDialog.Show("Open Failed", $"{ex.Message}\n\nPath: {filePath}");
                return Result.Failed;
            }

            // Step 4: Wait for save, offer sync
            var waitDlg = new TaskDialog($"Editing: {fileDef.file}");
            waitDlg.MainInstruction = "Edit, save, then click Sync";
            waitDlg.MainContent = $"File: {filePath}\n\nMake your changes, save the file, then click Sync Changes.";
            waitDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink1, "Sync Changes", "Reload STING with updated file");
            waitDlg.AddCommandLink(TaskDialogCommandLinkId.CommandLink2, "Skip (file saved, no reload)", "Keep changes but don't reload now");
            waitDlg.CommonButtons = TaskDialogCommonButtons.Cancel;

            if (waitDlg.Show() != TaskDialogResult.CommandLink1) return Result.Succeeded;

            // Step 5: Sync
            string syncMsg;
            DateTime modAfter = File.GetLastWriteTime(filePath);
            bool changed = modAfter > modBefore;
            try
            {
                switch (fileKey)
                {
                    case "config":
                        TagConfig.LoadFromFile(filePath);
                        Core.ComplianceScan.InvalidateCache();
                        Core.StingAutoTagger.InvalidateContext();
                        syncMsg = $"✅ project_config.json reloaded.\nTagConfig updated — {(changed ? "changes detected" : "no changes detected")}.";
                        break;
                    case "params":
                        syncMsg = "✅ MR_PARAMETERS.txt saved.\nRun 'Load Shared Params' to bind new parameters to Revit.";
                        break;
                    default:
                        syncMsg = $"✅ {fileDef.file} saved.{(changed ? "" : " No changes detected.")}";
                        break;
                }
            }
            catch (Exception ex) { syncMsg = $"Sync error: {ex.Message}"; }

            TaskDialog.Show("Sync Complete", syncMsg);
            return Result.Succeeded;
        }

        private static string PickOtherFile()
        {
            var td2 = new TaskDialog("Other Data Files");
            td2.MainInstruction = "Choose file";
            td2.AddCommandLink(TaskDialogCommandLinkId.CommandLink1, "PARAMETER_REGISTRY.json", "GUIDs and container definitions");
            td2.AddCommandLink(TaskDialogCommandLinkId.CommandLink2, "LABEL_DEFINITIONS.json",  "Tag label display names");
            td2.AddCommandLink(TaskDialogCommandLinkId.CommandLink3, "TAG_PLACEMENT_PRESETS_DEFAULT.json", "Tag placement coordinates");
            td2.AddCommandLink(TaskDialogCommandLinkId.CommandLink4, "WORKFLOW or cost_rates_5d files", "Workflow JSON or cost CSV");
            td2.CommonButtons = TaskDialogCommonButtons.Cancel;
            return td2.Show() switch
            {
                TaskDialogResult.CommandLink1 => "registry",
                TaskDialogResult.CommandLink2 => "labels",
                TaskDialogResult.CommandLink3 => "placement",
                TaskDialogResult.CommandLink4 => "workflow",
                _ => null
            };
        }

        private static string GetGuide(string key, string path) => key switch
        {
            "config" =>
                "KEY FIELDS:\n" +
                "  DISC_MAP:  {\"Ducts\": \"M\", \"Pipes\": \"P\", ...}\n" +
                "  SYS_MAP:   {\"HVAC\": [\"Air Terminals\", \"Ducts\", ...]}\n" +
                "  LOC_CODES: [\"BLD1\", \"BLD2\", ...]\n" +
                "  ZONE_CODES:[\"Z01\", \"Z02\", ...]\n" +
                "  SEPARATOR: \"-\"    NUM_PAD: 4\n" +
                "  COMPLIANCE_GATE_PCT: 80\n\n" +
                $"Path: {path}",
            "params" =>
                "FORMAT: Revit Shared Parameter .txt\n" +
                "  *GROUP ID NAME  →  GROUP 1 GroupName\n" +
                "  *PARAM GUID NAME DATATYPE DATACATEGORY GROUP VISIBLE DESC\n" +
                "  PARAM <guid> ASS_MY_PARAM TEXT  1 1 My description\n\n" +
                "After adding params, run 'Load Shared Params' to bind them.\n\n" +
                $"Path: {path}",
            "materials" =>
                "FORMAT: JSON object with material entries:\n" +
                "  {\"MaterialName\": {\"category\": \"Ducts\",\n" +
                "    \"unit_cost\": 45.0, \"unit\": \"m2\", \"spec\": \"...\"}}\n\n" +
                $"Path: {path}",
            _ => $"Path: {path}\n\nEdit and save, then click Sync."
        };
    }
```

## FIX 13B — Dispatch + button

**StingCommandHandler.cs** — add after `case "ConfigEditor":`:
```csharp
                    case "GuidedDataEditor": RunCommand<Tags.GuidedDataEditorCommand>(app); break;
```

**StingDockPanel.xaml** — find the ConfigEditor button and add after it:
```xml
<Button Style="{StaticResource ActionBtn}" Content="Edit Data Files" Tag="GuidedDataEditor"
        Click="Cmd_Click"
        ToolTip="Guided editor for project_config.json, MR_PARAMETERS.txt, materials, and all data files — with sync"/>
```


---

# ══════════════════════════════════════════════════════════════════════
# SECTION 14 — AUTOTAGGER SETTINGS PERSISTENCE ACROSS SESSIONS
# Files: StingAutoTagger.cs, TagConfig.cs
# Root cause: VisualTagging and StaleMarker booleans reset every Revit session.
# ══════════════════════════════════════════════════════════════════════

## FIX 14A — StingAutoTagger.cs: Persist SetVisualTagging to project_config.json

**Locate in StingAutoTagger.cs** (search for this exact text):
```csharp
        public static void SetVisualTagging(bool enabled) { _visualTaggingEnabled = enabled; }
```

**Replace with:**
```csharp
        public static void SetVisualTagging(bool enabled)
        {
            _visualTaggingEnabled = enabled;
            // FIX-14A: Persist across sessions
            try
            {
                string p = StingToolsApp.FindDataFile("project_config.json");
                if (!string.IsNullOrEmpty(p))
                {
                    string json = System.IO.File.Exists(p) ? System.IO.File.ReadAllText(p) : "{}";
                    var data = Newtonsoft.Json.JsonConvert
                        .DeserializeObject<System.Collections.Generic.Dictionary<string, object>>(json)
                        ?? new System.Collections.Generic.Dictionary<string, object>();
                    data["AUTO_TAGGER_VISUAL"] = enabled;
                    System.IO.File.WriteAllText(p,
                        Newtonsoft.Json.JsonConvert.SerializeObject(data,
                            Newtonsoft.Json.Formatting.Indented));
                }
            }
            catch (Exception ex) { StingLog.Warn($"SetVisualTagging persist: {ex.Message}"); }
        }
```

## FIX 14B — TagConfig.cs: Restore AutoTagger settings on config load

**Locate in TagConfig.cs** the line that sets `ComplianceGatePct` (approximately where
config values are parsed from JSON, search for):
```csharp
                ComplianceGatePct = data.TryGetValue("COMPLIANCE_GATE_PCT",
```

**After the `ComplianceGatePct` block, add:**
```csharp
                // FIX-14B: Restore auto-tagger runtime state from project_config
                if (data.TryGetValue("AUTO_TAGGER_VISUAL", out object _avtObj)
                    && _avtObj is bool _avtBool)
                {
                    try { Core.StingAutoTagger.SetVisualTagging(_avtBool); }
                    catch { }
                }
```

---

# ══════════════════════════════════════════════════════════════════════
# VERIFICATION CHECKLIST
# After applying ALL sections, run these checks.
# Report actual grep results in the Completion Report below.
# ══════════════════════════════════════════════════════════════════════

```
VERIFICATION COMMANDS (run from /mnt/project/):

# Section 1 — ExcelLink pipeline
grep -c "TypeTokenInherit\|string gridRef" ExcelLinkCommands.cs
# Expected: 4

# Section 2 — Theme DynamicResource
grep -c "DynamicResource" StingDockPanel.xaml
# Expected: ≥ 20

# Section 3 — Slider connections
grep -c "SetLeaderElbowParams\|SetTagStyleParams" StingDockPanel_xaml.cs
# Expected: ≥ 4 (declarations + calls)
grep -c "GetExtraParam.*ElbowMode\|FIX-3C" SmartTagPlacementCommand.cs
# Expected: ≥ 1

# Section 4 — Per-export folder
grep -c "PromptForExportPath" OutputLocationHelper.cs OperationsCommands.cs TagOperationCommands.cs
# Expected: ≥ 4 (definition + 3+ call sites)

# Section 5 — Crash fixes
grep -c "commandData.Application.ActiveUIDocument" RevisionManagementCommands.cs
# Expected: 0
grep -c "commandData.Application.ActiveUIDocument" IoTMaintenanceCommands.cs
# Expected: 0
grep -c "commandData.Application.ActiveUIDocument" DataPipelineEnhancementCommands.cs
# Expected: 0

# Section 6 — IoT dispatch
grep -c "AssetCondition\|MaintenanceSchedule\|DigitalTwin\|EnergyAnalysis" StingCommandHandler.cs
# Expected: ≥ 4
grep -c "AssetCondition\|MaintenanceSchedule\|DigitalTwin" StingDockPanel.xaml
# Expected: ≥ 3

# Section 7 — NLP functional
grep -c "ProcessQuery\|ResolveCommandPublic\|StingListPicker" NLPCommandProcessor.cs
# Expected: ≥ 2
grep -c "ResolveCommandPublic\|SystemParamPush.*BatchSystemPush" WorkflowEngine.cs
# Expected: ≥ 2

# Section 8 — ProSheets PDF
grep -c "class ProSheetsPDFCommand" OperationsCommands.cs
# Expected: 1
grep -c "ProSheetsPDF" StingCommandHandler.cs StingDockPanel.xaml
# Expected: ≥ 2

# Section 9 — PurgeSharedParams
grep -c "class PurgeSharedParamsCommand" LoadSharedParamsCommand.cs
# Expected: 1
grep -c "PurgeSharedParams" StingCommandHandler.cs StingDockPanel.xaml
# Expected: ≥ 2

# Section 10 — FamilyParamCreator purge
grep -c "PurgeFirst\|RemoveParameter\|FIX-10" FamilyParamCreatorCommand.cs
# Expected: ≥ 3

# Section 11 — MagicRename
grep -c "class MagicRenameCommand" ViewAutomationCommands.cs
# Expected: 1

# Section 12 — ViewTabColour
grep -c "class ViewTabColourCommand" ViewAutomationCommands.cs
# Expected: 1

# Section 13 — GuidedDataEditor
grep -c "class GuidedDataEditorCommand" ConfigEditorCommand.cs
# Expected: 1

# Section 14 — AutoTagger persistence
grep -c "AUTO_TAGGER_VISUAL" StingAutoTagger.cs TagConfig.cs
# Expected: ≥ 2
```

---

# ══════════════════════════════════════════════════════════════════════
# COMPLETION REPORT
# Fill in after applying all sections.
# ══════════════════════════════════════════════════════════════════════

```
SECTION  DESCRIPTION                          STATUS      FILES MODIFIED          LINES
───────  ───────────────────────────────────  ──────────  ──────────────────────  ─────
S01      ExcelLink TypeTokenInherit+GridRef   [ APPLIED ]  ExcelLinkCommands.cs    ___
S02      Theme DynamicResource system         [ APPLIED ]  ThemeManager+XAML+cs    ___
S03      Leader/Elbow sliders connected       [ APPLIED ]  StingDockPanel_xaml+SmartTag ___
S04      Per-export PromptForExportPath       [ APPLIED ]  OutputLocationHelper+Ops+TagOp ___
S05A     RevisionManagement crash fix (14)    [ APPLIED ]  RevisionManagementCommands.cs ___
S05B     IoTMaintenance crash fix (10)        [ APPLIED ]  IoTMaintenanceCommands.cs ___
S05C     DataPipelineEnhancement crash fix    [ APPLIED ]  DataPipelineEnhancementCommands.cs ___
S06A     IoT dispatch entries (10)            [ APPLIED ]  StingCommandHandler.cs  ___
S06B     DataPipeline dispatch entries (7)    [ APPLIED ]  StingCommandHandler.cs  ___
S06C     IoT + DataPipeline XAML buttons      [ APPLIED ]  StingDockPanel.xaml     ___
S07A     NLP functional execution             [ APPLIED ]  NLPCommandProcessor.cs  ___
S07B     WorkflowEngine.ResolveCommandPublic  [ APPLIED ]  WorkflowEngine.cs       ___
S07C     WorkflowEngine 20 missing tags       [ APPLIED ]  WorkflowEngine.cs       ___
S08A     PDF dispatch to ProSheetsPDF         [ APPLIED ]  StingCommandHandler.cs  ___
S08B     PDF XAML upgrade                     [ APPLIED ]  StingDockPanel.xaml     ___
S08C     ProSheetsPDFCommand class            [ APPLIED ]  OperationsCommands.cs   ___
S09      PurgeSharedParamsCommand             [ APPLIED ]  LoadSharedParamsCommand.cs ___
S10      FamilyParamCreator purge+folder      [ APPLIED ]  FamilyParamCreatorCommand.cs ___
S11      MagicRenameCommand                   [ APPLIED ]  ViewAutomationCommands.cs ___
S12      ViewTabColourCommand                 [ APPLIED ]  ViewAutomationCommands.cs ___
S13      GuidedDataEditorCommand              [ APPLIED ]  ConfigEditorCommand.cs  ___
S14      AutoTagger settings persistence      [ APPLIED ]  StingAutoTagger+TagConfig.cs ___

VERIFICATION RESULTS
────────────────────────────────────────────────────────
ExcelLink TypeTokenInherit count:    ___ (expect 4)
DynamicResource in XAML count:       ___ (expect ≥20)
Slider helpers count:                ___ (expect ≥4)
Crash lines RevisionMgmt:            ___ (expect 0)
Crash lines IoTMaintenance:          ___ (expect 0)
IoT dispatch count:                  ___ (expect ≥4)
NLP StingListPicker usage:           ___ (expect ≥1)
ProSheetsPDFCommand present:         ___ (expect 1)
PurgeSharedParamsCommand present:    ___ (expect 1)
MagicRenameCommand present:          ___ (expect 1)
ViewTabColourCommand present:        ___ (expect 1)
GuidedDataEditorCommand present:     ___ (expect 1)

TAGGING PIPELINE MATRIX — EXPECTED FINAL STATE
────────────────────────────────────────────────────────
AutoTag, BatchTag, TagAndCombine:            18/18 FULL ✅
StingAutoTagger:                             FULL ✅
TagSelected, ReTag, RetagStale:              FULL ✅
FullAutoPopulate, ResolveAllIssues:          FULL ✅
BuildTagsCommand:                            FULL ✅ (FIX-B01)
SystemParamPush, RepairDuplicateSeq:         FULL ✅
ExcelLinkImport (both paths):                FULL ✅ (after S01)
FamilyStagePopulate:                         TOKEN-ONLY (by design)
BulkParamWrite:                              BARE (by design)

PIPELINE: 100% ✅
```

