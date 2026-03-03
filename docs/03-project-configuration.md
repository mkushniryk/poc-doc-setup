> [← Back to Index](README.md)

# 3. .NET Project Configuration

Configure your projects for maximum safety.

---

## 3.1 Directory.Build.props

Place at repository root to apply settings to all projects.

```xml
<Project>
  <PropertyGroup>
    <!-- Target framework -->
    <TargetFramework>net8.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    
    <!-- Safety settings -->
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsAsErrors />
    <NoWarn />
    
    <!-- When to skip TreatWarningsAsErrors:
         - Brownfield projects with 1000+ existing warnings (use incremental approach in 06-standards-culture.md)
         - Rapid prototyping phases (re-enable before production)
         - Third-party generated code (use NoWarn in that project only) -->
    
    <!-- Code analysis -->
    <AnalysisLevel>latest-all</AnalysisLevel>
    <AnalysisMode>All</AnalysisMode>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    
    <!-- Other -->
    <ImplicitUsings>enable</ImplicitUsings>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
  
  <!-- Analyzers for all projects -->
  <ItemGroup>
    <PackageReference Include="Meziantou.Analyzer" Version="2.*">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

---

## 3.2 Global Usings

Create a `GlobalUsings.cs` file for common imports.

```csharp
// GlobalUsings.cs
global using System.Collections.Immutable;
global using Microsoft.Extensions.Logging;
global using YourApp.Domain.Common;
global using YourApp.Domain.ValueObjects;

// Aliases for disambiguation
global using JsonOptions = System.Text.Json.JsonSerializerOptions;
```

---

## 3.3 Static Code Analysis

Static analyzers catch bugs, enforce patterns, and maintain consistency without runtime overhead.

### Recommended Analyzer Packages

| Analyzer | Purpose | Install |
|----------|---------|--------|
| `Microsoft.CodeAnalysis.NetAnalyzers` | Built-in .NET rules (included in SDK) | Enabled by default |
| `Meziantou.Analyzer` | Best practices, security, performance | `dotnet add package Meziantou.Analyzer` |
| `SonarAnalyzer.CSharp` | Code quality, security vulnerabilities | `dotnet add package SonarAnalyzer.CSharp` |
| `Roslynator.Analyzers` | 500+ analyzers and refactorings | `dotnet add package Roslynator.Analyzers` |
| `StyleCop.Analyzers` | Code style and documentation | `dotnet add package StyleCop.Analyzers` |
| `Microsoft.VisualStudio.Threading.Analyzers` | Async/await correctness | `dotnet add package Microsoft.VisualStudio.Threading.Analyzers` |
| `ErrorProne.NET.Structs` | Struct and readonly correctness | `dotnet add package ErrorProne.NET.Structs` |
| `AsyncFixer` | Common async/await mistakes | `dotnet add package AsyncFixer` |

### Adding Analyzers to Directory.Build.props

```xml
<ItemGroup>
  <!-- Essential analyzers for all projects -->
  <PackageReference Include="Meziantou.Analyzer" Version="2.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
  
  <PackageReference Include="Roslynator.Analyzers" Version="4.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
  
  <PackageReference Include="SonarAnalyzer.CSharp" Version="9.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
  
  <PackageReference Include="Microsoft.VisualStudio.Threading.Analyzers" Version="17.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

### Configuring Analyzer Severity

Override rule severity in `.editorconfig`:

```ini
[*.cs]
# Treat specific warnings as errors
dotnet_diagnostic.CA1062.severity = error    # Validate arguments of public methods
dotnet_diagnostic.CA2007.severity = error    # Use ConfigureAwait
dotnet_diagnostic.CA1816.severity = error    # Dispose methods should call SuppressFinalize

# Disable rules that don't fit your project
dotnet_diagnostic.CA1014.severity = none     # Mark assemblies with CLSCompliant
dotnet_diagnostic.SA1600.severity = none     # Elements should be documented (if not needed)

# Downgrade noisy rules to suggestion
dotnet_diagnostic.IDE0058.severity = suggestion  # Expression value is never used
```

### Key Rules to Enable

| Rule | Category | What It Catches |
|------|----------|------------------|
| `CA2016` | Reliability | Missing CancellationToken forwarding |
| `CA1062` | Design | Null argument not validated |
| `CA2007` | Reliability | Missing ConfigureAwait |
| `CA1819` | Performance | Properties returning arrays |
| `CA2227` | Usage | Collection properties should be read-only |
| `CS8600-8604` | Nullable | Null reference warnings |
| `MA0004` | Meziantou | Use ConfigureAwait(false) |
| `MA0006` | Meziantou | Use string.Equals instead of == |
| `RCS1090` | Roslynator | Add ConfigureAwait |
| `S3925` | Sonar | ISerializable should be implemented correctly |

---

## 3.4 .editorconfig for C#

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.cs]
# Organize usings
dotnet_sort_system_directives_first = true
dotnet_separate_import_directive_groups = false

# this. preferences
dotnet_style_qualification_for_field = false:error
dotnet_style_qualification_for_property = false:error
dotnet_style_qualification_for_method = false:error
dotnet_style_qualification_for_event = false:error

# Use language keywords over framework types
dotnet_style_predefined_type_for_locals_parameters_members = true:error
dotnet_style_predefined_type_for_member_access = true:error

# Modifier preferences
dotnet_style_require_accessibility_modifiers = always:error
csharp_preferred_modifier_order = public,private,protected,internal,static,extern,new,virtual,abstract,sealed,override,readonly,unsafe,volatile,async:error

# Expression-level preferences
dotnet_style_object_initializer = true:warning
dotnet_style_collection_initializer = true:warning
dotnet_style_prefer_auto_properties = true:warning
dotnet_style_prefer_conditional_expression_over_assignment = true:suggestion

# Null-checking preferences
dotnet_style_coalesce_expression = true:warning
dotnet_style_null_propagation = true:warning
dotnet_style_prefer_is_null_check_over_reference_equality_method = true:error

# C# style preferences
csharp_style_var_for_built_in_types = true:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = true:suggestion

# Expression-bodied members
csharp_style_expression_bodied_methods = when_on_single_line:suggestion
csharp_style_expression_bodied_constructors = false:suggestion
csharp_style_expression_bodied_properties = true:warning

# Pattern matching
csharp_style_pattern_matching_over_is_with_cast_check = true:warning
csharp_style_pattern_matching_over_as_with_null_check = true:warning
csharp_style_prefer_switch_expression = true:warning
csharp_style_prefer_pattern_matching = true:warning

# Namespace declarations
csharp_style_namespace_declarations = file_scoped:error

# Braces
csharp_prefer_braces = true:warning

# Using directives
csharp_using_directive_placement = outside_namespace:error

# Naming conventions
dotnet_naming_rule.interface_should_be_begins_with_i.severity = error
dotnet_naming_rule.interface_should_be_begins_with_i.symbols = interface
dotnet_naming_rule.interface_should_be_begins_with_i.style = begins_with_i

dotnet_naming_symbols.interface.applicable_kinds = interface
dotnet_naming_style.begins_with_i.required_prefix = I
dotnet_naming_style.begins_with_i.capitalization = pascal_case

# Private fields with underscore
dotnet_naming_rule.private_fields_should_be_camel_case.severity = error
dotnet_naming_rule.private_fields_should_be_camel_case.symbols = private_fields
dotnet_naming_rule.private_fields_should_be_camel_case.style = camel_case_underscore

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.camel_case_underscore.required_prefix = _
dotnet_naming_style.camel_case_underscore.capitalization = camel_case
```

---

## 3.5 Code Formatting with dotnet format

Enforce consistent formatting automatically using `dotnet format`.

### Running dotnet format

```bash
# Check for formatting issues (CI-friendly, exits with error code)
dotnet format --verify-no-changes

# Auto-fix formatting issues
dotnet format

# Format only whitespace (faster)
dotnet format whitespace

# Format only style rules (var usage, expression bodies, etc.)
dotnet format style

# Format only analyzers (code quality)
dotnet format analyzers
```

### Pre-commit Hook with Husky.NET

Prevent unformatted code from being committed.

```bash
# Install Husky.NET
dotnet new tool-manifest
dotnet tool install Husky

# Initialize
dotnet husky install

# Add pre-commit hook
dotnet husky add .husky/pre-commit -c "dotnet format --verify-no-changes"
```

**`.husky/pre-commit`:**

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "Running dotnet format..."
dotnet format --verify-no-changes --no-restore

if [ $? -ne 0 ]; then
    echo "Code formatting issues found. Run 'dotnet format' to fix."
    exit 1
fi
```

### CI Pipeline Integration

```yaml
# GitHub Actions
jobs:
  format-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
      - run: dotnet format --verify-no-changes --no-restore
```

```yaml
# Azure DevOps
steps:
  - task: UseDotNet@2
    inputs:
      version: '8.0.x'
  - script: dotnet restore
  - script: dotnet format --verify-no-changes --no-restore
    displayName: 'Check code formatting'
```

### IDE Integration

**VS Code** (`.vscode/settings.json`):
```json
{
  "editor.formatOnSave": true,
  "omnisharp.enableEditorConfigSupport": true,
  "omnisharp.enableRoslynAnalyzers": true
}
```

**Visual Studio**: Options → Text Editor → C# → Code Style → *check* "Run code cleanup on save"

**JetBrains Rider**: Settings → Editor → Code Style → C# → *import from* `.editorconfig`

---

## 3.6 Complete Quality Pipeline

Combine all tools for comprehensive code quality.

```bash
#!/bin/bash
# scripts/quality-check.sh

set -e

echo "=== Restoring packages ==="
dotnet restore

echo "=== Checking code format ==="
dotnet format --verify-no-changes --no-restore

echo "=== Building with warnings as errors ==="
dotnet build --no-restore --configuration Release /p:TreatWarningsAsErrors=true

echo "=== Running tests ==="
dotnet test --no-build --configuration Release

echo "=== Quality check passed ==="
```

### Makefile for Common Tasks

```makefile
.PHONY: restore build test format lint quality

restore:
	dotnet restore

build: restore
	dotnet build --no-restore

test: build
	dotnet test --no-build

format:
	dotnet format

lint:
	dotnet format --verify-no-changes

quality: lint build test
	@echo "All quality checks passed"
```

Usage:
```bash
make quality    # Full pipeline
make format     # Auto-fix formatting
make lint       # Check without fixing
```

---

[← Language Features](02-language-features.md) | [Index](README.md) | [Next: Developer Experience →](04-developer-experience.md)
