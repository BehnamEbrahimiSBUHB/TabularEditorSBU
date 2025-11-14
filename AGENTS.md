# Tabular Editor 2.x - AI Agent Guide

## Project Snapshot
**Type**: Single-solution .NET Framework 4.8 WinForms desktop application  
**Purpose**: Open-source tool for editing SQL Server Analysis Services Tabular and Power BI XMLA models  
**Stack**: C#, WinForms, ANTLR4, Microsoft.AnalysisServices, NUnit  
**Sub-projects**: Each major project has its own AGENTS.md for detailed patterns

## Root Setup Commands

```powershell
# Restore NuGet packages
nuget restore TabularEditor.sln

# Build entire solution (Debug)
msbuild TabularEditor.sln /p:Configuration=Debug

# Build entire solution (Release)
msbuild TabularEditor.sln /p:Configuration=Release

# Run tests (requires VS Test or dotnet CLI)
dotnet test TabularEditor.sln

# Alternative: Open in Visual Studio and press F5
```

## Universal Conventions

### Code Style
- **Formatting**: Enforced via `.editorconfig` (4 spaces, CRLF, PascalCase)
- **C# Version**: C# 10.0 (TabularEditor), C# 12.0 (TOMWrapper)
- **Namespaces**: Block-scoped (`namespace TabularEditor { ... }`)
- **var usage**: Explicit types preferred (except when type is apparent)

### Commit Format
- No strict convention enforced
- Use descriptive commit messages referencing issue numbers when applicable

### Branch Strategy
- **main/master**: Stable releases
- **dev-ui-new**: Current development branch (active)
- Create feature branches from active dev branch

### PR Requirements
- Code must compile without errors
- Relevant tests should pass
- Follow existing code style (enforced by .editorconfig)

## Security & Secrets
- **Never commit**: Database connection strings, API keys, certificates
- **Secrets location**: Not stored in repo; use environment variables or secure vaults
- **Sensitive data**: Connection passwords handled via secure prompts at runtime
- **PII**: No user data collected or stored by default

## JIT Index (Directory Map)

### Project Structure
```
TabularEditor.sln                  Root solution file
??? TabularEditor/                 Main WinForms UI application ? [see TabularEditor/AGENTS.md]
?   ??? UI/                        Forms, dialogs, tree views
?   ??? Scripting/                 C# scripting engine and API
?   ??? BestPracticeAnalyzer/      BPA UI components
?   ??? PropertyGrid/              Custom property grid controls
?   ??? TreeViewAdv/               Advanced tree view control
??? TOMWrapper/                    Core TOM wrapper library ? [see TOMWrapper/AGENTS.md]
?   ??? TOMWrapper/                Main wrapper classes (Model, Table, Measure, etc.)
?   ??? Serialization/             JSON/TMDL serializers
?   ??? UndoFramework/             Undo/redo system
?   ??? PropertyGridUI/            Property grid type converters & editors
?   ??? Utils/                     DAX parsing, dependency analysis
??? AntlrGrammars/                 ANTLR4 grammars ? [see AntlrGrammars/AGENTS.md]
??? TabularEditorTest/             Tests for main app ? [see TabularEditorTest/AGENTS.md]
??? TOMWrapperTest/                Tests for TOM wrapper ? [see TOMWrapperTest/AGENTS.md]
??? TabularEditorInstaller/        MSI installer project (VSInstaller)
??? BPALib.csproj                  Best Practice Analyzer library (standalone)
```

### Quick Find Commands

```powershell
# Find a class definition
Get-ChildItem -Recurse -Filter "*.cs" | Select-String -Pattern "class YourClassName"

# Find a method
Get-ChildItem -Recurse -Filter "*.cs" | Select-String -Pattern "void YourMethodName"

# Find usage of a type
Get-ChildItem -Recurse -Filter "*.cs" | Select-String -Pattern "YourTypeName"

# Find XAML/RESX resources
Get-ChildItem -Recurse -Filter "*.resx"

# Find T4 templates (code generation)
Get-ChildItem -Recurse -Filter "*.tt"
```

### Cross-Project Dependencies
- **TabularEditor** depends on **TOMWrapper** (UI consumes business logic)
- **TOMWrapper** depends on **AntlrGrammars** (DAX parsing)
- Test projects depend on their respective main projects

## Definition of Done

Before submitting a PR:
- [ ] Code compiles without errors (`msbuild TabularEditor.sln`)
- [ ] No new warnings introduced (check build output)
- [ ] Relevant unit tests pass (`dotnet test` or VS Test Explorer)
- [ ] Code follows .editorconfig style
- [ ] No T4 generated files (*.generated.cs) manually edited
- [ ] Undo/redo works for model changes (if applicable)

## Common Global Patterns

### T4 Code Generation
- **DO**: Modify `.tt` files to change generated code
- **DON'T**: Edit `*.generated.cs` files directly (overwritten on build)
- **Regenerate**: Right-click `.tt` file ? "Run Custom Tool" in Visual Studio

### Undo/Redo System
- All model changes must go through `TabularModelHandler.UndoManager`
- Use `UndoPropertyChangedAction`, `UndoAddRemoveAction`, etc.
- See `TOMWrapper/UndoFramework/` for examples

### Resource Files
- UI strings go in `.resx` files (e.g., `TabularEditor/Strings.resx`)
- Access via generated designer class (e.g., `Strings.SomeMessage`)

### Property Grid Customization
- Use type converters in `TOMWrapper/PropertyGridUI/Converters/`
- Use collection editors in `TOMWrapper/PropertyGridUI/CollectionEditors/`
- Apply attributes like `[TypeConverter]`, `[Editor]` to wrapper properties

## External Documentation
- [Official Docs](https://docs.tabulareditor.com/te2/)
- [Getting Started](https://docs.tabulareditor.com/te2/Getting-Started.html)
- [Advanced Scripting](https://docs.tabulareditor.com/te2/Advanced-Scripting.html)
