> [← Back to Index](README.md) | [← Testing Strategy](testing-strategy.md)
>
> **Prerequisites:** [testing-strategy](testing-strategy.md)

# Advanced Testing Techniques

*Optional testing practices for mature teams seeking comprehensive coverage*

This guide covers advanced testing techniques. For essential testing (Unit, Integration, Architecture), see [Testing Strategy](testing-strategy.md).

---

## Table of Contents

1. [Property-Based Testing](#1-property-based-testing)
2. [Contract Testing](#2-contract-testing)
3. [Snapshot/Approval Testing](#3-snapshotapproval-testing)
4. [Mutation Testing](#4-mutation-testing)
5. [End-to-End Testing](#5-end-to-end-testing)

---

## 1. Property-Based Testing

### Purpose
Instead of testing specific examples, test *properties* that should always hold. The framework generates hundreds of random inputs to find edge cases.

### What It Catches
- Edge cases you didn't think of
- Off-by-one errors
- Boundary conditions
- Invariant violations
- Integer overflow

### Setup

```bash
dotnet add package FsCheck.Xunit
# Or
dotnet add package Hedgehog.Xunit
```

### Example: Testing Invariants

```csharp
using FsCheck;
using FsCheck.Xunit;

public class MoneyPropertyTests
{
    [Property]
    public Property Money_Addition_IsCommutative()
    {
        return Prop.ForAll<decimal, decimal>((a, b) =>
        {
            var moneyA = Money.From(Math.Abs(a) % 1_000_000); // Constrain to valid range
            var moneyB = Money.From(Math.Abs(b) % 1_000_000);

            return (moneyA + moneyB) == (moneyB + moneyA);
        });
    }

    [Property]
    public Property Money_Subtraction_NeverGoesNegative_WhenClamped()
    {
        return Prop.ForAll<decimal, decimal>((a, b) =>
        {
            var amount = Money.From(Math.Abs(a) % 1_000_000);
            var subtracted = Money.From(Math.Abs(b) % 1_000_000);

            var result = amount.SubtractClamped(subtracted);

            return result.Value >= 0;
        });
    }

    [Property]
    public Property Order_Total_EqualsSum_OfLineItems()
    {
        var genOrderLine = from productId in Arb.Generate<Guid>()
                           from quantity in Gen.Choose(1, 100)
                           from price in Gen.Choose(1, 10000)
                           select new OrderLine(
                               new ProductId(productId),
                               Quantity.From(quantity),
                               Money.From(price));

        var genOrder = from lines in Gen.ListOf(genOrderLine)
                       where lines.Count > 0
                       select new Order(lines.ToImmutableList());

        return Prop.ForAll(genOrder.ToArbitrary(), order =>
        {
            var expectedTotal = order.Lines.Sum(l => l.Price.Value * l.Quantity.Value);
            return order.Total.Value == expectedTotal;
        });
    }
}
```

### Example: Round-Trip Properties

```csharp
[Property]
public Property Json_RoundTrip_PreservesData()
{
    return Prop.ForAll<OrderDto>(dto =>
    {
        var json = JsonSerializer.Serialize(dto);
        var deserialized = JsonSerializer.Deserialize<OrderDto>(json);

        return dto.Equals(deserialized);
    });
}

[Property]
public Property Encryption_RoundTrip_PreservesData()
{
    return Prop.ForAll(
        Arb.Default.NonEmptyString().Generator,
        text =>
        {
            var encrypted = _crypto.Encrypt(text.Get);
            var decrypted = _crypto.Decrypt(encrypted);

            return text.Get == decrypted;
        });
}
```

### Custom Generators

```csharp
public static class CustomArbitraries
{
    public static Arbitrary<Email> EmailArbitrary()
    {
        var gen = from local in Arb.Generate<NonEmptyString>()
                  from domain in Gen.Elements("gmail.com", "outlook.com", "company.com")
                  select Email.Create($"{local.Get.Replace(" ", "")}@{domain}");

        return gen.Where(e => !e.IsError)
                  .Select(e => e.Value)
                  .ToArbitrary();
    }

    public static Arbitrary<Money> MoneyArbitrary()
    {
        var gen = from value in Gen.Choose(0, 1_000_000)
                  select Money.From(value / 100m);
        return gen.ToArbitrary();
    }
}

// Register in test class
public class OrderTests
{
    static OrderTests()
    {
        Arb.Register<CustomArbitraries>();
    }
}
```

### When to Use Property-Based Tests

| Use For | Example |
|---------|---------|
| Mathematical operations | Money arithmetic, percentage calculations |
| Serialization/deserialization | JSON round-trips, encryption |
| Data transformations | Parse → Format → Parse |
| Invariants | "Total is always non-negative" |
| Idempotent operations | "Applying twice equals applying once" |

---

## 2. Contract Testing

### Purpose
Ensure that service integrations (APIs you consume or provide) match expected contracts. Catch breaking changes before deployment.

### What It Catches
- API schema changes
- Missing fields in responses
- Type mismatches
- Broken backward compatibility

### Setup with Pact

```bash
dotnet add package PactNet
```

### Consumer Contract Test

```csharp
public class PaymentGatewayContractTests : IDisposable
{
    private readonly IPactBuilderV4 _pactBuilder;

    public PaymentGatewayContractTests()
    {
        var pact = Pact.V4("OrderService", "PaymentGateway", new PactConfig());
        _pactBuilder = pact.WithHttpInteractions();
    }

    [Fact]
    public async Task ProcessPayment_ReturnsApproval()
    {
        // Arrange
        _pactBuilder
            .UponReceiving("a valid payment request")
            .WithRequest(HttpMethod.Post, "/payments")
            .WithJsonBody(new
            {
                orderId = Match.Type("string"),
                amount = Match.Decimal(99.99m),
                currency = Match.Regex("USD", "^[A-Z]{3}$")
            })
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithJsonBody(new
            {
                transactionId = Match.Type("string"),
                status = Match.Regex("APPROVED", "^(APPROVED|DECLINED|PENDING)$"),
                processedAt = Match.Type(DateTime.UtcNow)
            });

        await _pactBuilder.VerifyAsync(async ctx =>
        {
            // Act
            var client = new PaymentGatewayClient(new HttpClient
            {
                BaseAddress = ctx.MockServerUri
            });

            var result = await client.ProcessPaymentAsync(new PaymentRequest
            {
                OrderId = "order-123",
                Amount = 99.99m,
                Currency = "USD"
            });

            // Assert
            result.Should().BeOfType<PaymentResult.Approved>();
        });
    }

    [Fact]
    public async Task ProcessPayment_WithInsufficientFunds_ReturnsDeclined()
    {
        _pactBuilder
            .Given("customer has insufficient funds")
            .UponReceiving("a payment request")
            .WithRequest(HttpMethod.Post, "/payments")
            .WithJsonBody(new { orderId = Match.Type(""), amount = Match.Decimal(), currency = "USD" })
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithJsonBody(new
            {
                status = "DECLINED",
                reason = Match.Type("Insufficient funds")
            });

        await _pactBuilder.VerifyAsync(async ctx =>
        {
            var client = new PaymentGatewayClient(new HttpClient { BaseAddress = ctx.MockServerUri });
            var result = await client.ProcessPaymentAsync(new PaymentRequest());

            result.Should().BeOfType<PaymentResult.Declined>();
        });
    }

    public void Dispose()
    {
        // Generates pact file for provider verification
    }
}
```

### Provider Verification

```csharp
[Trait("Category", "Contract")]
public class PaymentGatewayProviderTests : IClassFixture<PaymentApiFixture>
{
    private readonly PaymentApiFixture _fixture;

    [Fact]
    public void VerifyPactWithOrderService()
    {
        var verifier = new PactVerifier("PaymentGateway");

        verifier
            .WithHttpEndpoint(_fixture.ServerUri)
            .WithPactBrokerSource(new Uri("https://pact-broker.company.com"), options =>
            {
                options.ConsumerVersionSelectors(
                    new ConsumerVersionSelector { MainBranch = true },
                    new ConsumerVersionSelector { DeployedOrReleased = true }
                );
            })
            .WithProviderStateUrl(new Uri(_fixture.ServerUri, "/provider-states"))
            .Verify();
    }
}
```

---

## 3. Snapshot/Approval Testing

### Purpose
Verify that complex outputs (JSON, HTML, reports) don't change unexpectedly. Great for detecting regressions.

### What It Catches
- Unintended format changes
- Missing/extra fields in API responses
- Report layout regressions
- Serialization issues

### Setup

```bash
dotnet add package Verify.Xunit
```

### Example: API Response Snapshots

```csharp
[UsesVerify]
public class OrderApiSnapshotTests
{
    [Fact]
    public Task GetOrder_ResponseMatchesSnapshot()
    {
        var order = new OrderDto
        {
            Id = Guid.Parse("550e8400-e29b-41d4-a716-446655440000"),
            CustomerId = Guid.Parse("6ba7b810-9dad-11d1-80b4-00c04fd430c8"),
            Status = "Confirmed",
            Lines =
            [
                new() { ProductId = Guid.Parse("6ba7b811-9dad-11d1-80b4-00c04fd430c8"), Quantity = 2, Price = 29.99m },
                new() { ProductId = Guid.Parse("6ba7b812-9dad-11d1-80b4-00c04fd430c8"), Quantity = 1, Price = 49.99m }
            ],
            Total = 109.97m,
            CreatedAt = new DateTime(2024, 1, 15, 10, 30, 0, DateTimeKind.Utc)
        };

        return Verify(order);
    }
}

// First run creates: OrderApiSnapshotTests.GetOrder_ResponseMatchesSnapshot.verified.json
// Future runs compare against it
```

### Example: Report Snapshots

```csharp
[UsesVerify]
public class ReportGeneratorTests
{
    [Fact]
    public Task MonthlySalesReport_MatchesSnapshot()
    {
        var data = new SalesData
        {
            Period = new DateRange(new(2024, 1, 1), new(2024, 1, 31)),
            TotalRevenue = 125_430.50m,
            OrderCount = 1_234,
            TopProducts =
            [
                new("Widget Pro", 456, 45_678.00m),
                new("Gadget Plus", 321, 32_100.00m)
            ]
        };

        var report = _generator.Generate(data);

        return Verify(report);
    }
}
```

### Scrubbing Dynamic Values

```csharp
[ModuleInitializer]
public static void Initialize()
{
    VerifierSettings.ScrubMembers("CreatedAt", "UpdatedAt");
    VerifierSettings.ScrubInlineDateTimes("yyyy-MM-dd");
    VerifierSettings.AddScrubber(s => s.Replace(
        Guid.Empty.ToString(),
        "SCRUBBED-GUID"));
}
```

---

## 4. Mutation Testing

### Purpose
Verify that your tests actually catch bugs. Automatically introduces bugs ("mutants") and checks if tests fail.

### What It Catches
- Missing test coverage
- Weak assertions
- Tests that pass regardless of code changes
- Untested error paths

### Setup

```bash
dotnet tool install -g dotnet-stryker
```

### Configuration

```json
// stryker-config.json
{
    "stryker-config": {
        "project": "YourApp.Domain.csproj",
        "test-projects": ["YourApp.Domain.Tests.csproj"],
        "mutate": ["**/*.cs"],
        "reporters": ["html", "progress", "dashboard"],
        "threshold-high": 80,
        "threshold-low": 60,
        "threshold-break": 50
    }
}
```

### Running

```bash
cd src/YourApp.Domain
dotnet stryker

# Output:
# Mutants killed: 156
# Mutants survived: 12  ← These indicate weak tests!
# Mutation score: 92.8%
```

### Example: Finding Weak Tests

```csharp
// Your code
public bool IsExpired(License license, DateTimeOffset now)
{
    return license.ExpiresAt < now;  // Stryker changes < to <=
}

// Your test (WEAK - doesn't test boundary)
[Fact]
public void IsExpired_WhenPastExpiry_ReturnsTrue()
{
    var license = new License { ExpiresAt = DateTime.UtcNow.AddDays(-1) };
    var result = IsExpired(license, DateTime.UtcNow);
    result.Should().BeTrue();
}

// Fixed test (catches the mutation)
[Theory]
[InlineData(-1, true)]   // Past expiry
[InlineData(0, false)]   // Exactly at expiry - NOT expired
[InlineData(1, false)]   // Future expiry
public void IsExpired_ReturnsCorrectly(int daysFromNow, bool expected)
{
    var now = DateTimeOffset.UtcNow;
    var license = new License { ExpiresAt = now.AddDays(daysFromNow) };

    var result = IsExpired(license, now);

    result.Should().Be(expected);
}
```

### CI Integration

```yaml
- name: Mutation Testing
  run: |
    dotnet stryker --threshold-break 60
  continue-on-error: false  # Fail pipeline if mutation score too low
```

### When to Use

- **Weekly:** Run on critical domain modules
- **Before release:** Verify test quality
- **Not on every PR:** Too slow for continuous feedback

---

## 5. End-to-End Testing

### Purpose
Test complete user flows through the entire system. Last line of defense before production.

### What It Catches
- Full flow failures
- Integration between multiple services
- UI/UX issues
- Performance problems under realistic conditions

### Setup with Playwright

```bash
dotnet add package Microsoft.Playwright
dotnet new nunit -n E2ETests
```

### Example: Checkout Flow

```csharp
[Trait("Category", "E2E")]
public class CheckoutFlowTests : PageTest
{
    [Test]
    public async Task CompleteCheckout_AsNewCustomer()
    {
        // Navigate to product
        await Page.GotoAsync("https://staging.example.com/products");
        await Page.ClickAsync("[data-testid=product-widget-pro]");

        // Add to cart
        await Page.ClickAsync("[data-testid=add-to-cart]");
        await Page.ClickAsync("[data-testid=cart-icon]");

        // Proceed to checkout
        await Page.ClickAsync("[data-testid=checkout-button]");

        // Fill customer info
        await Page.FillAsync("[data-testid=email]", "test@example.com");
        await Page.FillAsync("[data-testid=name]", "Test User");
        await Page.FillAsync("[data-testid=address]", "123 Test St");

        // Fill payment (test card)
        await Page.FrameLocator("iframe").First.FillAsync(
            "[data-testid=card-number]",
            "4242424242424242");

        // Complete order
        await Page.ClickAsync("[data-testid=place-order]");

        // Verify success
        await Expect(Page.Locator("[data-testid=order-confirmation]")).ToBeVisibleAsync();
        await Expect(Page.Locator("[data-testid=order-number]")).ToContainTextAsync("ORD-");
    }
}
```

### When E2E Tests Are Worth It

| Scenario | Use E2E? |
|----------|----------|
| Critical revenue paths (checkout, signup) | Yes |
| Complex multi-step flows | Yes |
| Simple CRUD operations | No - Unit/Integration sufficient |
| Every feature | No - Too slow, too brittle |

### Best Practices

- **Keep minimal:** Only test critical paths
- **Use data-testid:** Avoid brittle CSS selectors
- **Run on staging:** Not against production
- **Parallelize:** Run in CI with multiple workers
- **Retry flaky tests:** But investigate and fix root cause

---

## Test Category Summary

| Category | Speed | Catches | When to Use |
|----------|-------|---------|-------------|
| Property | < 100ms | Edge cases | Complex calculations, invariants |
| Contract | < 10s | API changes | Service boundaries |
| Snapshot | < 100ms | Regressions | Complex outputs |
| Mutation | Minutes | Weak tests | Weekly/before release |
| E2E | < 60s | Flow failures | Critical paths only |

---

## Package Summary

| Package | Purpose |
|---------|---------|
| `FsCheck.Xunit` | Property-based testing |
| `PactNet` | Contract testing |
| `Verify.Xunit` | Snapshot testing |
| `Stryker` | Mutation testing |
| `Microsoft.Playwright` | E2E testing |

---

[← Testing Strategy](testing-strategy.md) | [Index](README.md)
