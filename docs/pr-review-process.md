> [← Back to Index](README.md)
>
> **Prerequisites:** [03-project-configuration](03-project-configuration.md), [07-anti-patterns](07-anti-patterns.md)

# PR Review Process

*Automated checks catch syntax; human review catches intent.*

---

## Two-Layer Enforcement

### Layer 1: Automated (CI Pipeline)

These run automatically on every PR—no human effort required:

- Build with `--warnaserror` (nullable, type safety)
- Unit, integration, and architecture tests
- Formatting (`dotnet format --verify-no-changes`)
- Analyzers (code quality rules)

**If CI passes, reviewers don't need to check these things.**

### Layer 2: Human Review (1 Reviewer + Automation)

Reviewers focus on what automation can't catch:

| Focus Area | What to Check |
|------------|---------------|
| **Intent** | Does this actually solve the problem in the ticket? |
| **Edge cases** | Did the author miss scenarios the tests don't cover? |
| **Design** | Is the abstraction level appropriate? Would a new team member understand this? |
| **Patterns** | Does it follow Result pattern, typed IDs, immutable DTOs? |

---

## PR Template

Copy to `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## What does this PR do?
Closes #

## How to verify
1.
2.
3.

## Type of change
- [ ] Bug fix
- [ ] New feature
- [ ] Refactoring
- [ ] Breaking change

## Author Checklist
- [ ] Feature works as described (verified manually)
- [ ] Tests cover happy path AND edge cases
- [ ] Self-reviewed against [anti-patterns](07-anti-patterns.md)
- [ ] AI-assisted: [ ] Yes / [ ] No (if yes, see [AI review guidelines](#reviewing-ai-generated-code))
```

---

## Review Focus

### What to Look For

1. **Feature correctness** — Does it actually solve the problem?
2. **Missing edge cases** — What happens with empty input? Null? Concurrent access?
3. **Anti-patterns** — See [Anti-Patterns](07-anti-patterns.md)

### What NOT to Check (Automation Handles It)

- Formatting and style
- Null reference possibilities (nullable analysis)
- Unused imports
- Naming conventions (analyzers)
- Build errors

---

## Review Comments

Use consistent format:

```
❌ Issue: [description] — see [link to docs]
✅ Suggest: [specific fix]
```

**Examples:**

```
❌ Issue: Using `Guid` for OrderId — see 07-anti-patterns.md
✅ Suggest: `readonly record struct OrderId(Guid Value)`
```

```
❓ Question: This handles case X, but what happens when Y?
```

---

## Process Rules

| Rule | Rationale |
|------|-----------|
| 1 reviewer + passing CI | Automation catches mechanical issues |
| Max 400 lines per PR | Larger PRs get superficial reviews |
| "How to verify" required | Forces author to think about behavior |
| Demo for major features | 5-min walkthrough before merge |

### When to Require 2 Reviewers

- Breaking API changes
- Security-sensitive code
- Infrastructure changes (CI, deployment)
- AI-generated code making significant changes (> 100 lines)

---

## Reviewing AI-Generated Code

AI-authored PRs require extra scrutiny. AI is good at syntax but weak at business intent.

> **Full guide:** [AI-Assisted Development](ai-assisted-development.md)

### AI-Specific Review Checklist

| Check | Why |
|-------|-----|
| **Intent match** | Does this actually solve the ticket, or just look like it does? |
| **Edge cases** | AI often handles happy path but misses boundaries, nulls, concurrency |
| **Pattern adherence** | Does it follow *your* patterns (Result types, typed IDs) or generic patterns? |
| **Test quality** | Tests written by AI may pass without testing meaningful behavior |
| **Unnecessary complexity** | AI may over-engineer simple solutions |

### Red Flags in AI Code

- Tests that mirror implementation too closely (testing mocks, not behavior)
- Generic variable names (`data`, `result`, `item`) instead of domain terms
- Missing error handling for expected failures
- Patterns not used elsewhere in codebase
- Comments explaining *what* instead of *why*

### Recommended Review Flow

1. Read the ticket acceptance criteria first
2. Review tests before implementation—do they verify the criteria?
3. Check implementation matches your team's patterns, not generic patterns
4. Run the code and verify behavior yourself

---

## Escalation

When reviewers disagree:

1. Discuss in PR comments (keep it technical)
2. If unresolved, 15-min sync call
3. If still unresolved, lead makes final call with documented reasoning

---

## Metrics

| Metric | Target | Why |
|--------|--------|-----|
| PR cycle time | < 24 hours | Fast feedback loop |
| Anti-patterns caught | Trending down | Standards being internalized |
| Bugs from merged PRs | < 1 per sprint | Review quality indicator |

---

[← Back to Index](README.md)
