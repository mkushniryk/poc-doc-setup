> [← Back to Index](README.md)
>
> **Prerequisites:** [01-core-philosophy](01-core-philosophy.md), [02-language-features](02-language-features.md), [03-project-configuration](03-project-configuration.md)

# Testing Strategy

*Testing is not about finding bugs—it's about making bugs impossible to ship.*

---

## Table of Contents

1. [Testing Priorities](#1-testing-priorities)
2. [Minimum Viable Test Suite](#2-minimum-viable-test-suite)
3. [Unit Testing](#3-unit-testing)
4. [Integration Testing](#4-integration-testing)
5. [Architecture Testing](#5-architecture-testing)
6. [CI Quality Gates](#6-ci-quality-gates)
7. [Quick Reference](#7-quick-reference)

For advanced testing techniques (Property-Based, Contract, Mutation, E2E), see [Advanced Testing](testing-advanced.md).

---

## 1. Testing Priorities

### The Testing Pyramid

```
        ╱╲
       ╱  ╲         E2E Tests (few, critical paths only)
      ╱────╲        ↑ Optional - see testing-advanced.md
     ╱      ╲
    ╱────────╲      Integration Tests (moderate, real dependencies)
   ╱          ╲     ↑ Required for DB/API code
  ╱────────────╲
 ╱              ╲   Unit Tests (many, fast, focused)
╱────────────────╲  ↑ Required for all business logic
```

### Priority Tiers

| Priority | Test Type | When Required | Speed |
|----------|-----------|---------------|-------|
| **P0** | Unit tests | All business logic | < 5ms |
| **P0** | Architecture tests | Always (CI gate) | < 1s |
| **P1** | Integration tests | Database/API code | < 5s |
| **P2** | Contract tests | External service boundaries | < 10s |
| **P3** | Property tests | Complex calculations | < 100ms |
| **P3** | Mutation tests | Weekly/pre-release | Minutes |
| **P3** | E2E tests | Critical revenue paths | < 60s |

> **Note on testing shapes:** Some teams prefer the "testing trophy" (more integration tests, fewer unit tests) popularized by Kent C. Dodds. The pyramid works well for .NET because: (1) xUnit/NUnit unit tests are extremely fast (<5ms each), (2) Testcontainers makes realistic integration tests practical, and (3) strongly-typed C# means many "integration" bugs are caught at compile time. Choose what fits your team—the key is fast feedback, not the shape.

### What Each Level Catches

| Level | Catches | Example |
|-------|---------|---------|
| Unit | Logic bugs, edge cases | `CalculateDiscount` returns wrong value |
| Integration | Wiring bugs, DB issues | Repository query returns wrong data |
| Architecture | Design drift, dependency violations | Domain depending on Infrastructure |

---

## 2. Minimum Viable Test Suite

Start here. These tests provide the best ROI for your time.

### Every Project Must Have

1. **Unit tests for business logic**
   - Domain entities and value objects
   - Calculation functions
   - State machine transitions

2. **Integration tests for data access**
   - Repository queries return correct data
   - Transactions commit properly
   - Migrations work

3. **Architecture tests for design rules**
   - Layer dependencies are correct
   - Naming conventions enforced

### PR Checklist

```markdown
## Required
- [ ] Unit tests for new/changed business logic
- [ ] Edge cases covered (null, empty, boundary values)
- [ ] Error paths tested (not just happy path)
- [ ] All tests pass locally

## For Database Changes
- [ ] Integration tests with real database
- [ ] Migration tested (up and down)
```

---

## 3. Unit Testing

### Purpose
Test individual units of logic in isolation. Fast feedback on correctness.

### Setup

```bash
dotnet add package xunit
dotnet add package FluentAssertions
dotnet add package NSubstitute
```

### Example: Testing Pure Business Logic

```csharp
public class DiscountCalculatorTests
{
    [Theory]
    [InlineData(100, 0, 100)]      // No discount
    [InlineData(100, 10, 90)]      // 10% off
    [InlineData(100, 100, 0)]      // Free
    [InlineData(0, 50, 0)]         // Zero amount
    public void CalculateDiscount_ReturnsCorrectAmount(
        decimal amount,
        decimal discountPercent,
        decimal expected)
    {
        var result = DiscountCalculator.Apply(
            Money.From(amount),
            Percentage.From(discountPercent));

        result.Value.Should().Be(expected);
    }

    [Fact]
    public void CalculateDiscount_WithNegativePercent_ReturnsError()
    {
        var result = Percentage.Create(-5);

        result.IsError.Should().BeTrue();
        result.FirstError.Code.Should().Be("Percentage.Invalid");
    }
}
```

### Example: Testing with Mocks

```csharp
public class OrderServiceTests
{
    private readonly IOrderRepository _repository = Substitute.For<IOrderRepository>();
    private readonly IEmailService _emailService = Substitute.For<IEmailService>();
    private readonly TimeProvider _timeProvider = Substitute.For<TimeProvider>();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _timeProvider.GetUtcNow().Returns(new DateTimeOffset(2024, 1, 15, 10, 0, 0, TimeSpan.Zero));
        _sut = new OrderService(_repository, _emailService, _timeProvider);
    }

    [Fact]
    public async Task PlaceOrder_WithValidOrder_SavesAndSendsConfirmation()
    {
        // Arrange
        var order = CreateValidOrder();
        _repository.SaveAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>())
            .Returns(Task.CompletedTask);

        // Act
        var result = await _sut.PlaceOrderAsync(order, CancellationToken.None);

        // Assert
        result.IsError.Should().BeFalse();
        await _repository.Received(1).SaveAsync(order, Arg.Any<CancellationToken>());
        await _emailService.Received(1).SendConfirmationAsync(
            order.CustomerEmail,
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task PlaceOrder_WithEmptyCart_ReturnsValidationError()
    {
        var order = CreateOrderWithNoItems();

        var result = await _sut.PlaceOrderAsync(order, CancellationToken.None);

        result.IsError.Should().BeTrue();
        result.FirstError.Type.Should().Be(ErrorType.Validation);
        await _repository.DidNotReceive().SaveAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
    }
}
```

### Best Practices

- **Test behavior, not implementation** — Test what the code does, not how
- **One assertion concept per test** — Multiple `Should()` calls are fine if testing one concept
- **Use meaningful names** — `PlaceOrder_WithExpiredPromo_ReturnsError` not `Test1`
- **Arrange-Act-Assert pattern** — Clear structure
- **No I/O in unit tests** — Mock all external dependencies

---

## 4. Integration Testing

### Purpose
Test that components work correctly together with real dependencies (databases, caches, message queues).

### Setup with Testcontainers

```bash
dotnet add package Testcontainers.PostgreSql
dotnet add package Respawn
```

### Database Test Fixture

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public string ConnectionString => _postgres.GetConnectionString();

    public AppDbContext CreateContext() => new(new DbContextOptionsBuilder<AppDbContext>()
        .UseNpgsql(ConnectionString)
        .Options);

    private Respawner _respawner = null!;

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        await using var context = CreateContext();
        await context.Database.MigrateAsync();

        _respawner = await Respawner.CreateAsync(ConnectionString, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = ["public"]
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await _respawner.ResetAsync(ConnectionString);
    }

    public async Task DisposeAsync()
    {
        await _postgres.DisposeAsync();
    }
}

[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }
```

### Integration Test Example

```csharp
[Collection("Database")]
[Trait("Category", "Integration")]
public class OrderRepositoryTests : IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;
    private AppDbContext _context = null!;
    private OrderRepository _sut = null!;

    public OrderRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    public async Task InitializeAsync()
    {
        await _fixture.ResetDatabaseAsync();
        _context = _fixture.CreateContext();
        _sut = new OrderRepository(_context);
    }

    public Task DisposeAsync()
    {
        return _context.DisposeAsync().AsTask();
    }

    [Fact]
    public async Task GetOrderWithItems_ReturnsFullAggregate()
    {
        // Arrange
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = CustomerId.New(),
            Lines =
            [
                new OrderLine { ProductId = ProductId.New(), Quantity = 2, Price = 10.00m },
                new OrderLine { ProductId = ProductId.New(), Quantity = 1, Price = 25.00m }
            ]
        };
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
        _context.ChangeTracker.Clear();

        // Act
        var result = await _sut.GetByIdAsync(order.Id, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.Lines.Should().HaveCount(2);
        result.Total.Should().Be(45.00m);
    }
}
```

### API Integration Tests

```csharp
public class ApiFixture : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_postgres.GetConnectionString()));
        });
    }

    public new async Task DisposeAsync()
    {
        await _postgres.DisposeAsync();
        await base.DisposeAsync();
    }
}

[Trait("Category", "Integration")]
public class OrdersApiTests : IClassFixture<ApiFixture>
{
    private readonly HttpClient _client;

    public OrdersApiTests(ApiFixture fixture)
    {
        _client = fixture.CreateClient();
    }

    [Fact]
    public async Task CreateOrder_ReturnsCreatedWithLocation()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = Guid.NewGuid(),
            Lines = [new() { ProductId = Guid.NewGuid(), Quantity = 2 }]
        };

        var response = await _client.PostAsJsonAsync("/api/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

---

## 5. Architecture Testing

### Purpose
Enforce architectural constraints as executable tests. Prevent design drift over time.

### Setup

```bash
dotnet add package ArchUnitNET.xUnit
```

### Dependency Rules

```csharp
using static ArchUnitNET.Fluent.ArchRuleDefinition;

[Trait("Category", "Architecture")]
public class ArchitectureTests
{
    private static readonly Architecture Architecture = new ArchLoader()
        .LoadAssemblies(
            typeof(Order).Assembly,           // Domain
            typeof(OrderHandler).Assembly,    // Application
            typeof(OrderRepository).Assembly, // Infrastructure
            typeof(OrdersController).Assembly // Api
        )
        .Build();

    private readonly IObjectProvider<IType> DomainLayer =
        Types().That().ResideInAssembly(typeof(Order).Assembly);

    private readonly IObjectProvider<IType> ApplicationLayer =
        Types().That().ResideInAssembly(typeof(OrderHandler).Assembly);

    private readonly IObjectProvider<IType> InfrastructureLayer =
        Types().That().ResideInAssembly(typeof(OrderRepository).Assembly);

    [Fact]
    public void Domain_ShouldNotDependOn_Application()
    {
        Types().That().Are(DomainLayer)
            .Should().NotDependOnAny(ApplicationLayer)
            .Check(Architecture);
    }

    [Fact]
    public void Domain_ShouldNotDependOn_Infrastructure()
    {
        Types().That().Are(DomainLayer)
            .Should().NotDependOnAny(InfrastructureLayer)
            .Check(Architecture);
    }

    [Fact]
    public void Application_ShouldNotDependOn_Infrastructure()
    {
        Types().That().Are(ApplicationLayer)
            .Should().NotDependOnAny(InfrastructureLayer)
            .Check(Architecture);
    }
}
```

### Naming Conventions

```csharp
[Fact]
public void Handlers_ShouldEndWithHandler()
{
    Classes().That()
        .ImplementInterface(typeof(IRequestHandler<,>))
        .Should().HaveNameEndingWith("Handler")
        .Check(Architecture);
}

[Fact]
public void Repositories_ShouldEndWithRepository()
{
    Classes().That()
        .ImplementInterface(typeof(IRepository<>))
        .Should().HaveNameEndingWith("Repository")
        .Check(Architecture);
}
```

---

## 6. CI Quality Gates

### Essential CI Pipeline

```yaml
# .github/workflows/pr.yml
name: PR Validation

on: [pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --warnaserror

      - name: Unit Tests
        run: dotnet test --filter Category=Unit --no-build --logger trx

      - name: Integration Tests
        run: dotnet test --filter Category=Integration --no-build --logger trx
        env:
          ConnectionStrings__Default: "Host=localhost;Database=test;Username=postgres;Password=test"

      - name: Architecture Tests
        run: dotnet test --filter Category=Architecture --no-build
```

### Required PR Checks

| Check | Purpose |
|-------|---------|
| Build with `--warnaserror` | Type safety enforced |
| Unit tests pass | Logic correctness verified |
| Integration tests pass | Database/API code works |
| Architecture tests pass | Design rules enforced |

---

## 7. Quick Reference

### Recommended Package Stack

| Package | Purpose |
|---------|---------|
| `xunit` | Test framework |
| `FluentAssertions` | Readable assertions |
| `NSubstitute` | Mocking |
| `Testcontainers.PostgreSql` | Real database in tests |
| `Respawn` | Database reset between tests |
| `ArchUnitNET.xUnit` | Architecture tests |

### Test Category Summary

| Category | Speed | When to Use |
|----------|-------|-------------|
| Unit | < 5ms | All business logic |
| Integration | < 5s | DB queries, external calls |
| Architecture | < 1s | Always (in CI) |

### Advanced Testing

For property-based, contract, mutation, and E2E testing, see [Advanced Testing](testing-advanced.md).

---

[← Back to Index](README.md) | [Advanced Testing →](testing-advanced.md)
