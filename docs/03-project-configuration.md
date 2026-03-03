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

## 3.6 Single Source of Truth: Local = CI

**The principle:** Every check that runs in CI must be runnable locally with the exact same command. CI pipelines call scripts; they don't contain logic.

### Why This Matters

| Problem | Without Script Sharing | With Script Sharing |
|---------|----------------------|---------------------|
| "Works on my machine" | CI uses different commands | Identical execution |
| Debugging CI failures | Reproduce in pipeline | Run `./scripts/ci.ps1` locally |
| Updating checks | Edit YAML + test in PR loop | Edit script + test locally |
| Onboarding | "Read the YAML to understand" | "Run the same script" |

**No surprises.** If it passes locally, it passes in CI. If it fails in CI, you can reproduce it instantly.

### PowerShell Scripts (Cross-Platform)

PowerShell Core runs on Windows, macOS, and Linux. Use it as the single script language.

```powershell
# scripts/ci.ps1
# Single source of truth for all CI checks

param(
    [switch]$SkipTests,
    [switch]$SkipFormat,
    [string]$Configuration = "Release"
)

$ErrorActionPreference = "Stop"

Write-Host "=== Restoring packages ===" -ForegroundColor Cyan
dotnet restore

if (-not $SkipFormat) {
    Write-Host "=== Checking code format ===" -ForegroundColor Cyan
    dotnet format --verify-no-changes --no-restore
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Format check failed. Run 'dotnet format' to fix." -ForegroundColor Red
        exit 1
    }
}

Write-Host "=== Building ===" -ForegroundColor Cyan
dotnet build --no-restore --configuration $Configuration /p:TreatWarningsAsErrors=true
if ($LASTEXITCODE -ne 0) {
    Write-Host "Build failed." -ForegroundColor Red
    exit 1
}

if (-not $SkipTests) {
    Write-Host "=== Running unit tests ===" -ForegroundColor Cyan
    dotnet test --no-build --configuration $Configuration --filter "Category!=Integration"
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Tests failed." -ForegroundColor Red
        exit 1
    }
}

Write-Host "=== All checks passed ===" -ForegroundColor Green
```

```powershell
# scripts/test-integration.ps1
# Integration tests with Testcontainers

param(
    [string]$Configuration = "Release"
)

$ErrorActionPreference = "Stop"

Write-Host "=== Running integration tests ===" -ForegroundColor Cyan
dotnet test --configuration $Configuration --filter "Category=Integration"
if ($LASTEXITCODE -ne 0) {
    Write-Host "Integration tests failed." -ForegroundColor Red
    exit 1
}

Write-Host "=== Integration tests passed ===" -ForegroundColor Green
```

### CI Pipelines Call Scripts (Don't Duplicate Logic)

**GitHub Actions:**
```yaml
name: CI

on: [push, pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      # Single line - calls the same script you run locally
      - name: Run CI checks
        run: pwsh ./scripts/ci.ps1
        
      - name: Run integration tests
        run: pwsh ./scripts/test-integration.ps1
```

**Azure DevOps:**
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: UseDotNet@2
    inputs:
      version: '8.0.x'
  
  # Single line - same script as local
  - pwsh: ./scripts/ci.ps1
    displayName: 'Run CI checks'
    
  - pwsh: ./scripts/test-integration.ps1
    displayName: 'Run integration tests'
```

### Local Development Workflow

```bash
# Run exact same checks as CI
pwsh ./scripts/ci.ps1

# Skip tests for quick format/build check
pwsh ./scripts/ci.ps1 -SkipTests

# Run integration tests (requires Docker)
pwsh ./scripts/test-integration.ps1
```

### Makefile Wrapper (Optional)

For developers who prefer `make`:

```makefile
.PHONY: ci test-integration format

ci:
	pwsh ./scripts/ci.ps1

test-integration:
	pwsh ./scripts/test-integration.ps1

format:
	dotnet format

# Aliases for convenience
check: ci
test: ci
```

Usage:
```bash
make ci              # Same as: pwsh ./scripts/ci.ps1
make test-integration # Same as: pwsh ./scripts/test-integration.ps1
```

### The Rule

| Location | Contains |
|----------|----------|
| `scripts/*.ps1` | All logic, all commands |
| CI YAML | `pwsh ./scripts/ci.ps1` — nothing else |
| Local terminal | `pwsh ./scripts/ci.ps1` — same thing |

**If you need to change a check, you change the script. CI and local update together.**

---

[← Language Features](02-language-features.md) | [Index](README.md) | [Next: Developer Experience →](04-developer-experience.md)
