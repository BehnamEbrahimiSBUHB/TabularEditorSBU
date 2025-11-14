# Tabular Editor 2.x - AI Coding Instructions

## Architecture Overview

**Type**: .NET Framework 4.8 WinForms desktop application for editing Analysis Services Tabular & Power BI models  
**Pattern**: Wrapper-based architecture with undo/redo, UI separation via partial classes

### Three-Layer Architecture

```
TabularEditor (UI Layer)
    ?? UIController (orchestrator) - split via partial classes by concern
    ?? TreeView (model navigation)
    ?? ScriptEngine (Roslyn-based C# execution)
       ?
TOMWrapper (Business Logic Layer)
    ?? TabularModelHandler (entry point)
    ?? Wrapper classes (never modify MetadataObject directly!)
    ?? UndoManager (tracks ALL model changes)
    ?? Serialization (JSON/TMDL formats)
       ?
AntlrGrammars (Parsing Layer)
    ?? DAXLexer.g4 ? generated lexer for DAX dependency analysis
```

## Critical Architecture Principles

### 1. **Wrapper Pattern (NEVER bypass!)**
```csharp
// ? WRONG - Bypasses undo tracking
measure.MetadataObject.Name = "NewName";

// ? CORRECT - Tracked by UndoManager
measure.Name = "NewName";
```
**Why**: Every property change MUST flow through wrapper setters to register undo actions.  
**See**: `TOMWrapper/AGENTS.md` for wrapper implementation details.

### 2. **Undo/Redo Framework (ALL changes tracked)**
```csharp
// Batch operations MUST be wrapped:
Handler.BeginUpdate("Batch rename measures");
try {
    foreach (var m in Model.AllMeasures)
        m.Name = "New_" + m.Name;
} finally {
    Handler.EndUpdate();
}
```
**Why**: Each property change creates an undo action. Batching combines them.  
**See**: `TOMWrapper/UndoFramework/UndoManager.cs`

### 3. **UI Controller Partial Classes (by concern)**
```
UIController.cs                    - Core orchestration
UIController_Tree.cs               - Tree view interactions
UIController_Files.cs              - File operations
UIController_Database.cs           - Database connections
UIController_ExpressionEditor.cs   - DAX editor
UIController_ScriptEditor.cs       - Script editor
UIController_PropertyGrid.cs       - Property grid
```
**Pattern**: Add methods to appropriate partial class based on responsibility.  
**See**: `TabularEditor/UI/UIController*.cs`

### 4. **T4 Code Generation (DO NOT EDIT GENERATED)**
```
TOMWrapper/Base/WrapperBase.tt     ? WrapperBase.generated.cs (regenerate, never edit)
Utils/DaxToken.tt                  ? DaxToken.Generated.cs
```
**Action**: Modify `.tt` templates, then right-click ? "Run Custom Tool"  
**See**: `TOMWrapper/AGENTS.md` for T4 pattern details

## Essential Developer Workflows

### Building & Testing
```powershell
# Full build (use msbuild, NOT dotnet build)
msbuild TabularEditor.sln /t:Rebuild /p:Configuration=Debug

# Run all tests
dotnet test TabularEditor.sln

# Run specific test project
dotnet test TOMWrapperTest/TOMWrapperTest.csproj
```

### Loading a Model
```csharp
// From file
var handler = new TabularModelHandler(@"path\to\model.bim");

// From database
var handler = new TabularModelHandler("localhost", "AdventureWorks");

// New blank model
var handler = new TabularModelHandler(compatibilityLevel: 1200);
```

### Scripting API (exposed to users)
```csharp
// Top-level objects available in scripts:
Model         // The tabular model
Selected      // Currently selected objects in UI (UITreeSelection)
Output()      // Write to script output window
Info()        // Info message
Warning()     // Warning message
Error()       // Error message
```
**Implementation**: `TabularEditor/Scripting/ScriptApi.cs`  
**Compilation**: `TabularEditor/Scripting/ScriptEngine.cs`

## Project-Specific Conventions

### Naming & Organization
- **Wrapper classes**: Match TOM object names (`Measure`, `Table`, `Column`)
- **Collections**: Plural + "Collection" (`MeasureCollection`, `ColumnCollection`)
- **UI partials**: `UIController_<Concern>.cs`
- **Test files**: `*Tests.cs` for NUnit tests

### Dependency Tracking
```csharp
// When renaming objects, DAX formulas auto-update:
measure.Name = "NewName"; 
// Formula: "[OldName] + 1" ? "[NewName] + 1"
```
**How**: `TOMWrapper/Utils/FormulaFixup.cs` + `DaxDependencyHelper.cs`  
**Trigger**: Property changes rebuild dependency tree (`FormulaFixup.BuildDependencyTree()`)

### Serialization Formats
1. **JSON (Model.bim)**: Legacy, entire model in one file
2. **Folder structure**: JSON split by object type (see `SerializeOptions.Levels`)
3. **TMDL**: New text-based format (CL 1600+)

**Save logic**: `TabularModelHandler.Save()` ? `Serialization/Serializer.cs`

### Error Handling
```csharp
// Errors stored per object:
measure.ErrorMessage  // TOM-reported errors
table.ErrorMessage

// Centralized error list:
Handler.Errors  // IReadOnlyCollection<IErrorMessageObject>
```
**UI display**: `FormMain` shows error count in status bar

## Integration Points

### External Dependencies
- **Microsoft.AnalysisServices.Tabular**: TOM library (core model API)
- **ANTLR4**: DAX/C# lexing (`AntlrGrammars/`)
- **FastColoredTextBox**: Syntax-highlighted editors
- **Aga.Controls.TreeViewAdv**: Custom tree view control
- **Newtonsoft.Json**: JSON serialization

### Cross-Component Communication
```
User edits property
    ?
PropertyGrid ? UIController ? TabularModelHandler
    ?
Wrapper property setter ? UndoManager.Add(action)
    ?
Model.MetadataObject updated ? TOM API
    ?
TreeModel.OnNodesChanged() ? UI refresh
```

### Plugin System
```csharp
// Plugins implement IPlugin interface:
public interface IPlugin {
    void Init(TabularModelHandler handler);
    void RegisterActions(Action<string, Action> callback);
}
```
**Location**: External DLLs loaded from app folder  
**See**: `TabularEditor/Program.cs` for plugin loading

## Common Pitfalls & Anti-Patterns

### ? Editing MetadataObject Directly
```csharp
measure.MetadataObject.Expression = "SUM(...)"; // NO UNDO!
```
**Fix**: Use wrapper properties (`measure.Expression = "SUM(...)"`)

### ? Forgetting BeginUpdate/EndUpdate
```csharp
foreach (var m in Model.AllMeasures)
    m.Name = "New_" + m.Name; // 100 undo actions!
```
**Fix**: Wrap in `Handler.BeginUpdate()` / `Handler.EndUpdate()`

### ? Editing T4 Generated Files
```csharp
// WrapperBase.generated.cs
public partial class Measure { /* DON'T EDIT HERE */ }
```
**Fix**: Edit `WrapperBase.tt` and regenerate

### ? Not Handling Model Changes During Undo/Redo
```csharp
if (Handler.UndoManager.UndoInProgress) return; // Skip logic
```
**Why**: Events fire during undo/redo; avoid recursive operations

### ? Storing State in Wrapper Objects
```csharp
measure.MyCustomField = value; // LOST on undo/redo!
```
**Fix**: Store state in `TabularModelHandler` or external cache

## Essential Files to Understand

### Core Entry Points
- `TabularEditor/Program.cs` - Application entry, plugin loading
- `TOMWrapper/TabularModelHandler.cs` - Main model handler
- `TabularEditor/UI/UIController.cs` - UI orchestrator

### Key Patterns
- `TOMWrapper/UndoFramework/UndoManager.cs` - Undo/redo implementation
- `TOMWrapper/Serialization/Serializer.cs` - Save/load logic
- `TabularEditor/Scripting/ScriptEngine.cs` - C# script compilation

### Testing Examples
- `TOMWrapperTest/TabularModelHandlerTests.cs` - Handler tests
- `TOMWrapperTest/UndoManagerTests.cs` - Undo/redo tests
- `TabularEditorTest/ScriptEngineTests.cs` - Scripting tests

## Quick Reference Commands

```powershell
# Find wrapper class
Get-ChildItem TOMWrapper -Filter "*.cs" | Select-String -Pattern "class.*: TabularObject"

# Find UI controller methods
Get-ChildItem TabularEditor/UI -Filter "UIController*.cs" | Select-String -Pattern "public.*void"

# Find undo actions
Get-ChildItem TOMWrapper/UndoFramework -Filter "Undo*.cs"

# Find serialization logic
Select-String -Path "TOMWrapper/Serialization/Serializer.cs" -Pattern "Serialize|Deserialize"
```

## Additional Resources
- **Root AGENTS.md**: Repository overview and setup
- **TOMWrapper/AGENTS.md**: Core business logic patterns
- **TabularEditor/AGENTS.md**: UI patterns and scripting
- **Official Docs**: https://docs.tabulareditor.com/te2/
