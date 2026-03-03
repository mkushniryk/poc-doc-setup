> [← Back to Index](README.md)

# 2. Advanced C# Safety Techniques

Beyond the basics—techniques that make invalid states unrepresentable and shift runtime errors to compile time.

---

## 2.1 Strongly-Typed IDs with Vogen

Skip the boilerplate, get compile-time validation.

```csharp
using Vogen;

// Generates all the boilerplate: converters, equality, validation
[ValueObject<Guid>]
public partial struct PatientId 
{
    private static Validation Validate(Guid input) =>
        input == Guid.Empty 
            ? Validation.Invalid("Cannot be empty GUID") 
            : Validation.Ok;
}

[ValueObject<string>]
public partial struct Email
{
    private static Validation Validate(string input) =>
        input.Contains('@') 
            ? Validation.Ok 
            : Validation.Invalid("Invalid email");
    
    private static string NormalizeInput(string input) => 
        input.Trim().ToLowerInvariant();
}

// Usage
var id = PatientId.From(Guid.NewGuid());  // ✅
var bad = PatientId.From(Guid.Empty);     // 💥 Throws ValueObjectValidationException

// JSON serialization, EF Core conversion, etc. - all generated
```

---

## 2.2 Discriminated Unions: Eliminating Invalid States

Use `OneOf` or custom union types to make illegal states unrepresentable.

```csharp
// ❌ Primitive obsession - runtime errors waiting to happen
public class ApiResponse
{
    public User? User { get; set; }
    public string? ErrorMessage { get; set; }
    public bool IsSuccess { get; set; }
    // Can be in invalid state: IsSuccess=true but User=null
}

// ✅ Discriminated union - invalid states are impossible
using OneOf;

public readonly struct ApiResponse : IOneOf
{
    private readonly OneOf<User, NotFound, ValidationError, Forbidden> _value;
    
    public ApiResponse(OneOf<User, NotFound, ValidationError, Forbidden> value) => _value = value;
    
    // Force exhaustive handling at compile time
    public TResult Match<TResult>(
        Func<User, TResult> user,
        Func<NotFound, TResult> notFound,
        Func<ValidationError, TResult> validation,
        Func<Forbidden, TResult> forbidden) => _value.Match(user, notFound, validation, forbidden);
}

// Usage - compiler forces you to handle ALL cases
var response = await GetUserAsync(id);
return response.Match(
    user => Ok(user),
    notFound => NotFound(),
    validation => BadRequest(validation.Errors),
    forbidden => Forbid()
);
```

**Roll your own minimal union:**

```csharp
// When you don't want a dependency
public abstract record Result<T>
{
    private Result() { }
    
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(Error Error) : Result<T>;
    
    public TResult Match<TResult>(Func<T, TResult> success, Func<Error, TResult> failure) =>
        this switch
        {
            Success s => success(s.Value),
            Failure f => failure(f.Error),
            _ => throw new InvalidOperationException("Exhaustive match failed")
        };
}

// Private constructor + sealed nested types = closed hierarchy
// No external code can add new cases
```

**Practical usage with ErrorOr library:**

```csharp
using ErrorOr;

public class PatientService(IPatientRepository repository)
{
    public async Task<ErrorOr<Patient>> GetPatientAsync(
        PatientId id, 
        CancellationToken ct)
    {
        var patient = await repository.FindAsync(id, ct);
        
        if (patient is null)
            return Error.NotFound("Patient.NotFound", $"Patient {id} not found");
            
        if (patient.IsArchived)
            return Error.Validation("Patient.Archived", "Patient record is archived");
            
        return patient;
    }
}

// In controller - compiler forces handling all outcomes
public async Task<IActionResult> GetPatient(Guid id, CancellationToken ct)
{
    var result = await _service.GetPatientAsync(new PatientId(id), ct);
    
    return result.Match(
        patient => Ok(patient),
        errors => errors.First().Type switch
        {
            ErrorType.NotFound => NotFound(errors),
            ErrorType.Validation => BadRequest(errors),
            _ => Problem()
        }
    );
}
```

> **When to skip discriminated unions:** Simple APIs with only success/fail outcomes (boolean or nullable return may suffice), or when the team is unfamiliar with functional patterns. Introduce gradually—start with `ErrorOr<T>` in new code before converting existing code.

---

## 2.3 Result Pattern with Predefined Errors

Centralize error definitions as constants for consistency and type safety.

```csharp
// Define domain errors as static constants
public static class PatientErrors
{
    public static readonly Error NotFound = Error.NotFound(
        code: "Patient.NotFound",
        description: "Patient was not found");
    
    public static readonly Error Archived = Error.Validation(
        code: "Patient.Archived",
        description: "Patient record is archived and cannot be modified");
    
    public static readonly Error InvalidEmail = Error.Validation(
        code: "Patient.InvalidEmail",
        description: "Patient email format is invalid");
    
    public static readonly Error DuplicateEmail = Error.Conflict(
        code: "Patient.DuplicateEmail",
        description: "A patient with this email already exists");
        
    // Factory method for errors that need dynamic data
    public static Error NotFoundById(PatientId id) => Error.NotFound(
        code: "Patient.NotFound",
        description: $"Patient with ID {id.Value} was not found");
}

public static class OrderErrors
{
    public static readonly Error NotFound = Error.NotFound(
        code: "Order.NotFound",
        description: "Order was not found");
    
    public static readonly Error AlreadyShipped = Error.Validation(
        code: "Order.AlreadyShipped",
        description: "Cannot modify an order that has already shipped");
    
    public static readonly Error InsufficientStock = Error.Validation(
        code: "Order.InsufficientStock",
        description: "Insufficient stock to fulfill order");
}
```

**Usage in services:**

```csharp
public class PatientService(IPatientRepository repository)
{
    public async Task<ErrorOr<Patient>> GetPatientAsync(
        PatientId id, 
        CancellationToken ct)
    {
        var patient = await repository.FindAsync(id, ct);
        
        if (patient is null)
            return PatientErrors.NotFoundById(id);  // Predefined error
            
        if (patient.IsArchived)
            return PatientErrors.Archived;  // Predefined error
            
        return patient;
    }
    
    public async Task<ErrorOr<Patient>> UpdateEmailAsync(
        PatientId id,
        Email newEmail,
        CancellationToken ct)
    {
        var patient = await repository.FindAsync(id, ct);
        
        if (patient is null)
            return PatientErrors.NotFound;
            
        if (await repository.EmailExistsAsync(newEmail, ct))
            return PatientErrors.DuplicateEmail;
            
        patient.UpdateEmail(newEmail);
        await repository.SaveAsync(patient, ct);
        
        return patient;
    }
}
```

**Benefits of predefined errors:**
- **Consistency** — same error code/message across the codebase
- **Discoverability** — IDE autocomplete shows available errors
- **Testability** — compare against known error constants
- **Documentation** — error catalog in one place
- **Refactoring** — change error message in one location

**Testing with predefined errors:**

```csharp
[Fact]
public async Task GetPatient_WhenNotFound_ReturnsNotFoundError()
{
    // Arrange
    var id = PatientId.New();
    _repository.FindAsync(id, Arg.Any<CancellationToken>())
        .Returns((Patient?)null);
    
    // Act
    var result = await _sut.GetPatientAsync(id, CancellationToken.None);
    
    // Assert
    result.IsError.Should().BeTrue();
    result.FirstError.Code.Should().Be(PatientErrors.NotFound.Code);
}
```

**Organizing errors by layer:**

```
src/
├── Domain/
│   └── Errors/
│       ├── PatientErrors.cs
│       ├── OrderErrors.cs
│       └── CommonErrors.cs
```

```csharp
// Common errors shared across domains
public static class CommonErrors
{
    public static readonly Error Unauthorized = Error.Unauthorized(
        code: "Auth.Unauthorized",
        description: "You are not authorized to perform this action");
    
    public static readonly Error Forbidden = Error.Forbidden(
        code: "Auth.Forbidden",
        description: "Access to this resource is forbidden");
    
    public static readonly Error ConcurrencyConflict = Error.Conflict(
        code: "Data.ConcurrencyConflict",
        description: "The record was modified by another user");
}
```

---

## 2.4 Static Analysis Attributes

Teach the compiler about your code's invariants.

```csharp
using System.Diagnostics.CodeAnalysis;

public static class Guard
{
    // Compiler knows this throws - dead code after is flagged
    [DoesNotReturn]
    public static void Fail(string message) => throw new InvalidOperationException(message);
    
    // Compiler knows: if returns true, 'value' is not null
    public static bool IsNotNull<T>([NotNullWhen(true)] T? value) => value is not null;
    
    // Compiler knows: when method returns, 'value' is not null
    public static void NotNull<T>([NotNull] T? value, [CallerArgumentExpression(nameof(value))] string? expr = null)
    {
        if (value is null)
            throw new ArgumentNullException(expr);
    }
    
    // Compiler understands string emptiness
    public static void NotEmpty([NotNull] string? value, [CallerArgumentExpression(nameof(value))] string? expr = null)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Cannot be empty", expr);
    }
}

// Usage - automatic parameter name in exceptions
public void Process(string input)
{
    Guard.NotEmpty(input); // Throws: "Cannot be empty (Parameter 'input')"
    // Compiler knows 'input' is non-null here
}

// MemberNotNull - guarantees field initialization
public class Connection
{
    private Socket? _socket;
    
    [MemberNotNull(nameof(_socket))]
    private void EnsureConnected()
    {
        _socket ??= new Socket(...);
    }
    
    public void Send(byte[] data)
    {
        EnsureConnected();
        _socket.Send(data); // No warning - compiler knows _socket is initialized
    }
}
```

**All critical attributes:**

| Attribute | Effect |
|-----------|--------|
| `[NotNull]` | Output is guaranteed non-null after call |
| `[NotNullWhen(bool)]` | Output non-null when return equals bool |
| `[NotNullIfNotNull("param")]` | Output non-null if param is non-null |
| `[DoesNotReturn]` | Method never returns (throws/exits) |
| `[DoesNotReturnIf(bool)]` | Throws when condition equals bool |
| `[MemberNotNull("field")]` | Field guaranteed initialized after call |
| `[MemberNotNullWhen(bool, "field")]` | Field initialized when return equals bool |

---

## 2.5 Phantom Types: Compile-Time State Machines

Encode state in the type system—invalid transitions become compile errors.

```csharp
// Marker types (no runtime cost)
public readonly struct Draft { }
public readonly struct Submitted { }
public readonly struct Approved { }
public readonly struct Rejected { }

// State encoded in generic parameter
public sealed class Document<TState> where TState : struct
{
    public DocumentId Id { get; }
    public string Content { get; }
    
    internal Document(DocumentId id, string content)
    {
        Id = id;
        Content = content;
    }
}

// Extension methods constrain valid transitions
public static class DocumentTransitions
{
    public static Document<Submitted> Submit(this Document<Draft> doc) =>
        new(doc.Id, doc.Content);
    
    public static Document<Approved> Approve(this Document<Submitted> doc) =>
        new(doc.Id, doc.Content);
    
    public static Document<Rejected> Reject(this Document<Submitted> doc) =>
        new(doc.Id, doc.Content);
    
    // Only approved documents can be published
    public static void Publish(this Document<Approved> doc) =>
        PublishingService.Publish(doc);
}

// Usage
var draft = new Document<Draft>(id, "content");
var submitted = draft.Submit();
var approved = submitted.Approve();
approved.Publish();  // ✅ Compiles

draft.Approve();     // ❌ Compile error - no Approve on Draft
submitted.Publish(); // ❌ Compile error - no Publish on Submitted
approved.Submit();   // ❌ Compile error - no Submit on Approved
```

---

## 2.6 Capability-Based Security with Interfaces

Encode permissions in the type system.

```csharp
// Capabilities as interfaces (no methods needed)
public interface ICanRead { }
public interface ICanWrite { }
public interface ICanDelete { }
public interface ICanAdmin : ICanRead, ICanWrite, ICanDelete { }

// Repository with capability constraints
public interface IRepository<T, TCapability> where T : class
{
    Task<T?> GetAsync(Guid id) 
        where TCapability : ICanRead;
    
    Task SaveAsync(T entity) 
        where TCapability : ICanWrite;
    
    Task DeleteAsync(Guid id) 
        where TCapability : ICanDelete;
}

// Context carries capability proof
public sealed class SecureContext<TCapability>(ClaimsPrincipal user)
{
    public ClaimsPrincipal User => user;
}

// Service methods require specific capabilities
public class DocumentService
{
    public Task<Document?> GetDocument(SecureContext<ICanRead> ctx, DocumentId id) =>
        _repo.GetAsync(id);
    
    public Task SaveDocument(SecureContext<ICanWrite> ctx, Document doc) =>
        _repo.SaveAsync(doc);
    
    // Only admin context can call this
    public Task DeleteDocument(SecureContext<ICanAdmin> ctx, DocumentId id) =>
        _repo.DeleteAsync(id);
}

// At the entry point - create context based on authentication
public IActionResult DeleteDoc(DocumentId id)
{
    if (User.IsInRole("Admin"))
    {
        var ctx = new SecureContext<ICanAdmin>(User);
        await _service.DeleteDocument(ctx, id);  // ✅ Compiles
        return Ok();
    }
    
    var readCtx = new SecureContext<ICanRead>(User);
    await _service.DeleteDocument(readCtx, id); // ❌ Compile error
}
```

---

## 2.7 Generic Math: Type-Safe Numeric Operations

Eliminate runtime type checks for numeric code (C# 11+).

```csharp
using System.Numerics;

// Works with any numeric type - int, double, decimal, BigInteger...
public static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var value in values)
        sum += value;
    return sum;
}

// Safe division with proper constraints
public static T SafeDivide<T>(T dividend, T divisor, T fallback = default!) 
    where T : INumber<T>
{
    return divisor == T.Zero ? fallback : dividend / divisor;
}

// Percentage type that's distinct from raw decimals
public readonly record struct Percentage : 
    INumber<Percentage>,
    IMinMaxValue<Percentage>
{
    private readonly decimal _value;
    
    public static Percentage MinValue => new(0);
    public static Percentage MaxValue => new(100);
    
    private Percentage(decimal value) => _value = Math.Clamp(value, 0, 100);
    
    public static Percentage FromDecimal(decimal value) => new(value);
    public decimal ToDecimal() => _value;
    
    // Implement INumber<T> members...
    // Benefits: Percentage + Percentage = Percentage (type-safe)
    // Money * Percentage = Money (via operator overloads)
}
```

---

## 2.8 Source Generators for Compile-Time Validation

Generate validation code that fails at compile time.

```csharp
// Attribute to trigger source generation
[GenerateStrongType]
public partial record struct Email
{
    private static partial Regex ValidationPattern();  // Generated
    
    [GeneratedRegex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$")]
    private static partial Regex ValidationPattern();
    
    private readonly string _value;
    
    private Email(string value) => _value = value;
    
    public static Result<Email> Create(string value) =>
        ValidationPattern().IsMatch(value)
            ? new Email(value)
            : Result.Failure<Email>("Invalid email format");
}

// Source generator can also create:
// - JSON converters
// - EF Core value converters  
// - Equality members
// - ToString()
// All at compile time with zero reflection
```

**Example: Generate exhaustive enum matching:**

```csharp
// Analyzer triggers compile error if you forget a case
[Closed] // Custom attribute checked by analyzer
public enum PaymentStatus
{
    Pending,
    Processing,
    Completed,
    Failed,
    Refunded
}

// Your analyzer enforces exhaustive matching
public string GetMessage(PaymentStatus status) => status switch
{
    PaymentStatus.Pending => "Waiting",
    PaymentStatus.Processing => "In progress",
    PaymentStatus.Completed => "Done",
    // ❌ Compile error: Missing cases: Failed, Refunded
};
```

---

## 2.9 Exhaustive Matching

Ensure all cases are handled at compile time.

```csharp
// C# requires a default case, but you can make it throw
public decimal CalculatePrice(SubscriptionTier tier) => tier switch
{
    SubscriptionTier.Free => 0m,
    SubscriptionTier.Basic => 9.99m,
    SubscriptionTier.Pro => 29.99m,
    SubscriptionTier.Enterprise => 99.99m,
    _ => throw new UnreachableException($"Unhandled tier: {tier}")
};

// Better: Use analyzer warning CS8509 (incomplete switch)
// Enable in .editorconfig:
// dotnet_diagnostic.CS8509.severity = error
```

---

## 2.10 Option Type: Model Optional Values Without Null

```csharp
// Simple Option implementation
public readonly struct Option<T> where T : notnull
{
    private readonly T? _value;
    private readonly bool _hasValue;
    
    private Option(T value)
    {
        _value = value;
        _hasValue = true;
    }
    
    public static Option<T> Some(T value) => new(value);
    public static Option<T> None => default;
    
    public TResult Match<TResult>(
        Func<T, TResult> onSome,
        Func<TResult> onNone) => _hasValue ? onSome(_value!) : onNone();
    
    public Option<TResult> Map<TResult>(Func<T, TResult> mapper) where TResult : notnull
        => _hasValue ? Option<TResult>.Some(mapper(_value!)) : Option<TResult>.None;
    
    public T GetValueOrDefault(T defaultValue) 
        => _hasValue ? _value! : defaultValue;
}

// Usage
public Option<Patient> FindPatient(PatientId id)
{
    var patient = _context.Patients.Find(id.Value);
    return patient is not null 
        ? Option<Patient>.Some(patient) 
        : Option<Patient>.None;
}

var name = FindPatient(id).Match(
    onSome: p => p.FullName,
    onNone: () => "Unknown"
);
```

---

## 2.11 Small, Focused Interfaces

Depend on behaviors, not large abstractions.

```csharp
// Instead of large interfaces
public interface IRepository<T>
{
    T GetById(Guid id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
    // ... 10 more methods
}

// Define small, focused interfaces
public interface IReader<T, TId>
{
    Task<T?> FindAsync(TId id, CancellationToken ct);
}

public interface IWriter<T>
{
    Task AddAsync(T entity, CancellationToken ct);
}

// Compose what you need
public class OrderService(
    IReader<Order, OrderId> orderReader,
    IWriter<Order> orderWriter)
{
    // Only has access to what it needs
}
```

---

[← Core Philosophy](01-core-philosophy.md) | [Index](README.md) | [Next: Project Configuration →](03-project-configuration.md)
