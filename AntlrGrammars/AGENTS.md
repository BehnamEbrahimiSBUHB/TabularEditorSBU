# AntlrGrammars (ANTLR4 Lexer/Parser Grammars)

## Package Identity
**Purpose**: ANTLR4 grammars for DAX (Data Analysis Expressions) and C# lexing  
**Framework**: .NET Framework 4.8  
**Output**: `AntlrGrammars.dll` (class library with generated lexers)  
**Key Dependency**: Antlr4.Runtime NuGet package

## Setup & Run

```powershell
# Build this project (auto-generates lexers from .g4 files)
msbuild AntlrGrammars/AntlrGrammars.csproj /p:Configuration=Debug

# If ANTLR grammars (.g4) are modified:
# 1. ANTLR tool automatically runs during build (via NuGet package)
# 2. Generated C# files appear in obj/Debug/

# Manual ANTLR generation (if needed)
java -jar antlr-4.x-complete.jar -Dlanguage=CSharp DAXLexer.g4
```

## Patterns & Conventions

### File Organization
```
AntlrGrammars/
??? DAXLexer.g4               DAX lexer grammar (source of truth)
??? CSharpLexer.g4            C# lexer grammar (for script editor)
??? obj/Debug/                Generated C# lexer files (auto-generated)
?   ??? DAXLexer.cs           Generated DAX lexer
?   ??? CSharpLexer.cs        Generated C# lexer
??? AntlrGrammars.csproj      Project file with ANTLR build tasks
```

### ANTLR Grammar Structure
ANTLR grammars define lexical tokens (keywords, operators, literals):

**Example** (from `DAXLexer.g4`):
```antlr
// Keywords
RETURN: 'RETURN';
EVALUATE: 'EVALUATE';
VAR: 'VAR';

// Operators
PLUS: '+';
MINUS: '-';
MULTIPLY: '*';
DIVIDE: '/';

// Literals
NUMBER: [0-9]+ ('.' [0-9]+)?;
STRING: '"' (~["])* '"';
IDENTIFIER: [a-zA-Z_][a-zA-Z0-9_]*;
```

### Generated Lexer Usage
Generated lexers are used in `TOMWrapper` for DAX parsing:

**Example** (see `TOMWrapper/TextServices/DAXLexerExtensions.cs`):
```csharp
var inputStream = new AntlrInputStream(daxExpression);
var lexer = new DAXLexer(inputStream);
var tokens = lexer.GetAllTokens();

foreach (var token in tokens)
{
    // Process DAX tokens (identifiers, keywords, etc.)
    if (token.Type == DAXLexer.IDENTIFIER)
    {
        // Handle identifier (measure name, table name, etc.)
    }
}
```

### Token Types
Each grammar rule generates a constant in the lexer class:
- `DAXLexer.RETURN` ? 'RETURN' keyword
- `DAXLexer.IDENTIFIER` ? Identifiers (measure names, column names)
- `DAXLexer.NUMBER` ? Numeric literals
- `DAXLexer.STRING` ? String literals

## Touch Points / Key Files

### Grammar Sources
- **`DAXLexer.g4`** - DAX lexer grammar (edit this for DAX syntax changes)
- **`CSharpLexer.g4`** - C# lexer grammar (edit this for C# syntax highlighting)

### Generated Files (DO NOT EDIT)
- **`obj/Debug/DAXLexer.cs`** - Generated DAX lexer (auto-regenerated on build)
- **`obj/Debug/CSharpLexer.cs`** - Generated C# lexer (auto-regenerated on build)

### Usage in TOMWrapper
- **`TOMWrapper/TextServices/DAXLexerExtensions.cs`** - Extensions for DAX token analysis
- **`TOMWrapper/Utils/DaxDependencyHelper.cs`** - Uses DAX lexer for dependency analysis
- **`TabularEditor/TextServices/ExpressionParser.cs`** - Uses DAX lexer for expression parsing

## JIT Index Hints

```powershell
# View DAX grammar
Get-Content AntlrGrammars/DAXLexer.g4

# View C# grammar
Get-Content AntlrGrammars/CSharpLexer.g4

# Find generated lexers
Get-ChildItem AntlrGrammars/obj -Recurse -Filter "*Lexer.cs"

# Find usage of DAX lexer in TOMWrapper
Get-ChildItem TOMWrapper -Recurse -Filter "*.cs" | Select-String -Pattern "DAXLexer"

# Check ANTLR version
Get-Content AntlrGrammars/packages.config | Select-String -Pattern "Antlr4"
```

## Common Gotchas

### 1. Generated Files
**Never edit** `obj/Debug/DAXLexer.cs` or `CSharpLexer.cs` directly:
```antlr
// ? BAD: Edit obj/Debug/DAXLexer.cs
public const int RETURN = 42;

// ? GOOD: Edit DAXLexer.g4 and rebuild
RETURN: 'RETURN';
```

### 2. ANTLR Build Integration
ANTLR generation happens automatically during build:
- `.g4` files ? ANTLR tool ? `.cs` files in `obj/`
- If grammar changes don't appear, clean and rebuild project

### 3. Token Conflicts
Avoid ambiguous grammar rules:
```antlr
// ? BAD: Ambiguous (NUMBER and IDENTIFIER overlap)
NUMBER: [0-9]+;
IDENTIFIER: [a-zA-Z0-9]+;

// ? GOOD: More specific rules
NUMBER: [0-9]+;
IDENTIFIER: [a-zA-Z_][a-zA-Z0-9_]*;
```

### 4. Case Sensitivity
DAX is case-insensitive, but grammar must handle this:
```antlr
// Case-insensitive keywords
RETURN: [Rr][Ee][Tt][Uu][Rr][Nn];

// Or use fragment rule for case-insensitivity
fragment A: [aA];
fragment R: [rR];
RETURN: R E T U R N;
```

### 5. Lexer vs. Parser
This project only contains **lexers** (tokenization):
- Lexer: Breaks text into tokens (keywords, identifiers, operators)
- Parser: Builds syntax tree from tokens (not used in this project)

For full DAX parsing, only lexing is needed (dependency analysis, syntax highlighting).

## Pre-PR Checks

```powershell
# Clean and rebuild to regenerate lexers
msbuild AntlrGrammars/AntlrGrammars.csproj /t:Clean
msbuild AntlrGrammars/AntlrGrammars.csproj /p:Configuration=Debug

# Verify generated files exist
Test-Path "AntlrGrammars/obj/Debug/DAXLexer.cs"
Test-Path "AntlrGrammars/obj/Debug/CSharpLexer.cs"

# Test DAX lexer in TOMWrapperTest
dotnet test TOMWrapperTest/TOMWrapperTest.csproj --filter "FullyQualifiedName~DAX"

# Manual test checklist:
# [ ] Grammar compiles without errors
# [ ] Generated lexer files exist in obj/
# [ ] DAX expressions still parse correctly in TabularEditor
# [ ] Syntax highlighting still works
```

## Anti-Patterns

? **DON'T**: Manually edit generated `.cs` files in `obj/`  
? **DO**: Edit `.g4` grammar files and rebuild

? **DON'T**: Add business logic to grammar files  
? **DO**: Keep grammars simple (tokenization only), add logic in consuming code

? **DON'T**: Create overlapping token rules  
? **DO**: Order rules from most specific to most general (ANTLR matches first rule)

? **DON'T**: Commit generated files to Git (they're in `.gitignore`)  
? **DO**: Commit only `.g4` source files

## Examples to Copy

### Adding a new DAX keyword
**Pattern**: See existing keywords in `DAXLexer.g4`
```antlr
// Add to keyword section
EVALUATE: [Ee][Vv][Aa][Ll][Uu][Aa][Tt][Ee];
DEFINE: [Dd][Ee][Ff][Ii][Nn][Ee];
// Add your new keyword:
NEWKEYWORD: [Nn][Ee][Ww][Kk][Ee][Yy][Ww][Oo][Rr][Dd];
```

### Adding a new operator
**Pattern**: See existing operators in `DAXLexer.g4`
```antlr
PLUS: '+';
MINUS: '-';
// Add your new operator:
POWER: '^';
```

### Handling whitespace
**Pattern**: See whitespace rules in `DAXLexer.g4`
```antlr
WS: [ \t\r\n]+ -> skip; // Skip whitespace
COMMENT: '//' ~[\r\n]* -> skip; // Skip single-line comments
BLOCK_COMMENT: '/*' .*? '*/' -> skip; // Skip block comments
```

### Fragment rules (reusable)
**Pattern**: Use fragments for common patterns
```antlr
fragment DIGIT: [0-9];
fragment LETTER: [a-zA-Z];
fragment UNDERSCORE: '_';

IDENTIFIER: (LETTER | UNDERSCORE) (LETTER | DIGIT | UNDERSCORE)*;
NUMBER: DIGIT+ ('.' DIGIT+)?;
```

## ANTLR Resources
- [ANTLR 4 Documentation](https://github.com/antlr/antlr4/blob/master/doc/index.md)
- [ANTLR 4 Getting Started](https://github.com/antlr/antlr4/blob/master/doc/getting-started.md)
- [Lexer Rules](https://github.com/antlr/antlr4/blob/master/doc/lexer-rules.md)
