# TOMWrapper (Tabular Object Model Wrapper Library)

## Package Identity
**Purpose**: Core business logic library wrapping Microsoft.AnalysisServices.Tabular APIs  
**Framework**: .NET Framework 4.8  
**Output**: `TOMWrapper.dll` (class library)  
**Key Features**: Model manipulation, serialization, undo/redo, dependency tracking

## Setup & Run

```powershell
# Build this project only
msbuild TOMWrapper/TOMWrapper.csproj /p:Configuration=Debug

# Run unit tests
dotnet test TOMWrapperTest/TOMWrapperTest.csproj

# Generate T4 templates (if modified)
# In Visual Studio: Right-click WrapperBase.tt ? Run Custom Tool
```

## Patterns & Conventions

### File Organization
```
TOMWrapper/
??? TOMWrapper/                   Core wrapper classes
?   ??? TabularModelHandler.cs    Main entry point (loads/saves models)
?   ??? TabularObject.cs          Base class for all wrappers
?   ??? TabularNamedObject.cs     Base for named objects (Table, Measure, etc.)
?   ??? Model.cs                  Model wrapper
?   ??? Table.cs                  Table wrapper
?   ??? Measure.cs                Measure wrapper
?   ??? Column.cs                 Column wrapper
?   ??? Relationship.cs           Relationship wrapper
?   ??? Base/WrapperBase.tt       T4 template for wrapper generation
?   ??? Indexers/                 Collection indexers (e.g., PerspectiveIndexer)
??? Serialization/                JSON/TMDL serializers
?   ??? Serializer.cs             Main serializer orchestrator
?   ??? SerializeOptions.cs       Serialization options
?   ??? TmdlSerializeOptions.cs   TMDL-specific options
?   ??? ObjectSerializer.cs       Per-object serialization logic
??? UndoFramework/                Undo/redo system
?   ??? UndoManager.cs            Central undo/redo manager
?   ??? IUndoAction.cs            Action interface
?   ??? UndoPropertyChangedAction.cs  Property change tracking
?   ??? UndoAddRemoveAction.cs    Object add/remove tracking
??? PropertyGridUI/               Property grid type converters & editors
?   ??? Converters/               Type converters (e.g., TableColumnConverter)
?   ??? CollectionEditors/        Collection editors (e.g., PartitionCollectionEditor)
??? TextServices/                 DAX parsing and lexing
?   ??? DAXLexer.cs               ANTLR-generated DAX lexer (linked from AntlrGrammars)
?   ??? DAXLexerExtensions.cs     Extensions for DAX token analysis
??? Utils/                        Utilities
    ??? DaxDependencyHelper.cs    DAX expression dependency analysis
    ??? FormulaFixup.cs           Rename/move fixup for DAX formulas
    ??? TabularDeployer.cs        Deployment to SSAS/Power BI
```

### Wrapper Pattern
Every Microsoft TOM object (e.g., `Microsoft.AnalysisServices.Tabular.Measure`) is wrapped:
- **Wrapper class**: `TabularEditor.TOMWrapper.Measure`
- **MetadataObject property**: Access underlying TOM object
- **Undo tracking**: All property changes go through `PropertyChanged` event ? `UndoManager`

**Example**:
```csharp
// DON'T: Directly modify TOM object
measure.MetadataObject.Expression = "SUM(...)"; // Bypasses undo tracking!

// DO: Use wrapper property
measure.Expression = "SUM(...)"; // Automatically tracked by UndoManager
```

### T4 Code Generation
- **Source template**: `TOMWrapper/Base/WrapperBase.tt`
- **Generated file**: `TOMWrapper/Base/WrapperBase.generated.cs` (DO NOT EDIT)
- **Regenerate**: Right-click `.tt` file ? "Run Custom Tool"

**Rules**:
```csharp
// Base/WrapperBase.tt
// ...existing code...
public partial class Measure : TabularNamedObject
{
    // Auto-generated properties and methods
}
```

### Undo/Redo Framework
All model changes must flow through `UndoManager`:

**Pattern**:
```csharp
// Property setter (auto-tracked)
public string Expression
{
    get => MetadataObject.Expression;
    set
    {
        var oldValue = Expression;
        if (oldValue == value) return;
        
        bool undoable = true;
        bool cancel = false;
        OnPropertyChanging("Expression", value, ref undoable, ref cancel);
        if (cancel) return;
        
        MetadataObject.Expression = value;
        
        if (undoable) Handler.UndoManager.Add(new UndoPropertyChangedAction(this, "Expression", oldValue, value));
        OnPropertyChanged("Expression", oldValue, value);
    }
}
```

**Batch operations**:
```csharp
Handler.BeginUpdate("Batch rename measures");
try
{
    foreach (var measure in Model.AllMeasures)
    {
        measure.Name = "New_" + measure.Name;
    }
}
finally
{
    Handler.EndUpdate();
}
```

### Serialization
Two main formats:
1. **JSON (Model.bim)**: Legacy format, folder-based serialization
2. **TMDL (Tabular Model Definition Language)**: New text-based format

**Serialization entry point**: `TabularModelHandler.Save()`

**Example** (see `Serialization/Serializer.cs`):
```csharp
// Save as folder structure
handler.Save(folder, SaveFormat.TabularEditorFolder, new SerializeOptions());

// Save as TMDL
handler.Save(folder, SaveFormat.TMDL, new TmdlSerializeOptions());
```

### Property Grid Customization
Type converters and editors live in `PropertyGridUI/`:

**Example type converter** (see `PropertyGridUI/Converters/TableColumnConverter.cs`):
```csharp
[TypeConverter(typeof(TableColumnConverter))]
public Column SortByColumn { get; set; }
```

**Example collection editor** (see `PropertyGridUI/CollectionEditors/PartitionCollectionEditor.cs`):
```csharp
[Editor(typeof(PartitionCollectionEditor), typeof(UITypeEditor))]
public PartitionCollection Partitions { get; }
```

## Touch Points / Key Files

### Core Handler
- **`TabularModelHandler.cs`** - Main entry point, model loading/saving
- **`TabularModelHandler.Database.cs`** - Database connection logic
- **`TabularModelHandler.FileHandling.cs`** - File I/O logic
- **`TabularModelHandler.Events.cs`** - Event handlers

### Wrapper Classes
- **`Model.cs`** - Root model object
- **`Table.cs`** - Table wrapper (includes calculated tables)
- **`Measure.cs`** - Measure wrapper
- **`Column.cs`** - Column wrapper (base for calculated columns)
- **`Relationship.cs`** - Relationship wrapper (base for single-column relationships)

### Serialization
- **`Serialization/Serializer.cs`** - Main serializer orchestrator
- **`Serialization/SplitModelSerializer.cs`** - Folder-based serialization
- **`Serialization/ObjectSerializer.cs`** - Per-object JSON serialization

### Undo Framework
- **`UndoFramework/UndoManager.cs`** - Undo/redo stack management
- **`UndoFramework/UndoPropertyChangedAction.cs`** - Property change tracking
- **`UndoFramework/UndoAddRemoveAction.cs`** - Object add/remove tracking

### Utilities
- **`Utils/DaxDependencyHelper.cs`** - Parse DAX and find dependencies
- **`Utils/FormulaFixup.cs`** - Rename/move fixup for DAX formulas
- **`Utils/TabularDeployer.cs`** - Deploy model to SSAS/Power BI

## JIT Index Hints

```powershell
# Find a wrapper class
Get-ChildItem TOMWrapper -Filter "*.cs" | Select-String -Pattern "class.*: TabularObject"

# Find property converters
Get-ChildItem PropertyGridUI/Converters -Filter "*.cs"

# Find undo actions
Get-ChildItem UndoFramework -Filter "Undo*.cs"

# Find serialization logic
Get-ChildItem Serialization -Filter "*.cs" | Select-String -Pattern "Serialize|Deserialize"

# Find dependency analysis
Select-String -Path "Utils/DaxDependencyHelper.cs" -Pattern "GetDependencies"

# Find DAX token types
Select-String -Path "TextServices/DAXLexerExtensions.cs" -Pattern "TokenType"
```

## Common Gotchas

### 1. Wrapper vs. MetadataObject
Always use wrapper properties, not `MetadataObject` directly:
```csharp
// ? BAD: Bypasses undo tracking
measure.MetadataObject.Name = "NewName";

// ? GOOD: Tracked by UndoManager
measure.Name = "NewName";
```

### 2. T4 Generated Code
Never edit `*.generated.cs` files:
```csharp
// ? BAD: Edit WrapperBase.generated.cs
public partial class Measure { /* changes here */ }

// ? GOOD: Edit WrapperBase.tt and regenerate
// Or create partial class in separate file
```

### 3. BeginUpdate/EndUpdate
Batch operations must be wrapped:
```csharp
// ? BAD: Each change creates undo action
foreach (var m in Model.AllMeasures)
    m.Name = "New_" + m.Name; // 100 undo actions!

// ? GOOD: One undo action for entire batch
Handler.BeginUpdate("Rename all measures");
try
{
    foreach (var m in Model.AllMeasures)
        m.Name = "New_" + m.Name;
}
finally
{
    Handler.EndUpdate();
}
```

### 4. Dependency Tracking
DAX formulas have dependencies (measures reference other measures):
- Use `DaxDependencyHelper.GetDependencies()` to find dependencies
- Use `FormulaFixup.BuildDependencyTree()` for rename operations

### 5. Annotations
Custom metadata stored as annotations (key-value pairs):
```csharp
measure.SetAnnotation("MyCustomKey", "MyValue");
string value = measure.GetAnnotation("MyCustomKey");
```

### 6. Power BI Constraints
Power BI has restrictions not in SSAS:
- Check `Model.Database.CompatibilityMode` (PowerBI vs. SSAS)
- Some features unavailable in Power BI (e.g., roles with RLS on DirectQuery)

## Pre-PR Checks

```powershell
# Build TOMWrapper
msbuild TOMWrapper/TOMWrapper.csproj /p:Configuration=Debug

# Run all TOMWrapper tests
dotnet test TOMWrapperTest/TOMWrapperTest.csproj

# If T4 templates modified, regenerate:
# Visual Studio: Right-click WrapperBase.tt ? Run Custom Tool

# Manual test checklist:
# [ ] Load a .bim file
# [ ] Make changes to model
# [ ] Undo changes
# [ ] Redo changes
# [ ] Save model (verify no corruption)
# [ ] Check dependency tree for measures
```

## Anti-Patterns

? **DON'T**: Modify `MetadataObject` properties directly  
? **DO**: Use wrapper properties (auto-tracked by undo framework)

? **DON'T**: Create wrapper objects manually (`new Measure()`)  
? **DO**: Use factory methods on collections (`table.Measures.Add()`)

? **DON'T**: Edit T4 generated files  
? **DO**: Modify `.tt` templates and regenerate

? **DON'T**: Forget `BeginUpdate/EndUpdate` for batch operations  
? **DO**: Wrap loops in `BeginUpdate/EndUpdate` to batch undo actions

? **DON'T**: Store state in wrapper objects (they're recreated on undo/redo)  
? **DO**: Store state in `TabularModelHandler` or external cache

## Examples to Copy

### Creating a new wrapper property
**Pattern**: `Measure.cs` (existing properties)
```csharp
[Category("Basic"), DisplayName("Expression")]
public string Expression
{
    get => MetadataObject.Expression;
    set => SetValue(Expression, value, v => MetadataObject.Expression = v);
}
```

### Adding a custom type converter
**Pattern**: `PropertyGridUI/Converters/TableColumnConverter.cs`
- Inherits from `TypeConverter`
- Overrides `GetStandardValues()` to provide dropdown options
- Apply with `[TypeConverter(typeof(YourConverter))]` attribute

### Adding a collection editor
**Pattern**: `PropertyGridUI/CollectionEditors/PartitionCollectionEditor.cs`
- Inherits from `CollectionEditor`
- Customize dialog behavior
- Apply with `[Editor(typeof(YourEditor), typeof(UITypeEditor))]`

### Serialization customization
**Pattern**: `Serialization/PerspectiveAnnotationSerializer.cs`
- Shows how to serialize custom annotations
- Implements custom JSON structure

### Undo action
**Pattern**: `UndoFramework/UndoPropertyChangedAction.cs`
- Implements `IUndoAction`
- Stores old/new values
- Implements `Undo()` and `Redo()` methods

### Dependency analysis
**Pattern**: `Utils/DaxDependencyHelper.cs`
- Parse DAX using `DAXLexer`
- Extract object references
- Build dependency graph
