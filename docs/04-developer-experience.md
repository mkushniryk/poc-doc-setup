> [← Back to Index](README.md)

# 4. Frictionless .NET Developer Experience

## Why Developer Experience Is Everything

**Time-to-feature is the ultimate competitive advantage.**

Every minute a developer spends fighting tooling, waiting for builds, or debugging environment issues is a minute not spent delivering value. Poor developer experience compounds:

- **Context switching kills productivity.** A 5-minute environment issue doesn't cost 5 minutes—it costs 23 minutes to regain focus.
- **Friction breeds shortcuts.** Developers skip tests when they're slow. They skip local validation when setup is painful. They ship bugs.
- **Onboarding cost multiplies.** If it takes a week to become productive, you've lost a week of salary for every new hire, every team rotation, every contractor.
- **Talent leaves.** Top engineers won't tolerate broken tooling. They'll find teams that respect their time.

The goal is radical simplicity: **clone, run, code, ship.** No Slack messages asking "how do I set up the database?" No wiki pages with 47 steps. No tribal knowledge.

> **The 10-minute rule:** A new developer should go from zero to running tests in under 10 minutes. If you can't achieve this, your architecture is fighting you.

---

## 4.1 Zero-Configuration Local Development

The holy grail: **everything works after clone.**

```bash
git clone <repo>
cd <repo>
docker compose up -d
dotnet run --project src/Api
# That's it. You're developing.
```

### Why This Matters for Feature Velocity

When dependencies are pre-configured:

| Scenario | Traditional Setup | Zero-Config Setup |
|----------|-------------------|-------------------|
| Start new feature | 30 min - 2 hours | 30 seconds |
| Switch branches with different deps | 15-30 min | `docker compose up -d` |
| Reproduce a bug | "Works on my machine" | Identical environments |
| Onboard new developer | 1-5 days | < 1 hour |
| Run integration tests locally | "I just push to CI" | `make test-integration` |

**The compound effect:** If you save 20 minutes per feature, and your team ships 50 features per month, that's 16+ hours saved monthly—two full developer days.

> **When to skip Docker setup:** Single-developer projects with no external dependencies, pure library packages, or when your dependencies are all in-memory (e.g., SQLite for simple apps). Also okay to defer during early prototyping—but add it before onboarding a second developer.

### Ideal .NET Repository Structure

```
/
├── src/
│   ├── Api/                          # ASP.NET Core API
│   ├── Domain/                       # Domain models, no dependencies
│   ├── Application/                  # Use cases, CQRS handlers
│   └── Infrastructure/               # EF Core, external services
├── tests/
│   ├── Unit/
│   ├── Integration/
│   └── Architecture/                 # ArchUnitNET tests
├── docker-compose.yml                # PostgreSQL, Redis, etc.
├── docker-compose.override.yml       # Local overrides (gitignored)
├── Directory.Build.props             # Shared project settings
├── Directory.Packages.props          # Central package management
├── .editorconfig                     # Code style
├── global.json                       # SDK version pinning
├── .env.example                      # Environment template
├── scripts/
│   ├── setup.sh                      # One-time setup
│   └── seed-data.sh                  # Test data
├── README.md                         # 5-minute quickstart
└── YourApp.sln
```

### Complete Local Infrastructure

**docker-compose.yml:**

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d app"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  seq:
    image: datalust/seq:latest
    environment:
      ACCEPT_EULA: "Y"
    ports:
      - "5341:5341"  # Ingestion
      - "8081:80"    # UI

  maildev:
    image: maildev/maildev
    ports:
      - "1080:1080"  # Web UI
      - "1025:1025"  # SMTP

volumes:
  postgres_data:
```

**appsettings.Development.json:**

```json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=app;Username=dev;Password=dev"
  },
  "Seq": {
    "ServerUrl": "http://localhost:5341"
  },
  "Email": {
    "SmtpHost": "localhost",
    "SmtpPort": 1025
  }
}
```

> **Metric:** If onboarding takes more than 30 minutes, productivity is leaking.

---

## 4.2 Instant Feedback Loops

**The faster the feedback, the faster the feature.**

Speed isn't just about build times—it's about the entire cycle from "I changed code" to "I know it works."

### Target Cycle Times

| Stage | Target | Reality Check |
|-------|--------|---------------|
| Hot reload | < 1 second | `dotnet watch` handles this |
| Unit tests (affected) | < 5 seconds | Test only what changed |
| Full unit test suite | < 30 seconds | Parallelize, no I/O |
| Build | < 15 seconds | Incremental compilation |
| Integration tests | < 2 minutes | Testcontainers, parallel |
| Full CI pipeline | < 10 minutes | Caching, selective testing |
| Deploy to staging | < 5 minutes | Pre-built containers |

### Hot Reload Workflow

```bash
# Terminal 1: Infrastructure
docker compose up -d

# Terminal 2: API with hot reload
dotnet watch --project src/Api

# Now every file save = instant update
```

### Optimizing Test Speed

**Separate fast from slow:**

```csharp
[Trait("Category", "Unit")]
public class OrderServiceTests { }

[Trait("Category", "Integration")]
public class OrderRepositoryTests { }

[Trait("Category", "E2E")]
public class CheckoutFlowTests { }
```

**Run only what you need:**

```bash
# During development - instant feedback
dotnet test --filter Category=Unit

# Before commit - thorough validation
dotnet test --filter "Category=Unit|Category=Integration"

# CI handles the rest
```

### Makefile for Frictionless Commands

```makefile
.PHONY: build test run clean

# Development workflow
dev:
	docker compose up -d
	dotnet watch --project src/Api

run:
	docker compose up -d
	dotnet run --project src/Api

# Testing
test:
	dotnet test --logger "console;verbosity=minimal"

test-unit:
	dotnet test --filter Category=Unit --logger "console;verbosity=minimal"

test-int:
	docker compose up -d
	dotnet test --filter Category=Integration
	docker compose down

test-watch:
	dotnet watch test --filter Category=Unit

# Build
build:
	dotnet build

publish:
	dotnet publish src/Api -c Release -o ./publish

# Utilities
format:
	dotnet format

clean:
	dotnet clean
	rm -rf **/bin **/obj

reset:
	docker compose down -v
	docker compose up -d
	dotnet ef database update --project src/Infrastructure

logs:
	docker compose logs -f

# Database
db-migrate:
	dotnet ef migrations add $(name) --project src/Infrastructure --startup-project src/Api

db-update:
	dotnet ef database update --project src/Infrastructure --startup-project src/Api
```

**Usage:**

```bash
make dev          # Start developing immediately
make test-unit    # Fast feedback
make reset        # Fresh database state
```

---

## 4.3 Central Package Management

**One place for all versions. No more dependency hell.**

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- ASP.NET Core -->
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="9.0.0" />
    
    <!-- Entity Framework -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.0" />
    
    <!-- Testing -->
    <PackageVersion Include="xunit" Version="2.9.0" />
    <PackageVersion Include="FluentAssertions" Version="7.0.0" />
    <PackageVersion Include="NSubstitute" Version="5.3.0" />
    <PackageVersion Include="Testcontainers" Version="4.0.0" />
    
    <!-- Analyzers - catch issues at compile time -->
    <PackageVersion Include="Meziantou.Analyzer" Version="2.0.180" />
    <PackageVersion Include="SonarAnalyzer.CSharp" Version="9.32.0" />
  </ItemGroup>
</Project>
```

**In project files—clean and simple:**

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore" />
<PackageReference Include="xunit" />
```

**Benefits:**
- Upgrade once, apply everywhere
- No version conflicts between projects
- Clear audit trail in a single file
- Renovate/Dependabot updates one file

---

## 4.4 SDK Version Pinning

**"Works on my machine" eliminated.**

```json
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "latestFeature",
    "allowPrerelease": false
  }
}
```

This ensures:
- Every developer uses the same SDK
- CI/CD uses the same SDK
- No surprises from SDK behavior changes

---

## 4.5 Pre-Commit Quality Gates

**Catch issues before they hit CI.**

**`.husky/pre-commit` (or use git hooks directly):**

```bash
#!/bin/sh
dotnet format --verify-no-changes
dotnet build --no-restore
dotnet test --filter Category=Unit --no-build
```

**Benefits:**
- Fast local validation
- No waiting for CI to tell you about formatting issues
- Confidence before pushing

---

## 4.6 IDE Configuration as Code

**Consistent experience across the team.**

**`.editorconfig`:**

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
dotnet_sort_system_directives_first = true
csharp_style_var_for_built_in_types = true
csharp_style_expression_bodied_methods = when_on_single_line
csharp_prefer_braces = true
```

**`.vscode/settings.json` (for VS Code users):**

```json
{
  "editor.formatOnSave": true,
  "omnisharp.enableRoslynAnalyzers": true,
  "omnisharp.enableEditorConfigSupport": true
}
```

---

## 4.7 Documentation That Stays Current

**README.md is the single source of truth.**

Every repo README must answer:
1. **What** does this do? (one paragraph)
2. **How** do I run it? (copy-paste commands)
3. **How** do I test it? (copy-paste commands)
4. **Where** is the architecture documentation?

**Template:**

```markdown
# Service Name

Brief description of what this service does.

## Quick Start

\`\`\`bash
docker compose up -d
dotnet run --project src/Api
\`\`\`

API available at: http://localhost:5000
Swagger: http://localhost:5000/swagger
Seq logs: http://localhost:8081

## Testing

\`\`\`bash
make test          # All tests
make test-unit     # Fast feedback
make test-int      # Integration tests
\`\`\`

## Architecture

See [architecture decision records](./docs/adr/) for design decisions.
```

---

## Summary: The Frictionless Checklist

| Requirement | Check |
|-------------|-------|
| Clone to running in < 10 min | ☐ |
| `docker compose up` = all dependencies | ☐ |
| Single command to run all tests | ☐ |
| Hot reload works | ☐ |
| Unit tests run in < 30 seconds | ☐ |
| No manual environment setup | ☐ |
| SDK version pinned | ☐ |
| Package versions centralized | ☐ |
| Code style automated | ☐ |
| README has working quick start | ☐ |

**If any box is unchecked, fix it before building features.**

---

[← Project Configuration](03-project-configuration.md) | [Index](README.md) | [Next: Economics →](05-economics.md)
