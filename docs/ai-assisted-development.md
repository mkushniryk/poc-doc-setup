> [← Back to Index](README.md)
>
> **Prerequisites:** [testing-strategy](testing-strategy.md), [pr-review-process](pr-review-process.md)

# AI-Assisted Development Guide

*AI is excellent at "how to code" but weak at "what to build." Structure accordingly.*

---

## The Core Problem

AI agents can produce code that:
- ✅ Compiles
- ✅ Passes tests it wrote
- ✅ Looks correct
- ❌ Doesn't actually solve the business problem
- ❌ Misses edge cases humans would catch
- ❌ Follows patterns it learned elsewhere, not yours

**Your guardrails must verify intent, not just syntax.**

---

## The Economics of AI-Assisted Development

AI subscriptions are expensive. GitHub Copilot, Claude, GPT-4 — costs add up quickly for a team.

**But here's what actually happens:**

### The Honest Budget Reality

Let's be clear about what AI costs look like for a manager (mid-level contractor at ~$35/hour):

| Line Item | Before AI | After AI |
|-----------|-----------|----------|
| Developer salary | $5,600/month | $5,600/month |
| AI subscriptions | $0 | $50–200/month |
| **Total** | $5,600 | $5,650–5,800 |

**You don't save money on your monthly bill.** The developer keeps working. The salary stays the same. AI is an addition, not a replacement.

### What You Actually Get

The value isn't cost reduction — it's **output multiplication**:

| What Changes | Before AI | After AI |
|--------------|-----------|----------|
| Features shipped per sprint | 3 | 5–7 |
| Bug rate | Baseline | Lower (less manual error) |
| Time stuck on syntax/boilerplate | 30% of day | 5% of day |
| Documentation quality | "We'll do it later" | Actually exists |
| Test coverage | "We'll add tests later" | Tests written upfront |

**Same cost. More output. Higher quality.**

### The Real Value Drivers

| Value | How AI Delivers It |
|-------|-------------------|
| **Pace** | Tasks done in minutes, not hours |
| **Performance** | Developers focus on hard problems, not typing |
| **Quality** | More time for review, less time on boilerplate |
| **Trust** | Deliverables are complete, not half-done |
| **Reputation** | Ship faster, respond to clients faster |
| **Morale** | Developers do interesting work, not tedious work |

### What AI Actually Saves Time On

| Task | Human time | AI-assisted time | Time freed |
|------|-----------|------------------|------------|
| Writing boilerplate code | 2 hours | 15 min | 1.75 hours |
| Writing unit tests | 3 hours | 30 min | 2.5 hours |
| Initial implementation draft | 4 hours | 45 min | 3.25 hours |
| Regex, SQL, config syntax | 30 min each | 2 min each | 28 min each |
| Documentation drafts | 1 hour | 10 min | 50 min |
| Code review prep/cleanup | 1 hour | 15 min | 45 min |

**That "saved" time doesn't disappear — it goes into the next feature, better testing, or fixing tech debt.**

### The ROI Calculation (Correct Framing)

```
Monthly AI cost:           $50
Developer rate:            $35/hour (mid-level)
Break-even point:          1.5 hours saved per month

Actual hours freed:        40+ hours/month
Value of freed time:       $1,400/month
ROI:                       28x return
```

**Break-even is trivially low.** If AI helps you write one unit test faster, skip one Stack Overflow session, or generate one boilerplate file — you've already paid for the month.

| At $35/hour | Saves | Pays for |
|-------------|-------|----------|
| One regex generation | 30 min | Half the subscription |
| One test file | 2.5 hours | 1.7 months |
| One boilerplate module | 2 hours | 1.3 months |

**Even at lower market rates, the economics are overwhelming.** AI gives you 40 hours of capacity for $50. Hiring gives you 160 hours for $5,600. That's $1.25/hour vs $35/hour for incremental capacity.

### What AI Eliminates

AI doesn't reduce headcount — it eliminates the work that drains developers:

| "Stupid stuff" AI handles | Why it matters |
|---------------------------|----------------|
| Syntax lookup | No more Stack Overflow tabs |
| Boilerplate typing | Focus on logic, not repetition |
| Test scaffolding | Tests get written instead of skipped |
| Documentation first drafts | Docs actually exist |
| Error message formatting | Consistent, helpful errors |
| Regex writing | No 30-minute debugging sessions |

**Developers do more meaningful work. That's the value.**

### The Competitive Reality

If your competitors use AI and you don't:

| Impact | Consequence |
|--------|-------------|
| They ship faster | You lose market position |
| They iterate more | You're stuck in longer cycles |
| Their devs are less fatigued | Your devs burn out on repetitive work |
| They attract talent | "No AI tools" is a red flag for candidates |

### The Investment Mindset

| Mindset | View |
|---------|------|
| **Cost center** | "AI adds $50/developer/month to my budget" |
| **Capacity investment** | "AI adds 40 hours/developer/month of output" |

**AI is not a cost reduction tool. It's a capacity multiplier.**

You're not paying less. You're getting more.

---

## AI Development Trust Levels

| Level | Description | Supervision Required |
|-------|-------------|---------------------|
| **L1: Autocomplete** | Copilot-style suggestions | Low - developer accepts/rejects inline |
| **L2: Assisted** | AI drafts, human refines | Medium - review each file |
| **L3: Supervised Autonomous** | AI completes tasks, human approves | High - structured verification |
| **L4: Fully Autonomous** | AI develops end-to-end | Very High - rigorous gates |

This guide focuses on **L3-L4** where AI produces PRs independently.

---

## The Inverted Development Flow

### Traditional Flow (Human-Driven)
```
Ticket → Developer thinks → Code → Tests → Review
```

### AI-Safe Flow (Specification-Driven)
```
Ticket → Human writes acceptance criteria → Human writes failing tests → AI implements → Automated verification → Human review
```

**Key principle:** Humans define "what correct looks like" BEFORE AI writes code.

---

## Phase 1: Specification Layer (Human Responsibility)

### 1.1 Ticket Requirements

Every ticket for AI must include:

```markdown
## User Story
As a [role], I want [feature] so that [benefit].

## Acceptance Criteria (Required for AI)
- [ ] Given [context], when [action], then [result]
- [ ] Given [edge case], when [action], then [specific handling]
- [ ] Given [error condition], when [action], then [error response]

## Technical Constraints
- Must use: [specific patterns, e.g., Result<T> for errors]
- Must NOT use: [anti-patterns to avoid]
- Reference: [link to relevant pattern docs]

## Test Scenarios (Human-Written)
1. Happy path: [description]
2. Edge case: [description]  
3. Error case: [description]

## Out of Scope
- [Explicitly what this ticket does NOT include]
```

### 1.2 Pre-Written Acceptance Tests

**Critical:** Write failing tests BEFORE AI implements.

```csharp
// Human writes this test first
public class CreateOrderTests
{
    [Fact]
    public async Task CreateOrder_WithValidItems_ReturnsOrderWithCorrectTotal()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = TestData.ValidCustomerId,
            Items = new[]
            {
                new OrderItem { ProductId = TestData.ProductA, Quantity = 2 },
                new OrderItem { ProductId = TestData.ProductB, Quantity = 1 }
            }
        };

        // Act
        var result = await _handler.Handle(request, CancellationToken.None);

        // Assert
        result.IsError.Should().BeFalse();
        result.Value.Total.Should().Be(Money.From(59.97m)); // 2x19.99 + 1x19.99
        result.Value.Items.Should().HaveCount(2);
        result.Value.Status.Should().Be(OrderStatus.Pending);
    }

    [Fact]
    public async Task CreateOrder_WithEmptyItems_ReturnsValidationError()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = TestData.ValidCustomerId,
            Items = Array.Empty<OrderItem>()
        };

        var result = await _handler.Handle(request, CancellationToken.None);

        result.IsError.Should().BeTrue();
        result.FirstError.Type.Should().Be(ErrorType.Validation);
    }

    [Fact]
    public async Task CreateOrder_WithNonexistentProduct_ReturnsNotFoundError()
    {
        // ... test for edge case
    }
}
```

**Why this works:** AI cannot "cheat" by writing tests that match its implementation. Tests define correctness independently.

---

## Phase 2: AI Implementation Layer

### 2.1 Structured Prompts

When invoking AI agents, always include:

```markdown
## Context
You are implementing a feature for a .NET 8 application.

## Required Reading (attach or reference)
- [02-language-features.md] - Use Result<T> pattern for all errors
- [07-anti-patterns.md] - Avoid these specific patterns
- [Project structure overview]

## Task
Implement [specific feature] to make the following tests pass:
[paste the pre-written tests]

## Constraints
- DO use: typed IDs, Result<T>, immutable records
- DO NOT use: exceptions for flow control, raw Guid for IDs, DateTime.Now
- Follow existing patterns in: [reference file path]

## Expected Output
- Implementation files
- Any additional unit tests for edge cases YOU identify
- Explanation of design decisions
```

### 2.2 AI Context Files

Create a `.ai-context/` folder with condensed guidelines:

```
.ai-context/
├── patterns.md          # Condensed version of 02-language-features.md
├── anti-patterns.md     # Condensed version of 07-anti-patterns.md
├── project-structure.md # Where things go
└── testing-conventions.md # How to write tests
```

These files are optimized for AI consumption (shorter, more examples, less prose).

---

## Phase 3: Verification Layer (Automated)

### 3.1 Extended CI Pipeline for AI PRs

```yaml
name: AI PR Verification

on:
  pull_request:
    # Trigger enhanced checks for AI-labeled PRs
    
jobs:
  standard-checks:
    # ... existing tests, format, analyzers
    
  ai-specific-checks:
    runs-on: ubuntu-latest
    if: contains(github.event.pull_request.labels.*.name, 'ai-generated')
    steps:
      - name: Verify acceptance tests exist
        run: |
          # Ensure pre-written acceptance tests still exist and pass
          # These should have been written BEFORE the AI PR
          
      - name: Mutation testing
        run: |
          dotnet stryker --threshold-high 80 --threshold-low 60
          # AI-generated code must survive mutation testing
          
      - name: Architecture tests
        run: |
          dotnet test --filter "Category=Architecture"
          # Verify AI didn't violate architectural boundaries
          
      - name: Anti-pattern scan
        run: |
          # Custom analyzer or script to catch common AI mistakes
          dotnet build /p:TreatWarningsAsErrors=true
          
      - name: Test coverage delta
        run: |
          # AI PR must not decrease coverage
          # New code must have >90% coverage
```

### 3.2 AI-Specific Analyzers

Create custom Roslyn analyzers for AI-prone mistakes:

```csharp
// Analyzer to catch AI's tendency to use generic exception handling
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class AICatchAllExceptionAnalyzer : DiagnosticAnalyzer
{
    // Flags: catch (Exception) without specific handling
}

// Analyzer to catch AI's tendency to ignore cancellation tokens
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class AIIgnoredCancellationTokenAnalyzer : DiagnosticAnalyzer
{
    // Flags: async methods that don't propagate CancellationToken
}
```

### 3.3 Mandatory Mutation Testing

AI-written tests often have weak assertions. Mutation testing catches this:

```bash
# In CI for AI PRs
dotnet stryker \
  --project "YourProject.csproj" \
  --threshold-high 80 \  # Fail if mutation score below 80%
  --reporters "['html', 'json']"
```

If AI wrote tests that don't actually verify behavior, mutations will survive.

---

## Phase 4: Human Review Layer (Enhanced for AI)

### 4.1 AI PR Review Checklist

```markdown
## AI-Generated PR Review Checklist

### Intent Verification (Most Critical)
- [ ] Does this actually solve the problem described in the ticket?
- [ ] Are there requirements in the ticket that were missed?
- [ ] Did AI make assumptions that weren't in the spec?

### Edge Case Analysis
- [ ] What inputs could break this? (empty, null, negative, huge)
- [ ] What happens under concurrent access?
- [ ] What happens when dependencies fail?

### Pattern Compliance
- [ ] Uses Result<T>, not exceptions for expected failures
- [ ] Uses typed IDs, not raw Guid/int
- [ ] Immutable where appropriate
- [ ] No anti-patterns from 07-anti-patterns.md

### AI-Specific Red Flags
- [ ] Overly generic variable names (data, result, item)
- [ ] Comments that restate the obvious
- [ ] Unused parameters or dead code
- [ ] Hardcoded values that should be configurable
- [ ] Missing logging/observability
- [ ] Tests that only test the happy path
```

### 4.2 Mandatory Sections in AI PRs

```markdown
## AI-Generated PR

### Prompt Used
<!-- What instructions were given to the AI? -->

### Tests Written Before AI Implementation
<!-- Link to commit with pre-written tests -->

### AI's Design Decisions
<!-- AI should explain WHY it chose this approach -->

### Human Verification
- [ ] Ran the feature locally and verified behavior
- [ ] Compared implementation against acceptance criteria
- [ ] Reviewed AI's explanation for reasonableness
```

### 4.3 Review Time Multiplier

| PR Source | Review Time |
|-----------|-------------|
| Senior engineer | 1x |
| Junior engineer | 1.5x |
| AI-generated | 2x |

**AI PRs require MORE scrutiny, not less.**

---

## Phase 5: Production Safeguards

### 5.1 Feature Flags for AI Code

```csharp
// All AI-implemented features behind flags initially
if (await _featureManager.IsEnabledAsync("AI_OrderProcessingV2"))
{
    return await _aiImplementedHandler.Handle(request, ct);
}
return await _humanImplementedHandler.Handle(request, ct);
```

### 5.2 Shadow Mode Testing

Run AI implementation in parallel with existing code:

```csharp
public async Task<Result<Order>> CreateOrder(CreateOrderRequest request, CancellationToken ct)
{
    var originalResult = await _originalHandler.Handle(request, ct);
    
    // Shadow execution - log differences, don't use result
    _ = Task.Run(async () =>
    {
        var aiResult = await _aiHandler.Handle(request, ct);
        if (!ResultsMatch(originalResult, aiResult))
        {
            _logger.LogWarning("AI implementation diverged: {Diff}", 
                ComputeDiff(originalResult, aiResult));
        }
    });
    
    return originalResult;
}
```

### 5.3 Gradual Rollout

```
Week 1: 1% traffic to AI implementation, monitor
Week 2: 10% if no issues
Week 3: 50% if no issues
Week 4: 100% with instant rollback capability
```

---

## Anti-Patterns in AI Development

| Anti-Pattern | Why It's Dangerous | Better Approach |
|--------------|-------------------|-----------------|
| "AI, implement this ticket" | No verification of intent | Spec-first with pre-written tests |
| Trusting AI-written tests | AI writes tests that pass its code | Human writes acceptance tests first |
| Reviewing AI PRs casually | "It's just AI, it probably works" | 2x review time, enhanced checklist |
| No feature flags | Can't roll back AI bugs | Always flag AI features initially |
| AI writes entire PR | No human context | AI implements, human refines |

---

## Metrics for AI Development

| Metric | Target | Action if Missed |
|--------|--------|------------------|
| AI PR acceptance rate | > 70% | Improve prompts/context |
| AI bugs in production | 0 | Add more gates |
| AI PR review time | < 2 hours | Better spec quality |
| Mutation test score | > 80% | Require better tests |
| Rollback rate for AI features | < 5% | Longer shadow testing |

---

## Quick Start: Enabling AI for a Feature

1. **Write the ticket** with full acceptance criteria
2. **Write failing acceptance tests** (human responsibility)
3. **Create AI prompt** with context files and constraints
4. **Run AI** to generate implementation
5. **CI verifies** all tests pass + mutation testing
6. **Human reviews** with AI checklist (2x time)
7. **Deploy behind feature flag**
8. **Monitor in shadow mode**
9. **Gradual rollout**

---

## Summary

| Layer | Who | Responsibility |
|-------|-----|----------------|
| Specification | Human | Define WHAT correct looks like |
| Acceptance Tests | Human | Encode correctness before AI runs |
| Implementation | AI | Write code to pass tests |
| Verification | Automated | Extended CI + mutation testing |
| Review | Human | Intent verification, 2x scrutiny |
| Production | Automated | Feature flags, shadow mode, gradual rollout |

**The key insight:** AI should never define what "correct" means. Humans define correctness via specs and tests. AI implements. Automation verifies. Humans approve.

---

## AI Usage Policy by Experience Level

*The hardest question in AI-assisted development isn't technical—it's developmental.*

### The Core Tension

| Perspective | Argument |
|-------------|----------|
| **AI stunts growth** | Juniors who rely on AI skip the struggle that builds deep understanding. They become "prompt engineers" who can't debug when AI fails. |
| **AI is the new baseline** | We don't write assembly anymore. Maybe writing boilerplate from scratch is the new assembly—a skill that's educational but not professionally necessary. |

**Both perspectives have merit.** The answer isn't binary prohibition or full access—it's structured progression.

### What AI Cannot Teach

These skills require human struggle and cannot be shortcut:

| Skill | Why AI Can't Teach It |
|-------|----------------------|
| **Debugging intuition** | Knowing WHERE to look when things break |
| **System mental models** | Understanding how components interact |
| **Reading others' code** | Navigating unfamiliar codebases |
| **Failure pattern recognition** | "I've seen this bug shape before" |
| **Trade-off reasoning** | Why we chose X over Y |
| **Requirements translation** | Business problem → technical solution |
| **Code review judgment** | Is this good? Is this maintainable? |

### What AI Makes Less Critical

These skills are becoming commoditized:

| Skill | New Reality |
|-------|-------------|
| Syntax memorization | AI knows every language |
| Boilerplate writing | AI generates it instantly |
| API documentation lookup | AI has it internalized |
| CRUD implementation | Nearly fully automatable |
| Regex writing | AI is better than most humans |

### The Real Risk

A junior who uses AI for everything becomes:
- Unable to debug when AI produces subtly wrong code
- Unable to review others' code (including AI's)
- Dependent on AI availability
- Lacking the "taste" that comes from writing bad code and learning

**The failure mode isn't "they write slower"—it's "they can't recognize wrong."**

---

## Tiered AI Policy by Level

### Junior Engineers (0-2 years)

**Philosophy:** Build foundations first. AI is a teacher, not a replacement.

| Zone | AI Usage | Rationale |
|------|----------|-----------|
| **🔴 Prohibited** | Core algorithms, data structures, debugging | Must build mental models through struggle |
| **🟡 Explain-to-use** | Implementation after design is complete | Can use AI, but must explain every line in PR |
| **🟢 Permitted** | Documentation, test boilerplate, formatting | Low learning value, high tedium |

**Specific Rules:**
- First 6 months: No AI for implementation (autocomplete OK)
- Months 6-12: AI allowed if they can explain the code in review
- After 12 months: Graduate to mid-level policy based on demonstrated understanding

**Required Exercises (No AI):**
```
□ Implement a linked list from scratch
□ Debug a production issue using only logs and debugger
□ Read and summarize an unfamiliar codebase
□ Write a PR review that catches real issues
□ Explain a complex piece of existing code to the team
```

**Learning Checkpoints:**
- Can they debug AI-generated code when it fails?
- Can they explain WHY the code works, not just WHAT it does?
- Can they spot anti-patterns in AI output?

### Mid-Level Engineers (2-5 years)

**Philosophy:** AI as a tool, not a crutch. Use it, but maintain ability to work without it.

| Zone | AI Usage | Rationale |
|------|----------|-----------|
| **🔴 Prohibited** | Architecture decisions, critical debugging | Must own system understanding |
| **🟡 Review-required** | Feature implementation | Can use AI freely, but must deeply review output |
| **🟢 Permitted** | Boilerplate, tests, documentation, refactoring | Efficiency gain with low risk |

**Specific Rules:**
- All AI-assisted PRs must be self-reviewed before requesting review
- Must be able to implement a feature without AI once per quarter (spot check)
- Must review junior AI-generated PRs (teaches recognition)

**Growth Requirements:**
```
□ One "no-AI week" per quarter to maintain fundamentals
□ Lead debugging sessions without AI assistance
□ Mentor juniors on when NOT to use AI
□ Identify and document cases where AI produced incorrect code
```

### Senior Engineers (5+ years)

**Philosophy:** AI is a force multiplier. Use judgment on when it helps vs. hinders.

| Zone | AI Usage | Rationale |
|------|----------|-----------|
| **🔴 Prohibited** | Security-critical code (verify manually) | AI can introduce subtle vulnerabilities |
| **🟢 Permitted** | Everything else | Trusted judgment |

**Responsibilities:**
- Review AI-generated PRs from all levels
- Maintain team's AI context files and prompts
- Identify when AI is producing systematic errors
- Mentor on AI-human collaboration patterns

---

## The "Driving Test" Approach

Before granting AI privileges, verify foundational competence:

### Junior → Mid AI Assessment

```markdown
## AI Readiness Assessment (Quarterly)

### Practical Test (No AI allowed, 2 hours)
1. Implement [specific feature] from scratch
2. Debug [intentionally broken code]
3. Review [AI-generated PR] and identify issues

### Knowledge Check
- Explain how [core system component] works
- What would you check if [failure scenario] occurred?
- Why do we use [pattern X] instead of [pattern Y]?

### Result
□ Pass: Graduate to mid-level AI policy
□ Partial: Specific areas need more no-AI practice
□ Fail: Extend foundational period
```

### Mid → Senior AI Assessment

```markdown
## Senior AI Trust Assessment

### Demonstrated Competencies
- [ ] Successfully debugged AI-generated code in production
- [ ] Identified systematic AI errors and documented solutions
- [ ] Mentored junior on AI code review
- [ ] Made architecture decisions that AI couldn't have made

### Judgment Check
- When would you NOT use AI for a task?
- How do you verify AI-generated code correctness?
- What patterns does AI frequently get wrong in our codebase?
```

---

## Implementation: Gradual AI Enablement

### Week 1-4: Establish Baseline
- All engineers work without AI for one sprint
- Document: What's hard? What's tedious?
- Baseline velocity measurement

### Week 5-8: Seniors Only
- Seniors get full AI access
- Document: Productivity gains, error patterns
- Seniors build AI context files

### Week 9-12: Mid-Level Rollout
- Mid-level engineers get AI with review-required zones
- Seniors review all AI-assisted PRs
- Weekly retro: What AI got wrong this week?

### Week 13+: Junior Graduated Access
- Juniors get explain-to-use zones
- Quarterly assessments begin
- Track: Can juniors debug AI failures?

---

## The "Why" Conversation

When a junior asks "Why can't I use AI for this?", the answer is:

> "AI generates code. Engineers understand systems. If you can't debug when AI fails—and it will fail—you're not an engineer, you're a prompt operator. We're investing in making you someone who can work WITH AI effectively, which requires first understanding what AI is doing. It's like learning to drive manual before automatic—you develop intuition you'll use forever."

When a senior says "Let them use AI, that's the future":

> "The future requires engineers who can verify AI output, debug AI failures, and recognize when AI is confidently wrong. That judgment only comes from having written and debugged code yourself. We're not anti-AI—we're pro-competence."

---

## Metrics to Track

| Metric | Healthy Range | Warning Sign |
|--------|---------------|--------------|
| Junior debug success rate (without AI) | > 70% | Dropping means AI dependency |
| AI PR rejection rate by level | Juniors > Seniors | Juniors same as seniors = rubber-stamping |
| Time to resolve AI-generated bugs | Decreasing | Increasing = skill atrophy |
| "No-AI" assessment pass rate | > 80% | Below = policy too permissive |

---

## Open Questions (Revisit Quarterly)

- Is "writing code from scratch" becoming like "writing assembly"—educational but professionally optional?
- Are we seeing skill atrophy in mid-level engineers who had early AI access?
- What new skills are emerging that we should assess?
- Should the prohibited zones shrink as AI improves?

**This policy is not permanent.** Re-evaluate every quarter based on:
1. AI capability improvements
2. Industry skill expectations
3. Observed developer growth patterns
4. Production incident patterns

