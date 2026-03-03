# C# .NET Velocity & Quality Playbook

*Ship faster by catching bugs before they ship. A practical guide to .NET development that emphasizes both speed and quality.*

**Target:** .NET 8+ / C# 12+

---

## Start Here

| Document | Who It's For | Time |
|----------|--------------|------|
| [Executive Summary](EXECUTIVE-SUMMARY.md) | Managers, tech leads, stakeholders | 5 min |
| [Quick Start](QUICK-START.md) | Developers ready to implement | 10 min |

---

## Core Playbook (Required Reading)

| # | Topic | Description |
|---|-------|-------------|
| 1 | [Core Safety Philosophy](01-core-philosophy.md) | First principles: type safety, explicit failure handling |
| 2 | [C# Language Features](02-language-features.md) | Nullable, records, pattern matching, Result types |
| 3 | [Project Configuration](03-project-configuration.md) | Directory.Build.props, analyzers, .editorconfig |
| 4 | [Developer Experience](04-developer-experience.md) | Docker, local dev, fast feedback loops |
| 5 | [Economics](05-economics.md) | ROI of quality practices, cost of bugs |
| 6 | [Standards & Culture](06-standards-culture.md) | Raising standards without burnout |
| 7 | [Anti-Patterns](07-anti-patterns.md) | Common mistakes to avoid (quick reference) |

---

## Implementation Guides

| Topic | When to Use | Priority |
|-------|-------------|----------|
| [Testing Strategy](testing-strategy.md) | Setting up unit/integration/architecture tests | Required |
| [PR Review Process](pr-review-process.md) | Establishing code review practices | Required |
| [Migration Guide](migration-guide.md) | Adopting playbook in existing codebases | As needed |
| [Testing Advanced](testing-advanced.md) | Property, mutation, contract, E2E testing | As needed |
| [External Integration](external-integration.md) | SOAP, OpenAPI, resilience patterns | As needed |
| [AI-Assisted Development](ai-assisted-development.md) | AI agent guardrails, verification workflows | As needed |

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────┐
│              .NET VELOCITY PRINCIPLES                        │
├─────────────────────────────────────────────────────────────┤
│  <Nullable>enable</Nullable>          ← Catch nulls at compile time │
│  <TreatWarningsAsErrors>true          ← Force clean code     │
│  readonly record struct Id(Guid Value) ← Prevent ID mixups   │
│  required + init                       ← Immutable by default │
│  ErrorOr<T>, OneOf                     ← Explicit error paths │
│  TimeProvider                          ← Testable time        │
└─────────────────────────────────────────────────────────────┘
```

---

## ROI Summary

| Investment | Effort | Annual Return |
|------------|--------|---------------|
| Nullable + TreatWarningsAsErrors | 1 day | $3,000 |
| Docker Compose setup | 1 day | $8,000 |
| Typed IDs (10 types) | 4 hours | $2,000 |
| Fast CI pipeline | 2 days | $20,000 |
| **Total** | ~1 week | **$34,000/year** |

See [Executive Summary](EXECUTIVE-SUMMARY.md) for full business case.

---

## Reading Path

```
                    ┌──────────────────┐
                    │ Executive Summary │ ← Start here (managers)
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │    Quick Start    │ ← Start here (developers)
                    └────────┬─────────┘
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
    ▼                        ▼                        ▼
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│   01   │→│   02   │→│   03   │→│   04   │→│   05   │→│   06   │→│   07   │
│Philos. │  │Language│  │Project │  │  Dev   │  │ Econ.  │  │Culture │  │ Anti-  │
│        │  │Features│  │ Config │  │  Exp.  │  │        │  │        │  │patterns│
└────────┘  └───┬────┘  └───┬────┘  └────────┘  └────────┘  └────────┘  └────────┘
                │           │
                │           └──────────────────┐
                │                              │
                ▼                              ▼
        ┌───────────────┐              ┌──────────────┐
        │testing-strategy│─────────────▶│ pr-review-   │
        └───────┬───────┘              │   process    │
                │                      └──────────────┘
                ▼                              │
        ┌───────────────┐              ┌──────▼───────┐
        │testing-advanced│             │ ai-assisted- │
        └───────────────┘              │ development  │
                │                      └──────────────┘
                ▼
        ┌───────────────┐
        │   external-   │
        │  integration  │
        └───────────────┘
```

**Reading order:**
1. Executives/managers: [EXECUTIVE-SUMMARY](EXECUTIVE-SUMMARY.md) only
2. Developers adopting: [QUICK-START](QUICK-START.md) → Core playbook (01-07) in order
3. Deep dive: Implementation guides based on need (check prerequisites at top of each)

---

## File Structure

```
/
├── README.md                      ← You are here
├── EXECUTIVE-SUMMARY.md           ← Management pitch
├── QUICK-START.md                 ← Implementation checklist
│
├── 01-core-philosophy.md          ← Required: First principles
├── 02-language-features.md        ← Required: C# features
├── 03-project-configuration.md    ← Required: Project setup
├── 04-developer-experience.md     ← Required: Local dev, DX
├── 05-economics.md                ← Required: Cost of change
├── 06-standards-culture.md        ← Required: Team culture
├── 07-anti-patterns.md            ← Required: What to avoid
│
├── testing-strategy.md            ← Required: Core testing
├── testing-advanced.md            ← Optional: Advanced testing
├── pr-review-process.md           ← Required: Review process
├── migration-guide.md             ← Optional: Brownfield adoption
├── external-integration.md        ← Optional: External APIs
└── ai-assisted-development.md     ← Optional: AI workflows
```

---

*Last updated: March 2026*
