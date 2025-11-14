# TabularEditor (Main UI Application)

## Package Identity
**Purpose**: WinForms desktop application for editing Analysis Services Tabular and Power BI models  
**Framework**: .NET Framework 4.8 WinForms  
**Output**: `TabularEditor.exe` (executable application)

## Setup & Run

```powershell
# Build this project only
msbuild TabularEditor/TabularEditor.csproj /p:Configuration=Debug

# Run from Visual Studio
# Set TabularEditor as startup project and press F5

# Run compiled binary
./TabularEditor/bin/Debug/TabularEditor.exe

# Run with command-line arguments (for testing)
./TabularEditor/bin/Debug/TabularEditor.exe "path\to\model.bim"
```

## Patterns & Conventions

### File Organization
```
TabularEditor/
??? FormMain.cs                   Main application window
??? Program.cs                    Entry point
??? UI/                           Core UI controllers and helpers
?   ??? UIController.cs           Main UI orchestrator
?   ??? UIController_*.cs         Partial classes for different UI areas
?   ??? Dialogs/                  All dialog forms
?   ??? Actions/                  Action classes (Cut/Copy/Paste/Undo/etc.)
?   ??? Extensions/               Control extensions (WatermarkTextBox, etc.)
??? Scripting/                    C# scripting engine
?   ??? ScriptEngine.cs           Roslyn-based script execution
?   ??? ScriptApi.cs              API exposed to scripts (Selected, Model, etc.)
?   ??? ScriptOutputForm.cs       Script output window
??? BestPracticeAnalyzer/         BPA UI
?   ??? BPAForm.cs                BPA results dialog
?   ??? BPAEditorForm.cs          BPA rule editor
?   ??? BPAManager.cs             BPA collection manager
??? TreeViewAdv/                  Custom tree view control (3rd party)
??? TreeViewAdvExtension/         Tabular-specific tree view extensions
??? PropertyGrid/                 Custom property grid
??? TextServices/                 DAX/C# syntax highlighting
```

### UI Controller Pattern
- **UIController** is the central orchestrator (singleton-like)
- Partial classes split by concern:
  - `UIController_Tree.cs` - Tree view interactions
  - `UIController_Files.cs` - File operations
  - `UIController_Database.cs` - Database connections
  - `UIController_ScriptEditor.cs` - Script editor
  - `UIController_ExpressionEditor.cs` - DAX editor
  - `UIController_PropertyGrid.cs` - Property grid
  - `UIController_Translations.cs` - Translation grid
  - `UIController_ContextMenu.cs` - Right-click menus

**Example**: To handle a new tree action, add to `UIController_Tree.cs`

### Scripting API
Scripts expose the following top-level objects:
- `Selected` - Currently selected objects
- `Model` - The tabular model
- `Info()`, `Warning()`, `Error()` - Output functions
- `Output()` - Generic output
- `SaveFile()`, `ReadFile()` - File I/O

**Example**: See `Scripting/ScriptApi.cs` for full API surface

### Forms & Dialogs
- ? **DO**: Inherit from `Form` or `UserControl`
- ? **DO**: Place in `UI/Dialogs/` directory
- ? **DO**: Use `.Designer.cs` for auto-generated UI code
- ? **DON'T**: Put business logic in `.Designer.cs`

**Example Form**:
```csharp
// UI/Dialogs/MyCustomDialog.cs
public partial class MyCustomDialog : Form
{
    public MyCustomDialog()
    {
        InitializeComponent();
    }

    private void btnOk_Click(object sender, EventArgs e)
    {
        // Handle logic here
        DialogResult = DialogResult.OK;
        Close();
    }
}
```

### Best Practice Analyzer
- Rules stored as JSON in annotations (see `BestPracticeAnalyzer/BestPracticeRule.cs`)
- Analyzer runs LINQ expressions against model objects
- UI displays results in `BPAForm.cs`

**Example**: Copy rule pattern from existing rules in BPA collections

### Resource Usage
- **UI Strings**: `Strings.resx` ? `Strings.SomeMessage`
- **Images**: `Properties/Resources.resx` ? `Properties.Resources.IconName`
- Embedded resources compiled into assembly

## Touch Points / Key Files

### Application Entry
- **`Program.cs`** - Main entry point, handles single instance, command-line args
- **`FormMain.cs`** - Main window, toolbar, menus, docking

### Core UI Logic
- **`UI/UIController.cs`** - Central UI orchestrator
- **`UI/UITreeSelection.cs`** - Tree selection state management
- **`UI/TabularUITree.cs`** - Tree population logic

### Scripting
- **`Scripting/ScriptEngine.cs`** - Roslyn-based C# script compilation/execution
- **`Scripting/ScriptApi.cs`** - API surface exposed to scripts
- **`Scripting/ScriptHelper.cs`** - Helper methods for scripts

### Property Grid
- **`PropertyGrid/NavigatablePropertyGrid.cs`** - Custom property grid with navigation

### DAX Editor
- **`TextServices/ExpressionParser.cs`** - DAX expression parsing
- **`UI/UIController_ExpressionEditor.cs`** - DAX editor UI logic

## JIT Index Hints

```powershell
# Find a form/dialog
Get-ChildItem UI/Dialogs -Filter "*.cs" -Recurse | Select-String -Pattern "class.*Form"

# Find an action handler
Get-ChildItem UI/Actions -Filter "*.cs" | Select-String -Pattern "class.*Action"

# Find script API methods
Select-String -Path "Scripting/ScriptApi.cs" -Pattern "public.*void|public.*string"

# Find UI controller methods
Get-ChildItem UI -Filter "UIController*.cs" | Select-String -Pattern "public.*void"

# Find tree view node types
Get-ChildItem TreeViewAdvExtension -Filter "*.cs" | Select-String -Pattern "class.*Node"
```

## Common Gotchas

### 1. Action Framework
All UI actions (Cut/Copy/Paste/Delete) go through `ITextBox` or `ModelAction` interfaces:
- Text actions: `UI/Actions/TextboxAction.cs`
- Model actions: `UI/Actions/ModelAction.cs`
- Register in `UI/Actions/ModelActionManager.cs`

### 2. Tree View Extensions
Custom tree node controls live in `TreeViewAdvExtension/`:
- `TabularIcon.cs` - Icon renderer
- `TabularName.cs` - Name renderer (with error icons)
- Always use `TreeNodeAdv` from `TreeViewAdv/` library

### 3. Undo/Redo
UI changes to model objects automatically tracked via `UndoManager` in `TOMWrapper`
- No manual undo actions needed for property changes
- Batch actions: Use `Handler.BeginUpdate()` / `Handler.EndUpdate()`

### 4. Scripting Security
Scripts run with full trust (no sandboxing):
- Warn users about untrusted scripts
- Scripts can access file system, network, etc.

### 5. WinForms Designer
- Always use Designer for UI layout (don't hand-code controls)
- Designer code goes in `.Designer.cs` (auto-generated)
- Custom logic goes in main `.cs` file

## Pre-PR Checks

```powershell
# Build TabularEditor project
msbuild TabularEditor/TabularEditor.csproj /p:Configuration=Debug

# Run UI tests (if any)
dotnet test TabularEditorTest/TabularEditorTest.csproj

# Manual smoke test checklist:
# [ ] Application launches without errors
# [ ] Can open a .bim file
# [ ] Can connect to SSAS/Power BI
# [ ] Tree view renders correctly
# [ ] Property grid shows selected object
# [ ] Script editor works (try a simple script)
# [ ] Undo/redo works
```

## Anti-Patterns

? **DON'T**: Bypass `UIController` and directly manipulate `FormMain` controls  
? **DO**: Add methods to `UIController` for UI state changes

? **DON'T**: Create model objects directly via `new Measure()`, etc.  
? **DO**: Use `Handler.Model.AddMeasure()` to ensure undo tracking

? **DON'T**: Hardcode strings in UI code  
? **DO**: Use resource files (`Strings.resx`)

? **DON'T**: Store state in static fields  
? **DO**: Use `UIController` instance fields or `Handler` context

## Examples to Copy

### Creating a new dialog
**Pattern**: `UI/Dialogs/DeployForm.cs`
- Shows dialog creation with validation
- Uses data binding to controls
- Implements `DialogResult` pattern

### Adding a tree context menu action
**Pattern**: `UI/UIController_ContextMenu.cs`
- Look for `BuildContextMenu()` method
- Add menu items conditionally based on selection

### Creating a custom action
**Pattern**: `UI/Actions/CreateRelationshipAction.cs`
- Shows how to create a custom action
- Integrates with undo/redo framework

### Scripting API extension
**Pattern**: `Scripting/ScriptApi.cs`
- Add public methods to expose to scripts
- Use `[ScriptMethod]` attribute for documentation
