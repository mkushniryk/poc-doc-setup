> [← Back to Index](README.md)

# 6. Raising .NET Standards Without Burnout

High standards poorly implemented create burnout. Well-implemented standards reduce burnout.

---

## Root Causes of Burnout
- **Ambiguity** — unclear expectations, no golden path
- **Unpredictability** — firefighting, surprise production bugs
- **Invisible work** — manual deploys, credential hunting, environment setup

---

## Raising Standards Safely in .NET

| Practice | Example Implementation |
|----------|------------------------|
| One major discipline change per quarter | Q1: Enable nullable. Q2: Typed IDs. Q3: Result pattern. |
| Pair strictness with faster feedback | `TreatWarningsAsErrors` + good IDE config = instant feedback |
| Automate invisible friction | Docker compose, analyzers, pre-commit hooks |
| Preserve autonomy in implementation | "Use Result types" not "Use ErrorOr library specifically" |
| Celebrate simplification | Track lines of code deleted, complexity reduced |

---

## Warning Signs
- Engineers fear enabling warnings as errors → too many existing violations
- "That's not how we do things" without explanation → dogma forming
- Increasing time-to-compile due to analyzers → analyzer choice needs review

---

## Incremental Adoption Strategy

**Week 1-2: Audit**
```bash
# Count current warnings
dotnet build 2>&1 | grep -c "warning"

# List warning types
dotnet build 2>&1 | grep "warning" | sed 's/.*warning \([A-Z]*[0-9]*\).*/\1/' | sort | uniq -c | sort -rn
```

**Week 3-4: Fix Top Violations**
```csharp
// Fix specific warnings, keep others as warnings for now
<PropertyGroup>
  <WarningsAsErrors>CS8600;CS8601;CS8602;CS8603</WarningsAsErrors>
</PropertyGroup>
```

**Week 5+: Gradually Expand**
```csharp
// Eventually enable all
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

> **Strictness should increase calm, not fear.**
>
> **For existing codebases:** See [Migration Guide](migration-guide.md) for detailed tactical steps.

---

## Avoiding Dogma

Philosophy must serve outcomes, not the other way around.

### Healthy Practices

| Do | Don't |
|----|-------|
| Re-evaluate rules quarterly | Treat rules as sacred |
| Allow intentional, documented suppressions | "Never suppress warnings" |
| Encourage constructive dissent | Shut down questions |
| Justify with outcomes | Justify with authority |

### Signs of Dogma in .NET Teams
- "We always use MediatR" (instead of "we value decoupling")
- "Every method must be async" (even when sync is simpler)
- "Everything must have an interface" (even with single implementation)
- Copying patterns without understanding why

### Evaluation Framework

**Keep a rule if it increases:**
- Compile-time safety
- Test confidence
- Refactoring ease
- Onboarding speed

**Re-evaluate a rule if it increases:**
- Boilerplate without safety benefit
- Compile time significantly
- Cognitive load without payoff

---

[← Economics](05-economics.md) | [Index](README.md) | [Next: Anti-Patterns →](07-anti-patterns.md)
