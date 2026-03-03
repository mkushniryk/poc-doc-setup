> [← Back to Index](README.md)

# Quick Start: Implementation Roadmap

*Prioritized checklist to adopt the playbook practices*

---

## Day 1: Immediate Wins

These take minutes and provide instant value.

### Project Configuration
- [ ] Create `Directory.Build.props` in solution root:
```xml
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <ImplicitUsings>enable</ImplicitUsings>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
  </PropertyGroup>
</Project>
```

- [ ] Create `Directory.Packages.props` for central package management:
```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- Add your packages here -->
  </ItemGroup>
</Project>
```

- [ ] Add `.editorconfig` with team conventions

**Result:** Compiler now catches null reference bugs and enforces consistency.

---

## Week 1: Local Development

### Docker Compose Setup
- [ ] Create `docker-compose.yml` for dependencies:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app
    ports:
      - "5432:5432"
```

- [ ] Update README with quick start:
```markdown
## Quick Start
git clone <repo>
docker compose up -d
dotnet run
```

- [ ] Verify new developer can run in < 5 minutes

### Essential Analyzers
- [ ] Add to `Directory.Build.props`:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="9.*" />
</ItemGroup>
```

**Result:** Zero-config onboarding, no "works on my machine" issues.

---

## Week 2-4: Code Quality Foundations

### Typed IDs (New Code Only)
- [ ] For each new domain entity, create typed ID:
```csharp
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
}
```

- [ ] Use in new method signatures:
```csharp
// Before: void Process(Guid orderId, Guid customerId)
// After:
void Process(OrderId orderId, CustomerId customerId)
```

### Pre-commit Hooks
- [ ] Install Husky.Net:
```bash
dotnet tool install Husky
dotnet husky install
```

- [ ] Configure `.husky/pre-commit`:
```bash
dotnet format --verify-no-changes
dotnet build --no-restore
```

### Basic Testing
- [ ] Add test project with:
```bash
dotnet add package xunit
dotnet add package FluentAssertions
dotnet add package NSubstitute
```

- [ ] Write unit tests for new business logic
- [ ] See [Testing Strategy](testing-strategy.md) for patterns

**Result:** Compiler prevents ID mixups; formatting issues never reach PR.

---

## Month 1-3: Advanced Practices

### Result Pattern for Error Handling
- [ ] Add ErrorOr package:
```bash
dotnet add package ErrorOr
```

- [ ] Use for operations that can fail:
```csharp
// Before: throws exceptions
public Order GetOrder(OrderId id) { ... }

// After: explicit errors
public ErrorOr<Order> GetOrder(OrderId id) { ... }
```

### Integration Tests
- [ ] Add Testcontainers:
```bash
dotnet add package Testcontainers.PostgreSql
dotnet add package Respawn
```

- [ ] Write integration tests for critical paths
- [ ] See [Testing Strategy](testing-strategy.md#4-integration-testing)

### Architecture Tests
- [ ] Add ArchUnitNET:
```bash
dotnet add package ArchUnitNET.xUnit
```

- [ ] Enforce layer dependencies:
```csharp
[Fact]
public void Domain_ShouldNotDependOn_Infrastructure()
{
    Types().That().Are(DomainLayer)
        .Should().NotDependOnAny(InfrastructureLayer)
        .Check(Architecture);
}
```

**Result:** Errors are explicit in signatures; tests catch wiring issues.

---

## Ongoing Practices

### Every PR
- [ ] Run tests locally before pushing
- [ ] Self-review against [Anti-Patterns](07-anti-patterns.md)
- [ ] Keep PRs under 400 lines

### Every Sprint
- [ ] Review which anti-patterns appear most → add analyzers
- [ ] Check CI pipeline speed → optimize if > 10 minutes

### Monthly
- [ ] Run mutation testing on critical modules
- [ ] Review and update this checklist based on team learnings

### Quarterly
- [ ] Re-evaluate practices per [Standards & Culture](06-standards-culture.md)
- [ ] Assess ROI using [Economics](05-economics.md) framework

---

## Priority Matrix

| Priority | Practice | Effort | Impact |
|----------|----------|--------|--------|
| **P0** | Nullable + TreatWarningsAsErrors | 1 hour | Eliminates NullReferenceExceptions |
| **P0** | Docker Compose | 4 hours | Eliminates onboarding friction |
| **P1** | Typed IDs (new code) | Ongoing | Prevents ID mixup bugs |
| **P1** | Unit tests | Ongoing | Catches logic errors early |
| **P2** | Result pattern | 1-2 days | Makes errors explicit |
| **P2** | Integration tests | 1-2 days | Catches wiring issues |
| **P3** | Architecture tests | 4 hours | Prevents design drift |
| **P3** | Pre-commit hooks | 1 hour | Catches issues before push |

---

## Measuring Progress

Track these metrics weekly:

| Metric | Starting | Week 4 Target | Month 3 Target |
|--------|----------|---------------|----------------|
| Onboarding time | __ days | < 1 day | < 1 hour |
| Production bugs/month | __ | -25% | -50% |
| "Works on my machine" incidents | __/week | -50% | 0 |
| CI pipeline duration | __ min | < 15 min | < 10 min |

---

## Next Steps

1. **Today:** Complete the Day 1 checklist
2. **This week:** Complete Week 1 checklist
3. **Deep dive:** Read [Core Philosophy](01-core-philosophy.md) to understand the "why"
4. **Reference:** Bookmark [Anti-Patterns](07-anti-patterns.md) for PR reviews

---

*Part of the [C# .NET Velocity & Quality Playbook](README.md)*
