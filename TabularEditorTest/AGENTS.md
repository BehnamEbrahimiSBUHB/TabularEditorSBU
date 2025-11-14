# TabularEditorTest (Main Application Tests)

## Package Identity
**Purpose**: Unit and integration tests for TabularEditor (main UI application)  
**Framework**: NUnit 3.x  
**Test Runner**: NUnit Console, Visual Studio Test Explorer, or `dotnet test`

## Setup & Run

```powershell
# Build test project
msbuild TabularEditorTest/TabularEditorTest.csproj /p:Configuration=Debug

# Run all tests (via dotnet CLI)
dotnet test TabularEditorTest/TabularEditorTest.csproj

# Run specific test class
dotnet test TabularEditorTest/TabularEditorTest.csproj --filter "FullyQualifiedName~ScriptEngineTests"

# Run in Visual Studio Test Explorer
# Test > Run All Tests (or Ctrl+R, A)

# Debug a single test
# Right-click test in Test Explorer > Debug Selected Tests
```

## Patterns & Conventions

### File Organization
```
TabularEditorTest/
??? AdventureWorks.bim        Sample model for testing
??? Constants.cs              Test constants (file paths, connection strings)
??? CLITests.cs               Command-line interface tests
??? ScriptEngineTests.cs      C# scripting engine tests
??? ScriptHelperTests.cs      Script helper method tests
??? UpdateServiceTests.cs     Update service tests
??? UiTester.cs               UI automation/smoke tests (if any)
```

### Test Class Structure
Standard NUnit pattern:

**Example** (see `ScriptEngineTests.cs`):
```csharp
[TestFixture]
public class ScriptEngineTests
{
    [SetUp]
    public void SetUp()
    {
        // Initialize test data, load model, etc.
    }

    [Test]
    public void TestScriptExecution()
    {
        // Arrange
        var script = "Selected.Measures[0].Name = 'TestMeasure';";
        
        // Act
        var result = scriptEngine.Execute(script);
        
        // Assert
        Assert.That(result.Success, Is.True);
    }

    [TearDown]
    public void TearDown()
    {
        // Clean up resources
    }
}
```

### Test Data
- **`AdventureWorks.bim`**: Sample Analysis Services model
- Used as test fixture for most tests
- Contains tables, measures, relationships for realistic testing

**Loading test model**:
```csharp
var modelPath = Path.Combine(Constants.TestDataPath, "AdventureWorks.bim");
var handler = new TabularModelHandler(modelPath);
```

### Mock/Stub Patterns
For UI components that can't run in headless tests:
- Mock `UIController` interactions
- Use `TabularModelHandler` directly (no UI)
- Avoid testing actual WinForms rendering (too brittle)

## Touch Points / Key Files

### Test Classes
- **`ScriptEngineTests.cs`** - Test C# scripting engine (Roslyn-based execution)
- **`ScriptHelperTests.cs`** - Test script helper methods (Selected, Model, Info, etc.)
- **`CLITests.cs`** - Test command-line argument parsing and execution
- **`UpdateServiceTests.cs`** - Test update checking logic

### Test Data
- **`AdventureWorks.bim`** - Sample tabular model
- **`Constants.cs`** - Test configuration (paths, connection strings)

### External Dependencies
- **TabularEditor.exe** - Main application (referenced as dependency)
- **TOMWrapper.dll** - Core library (referenced as dependency)

## JIT Index Hints

```powershell
# Find all test classes
Get-ChildItem TabularEditorTest -Filter "*Tests.cs"

# Find all tests
Get-ChildItem TabularEditorTest -Filter "*.cs" | Select-String -Pattern "\[Test\]"

# Find setup/teardown methods
Get-ChildItem TabularEditorTest -Filter "*.cs" | Select-String -Pattern "\[SetUp\]|\[TearDown\]"

# Find test fixtures
Get-ChildItem TabularEditorTest -Filter "*.cs" | Select-String -Pattern "\[TestFixture\]"

# View test model
Get-Content TabularEditorTest/AdventureWorks.bim
```

## Common Gotchas

### 1. UI Tests are Fragile
Avoid testing actual UI rendering:
```csharp
// ? BAD: Test actual form rendering (brittle, headless CI fails)
var form = new FormMain();
form.Show();
Assert.That(form.treeView.Nodes.Count, Is.GreaterThan(0));

// ? GOOD: Test underlying logic
var handler = new TabularModelHandler("test.bim");
Assert.That(handler.Model.Tables.Count, Is.GreaterThan(0));
```

### 2. Test Isolation
Each test should be independent:
```csharp
// ? BAD: Tests depend on each other
[Test]
public void Test1() { sharedState = "value"; }

[Test]
public void Test2() { Assert.That(sharedState, Is.EqualTo("value")); }

// ? GOOD: Each test sets up its own state
[Test]
public void Test1()
{
    var state = "value";
    Assert.That(state, Is.EqualTo("value"));
}

[Test]
public void Test2()
{
    var state = "value";
    Assert.That(state, Is.EqualTo("value"));
}
```

### 3. File Paths
Use `Constants.cs` for test data paths:
```csharp
// ? BAD: Hardcoded paths (breaks on different machines)
var modelPath = @"C:\MyFolder\test.bim";

// ? GOOD: Relative paths via Constants
var modelPath = Path.Combine(Constants.TestDataPath, "AdventureWorks.bim");
```

### 4. Async Tests
Mark async tests properly:
```csharp
[Test]
public async Task TestAsyncOperation()
{
    var result = await SomeAsyncMethod();
    Assert.That(result, Is.Not.Null);
}
```

### 5. Test Data Cleanup
Clean up temp files in `[TearDown]`:
```csharp
[TearDown]
public void TearDown()
{
    if (File.Exists(tempFilePath))
        File.Delete(tempFilePath);
}
```

## Pre-PR Checks

```powershell
# Run all tests
dotnet test TabularEditorTest/TabularEditorTest.csproj

# Run with detailed output
dotnet test TabularEditorTest/TabularEditorTest.csproj --logger "console;verbosity=detailed"

# Generate code coverage (if configured)
dotnet test TabularEditorTest/TabularEditorTest.csproj --collect:"XPlat Code Coverage"

# Manual test checklist:
# [ ] All tests pass
# [ ] No test dependencies on external services (SSAS/Power BI)
# [ ] Tests run in headless CI environment
# [ ] New features have corresponding tests
```

## Anti-Patterns

? **DON'T**: Test UI rendering directly (brittle, CI-hostile)  
? **DO**: Test business logic, use mocks for UI components

? **DON'T**: Write tests that depend on external services (SSAS, network)  
? **DO**: Mock external dependencies or skip tests with `[Ignore]` if needed

? **DON'T**: Use hardcoded file paths  
? **DO**: Use `Constants.cs` and relative paths

? **DON'T**: Share state between tests  
? **DO**: Make each test independent (use `[SetUp]` to initialize)

? **DON'T**: Leave temp files after tests  
? **DO**: Clean up in `[TearDown]`

## Examples to Copy

### Basic test structure
**Pattern**: `ScriptEngineTests.cs`
```csharp
[TestFixture]
public class MyFeatureTests
{
    private TabularModelHandler handler;

    [SetUp]
    public void SetUp()
    {
        var modelPath = Path.Combine(Constants.TestDataPath, "AdventureWorks.bim");
        handler = new TabularModelHandler(modelPath);
    }

    [Test]
    public void TestSomething()
    {
        // Arrange
        var measure = handler.Model.Tables[0].Measures[0];
        
        // Act
        measure.Name = "NewName";
        
        // Assert
        Assert.That(measure.Name, Is.EqualTo("NewName"));
    }

    [TearDown]
    public void TearDown()
    {
        handler?.Dispose();
    }
}
```

### Testing scripting engine
**Pattern**: `ScriptEngineTests.cs`
```csharp
[Test]
public void TestScriptExecution()
{
    var script = @"
        foreach(var m in Model.AllMeasures)
        {
            m.Description = 'Generated description';
        }
    ";
    
    var result = ScriptEngine.Execute(script, handler);
    
    Assert.That(result.Success, Is.True);
    Assert.That(handler.Model.AllMeasures.All(m => m.Description == "Generated description"), Is.True);
}
```

### Testing command-line interface
**Pattern**: `CLITests.cs`
```csharp
[Test]
public void TestCLIArguments()
{
    var args = new[] { "model.bim", "-S", "script.csx" };
    var result = CommandLineHandler.Parse(args);
    
    Assert.That(result.ModelPath, Is.EqualTo("model.bim"));
    Assert.That(result.ScriptPath, Is.EqualTo("script.csx"));
}
```

### Testing undo/redo
**Pattern**: Test undo framework
```csharp
[Test]
public void TestUndo()
{
    var measure = handler.Model.Tables[0].Measures[0];
    var originalName = measure.Name;
    
    measure.Name = "NewName";
    Assert.That(measure.Name, Is.EqualTo("NewName"));
    
    handler.UndoManager.Undo();
    Assert.That(measure.Name, Is.EqualTo(originalName));
}
```

## NUnit Attributes Reference

- `[TestFixture]` - Marks a class as a test fixture
- `[Test]` - Marks a method as a test
- `[SetUp]` - Runs before each test
- `[TearDown]` - Runs after each test
- `[OneTimeSetUp]` - Runs once before all tests in fixture
- `[OneTimeTearDown]` - Runs once after all tests in fixture
- `[Ignore("Reason")]` - Skips a test
- `[Category("CategoryName")]` - Groups tests by category
- `[TestCase(arg1, arg2)]` - Parameterized test

## NUnit Resources
- [NUnit Documentation](https://docs.nunit.org/)
- [NUnit Assertions](https://docs.nunit.org/articles/nunit/writing-tests/assertions/assertion-models/constraint.html)
