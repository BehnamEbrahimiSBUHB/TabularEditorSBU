# TOMWrapperTest (Core Library Tests)

## Package Identity
**Purpose**: Unit tests for TOMWrapper library (core business logic)  
**Framework**: NUnit 3.x  
**Focus**: Model manipulation, serialization, undo/redo, dependency tracking  
**Test Runner**: NUnit Console, Visual Studio Test Explorer, or `dotnet test`

## Setup & Run

```powershell
# Build test project
msbuild TOMWrapperTest/TOMWrapperTest.csproj /p:Configuration=Debug

# Run all tests
dotnet test TOMWrapperTest/TOMWrapperTest.csproj

# Run specific test class
dotnet test TOMWrapperTest/TOMWrapperTest.csproj --filter "FullyQualifiedName~UndoManagerTests"

# Run tests for specific compatibility level
dotnet test TOMWrapperTest/TOMWrapperTest.csproj --filter "FullyQualifiedName~CL1200"

# Debug a test in Visual Studio
# Right-click test in Test Explorer > Debug Selected Tests
```

## Patterns & Conventions

### File Organization
```
TOMWrapperTest/
??? Constants.cs                      Test constants (model paths, etc.)
??? TabularModelHandlerTests.cs      Tests for model loading/saving
??? UndoManagerTests.cs              Tests for undo/redo framework
??? ObjectCreationTests.cs           Tests for object creation/deletion
??? ObjectHandlingTests.cs           Tests for object manipulation
??? DeserializerTests.cs             Tests for JSON/TMDL deserialization
??? FixUpTests.cs                    Tests for formula fixup (rename/move)
??? DatabaseHelperTest.cs            Tests for database operations
??? ExternalChangeTests.cs           Tests for external change detection
??? PowerBITests.cs                  Tests for Power BI specific features
??? PropertyChangeTestGenerator.cs   Auto-generated property change tests
??? PropertyChangeTestGenerator.tt   T4 template for test generation
??? TestGenerator.cs                 Auto-generated tests
??? TestGenerator.tt                 T4 template for test generation
```

### Test Class Structure
Standard NUnit pattern with compatibility level attributes:

**Example** (see `UndoManagerTests.cs`):
```csharp
[TestFixture]
public class UndoManagerTests
{
    private TabularModelHandler handler;

    [SetUp]
    public void SetUp()
    {
        // Create empty model for testing
        handler = new TabularModelHandler(1200); // Compatibility level 1200
    }

    [Test]
    public void TestUndo()
    {
        // Arrange
        var table = handler.Model.AddTable("TestTable");
        var originalCount = handler.Model.Tables.Count;
        
        // Act
        handler.UndoManager.Undo();
        
        // Assert
        Assert.That(handler.Model.Tables.Count, Is.EqualTo(originalCount - 1));
    }

    [TearDown]
    public void TearDown()
    {
        handler?.Dispose();
    }
}
```

### T4 Generated Tests
This project uses T4 templates to auto-generate repetitive tests:

**Pattern**:
- **`TestGenerator.tt`** - Generates tests for all wrapper classes
- **`PropertyChangeTestGenerator.tt`** - Generates property change tests

**Regenerate** (when wrapper classes change):
```powershell
# In Visual Studio:
# Right-click TestGenerator.tt ? Run Custom Tool
# Right-click PropertyChangeTestGenerator.tt ? Run Custom Tool
```

### Compatibility Level Tests
Tests can target specific compatibility levels:

**Example**:
```csharp
[Test]
[CompatibilityLevel(1200)] // Test only on CL 1200
public void TestFeatureForCL1200()
{
    // Test logic
}

[Test]
[CompatibilityLevel(1400, 1500)] // Test on CL 1400 and 1500
public void TestFeatureForHigherCL()
{
    // Test logic
}
```

### Test Data Setup
Create models programmatically (no external .bim files needed):

**Example**:
```csharp
[SetUp]
public void SetUp()
{
    handler = new TabularModelHandler(1200);
    
    // Create test data
    var table = handler.Model.AddTable("TestTable");
    var column = table.AddDataColumn("TestColumn");
    var measure = table.AddMeasure("TestMeasure", "SUM('TestTable'[TestColumn])");
}
```

## Touch Points / Key Files

### Core Test Classes
- **`TabularModelHandlerTests.cs`** - Model loading, saving, serialization
- **`UndoManagerTests.cs`** - Undo/redo functionality
- **`ObjectCreationTests.cs`** - Creating tables, measures, columns, etc.
- **`ObjectHandlingTests.cs`** - Modifying, deleting, moving objects

### Serialization Tests
- **`DeserializerTests.cs`** - JSON/TMDL deserialization
- **`DatabaseHelperTest.cs`** - Database connection and deployment

### Advanced Tests
- **`FixUpTests.cs`** - Formula fixup when renaming/moving objects
- **`ExternalChangeTests.cs`** - Detecting external changes to model
- **`PowerBITests.cs`** - Power BI specific features and constraints

### Generated Tests
- **`PropertyChangeTestGenerator.cs`** - Auto-generated property change tests (DO NOT EDIT)
- **`TestGenerator.cs`** - Auto-generated wrapper tests (DO NOT EDIT)

## JIT Index Hints

```powershell
# Find all test classes
Get-ChildItem TOMWrapperTest -Filter "*Tests.cs"

# Find all tests
Get-ChildItem TOMWrapperTest -Filter "*.cs" | Select-String -Pattern "\[Test\]"

# Find T4 templates
Get-ChildItem TOMWrapperTest -Filter "*.tt"

# Find generated test files
Get-ChildItem TOMWrapperTest -Filter "*Generator.cs"

# Find tests for specific compatibility level
Get-ChildItem TOMWrapperTest -Filter "*.cs" | Select-String -Pattern "\[CompatibilityLevel"

# Find undo/redo tests
Select-String -Path "TOMWrapperTest/UndoManagerTests.cs" -Pattern "\[Test\]"
```

## Common Gotchas

### 1. Generated Test Files
Never edit `PropertyChangeTestGenerator.cs` or `TestGenerator.cs` directly:
```csharp
// ? BAD: Edit PropertyChangeTestGenerator.cs
[Test]
public void TestMeasureNameChange() { /* changes here */ }

// ? GOOD: Edit PropertyChangeTestGenerator.tt and regenerate
// Or add custom tests in separate file
```

### 2. Compatibility Levels
Some features only work on certain compatibility levels:
```csharp
// ? BAD: Test feature not available on CL 1200
[Test]
public void TestCalculationGroups()
{
    var table = handler.Model.AddCalculationGroupTable(); // Fails on CL 1200!
}

// ? GOOD: Use CompatibilityLevel attribute
[Test]
[CompatibilityLevel(1470)] // Calculation groups require CL 1470+
public void TestCalculationGroups()
{
    var table = handler.Model.AddCalculationGroupTable();
}
```

### 3. Undo/Redo State
Undo manager must be in correct state for tests:
```csharp
// ? BAD: Undo stack might be dirty from previous test
[Test]
public void TestUndo()
{
    handler.UndoManager.Undo(); // Might undo from previous test!
}

// ? GOOD: Clear undo stack in SetUp
[SetUp]
public void SetUp()
{
    handler = new TabularModelHandler(1200);
    handler.UndoManager.Clear(); // Start with clean slate
}
```

### 4. Dependency Tracking
When testing formula fixup, ensure dependencies are tracked:
```csharp
[Test]
public void TestRename()
{
    var measure1 = handler.Model.Tables[0].AddMeasure("Measure1", "1");
    var measure2 = handler.Model.Tables[0].AddMeasure("Measure2", "[Measure1] + 1");
    
    // Rename Measure1
    measure1.Name = "RenamedMeasure";
    
    // Formula should be fixed up
    Assert.That(measure2.Expression, Does.Contain("RenamedMeasure"));
}
```

### 5. Serialization Roundtrip
Test that objects survive serialization/deserialization:
```csharp
[Test]
public void TestSerializationRoundtrip()
{
    var table = handler.Model.AddTable("TestTable");
    table.AddDataColumn("TestColumn");
    
    // Serialize to JSON
    var json = handler.Serialize();
    
    // Deserialize into new handler
    var handler2 = new TabularModelHandler();
    handler2.Deserialize(json);
    
    // Verify object still exists
    Assert.That(handler2.Model.Tables["TestTable"], Is.Not.Null);
}
```

## Pre-PR Checks

```powershell
# Run all tests
dotnet test TOMWrapperTest/TOMWrapperTest.csproj

# Run tests with verbose output
dotnet test TOMWrapperTest/TOMWrapperTest.csproj --logger "console;verbosity=detailed"

# If T4 templates modified, regenerate:
# Visual Studio: Right-click .tt files ? Run Custom Tool

# Run tests for all compatibility levels
dotnet test TOMWrapperTest/TOMWrapperTest.csproj --filter "FullyQualifiedName~CL1200"
dotnet test TOMWrapperTest/TOMWrapperTest.csproj --filter "FullyQualifiedName~CL1400"
dotnet test TOMWrapperTest/TOMWrapperTest.csproj --filter "FullyQualifiedName~CL1500"

# Manual test checklist:
# [ ] All tests pass
# [ ] Undo/redo tests pass
# [ ] Serialization roundtrip tests pass
# [ ] Formula fixup tests pass
# [ ] No external dependencies (SSAS, network)
```

## Anti-Patterns

? **DON'T**: Edit T4 generated test files directly  
? **DO**: Modify `.tt` templates and regenerate

? **DON'T**: Test features without checking compatibility level  
? **DO**: Use `[CompatibilityLevel]` attribute for version-specific features

? **DON'T**: Share handler instances between tests  
? **DO**: Create fresh handler in `[SetUp]` for each test

? **DON'T**: Ignore undo/redo in tests  
? **DO**: Test that changes are undoable/redoable

? **DON'T**: Test only happy paths  
? **DO**: Test error cases, edge cases, invalid inputs

## Examples to Copy

### Basic object creation test
**Pattern**: `ObjectCreationTests.cs`
```csharp
[Test]
public void TestCreateTable()
{
    // Arrange
    var tableName = "NewTable";
    
    // Act
    var table = handler.Model.AddTable(tableName);
    
    // Assert
    Assert.That(table, Is.Not.Null);
    Assert.That(table.Name, Is.EqualTo(tableName));
    Assert.That(handler.Model.Tables.Contains(tableName), Is.True);
}
```

### Undo/redo test
**Pattern**: `UndoManagerTests.cs`
```csharp
[Test]
public void TestUndoRedo()
{
    // Arrange
    var table = handler.Model.AddTable("TestTable");
    var originalCount = handler.Model.Tables.Count;
    
    // Act - Undo
    handler.UndoManager.Undo();
    Assert.That(handler.Model.Tables.Count, Is.EqualTo(originalCount - 1));
    
    // Act - Redo
    handler.UndoManager.Redo();
    Assert.That(handler.Model.Tables.Count, Is.EqualTo(originalCount));
}
```

### Formula fixup test
**Pattern**: `FixUpTests.cs`
```csharp
[Test]
public void TestRenameFixup()
{
    // Arrange
    var measure1 = handler.Model.Tables[0].AddMeasure("Measure1", "1");
    var measure2 = handler.Model.Tables[0].AddMeasure("Measure2", "[Measure1] + 1");
    
    // Act - Rename Measure1
    measure1.Name = "RenamedMeasure";
    
    // Assert - Measure2 formula should be updated
    Assert.That(measure2.Expression, Does.Contain("[RenamedMeasure]"));
}
```

### Serialization test
**Pattern**: `DeserializerTests.cs`
```csharp
[Test]
public void TestSerialize()
{
    // Arrange
    var table = handler.Model.AddTable("TestTable");
    table.AddDataColumn("TestColumn");
    
    // Act
    var json = handler.SerializeToJson();
    
    // Assert
    Assert.That(json, Is.Not.Null);
    Assert.That(json, Does.Contain("TestTable"));
    Assert.That(json, Does.Contain("TestColumn"));
}
```

### Compatibility level test
**Pattern**: `PowerBITests.cs`
```csharp
[Test]
[CompatibilityLevel(1470)]
public void TestCalculationGroups()
{
    // Arrange & Act
    var table = handler.Model.AddCalculationGroupTable("CalcGroup");
    
    // Assert
    Assert.That(table, Is.Not.Null);
    Assert.That(table.IsCalculationGroup, Is.True);
}
```

### Property change test (auto-generated pattern)
**Pattern**: `PropertyChangeTestGenerator.cs` (generated from .tt)
```csharp
[Test]
public void TestMeasureNameChange()
{
    // Arrange
    var measure = handler.Model.Tables[0].AddMeasure("TestMeasure");
    var originalName = measure.Name;
    
    // Act
    measure.Name = "NewName";
    
    // Assert
    Assert.That(measure.Name, Is.EqualTo("NewName"));
    
    // Undo
    handler.UndoManager.Undo();
    Assert.That(measure.Name, Is.EqualTo(originalName));
}
```

## T4 Template Usage

### Regenerating Tests
When wrapper classes change, regenerate tests:

1. **Edit wrapper class** (e.g., add new property to `Measure.cs`)
2. **Regenerate T4 templates**:
   - Right-click `TestGenerator.tt` ? Run Custom Tool
   - Right-click `PropertyChangeTestGenerator.tt` ? Run Custom Tool
3. **Run tests** to verify new properties are tested

### T4 Template Structure
**Pattern**: `TestGenerator.tt`
```
<#@ template ... #>
<#@ output extension=".cs" #>
<#
    // Template logic to generate tests
    foreach (var wrapperClass in wrapperClasses)
    {
#>
[Test]
public void Test<#= wrapperClass.Name #>Creation()
{
    // Generated test code
}
<#
    }
#>
```

## NUnit Resources
- [NUnit Documentation](https://docs.nunit.org/)
- [NUnit Assertions](https://docs.nunit.org/articles/nunit/writing-tests/assertions/assertion-models/constraint.html)
- [NUnit Attributes](https://docs.nunit.org/articles/nunit/writing-tests/attributes.html)
