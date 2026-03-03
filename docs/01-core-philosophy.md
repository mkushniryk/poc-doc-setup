> [← Back to Index](README.md)

# 1. Core Safety Philosophy for C#

Modern C# provides powerful tools for compile-time safety. The key is using them intentionally.

---

## 1.1 First Principles in C#

### 1. Illegal States Must Be Unrepresentable

Use C#'s type system to make invalid states impossible to construct.

**Weak modeling:**

```csharp
public class Appointment
{
    public string Status { get; set; } // "scheduled", "cancelled", etc.
    public Guid PatientId { get; set; }
    public Guid DoctorId { get; set; }
}
```

Problems:
- `Status` allows typos (`"Schedled"`)
- `PatientId` and `DoctorId` are both `Guid` — easy to swap by accident
- Everything is mutable

**Strong modeling with C# features:**

```csharp
// Strongly-typed IDs prevent mixing up different ID types
public readonly record struct PatientId(Guid Value);
public readonly record struct DoctorId(Guid Value);
public readonly record struct AppointmentId(Guid Value);

// Enum prevents invalid status values
public enum AppointmentStatus
{
    Scheduled,
    Confirmed,
    Cancelled,
    Completed
}

// Immutable record with required properties
public record Appointment
{
    public required AppointmentId Id { get; init; }
    public required PatientId PatientId { get; init; }
    public required DoctorId DoctorId { get; init; }
    public required AppointmentStatus Status { get; init; }
    public required DateTimeOffset ScheduledAt { get; init; }
}
```

**C# features used:**
- `readonly record struct` — value-type wrapper with equality
- `record` — immutable by convention, value-based equality
- `required` — compiler enforces initialization
- `init` — can only be set during construction
- `enum` — compiler-checked finite set of values

> **Goal:** The compiler catches bugs before runtime.
>
> **See also:** [Strongly-Typed IDs with Vogen](02-language-features.md#211-strongly-typed-ids-with-vogen) for production implementation, [Economics](05-economics.md#strongly-typed-ids) for ROI analysis.

> **When to skip typed IDs:** Internal tools with single-developer ownership, scripts or one-off utilities, or when all ID parameters are already different types (e.g., `userId: string`, `orderId: int`). If you're unsure, default to typed IDs—the cost is low.

---

### 2. Failure Must Be Explicit

Use Result types instead of exceptions for expected failures.

**Implicit failure (anti-pattern):**

```csharp
public Patient GetPatient(Guid id)
{
    var patient = _dbContext.Patients.Find(id);
    if (patient == null)
        throw new NotFoundException($"Patient {id} not found");
    return patient;
}
```

Problems:
- Caller doesn't know this can fail from the signature
- Exception handling is expensive and unpredictable
- Business errors become infrastructure errors

**Explicit failure with Result pattern:**

```csharp
// Use a library like ErrorOr, OneOf, or FluentResults
public Result<Patient> GetPatient(PatientId id)
{
    var patient = _dbContext.Patients.Find(id.Value);
    return patient is not null
        ? new Result<Patient>.Success(patient)
        : new Result<Patient>.Failure(new Error("NotFound", $"Patient {id.Value} not found"));
}

// Caller is forced to handle both outcomes
public IActionResult GetPatient(Guid id)
{
    return _repository.GetPatient(new PatientId(id)).Match(
        onSuccess: patient => Ok(patient),
        onFailure: error => error.Code switch
        {
            "NotFound" => NotFound(error.Message),
            _ => BadRequest(error.Message)
        }
    );
}
```

> For full Result type implementations, see [Section 2.1: Discriminated Unions](02-language-features.md#21-discriminated-unions-eliminating-invalid-states).

**Recommended libraries:**
- [`ErrorOr`](https://github.com/amantinband/error-or) — lightweight, discriminated union style
- [`OneOf`](https://github.com/mcintyre321/OneOf) — general discriminated unions
- [`FluentResults`](https://github.com/altmann/FluentResults) — rich result pattern
- [`LanguageExt`](https://github.com/louthy/language-ext) — full functional programming

> **When to skip Result types:** Infrastructure code where exceptions are appropriate (e.g., database connection failures), third-party library boundaries that expect exceptions, or simple CRUD apps where the overhead doesn't justify the benefit. Use exceptions for "should never happen" bugs; use Results for expected business failures.

---

### 3. Side Effects Must Be Isolated

Separate pure business logic from I/O operations.

**Mixed logic (impure):**

```csharp
public class AppointmentService
{
    public async Task ProcessReminders()
    {
        var appointments = await _db.Appointments
            .Where(a => a.ScheduledAt < DateTime.UtcNow.AddHours(24))
            .ToListAsync();
            
        foreach (var apt in appointments)
        {
            await _emailService.SendReminderAsync(apt.PatientEmail);
            _logger.LogInformation("Sent reminder for {Id}", apt.Id);
        }
    }
}
```

Problems:
- `DateTime.UtcNow` makes tests non-deterministic
- Can't test logic without mocking database and email
- Business rules hidden inside infrastructure code

**Pure logic + impure orchestration:**

```csharp
// Pure domain logic - no dependencies, fully testable
public static class ReminderPolicy
{
    public record ReminderDecision(
        AppointmentId AppointmentId,
        bool ShouldSend,
        string? Reason);
    
    public static ReminderDecision ShouldSendReminder(
        Appointment appointment,
        DateTimeOffset now,
        TimeSpan reminderWindow)
    {
        var timeUntilAppointment = appointment.ScheduledAt - now;
        
        if (appointment.Status == AppointmentStatus.Cancelled)
            return new(appointment.Id, false, "Appointment cancelled");
            
        if (timeUntilAppointment <= TimeSpan.Zero)
            return new(appointment.Id, false, "Appointment already passed");
            
        if (timeUntilAppointment <= reminderWindow)
            return new(appointment.Id, true, $"Appointment in {timeUntilAppointment.Hours} hours");
            
        return new(appointment.Id, false, "Too early for reminder");
    }
}

// Impure orchestration layer
public class ReminderOrchestrator
{
    private readonly IAppointmentRepository _repository;
    private readonly IEmailService _emailService;
    private readonly TimeProvider _timeProvider; // .NET 8 abstraction
    
    public async Task ProcessRemindersAsync(CancellationToken ct)
    {
        var now = _timeProvider.GetUtcNow();
        var appointments = await _repository.GetUpcomingAsync(now, ct);
        
        foreach (var apt in appointments)
        {
            var decision = ReminderPolicy.ShouldSendReminder(
                apt, now, TimeSpan.FromHours(24));
                
            if (decision.ShouldSend)
                await _emailService.SendReminderAsync(apt, ct);
        }
    }
}
```

**C# features used:**
- `TimeProvider` (.NET 8) — testable time abstraction
- `static class` — enforces no instance state
- `record` — immutable decision objects

---

### 4. Boundaries Must Be Narrow and Typed

Use explicit DTOs and validation at system boundaries.

**Weak boundary:**

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder([FromBody] JsonElement body)
{
    var customerId = body.GetProperty("customerId").GetGuid();
    var items = body.GetProperty("items").EnumerateArray();
    // Hope everything exists and is the right type...
}
```

**Strong boundary with validation:**

```csharp
// Request DTO with validation attributes
public record CreateOrderRequest
{
    [Required]
    public required Guid CustomerId { get; init; }
    
    [Required, MinLength(1)]
    public required List<OrderLineRequest> Lines { get; init; }
    
    [Range(0, 100)]
    public decimal? DiscountPercent { get; init; }
}

public record OrderLineRequest
{
    [Required]
    public required Guid ProductId { get; init; }
    
    [Range(1, 1000)]
    public required int Quantity { get; init; }
}

// Controller with model binding
[HttpPost]
public async Task<IActionResult> CreateOrder(
    [FromBody] CreateOrderRequest request,
    CancellationToken ct)
{
    // Validation already happened via model binding
    // Convert to domain types at the boundary
    var command = new CreateOrderCommand(
        CustomerId: new CustomerId(request.CustomerId),
        Lines: request.Lines.Select(l => new OrderLine(
            new ProductId(l.ProductId),
            Quantity.Create(l.Quantity))).ToList()
    );
    
    var result = await _handler.HandleAsync(command, ct);
    return result.Match(
        success: order => CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order),
        failure: error => BadRequest(error)
    );
}
```

**Best practices:**
- Separate API DTOs from domain models
- Validate at the boundary, trust internally
- Use `FluentValidation` for complex validation rules
- Generate OpenAPI contracts from typed endpoints

---

### 5. Automation Over Intention

Humans forget. The compiler and analyzers enforce.

**Key configuration principles:**
- `<Nullable>enable</Nullable>` — catch null bugs at compile time
- `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` — no ignored warnings
- `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>` — consistent style
- Roslyn analyzers — catch anti-patterns automatically

> See [Chapter 3: Project Configuration](03-project-configuration.md) for complete `Directory.Build.props`, `.editorconfig`, and analyzer setup.

---

[← Back to Index](README.md) | [Next: Language Features →](02-language-features.md)
