# Production-Ready Health Checks for .NET

Health checks are not just "is the service alive?" — they are **early warning systems** that catch problems before users do, reduce debugging time, and prevent 3am incidents.

## Why Health Checks Save Development Time

| Benefit | How |
|---------|-----|
| **Faster bug detection** | Issues surface in health endpoints before users report them |
| **Reduced debugging time** | Know immediately which dependency failed |
| **Fewer production incidents** | Circuit breakers prevent cascade failures |
| **Clearer incident response** | Health state tells you exactly what's wrong |
| **Less firefighting** | Proactive monitoring instead of reactive debugging |

**Real impact:** A properly designed health check system turns a 2-hour "why is the app slow?" investigation into a 30-second dashboard glance.

---

## 1. Health Check Fundamentals

### 1.1 Naming Conventions

Health check names should be:

- **Stable** — don't change between environments
- **Role-based** — represent logical dependency, not technology
- **Machine-parseable** — usable in logs, alerts, dashboards

**Naming Pattern:**

```
<dependency-type>-<logical-role>
```

| Dependency | Good Name |
|------------|-----------|
| Main DB | `database-primary` |
| Cache layer | `cache-primary` |
| External system | `core-system` |

> **Time saver:** Consistent naming means your monitoring dashboards and alerts work across all services without per-service configuration.

### 1.2 Health Check Types (Tags)

Tags control endpoint behavior and are **orchestration signals**:

| Type | Purpose | Should Include |
|------|---------|----------------|
| **Liveness** (`live`) | Is process alive? | Nothing — process only |
| **Readiness** (`ready`) | Can app serve traffic? | SQL, Redis, required APIs |
| **Startup** (`startup`) | Has initialization completed? | Migrations, schema validation |

**Critical Rule:** Never tag SQL/Redis as `live` — DB outages should not trigger container restarts.

### 1.3 Basic Setup

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: configuration.GetConnectionString("DefaultConnection")!,
        name: "database-primary",
        timeout: TimeSpan.FromSeconds(5),
        tags: ["db", "sql", "ready"]
    );

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // process only
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = r => r.Tags.Contains("ready")
});
```

---

## 2. Azure App Service Strategy

Azure App Service health checks differ from Kubernetes:

- Only pings **one endpoint** (e.g., `/health`)
- Returns 200–299 = healthy, otherwise instance removed from load balancer
- Does **not** understand readiness vs liveness

### 2.1 Three-Endpoint Architecture

| Endpoint | Purpose | Exposed? |
|----------|---------|----------|
| `/health/live` | Process alive | No |
| `/health/ready` | Dependencies ready | No |
| `/health` | Azure App Service | Yes (restricted) |

### 2.2 Azure-Safe Health Endpoint

**Critical:** Do NOT map `/health` directly to full readiness.

If SQL has a 30-second transient issue → all instances return 500 → Azure removes all → **self-inflicted outage**.

**Solution:** Use `Degraded` status for transient failures:

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString,
        name: "database-primary",
        timeout: TimeSpan.FromSeconds(2),
        failureStatus: HealthStatus.Degraded,  // Won't remove instance
        tags: ["appservice"]
    );

app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = r => r.Tags.Contains("appservice"),
    ResponseWriter = (_, __) => Task.CompletedTask
});
```

> **Bug prevention:** This pattern prevents the #1 cause of self-inflicted outages — cascading instance removal during transient dependency issues.

### 2.3 Performance Rules

Health endpoints must:

- Complete in < 2 seconds
- Never run migrations or heavy queries
- Use `SELECT 1` only

---

## 3. External System Health Checks (SOAP/OData)

For mandatory external dependencies, design carefully to avoid cascade failures.

```csharp
public class ExternalSystemConnectivityHealthCheck : IHealthCheck
{
    private readonly HttpClient _client;

    public ExternalSystemConnectivityHealthCheck(HttpClient client)
        => _client = client;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var response = await _client.GetAsync("/odata/$metadata", cancellationToken);
            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy()
                : HealthCheckResult.Degraded("External system returned non-success.");
        }
        catch
        {
            return HealthCheckResult.Degraded("External system unreachable.");
        }
    }
}
```

> **Faster debugging:** When external system is down, health endpoint immediately tells you — no need to dig through logs or reproduce the issue.

---

## 4. Cached Health State (Background Service)

**Problem:** Running health checks on every request creates dependency pressure and can cause thundering herd issues.

**Solution:** Background service runs checks periodically, `/health` returns cached result instantly.

### 4.1 Implementation

```csharp
public class CachedHealthState
{
    public HealthReport? Report { get; private set; }
    public DateTimeOffset LastUpdated { get; private set; }

    public void Set(HealthReport report)
    {
        Report = report;
        LastUpdated = DateTimeOffset.UtcNow;
    }
}

public class HealthCheckBackgroundService : BackgroundService
{
    private readonly HealthCheckService _healthCheckService;
    private readonly CachedHealthState _cachedState;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var report = await _healthCheckService.CheckHealthAsync(
                predicate: r => r.Tags.Contains("cached"),
                cancellationToken: stoppingToken);

            _cachedState.Set(report);
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}
```

### 4.2 Cached Endpoint

```csharp
app.MapGet("/health", (CachedHealthState cachedState) =>
{
    var report = cachedState.Report;
    if (report == null) return Results.StatusCode(503);

    return report.Status switch
    {
        HealthStatus.Healthy => Results.Ok(),
        HealthStatus.Degraded => Results.Ok(),
        HealthStatus.Unhealthy => Results.StatusCode(503),
        _ => Results.Ok()
    };
});
```

> **Cost reduction:** Prevents health check storms that can overwhelm dependencies during high load.

---

## 5. Circuit Breaker Integration

**Goal:** Health should reflect runtime protection state, not make additional probing calls.

### 5.1 Circuit State Holder

```csharp
public class SqlCircuitState
{
    private readonly object _lock = new();
    private CircuitState _state = CircuitState.Closed;
    private DateTimeOffset _lastSuccessUtc = DateTimeOffset.UtcNow;

    public CircuitState State
    {
        get { lock (_lock) return _state; }
        set { lock (_lock) _state = value; }
    }

    public DateTimeOffset LastSuccessUtc
    {
        get { lock (_lock) return _lastSuccessUtc; }
        set { lock (_lock) _lastSuccessUtc = value; }
    }
}
```

### 5.2 State-Based Health Check

```csharp
public class SqlCircuitHealthCheck : IHealthCheck
{
    private readonly SqlCircuitState _state;

    public SqlCircuitHealthCheck(SqlCircuitState state) => _state = state;

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var timeSinceSuccess = DateTimeOffset.UtcNow - _state.LastSuccessUtc;

        return _state.State switch
        {
            CircuitState.Closed =>
                Task.FromResult(HealthCheckResult.Healthy()),
            
            CircuitState.HalfOpen =>
                Task.FromResult(HealthCheckResult.Degraded("Circuit half-open")),
            
            CircuitState.Open when timeSinceSuccess < TimeSpan.FromMinutes(5) =>
                Task.FromResult(HealthCheckResult.Degraded("Circuit open")),
            
            CircuitState.Open =>
                Task.FromResult(HealthCheckResult.Unhealthy("Unavailable")),
            
            _ => Task.FromResult(HealthCheckResult.Unhealthy())
        };
    }
}
```

> **Incident prevention:** Circuit breaker stops cascading failures. Health check exposes this state so you know protection is active.

---

## 6. Resilience Pipeline

### 6.1 Policy Wrap Order

```csharp
var pipeline = Policy.WrapAsync(retry, circuit, timeout);
```

Execution order is **outside → inside**:

```
Retry → CircuitBreaker → Timeout → SQL Operation
```

| Policy | Why |
|--------|-----|
| **Timeout** (innermost) | Each attempt individually bounded |
| **Circuit** (middle) | Counts each failed attempt |
| **Retry** (outermost) | Absorbs transient noise |

### 6.2 Complete Configuration

```csharp
var retry = Policy
    .Handle<SqlException>()
    .Or<TimeoutRejectedException>()
    .WaitAndRetryAsync(3, attempt =>
        TimeSpan.FromMilliseconds(200 * attempt + Random.Shared.Next(0, 100)));

var circuit = Policy
    .Handle<SqlException>()
    .Or<TimeoutRejectedException>()
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (_, _) => state.State = CircuitState.Open,
        onReset: () =>
        {
            state.State = CircuitState.Closed;
            state.LastSuccessUtc = DateTimeOffset.UtcNow;
        },
        onHalfOpen: () => state.State = CircuitState.HalfOpen);

var timeout = Policy.TimeoutAsync(TimeSpan.FromSeconds(5));

var pipeline = Policy.WrapAsync(retry, circuit, timeout);
```

> **Less debugging:** When SQL is down, circuit opens after 5 failures. Health immediately shows "circuit open" — no log diving required.

---

## 7. Exception Handling

Users should never see infrastructure exceptions.

```csharp
public class ResilienceExceptionMiddleware
{
    private readonly RequestDelegate _next;

    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (BrokenCircuitException)
        {
            context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Service temporarily unavailable",
                code = "DEPENDENCY_UNAVAILABLE"
            });
        }
    }
}
```

**Use 503** — signals retryable condition.

---

## 8. .NET 8 Resilience Pipelines

Replace Polly with native `Microsoft.Extensions.Resilience` for cleaner code:

```csharp
builder.Services.AddResiliencePipeline("sql-pipeline", builder =>
{
    builder
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(200),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 5,
            BreakDuration = TimeSpan.FromSeconds(30),
            OnOpened = args =>
            {
                var state = args.Context.ServiceProvider
                    .GetRequiredService<SqlCircuitState>();
                state.State = CircuitState.Open;
                return default;
            },
            OnClosed = args =>
            {
                var state = args.Context.ServiceProvider
                    .GetRequiredService<SqlCircuitState>();
                state.State = CircuitState.Closed;
                return default;
            }
        })
        .AddTimeout(TimeSpan.FromSeconds(5));
});
```

---

## 9. Anti-Patterns

| Anti-Pattern | Impact |
|--------------|--------|
| Mapping `/health` to full readiness | Self-inflicted cascading outage |
| Long SQL timeout (30s) | Thread pool exhaustion, slow failures |
| No timeout specified | Requests hang indefinitely |
| DB check in liveness | Unnecessary restarts on DB blip |
| Exposing `/health` publicly | Security risk |

---

## 10. Quick Wins Checklist

- [ ] Add `timeout: TimeSpan.FromSeconds(5)` to all health checks
- [ ] Use `HealthStatus.Degraded` for non-critical dependencies
- [ ] Separate `/health` (Azure) from `/health/ready` (monitoring)
- [ ] Add circuit breaker state to health response
- [ ] Cache health state in background service
- [ ] Return 503 with error code (not stack traces) on failures

---

## Summary

| Component | Responsibility |
|-----------|----------------|
| **Retry** | Absorb transient noise |
| **Timeout** | Bound latency |
| **Circuit Breaker** | Prevent cascade failures |
| **Health Check** | Expose system state for fast debugging |
| **Background Cache** | Stabilize under load |

**Bottom line:** Properly designed health checks turn "why is production broken?" into a dashboard glance instead of a 2-hour investigation.
