> [← Back to Index](README.md)

# 7. C# Anti-Patterns to Avoid

Learn from common C#/.NET mistakes.

## Code Anti-Patterns

> **Full explanations:** See [01-core-philosophy](01-core-philosophy.md) for principles, [02-language-features](02-language-features.md) for implementations.

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| `Guid` for all IDs | ID mixups compile fine | `readonly record struct PatientId(Guid Value)` |
| `string` for enums | Typos, case issues | `enum`, then pattern match exhaustively |
| `catch (Exception)` | Hidden failures | Explicit Result types, let infrastructure exceptions bubble |
| `public set` everywhere | Unexpected mutation | `init`, `required`, records |
| `DateTime.Now` in logic | Non-deterministic tests | Inject `TimeProvider` |
| Service locator | Hidden dependencies | Constructor injection |
| `async void` | Unobserved exceptions | `async Task` always |
| `Task.Result` / `.Wait()` | Deadlocks, thread starvation | `await` throughout |

---

## Detailed Examples

### Anti-Pattern: Generic ID Types

```csharp
// ❌ Wrong - IDs are interchangeable
public void TransferMoney(Guid fromAccountId, Guid toAccountId, decimal amount)
{
    // Easy to swap parameters by mistake!
}

// ✅ Better - Compile-time safety
public void TransferMoney(AccountId from, AccountId to, Money amount)
{
    // Compiler prevents parameter swapping
}
```

### Anti-Pattern: Null Returns for "Not Found"

```csharp
// ❌ Wrong - null hides meaning
public Patient? GetPatient(Guid id)
{
    return _context.Patients.Find(id); // null if not found
}

// ✅ Better - explicit Result
public async Task<ErrorOr<Patient>> GetPatientAsync(PatientId id, CancellationToken ct)
{
    var patient = await _context.Patients.FindAsync(id.Value, ct);
    return patient is null 
        ? Error.NotFound("Patient.NotFound", $"Patient {id} not found")
        : patient;
}
```

### Anti-Pattern: Mutable DTOs

```csharp
// ❌ Wrong - can be modified after creation
public class CreateOrderRequest
{
    public Guid CustomerId { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
}

// ✅ Better - immutable, validated
public record CreateOrderRequest
{
    [Required]
    public required Guid CustomerId { get; init; }
    
    [Required, MinLength(1)]
    public required IReadOnlyList<OrderLine> Lines { get; init; }
}
```

### Anti-Pattern: DateTime.Now in Business Logic

```csharp
// ❌ Wrong - not testable
public bool IsExpired(License license)
{
    return license.ExpiresAt < DateTime.UtcNow;
}

// ✅ Better - testable
public bool IsExpired(License license, DateTimeOffset now)
{
    return license.ExpiresAt < now;
}

// Or inject TimeProvider (.NET 8)
public class LicenseService(TimeProvider timeProvider)
{
    public bool IsExpired(License license)
    {
        return license.ExpiresAt < timeProvider.GetUtcNow();
    }
}
```

### Anti-Pattern: Service Locator

```csharp
// ❌ Wrong - hidden dependencies
public class OrderService
{
    public void Process(Order order)
    {
        var emailService = ServiceLocator.Get<IEmailService>(); // Hidden!
        var logger = ServiceLocator.Get<ILogger>(); // Hidden!
    }
}

// ✅ Better - explicit dependencies
public class OrderService(
    IEmailService emailService,
    ILogger<OrderService> logger)
{
    public void Process(Order order)
    {
        // Dependencies are visible in constructor
    }
}
```

### Anti-Pattern: Async Void

```csharp
// ❌ Wrong - exceptions are lost
public async void SendNotification(User user)
{
    await _emailService.SendAsync(user.Email); // Exception goes nowhere!
}

// ✅ Better - exceptions can be observed
public async Task SendNotificationAsync(User user, CancellationToken ct)
{
    await _emailService.SendAsync(user.Email, ct);
}
```

### Anti-Pattern: Blocking on Async Code

```csharp
// ❌ Wrong - can cause deadlocks
public User GetUser(Guid id)
{
    return _userService.GetUserAsync(id).Result; // Deadlock risk!
}

// ✅ Better - async all the way
public async Task<User> GetUserAsync(Guid id, CancellationToken ct)
{
    return await _userService.GetUserAsync(id, ct);
}
```

---

## Quick Checklist

Before submitting a PR, verify:

- [ ] No raw `Guid`/`int` for domain IDs (use typed IDs)
- [ ] No `catch (Exception)` without re-throw or explicit handling
- [ ] No `DateTime.Now` or `DateTime.UtcNow` (use `TimeProvider`)
- [ ] No `async void` methods
- [ ] No `.Result` or `.Wait()` on tasks
- [ ] No service locator usage
- [ ] All DTOs use `init` and `required` where appropriate

---

[← Standards & Culture](06-standards-culture.md) | [Index](README.md)
