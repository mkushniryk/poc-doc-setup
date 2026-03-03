> [← Back to Index](README.md)
>
> **Prerequisites:** [03-project-configuration](03-project-configuration.md), [06-standards-culture](06-standards-culture.md)

# Migration Guide: Adopting the Playbook in Brownfield Projects

*Tactical steps for introducing these practices to existing codebases.*

---

## The Challenge

New projects can adopt these practices from day one. Existing codebases face:
- Thousands of existing warnings
- Fear of breaking changes
- Team resistance to "yet another initiative"
- Pressure to keep shipping features

**The solution:** Incremental adoption with clear milestones and measurable progress.

---

## Phase 0: Audit (Day 1)

Before changing anything, understand your starting point.

### Measure Current State

```bash
# Count total warnings
dotnet build 2>&1 | grep -c "warning"

# Group warnings by type
dotnet build 2>&1 | grep "warning" | \
  sed 's/.*warning \([A-Z]*[0-9]*\).*/\1/' | \
  sort | uniq -c | sort -rn | head -20

# Count nullable warnings specifically
dotnet build 2>&1 | grep -c "CS86"

# Count projects
find . -name "*.csproj" | wc -l
```

### Document Baseline

| Metric | Current | Target | Date |
|--------|---------|--------|------|
| Total warnings | ___ | 0 | ___ |
| CS8600-8603 (nullable) | ___ | 0 | ___ |
| Projects with Nullable=enable | 0 | All | ___ |
| Typed ID types | 0 | Top 10 entities | ___ |

---

## Phase 1: Foundation (Week 1-2)

Low-risk changes that provide immediate value.

### 1.1 Create Directory.Build.props (If Missing)

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <ImplicitUsings>enable</ImplicitUsings>
    
    <!-- Start permissive, tighten later -->
    <Nullable>warnings</Nullable>
    <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

### 1.2 Central Package Management

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- Move all PackageReference versions here -->
  </ItemGroup>
</Project>
```

Run: `dotnet list package --outdated` to identify version drift.

### 1.3 Docker Compose for Dependencies

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
```

Update README to document the new workflow.

---

## Phase 2: Nullable Adoption (Week 2-4)

### Strategy: Project-by-Project

Don't enable nullable globally. Start with leaf projects (no internal dependencies).

```
Leaf projects (start here)     Core projects (do last)
├── Tests.Unit              →  ├── Domain
├── Tests.Integration       →  ├── Application
└── Api (if thin)           →  └── Infrastructure
```

### Step-by-Step for Each Project

1. **Enable warnings only:**
   ```xml
   <Nullable>warnings</Nullable>
   ```

2. **Fix easy warnings** (10-20 at a time):
   - Add `?` to nullable fields
   - Add null checks where needed
   - Use `required` for non-nullable properties

3. **Enable enforcement:**
   ```xml
   <Nullable>enable</Nullable>
   ```

4. **Re-run build, fix remaining issues**

### Temporary Suppressions

For legacy code you can't fix immediately:

```csharp
// Documented suppression - tracked in backlog
#pragma warning disable CS8602 // Dereference of possibly null reference
var result = legacyService.GetData().Value;
#pragma warning restore CS8602
// TODO: Fix in PROJ-1234
```

Track suppressions and schedule removal:

```bash
# Count remaining suppressions
grep -r "pragma warning disable CS86" --include="*.cs" | wc -l
```

---

## Phase 3: Warnings as Errors (Week 4-6)

### Gradual Escalation

```xml
<!-- Phase 3a: Specific warnings only -->
<PropertyGroup>
  <WarningsAsErrors>CS8600;CS8601;CS8602;CS8603;CS8604</WarningsAsErrors>
</PropertyGroup>

<!-- Phase 3b: All nullable warnings -->
<PropertyGroup>
  <WarningsAsErrors>nullable</WarningsAsErrors>
</PropertyGroup>

<!-- Phase 3c: All warnings -->
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

### Handling Third-Party Generated Code

If you have generated code causing warnings:

```xml
<!-- In the specific project, not Directory.Build.props -->
<PropertyGroup Condition="'$(Configuration)' == 'Debug'">
  <NoWarn>$(NoWarn);CS8618</NoWarn>
</PropertyGroup>
```

---

## Phase 4: Typed IDs (Week 6-8)

### Start with New Code Only

**Rule:** All new entities get typed IDs. Don't refactor existing code yet.

```csharp
// New entity - uses typed ID
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
}

// Existing entity - leave as-is for now
public class LegacyCustomer
{
    public Guid Id { get; set; }  // Keep for now
}
```

### Identify High-Risk Areas

Audit existing code for parameter-swap risk:

```csharp
// HIGH RISK - two Guids adjacent, easy to swap
void Transfer(Guid from, Guid to, decimal amount);

// Lower risk - different types naturally
void CreateUser(string email, int departmentId);
```

Prioritize typed IDs for high-risk methods.

### Incremental Migration Pattern

When you do migrate existing code:

```csharp
// Step 1: Add new method with typed ID
public Order GetOrder(OrderId id) => 
    GetOrderLegacy(id.Value);

// Step 2: Mark old method obsolete
[Obsolete("Use OrderId overload")]
public Order GetOrderLegacy(Guid id) => ...

// Step 3: Migrate callers over time

// Step 4: Remove obsolete method
```

---

## Phase 5: Result Types (Week 8-12)

### Start at Boundaries

Introduce Result types at API boundaries first:

```csharp
// API layer - start here
public async Task<ErrorOr<OrderDto>> GetOrder(Guid id)

// Domain layer - add later
// Keep existing exception-based code working
```

### Interop with Existing Exception Code

```csharp
public async Task<ErrorOr<Order>> GetOrderSafe(OrderId id)
{
    try
    {
        var order = await _legacyService.GetOrder(id.Value);
        return order;
    }
    catch (NotFoundException)
    {
        return Error.NotFound("Order.NotFound", $"Order {id} not found");
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Unexpected error getting order {Id}", id);
        throw; // Let unexpected errors bubble
    }
}
```

---

## Timeline Templates

### 4-Week Sprint (Small Team, Urgent)

| Week | Focus | Deliverable |
|------|-------|-------------|
| 1 | Audit + Foundation | Directory.Build.props, Docker compose |
| 2 | Nullable (leaf projects) | Tests + 1 project fully nullable |
| 3 | Nullable (remaining) + Warnings as errors | All projects nullable-enabled |
| 4 | Typed IDs (new code policy) | Guidelines documented, 3-5 ID types created |

### 12-Week Quarter (Full Adoption)

| Week | Focus |
|------|-------|
| 1-2 | Audit, foundation, Docker setup |
| 3-4 | Nullable for test projects |
| 5-6 | Nullable for all projects |
| 7-8 | TreatWarningsAsErrors enabled |
| 9-10 | Typed IDs for new entities + high-risk existing |
| 11-12 | Result types at API boundaries |

---

## Tracking Progress

### Weekly Metrics Dashboard

```markdown
## Migration Progress - Week N

| Metric | Last Week | This Week | Target |
|--------|-----------|-----------|--------|
| Total warnings | 1,234 | 987 | 0 |
| Projects with Nullable=enable | 4/12 | 7/12 | 12/12 |
| Typed ID types | 5 | 8 | 15 |
| Pragma suppressions | 45 | 38 | 0 |
```

### Celebrate Milestones

- First project with zero warnings
- First PR rejected by CI for nullable violation (the system works!)
- First bug prevented by typed ID (track in postmortem)

---

## Common Obstacles and Solutions

| Obstacle | Solution |
|----------|----------|
| "We don't have time" | Start with 30 min/day. Compound returns. |
| "Too many warnings" | Fix 10-20 per day. Track progress weekly. |
| "Team resistance" | Demonstrate ROI on one project first. |
| "Generated code has warnings" | Isolate in separate project, use NoWarn. |
| "Leadership won't approve" | Use [economics arguments](05-economics.md#510-making-the-case-to-stakeholders) |

---

## Related Resources

- [06-standards-culture](06-standards-culture.md) — Cultural approach to raising standards
- [03-project-configuration](03-project-configuration.md) — Target configuration
- [QUICK-START](QUICK-START.md) — For new projects

---

[← Back to Index](README.md)
