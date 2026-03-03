> [← Back to Index](README.md)
>
> **Prerequisites:** [01-core-philosophy](01-core-philosophy.md), [02-language-features](02-language-features.md), [03-project-configuration](03-project-configuration.md), [testing-strategy](testing-strategy.md)

# Robust External System Integration

*A comprehensive guide to integrating with external SOAP and REST/OpenAPI services in .NET*

---

## Table of Contents

1. [Integration Architecture Principles](#1-integration-architecture-principles)
2. [OpenAPI/REST Integration](#2-openapirest-integration)
3. [SOAP Integration](#3-soap-integration)
4. [Contract Validation and Testing](#4-contract-validation-and-testing)
5. [Resilience Patterns](#5-resilience-patterns)
6. [Versioning and Change Management](#6-versioning-and-change-management)
7. [Observability for External Integrations](#7-observability-for-external-integrations)
8. [Testing Strategy Summary](#8-testing-strategy-summary)
9. [External Integration Checklist](#9-external-integration-checklist)

---

## Introduction

Integrating with external APIs (SOAP, REST/OpenAPI) introduces risk. External systems are outside your control—they can change, fail, or behave unexpectedly. This guide covers how to build robust, testable, changeable integrations.

**Key principles:**
- Isolate external dependencies behind abstractions
- Use contract-first client generation
- Validate contracts automatically
- Plan for failure with resilience patterns
- Make changes easy through versioning

---

# 1. Integration Architecture Principles

## Isolate External Dependencies

Never let external API details leak into your domain.

```
┌─────────────────────────────────────────────────────────────┐
│                      Your Application                        │
├─────────────────────────────────────────────────────────────┤
│  Domain Layer          │  No external dependencies          │
│  ├── Entities          │  Pure business logic               │
│  ├── Value Objects     │                                    │
│  └── Domain Services   │                                    │
├─────────────────────────────────────────────────────────────┤
│  Application Layer     │  Orchestration, use cases          │
│  ├── Commands/Queries  │  Depends on abstractions           │
│  └── Handlers          │                                    │
├─────────────────────────────────────────────────────────────┤
│  Infrastructure Layer  │  External system adapters          │
│  ├── PaymentGateway    │  ← SOAP/REST clients live here     │
│  ├── ShippingProvider  │                                    │
│  └── InventorySystem   │                                    │
└─────────────────────────────────────────────────────────────┘
```

## The Anti-Corruption Layer Pattern

Create a boundary that translates between your domain and external systems.

```csharp
// Your domain model - stable, under your control
public record PaymentRequest(
    OrderId OrderId,
    Money Amount,
    PaymentMethod Method);

public record PaymentResult
{
    public sealed record Approved(TransactionId TransactionId, DateTimeOffset ProcessedAt) : PaymentResult;
    public sealed record Declined(string Reason, string? ErrorCode) : PaymentResult;
    public sealed record PendingReview(string ReviewId) : PaymentResult;
    public sealed record Failed(string Error) : PaymentResult;
    
    private PaymentResult() { }
}

// Domain interface - your contract, independent of external API
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken ct);
}
```

---

# 2. OpenAPI/REST Integration

## Contract-First Client Generation

Use OpenAPI specs to generate strongly-typed clients.

**NSwag (recommended for .NET):**

```bash
# Install NSwag CLI
dotnet tool install -g NSwag.ConsoleCore

# Generate client from OpenAPI spec
nswag openapi2csclient \
  /input:https://api.example.com/swagger/v1/swagger.json \
  /output:Generated/ExampleApiClient.cs \
  /namespace:YourApp.Infrastructure.ExternalApis.Example \
  /className:ExampleApiClient \
  /generateClientInterfaces:true \
  /useBaseUrl:false
```

**Or use Kiota (Microsoft's modern generator):**

```bash
dotnet tool install -g Microsoft.OpenApi.Kiota

kiota generate \
  --openapi https://api.example.com/swagger/v1/swagger.json \
  --language CSharp \
  --output ./Generated/ExampleApi \
  --namespace-name YourApp.Infrastructure.ExternalApis.Example
```

## Anti-Corruption Layer Implementation

```csharp
// Generated client interface (from NSwag)
public interface IExampleApiClient
{
    Task<ExternalPaymentResponse> ProcessPaymentAsync(
        ExternalPaymentRequest request, 
        CancellationToken ct);
}

// Your adapter - translates between your domain and external API
public class ExamplePaymentGatewayAdapter : IPaymentGateway
{
    private readonly IExampleApiClient _client;
    private readonly ILogger<ExamplePaymentGatewayAdapter> _logger;
    
    public ExamplePaymentGatewayAdapter(
        IExampleApiClient client,
        ILogger<ExamplePaymentGatewayAdapter> logger)
    {
        _client = client;
        _logger = logger;
    }
    
    public async Task<PaymentResult> ProcessPaymentAsync(
        PaymentRequest request, 
        CancellationToken ct)
    {
        // Translate domain model → external API model
        var externalRequest = MapToExternalRequest(request);
        
        try
        {
            var response = await _client.ProcessPaymentAsync(externalRequest, ct);
            
            // Translate external API model → domain model
            return MapToPaymentResult(response);
        }
        catch (ApiException ex) when (ex.StatusCode == 400)
        {
            _logger.LogWarning(ex, "Payment validation failed for order {OrderId}", request.OrderId);
            return new PaymentResult.Failed($"Validation error: {ex.Message}");
        }
        catch (ApiException ex) when (ex.StatusCode >= 500)
        {
            _logger.LogError(ex, "Payment gateway error for order {OrderId}", request.OrderId);
            throw new ExternalServiceException("Payment gateway unavailable", ex);
        }
    }
    
    private static ExternalPaymentRequest MapToExternalRequest(PaymentRequest request)
    {
        return new ExternalPaymentRequest
        {
            ReferenceId = request.OrderId.Value.ToString(),
            AmountCents = (long)(request.Amount.Amount * 100),
            Currency = request.Amount.Currency.Code,
            PaymentType = request.Method switch
            {
                PaymentMethod.CreditCard => "CARD",
                PaymentMethod.BankTransfer => "BANK",
                PaymentMethod.Wallet => "WALLET",
                _ => throw new ArgumentOutOfRangeException(nameof(request.Method))
            }
        };
    }
    
    private static PaymentResult MapToPaymentResult(ExternalPaymentResponse response)
    {
        return response.Status switch
        {
            "APPROVED" => new PaymentResult.Approved(
                new TransactionId(response.TransactionId!),
                DateTimeOffset.Parse(response.ProcessedAt!)),
            "DECLINED" => new PaymentResult.Declined(
                response.DeclineReason ?? "Unknown",
                response.ErrorCode),
            "PENDING_REVIEW" => new PaymentResult.PendingReview(response.ReviewId!),
            "ERROR" => new PaymentResult.Failed(response.ErrorMessage ?? "Unknown error"),
            _ => new PaymentResult.Failed($"Unknown status: {response.Status}")
        };
    }
}
```

## Save OpenAPI Specs Locally

Don't rely on remote specs being available during build.

```
/src
  /Infrastructure
    /ExternalApis
      /PaymentGateway
        /Contracts
          payment-api-v1.json       ← Checked into source control
          payment-api-v2.json
        /Generated
          PaymentApiClient.cs       ← Generated, .gitignore'd or committed
        PaymentGatewayAdapter.cs
```

**MSBuild integration for regeneration:**

```xml
<!-- In .csproj -->
<Target Name="GenerateApiClients" BeforeTargets="BeforeBuild">
  <Exec Command="nswag openapi2csclient /input:Contracts/payment-api-v1.json /output:Generated/PaymentApiClient.cs /namespace:YourApp.Infrastructure.ExternalApis.PaymentGateway" />
</Target>
```

---

# 3. SOAP Integration

SOAP APIs require different tooling but the same principles apply.

## Generate WCF Client

```bash
# Using dotnet-svcutil
dotnet tool install -g dotnet-svcutil

dotnet-svcutil https://example.com/service.wsdl \
  --outputDir Generated \
  --namespace "*,YourApp.Infrastructure.ExternalApis.LegacySystem"
```

**Or reference the WSDL directly:**

```xml
<!-- In .csproj -->
<ItemGroup>
  <PackageReference Include="System.ServiceModel.Http" Version="6.*" />
  <PackageReference Include="System.ServiceModel.Primitives" Version="6.*" />
</ItemGroup>
```

## SOAP Anti-Corruption Layer

```csharp
// Generated SOAP client
public interface ILegacyInventoryServiceClient
{
    Task<CheckInventoryResponse> CheckInventoryAsync(CheckInventoryRequest request);
}

// Your domain interface
public interface IInventoryService
{
    Task<InventoryStatus> CheckAvailabilityAsync(
        ProductId productId, 
        Quantity quantity, 
        CancellationToken ct);
}

public record InventoryStatus(
    bool IsAvailable,
    int AvailableQuantity,
    DateTimeOffset? ExpectedRestockDate);

// Adapter
public class LegacyInventoryAdapter : IInventoryService
{
    private readonly ILegacyInventoryServiceClient _soapClient;
    private readonly ILogger<LegacyInventoryAdapter> _logger;
    
    public async Task<InventoryStatus> CheckAvailabilityAsync(
        ProductId productId,
        Quantity quantity,
        CancellationToken ct)
    {
        var request = new CheckInventoryRequest
        {
            SKU = productId.Value,
            RequestedQty = quantity.Value,
            RequestTimestamp = DateTime.UtcNow
        };
        
        try
        {
            var response = await _soapClient.CheckInventoryAsync(request);
            
            return new InventoryStatus(
                IsAvailable: response.AvailableFlag == "Y",
                AvailableQuantity: response.OnHandQty,
                ExpectedRestockDate: ParseNullableDate(response.NextDeliveryDate));
        }
        catch (CommunicationException ex)
        {
            _logger.LogError(ex, "SOAP communication error checking inventory for {ProductId}", productId);
            throw new ExternalServiceException("Inventory service unavailable", ex);
        }
    }
    
    private static DateTimeOffset? ParseNullableDate(string? dateString)
    {
        if (string.IsNullOrEmpty(dateString)) return null;
        return DateTimeOffset.TryParse(dateString, out var date) ? date : null;
    }
}
```

---

# 4. Contract Validation and Testing

External APIs can change without notice. Test your assumptions.

## Contract Tests with Saved Responses

Save real API responses and test your mapping logic against them.

```csharp
// Test data structure
/tests
  /Integration
    /PaymentGateway
      /Fixtures
        approved-response.json
        declined-response.json
        pending-review-response.json
        error-response-validation.json
        error-response-server.json
```

```csharp
public class PaymentGatewayContractTests
{
    private readonly ExamplePaymentGatewayAdapter _adapter;
    
    [Theory]
    [InlineData("approved-response.json", typeof(PaymentResult.Approved))]
    [InlineData("declined-response.json", typeof(PaymentResult.Declined))]
    [InlineData("pending-review-response.json", typeof(PaymentResult.PendingReview))]
    public async Task Should_Map_Response_To_Expected_Result_Type(
        string fixtureFile, 
        Type expectedType)
    {
        // Arrange
        var responseJson = await File.ReadAllTextAsync($"Fixtures/{fixtureFile}");
        var mockClient = CreateMockClientReturning(responseJson);
        var adapter = new ExamplePaymentGatewayAdapter(mockClient, NullLogger.Instance);
        
        // Act
        var result = await adapter.ProcessPaymentAsync(CreateTestRequest(), CancellationToken.None);
        
        // Assert
        result.Should().BeOfType(expectedType);
    }
    
    [Fact]
    public async Task Approved_Response_Should_Have_TransactionId()
    {
        // Arrange
        var responseJson = await File.ReadAllTextAsync("Fixtures/approved-response.json");
        var mockClient = CreateMockClientReturning(responseJson);
        var adapter = new ExamplePaymentGatewayAdapter(mockClient, NullLogger.Instance);
        
        // Act
        var result = await adapter.ProcessPaymentAsync(CreateTestRequest(), CancellationToken.None);
        
        // Assert
        var approved = result.Should().BeOfType<PaymentResult.Approved>().Subject;
        approved.TransactionId.Should().NotBe(default);
        approved.ProcessedAt.Should().BeAfter(DateTimeOffset.MinValue);
    }
}
```

## Schema Validation Tests

Validate that the external API response matches your expected schema.

```csharp
public class PaymentApiSchemaTests
{
    [Fact]
    public async Task Response_Schema_Should_Match_Expected_Contract()
    {
        // Load saved OpenAPI spec
        var spec = await File.ReadAllTextAsync("Contracts/payment-api-v1.json");
        var openApiDocument = new OpenApiStringReader().Read(spec, out var diagnostic);
        
        // Load actual response from fixture
        var responseJson = await File.ReadAllTextAsync("Fixtures/approved-response.json");
        var response = JsonDocument.Parse(responseJson);
        
        // Validate against schema
        var schema = openApiDocument.Paths["/payments"]
            .Operations[OperationType.Post]
            .Responses["200"]
            .Content["application/json"]
            .Schema;
            
        var validator = new JsonSchemaValidator();
        var result = validator.Validate(response, schema);
        
        result.IsValid.Should().BeTrue(
            because: $"Response should match schema. Errors: {string.Join(", ", result.Errors)}");
    }
}
```

## Live Contract Verification Tests

Periodically verify that the external API still matches your expectations.

```csharp
[Trait("Category", "ExternalContract")]
public class PaymentApiLiveContractTests
{
    private readonly IExampleApiClient _client;
    
    public PaymentApiLiveContractTests()
    {
        // Real HTTP client pointing to sandbox/test environment
        _client = CreateRealClient(Environment.GetEnvironmentVariable("PAYMENT_API_SANDBOX_URL"));
    }
    
    [Fact]
    public async Task Should_Return_Expected_Response_Structure()
    {
        // Arrange - use test credentials
        var request = new ExternalPaymentRequest
        {
            ReferenceId = $"test-{Guid.NewGuid()}",
            AmountCents = 1000,
            Currency = "USD",
            PaymentType = "CARD",
            // Test card that always approves
            CardToken = "tok_test_visa_approve"
        };
        
        // Act
        var response = await _client.ProcessPaymentAsync(request, CancellationToken.None);
        
        // Assert - verify structure, not specific values
        response.Status.Should().BeOneOf("APPROVED", "DECLINED", "PENDING_REVIEW", "ERROR");
        response.TransactionId.Should().NotBeNullOrEmpty();
        response.ProcessedAt.Should().NotBeNullOrEmpty();
    }
    
    [Fact]
    public async Task Should_Return_Decline_For_Test_Decline_Card()
    {
        var request = new ExternalPaymentRequest
        {
            ReferenceId = $"test-{Guid.NewGuid()}",
            AmountCents = 1000,
            Currency = "USD",
            PaymentType = "CARD",
            CardToken = "tok_test_visa_decline"
        };
        
        var response = await _client.ProcessPaymentAsync(request, CancellationToken.None);
        
        response.Status.Should().Be("DECLINED");
        response.DeclineReason.Should().NotBeNullOrEmpty();
    }
}
```

**Run contract tests on a schedule:**

```yaml
# .github/workflows/contract-tests.yml
name: External Contract Tests

on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM
  workflow_dispatch:      # Manual trigger

jobs:
  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
      - run: dotnet test --filter Category=ExternalContract
        env:
          PAYMENT_API_SANDBOX_URL: ${{ secrets.PAYMENT_API_SANDBOX_URL }}
```

---

# 5. Resilience Patterns

External systems fail. Plan for it.

## Configure Resilience with Polly

```csharp
// Install packages
// dotnet add package Microsoft.Extensions.Http.Resilience
// dotnet add package Polly

public static class HttpClientConfiguration
{
    public static IServiceCollection AddPaymentGatewayClient(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddHttpClient<IExampleApiClient, ExampleApiClient>(client =>
        {
            client.BaseAddress = new Uri(configuration["PaymentGateway:BaseUrl"]!);
            client.Timeout = TimeSpan.FromSeconds(30);
        })
        .AddStandardResilienceHandler()  // .NET 8 built-in resilience
        .Configure(options =>
        {
            // Retry configuration
            options.Retry.MaxRetryAttempts = 3;
            options.Retry.Delay = TimeSpan.FromMilliseconds(500);
            options.Retry.UseJitter = true;
            options.Retry.ShouldHandle = args => ValueTask.FromResult(
                args.Outcome.Result?.StatusCode >= HttpStatusCode.InternalServerError ||
                args.Outcome.Exception is HttpRequestException);
            
            // Circuit breaker
            options.CircuitBreaker.FailureRatio = 0.5;
            options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
            options.CircuitBreaker.MinimumThroughput = 10;
            options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);
            
            // Timeout
            options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(10);
            options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(60);
        });
        
        return services;
    }
}
```

## Custom Resilience with Polly v8

```csharp
public static class ResiliencePolicies
{
    public static ResiliencePipeline<HttpResponseMessage> CreatePaymentGatewayPipeline()
    {
        return new ResiliencePipelineBuilder<HttpResponseMessage>()
            // Rate limiter
            .AddRateLimiter(new SlidingWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6
            })
            // Circuit breaker
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                FailureRatio = 0.5,
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 10,
                BreakDuration = TimeSpan.FromSeconds(30),
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .HandleResult(r => r.StatusCode >= HttpStatusCode.InternalServerError)
                    .Handle<HttpRequestException>()
            })
            // Retry
            .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromMilliseconds(500),
                UseJitter = true,
                BackoffType = DelayBackoffType.Exponential,
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .HandleResult(r => r.StatusCode >= HttpStatusCode.InternalServerError)
                    .Handle<HttpRequestException>()
            })
            // Timeout per attempt
            .AddTimeout(TimeSpan.FromSeconds(10))
            .Build();
    }
}
```

## Graceful Degradation

```csharp
public class ResilientPaymentGateway : IPaymentGateway
{
    private readonly IPaymentGateway _primaryGateway;
    private readonly IPaymentGateway _fallbackGateway;
    private readonly ILogger<ResilientPaymentGateway> _logger;
    
    public async Task<PaymentResult> ProcessPaymentAsync(
        PaymentRequest request, 
        CancellationToken ct)
    {
        try
        {
            return await _primaryGateway.ProcessPaymentAsync(request, ct);
        }
        catch (ExternalServiceException ex)
        {
            _logger.LogWarning(ex, 
                "Primary gateway failed for order {OrderId}, trying fallback", 
                request.OrderId);
            
            try
            {
                return await _fallbackGateway.ProcessPaymentAsync(request, ct);
            }
            catch (ExternalServiceException fallbackEx)
            {
                _logger.LogError(fallbackEx, 
                    "Both gateways failed for order {OrderId}", 
                    request.OrderId);
                
                // Return a domain result, don't throw
                return new PaymentResult.Failed(
                    "Payment processing temporarily unavailable. Please try again later.");
            }
        }
    }
}
```

---

# 6. Versioning and Change Management

External APIs evolve. Design for change.

## Version Your Adapters

```
/src
  /Infrastructure
    /ExternalApis
      /PaymentGateway
        /V1
          PaymentGatewayAdapterV1.cs
          /Generated
            PaymentApiClientV1.cs
          /Contracts
            payment-api-v1.json
        /V2
          PaymentGatewayAdapterV2.cs
          /Generated
            PaymentApiClientV2.cs
          /Contracts
            payment-api-v2.json
        IPaymentGatewayFactory.cs
```

## Feature Flag for Migration

```csharp
public interface IPaymentGatewayFactory
{
    IPaymentGateway Create();
}

public class PaymentGatewayFactory : IPaymentGatewayFactory
{
    private readonly IPaymentGateway _v1;
    private readonly IPaymentGateway _v2;
    private readonly IFeatureManager _featureManager;
    
    public IPaymentGateway Create()
    {
        return _featureManager.IsEnabled("PaymentGatewayV2")
            ? _v2
            : _v1;
    }
}
```

## Contract Change Detection

```csharp
public class ContractChangeDetector
{
    public static async Task<ContractDiff> CompareContracts(
        string previousSpecPath, 
        string currentSpecPath)
    {
        var previous = await LoadSpec(previousSpecPath);
        var current = await LoadSpec(currentSpecPath);
        
        var diff = new ContractDiff();
        
        // Check for removed endpoints
        foreach (var path in previous.Paths.Keys)
        {
            if (!current.Paths.ContainsKey(path))
            {
                diff.RemovedEndpoints.Add(path);
            }
        }
        
        // Check for changed response schemas
        foreach (var (path, operations) in current.Paths)
        {
            if (previous.Paths.TryGetValue(path, out var previousOps))
            {
                foreach (var (method, operation) in operations.Operations)
                {
                    if (previousOps.Operations.TryGetValue(method, out var prevOp))
                    {
                        var schemaChanges = CompareSchemas(
                            prevOp.Responses, 
                            operation.Responses);
                        if (schemaChanges.Any())
                        {
                            diff.ChangedSchemas.Add($"{method} {path}", schemaChanges);
                        }
                    }
                }
            }
        }
        
        return diff;
    }
}

// Run as part of CI when specs are updated
[Fact]
public async Task Contract_Changes_Should_Be_Backward_Compatible()
{
    var diff = await ContractChangeDetector.CompareContracts(
        "Contracts/payment-api-v1.json",
        "Contracts/payment-api-v1-updated.json");
    
    diff.RemovedEndpoints.Should().BeEmpty(
        "Removing endpoints is a breaking change");
    diff.RemovedRequiredFields.Should().BeEmpty(
        "Removing required response fields is a breaking change");
}
```

---

# 7. Observability for External Integrations

## Structured Logging

```csharp
public class LoggingPaymentGateway : IPaymentGateway
{
    private readonly IPaymentGateway _inner;
    private readonly ILogger<LoggingPaymentGateway> _logger;
    
    public async Task<PaymentResult> ProcessPaymentAsync(
        PaymentRequest request, 
        CancellationToken ct)
    {
        using var activity = Activity.Current?.Source.StartActivity("PaymentGateway.ProcessPayment");
        activity?.SetTag("order.id", request.OrderId.Value);
        activity?.SetTag("payment.amount", request.Amount.Amount);
        activity?.SetTag("payment.currency", request.Amount.Currency.Code);
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var result = await _inner.ProcessPaymentAsync(request, ct);
            
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Payment processed for Order {OrderId}: {ResultType} in {ElapsedMs}ms",
                request.OrderId,
                result.GetType().Name,
                stopwatch.ElapsedMilliseconds);
            
            activity?.SetTag("payment.result", result.GetType().Name);
            activity?.SetStatus(ActivityStatusCode.Ok);
            
            return result;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            
            _logger.LogError(ex,
                "Payment failed for Order {OrderId} after {ElapsedMs}ms",
                request.OrderId,
                stopwatch.ElapsedMilliseconds);
            
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            
            throw;
        }
    }
}
```

## Metrics

```csharp
public class MetricsPaymentGateway : IPaymentGateway
{
    private readonly IPaymentGateway _inner;
    private readonly Counter<long> _requestCounter;
    private readonly Histogram<double> _requestDuration;
    
    public MetricsPaymentGateway(
        IPaymentGateway inner,
        IMeterFactory meterFactory)
    {
        _inner = inner;
        
        var meter = meterFactory.Create("YourApp.PaymentGateway");
        _requestCounter = meter.CreateCounter<long>(
            "payment_gateway_requests_total",
            description: "Total payment gateway requests");
        _requestDuration = meter.CreateHistogram<double>(
            "payment_gateway_request_duration_seconds",
            description: "Payment gateway request duration");
    }
    
    public async Task<PaymentResult> ProcessPaymentAsync(
        PaymentRequest request, 
        CancellationToken ct)
    {
        var tags = new TagList
        {
            { "gateway", "example" },
            { "currency", request.Amount.Currency.Code }
        };
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var result = await _inner.ProcessPaymentAsync(request, ct);
            
            tags.Add("result", result switch
            {
                PaymentResult.Approved => "approved",
                PaymentResult.Declined => "declined",
                PaymentResult.PendingReview => "pending",
                PaymentResult.Failed => "failed",
                _ => "unknown"
            });
            
            _requestCounter.Add(1, tags);
            _requestDuration.Record(stopwatch.Elapsed.TotalSeconds, tags);
            
            return result;
        }
        catch (Exception)
        {
            tags.Add("result", "error");
            _requestCounter.Add(1, tags);
            _requestDuration.Record(stopwatch.Elapsed.TotalSeconds, tags);
            throw;
        }
    }
}
```

---

# 8. Testing Strategy Summary

| Test Type | Purpose | Frequency | Environment |
|-----------|---------|-----------|-------------|
| **Unit Tests** | Test adapter mapping logic | Every build | Local, mocked |
| **Contract Tests** | Verify response parsing | Every build | Local, fixtures |
| **Schema Validation** | Detect schema drift | Every build | Local, saved specs |
| **Integration Tests** | Test HTTP client config | Every build | Local, WireMock |
| **Live Contract Tests** | Verify external API | Daily/Weekly | Sandbox/Test env |
| **Load Tests** | Verify resilience | Pre-release | Sandbox/Test env |

## WireMock for Integration Tests

```csharp
public class PaymentGatewayIntegrationTests : IAsyncLifetime
{
    private WireMockServer _server = null!;
    private IPaymentGateway _gateway = null!;
    
    public Task InitializeAsync()
    {
        _server = WireMockServer.Start();
        
        var client = new ExampleApiClient(new HttpClient
        {
            BaseAddress = new Uri(_server.Urls[0])
        });
        
        _gateway = new ExamplePaymentGatewayAdapter(client, NullLogger.Instance);
        
        return Task.CompletedTask;
    }
    
    [Fact]
    public async Task Should_Handle_Timeout_Gracefully()
    {
        _server.Given(Request.Create().WithPath("/payments").UsingPost())
            .RespondWith(Response.Create()
                .WithDelay(TimeSpan.FromSeconds(30))); // Longer than timeout
        
        var act = () => _gateway.ProcessPaymentAsync(CreateTestRequest(), CancellationToken.None);
        
        await act.Should().ThrowAsync<TimeoutException>();
    }
    
    [Fact]
    public async Task Should_Handle_ServerError_With_Retry()
    {
        _server.Given(Request.Create().WithPath("/payments").UsingPost())
            .InScenario("RetryScenario")
            .WillSetStateTo("FirstFail")
            .RespondWith(Response.Create().WithStatusCode(500));
        
        _server.Given(Request.Create().WithPath("/payments").UsingPost())
            .InScenario("RetryScenario")
            .WhenStateIs("FirstFail")
            .WillSetStateTo("Success")
            .RespondWith(Response.Create()
                .WithStatusCode(200)
                .WithBody(LoadFixture("approved-response.json")));
        
        var result = await _gateway.ProcessPaymentAsync(CreateTestRequest(), CancellationToken.None);
        
        result.Should().BeOfType<PaymentResult.Approved>();
    }
    
    public Task DisposeAsync()
    {
        _server.Stop();
        return Task.CompletedTask;
    }
}
```

---

# 9. External Integration Checklist

Use this checklist when integrating with a new external system:

## Initial Setup
- [ ] Save OpenAPI/WSDL spec to source control
- [ ] Generate strongly-typed client
- [ ] Create domain interface (anti-corruption layer)
- [ ] Implement adapter with mapping logic
- [ ] Add unit tests for mapping

## Contract Validation
- [ ] Save example responses as fixtures
- [ ] Add contract tests for all response types
- [ ] Add schema validation tests
- [ ] Set up live contract verification (sandbox)
- [ ] Schedule daily contract test runs

## Resilience
- [ ] Configure retry policy
- [ ] Configure circuit breaker
- [ ] Configure timeouts (per-attempt and total)
- [ ] Add fallback strategy if applicable
- [ ] Test failure scenarios with WireMock

## Observability
- [ ] Add structured logging
- [ ] Add metrics (request count, duration, errors)
- [ ] Add distributed tracing
- [ ] Create dashboard/alerts

## Documentation
- [ ] Document API credentials management
- [ ] Document rate limits
- [ ] Document error handling behavior
- [ ] Document versioning strategy

---

## Related Reading

- [Main C# Safety & Maturity Playbook](README.md)
- [Microsoft: Make HTTP requests with resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)
- [Polly Documentation](https://www.pollydocs.org/)
- [NSwag Documentation](https://github.com/RicoSuter/NSwag)

---

*Last updated: March 2026 | Target: .NET 8+ / C# 12+*

---

[← Back to Index](README.md)
