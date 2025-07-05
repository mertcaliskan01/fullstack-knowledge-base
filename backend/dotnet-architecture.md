# Scalable Backend Architecture with .NET

## Introduction

Building scalable .NET applications requires more than just writing functional code. A well-architected backend serves as the foundation for maintainability, testability, and future growth. This guide focuses on architectural patterns that have proven successful in production environments, moving beyond basic tutorials to address real-world challenges that senior developers face.

## Project Structure and Layering

### Clean Architecture Layers

```
YourProject/
├── src/
│   ├── YourProject.API/                 # Presentation Layer
│   ├── YourProject.Application/         # Application Layer
│   ├── YourProject.Domain/              # Domain Layer
│   └── YourProject.Infrastructure/      # Infrastructure Layer
├── tests/
│   ├── YourProject.UnitTests/
│   ├── YourProject.IntegrationTests/
│   └── YourProject.FunctionalTests/
└── docs/
```

### Layer Responsibilities

**Domain Layer** - Your business logic core:
- Entities and value objects
- Domain services
- Domain events
- Business rules and invariants

**Application Layer** - Orchestration and use cases:
- Command/Query handlers (CQRS)
- Application services
- DTOs and mapping
- Validation logic

**Infrastructure Layer** - External concerns:
- Database context and repositories
- External API clients
- Message queues
- File storage

**Presentation Layer** - API surface:
- Controllers and endpoints
- Middleware configuration
- Authentication/Authorization
- API documentation

## Core Design Principles

### Dependency Injection

.NET's built-in DI container is powerful, but its effectiveness depends on proper configuration:

```csharp
// Program.cs
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IEmailService, EmailService>();

// Use constructor injection consistently
public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;
    private readonly IEmailService _emailService;
    
    public UserService(IUserRepository userRepository, IEmailService emailService)
    {
        _userRepository = userRepository;
        _emailService = emailService;
    }
}
```

### SOLID Principles in Practice

**Single Responsibility Principle** - Each class has one reason to change:

```csharp
// Good: Separate concerns
public class UserRegistrationService
{
    public async Task<Result<User>> RegisterUserAsync(RegisterUserCommand command)
    {
        // Only handles user registration logic
    }
}

public class UserValidationService
{
    public ValidationResult ValidateUser(RegisterUserCommand command)
    {
        // Only handles validation logic
    }
}
```

**Open/Closed Principle** - Extend without modification:

```csharp
public interface INotificationService
{
    Task SendNotificationAsync(Notification notification);
}

public class EmailNotificationService : INotificationService { }
public class SmsNotificationService : INotificationService { }
public class PushNotificationService : INotificationService { }

// Easy to add new notification types without changing existing code
```

## API Design Best Practices

### Controller Design

Controllers should be thin and focused on HTTP concerns:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public UsersController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    [HttpPost]
    public async Task<ActionResult<UserDto>> CreateUser([FromBody] CreateUserCommand command)
    {
        var result = await _mediator.Send(command);
        
        if (result.IsSuccess)
            return CreatedAtAction(nameof(GetUser), new { id = result.Value.Id }, result.Value);
            
        return BadRequest(result.Errors);
    }
}
```

### Middleware Strategy

Custom middleware for cross-cutting concerns:

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;
    
    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation(
                "Request {Method} {Path} completed in {ElapsedMs}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}
```

### Action Filters

Filters for common concerns like validation and caching:

```csharp
public class ValidationFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            var errors = context.ModelState
                .Where(x => x.Value.Errors.Count > 0)
                .Select(x => new { Field = x.Key, Errors = x.Value.Errors.Select(e => e.ErrorMessage) });
                
            context.Result = new BadRequestObjectResult(errors);
        }
    }
    
    public void OnActionExecuted(ActionExecutedContext context) { }
}
```

## Error Handling and Logging

### Global Exception Handling

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "An unhandled exception occurred");
        
        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An error occurred while processing your request.",
            Detail = exception.Message
        };
        
        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);
        
        return true;
    }
}
```

### Structured Logging

```csharp
// Use structured logging for better observability
_logger.LogInformation(
    "User {UserId} created order {OrderId} with {ItemCount} items for {TotalAmount:C}",
    userId,
    orderId,
    itemCount,
    totalAmount);
```

## Database Patterns

### Repository Pattern with Entity Framework Core

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly DbContext _context;
    private readonly DbSet<T> _dbSet;
    
    public Repository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }
    
    public async Task<T?> GetByIdAsync(object id)
    {
        return await _dbSet.FindAsync(id);
    }
    
    public async Task<T> AddAsync(T entity)
    {
        var entry = await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
        return entry.Entity;
    }
}
```

### Unit of Work Pattern

```csharp
public interface IUnitOfWork : IDisposable
{
    IRepository<T> Repository<T>() where T : class;
    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly DbContext _context;
    private readonly Dictionary<Type, object> _repositories;
    private IDbContextTransaction? _transaction;
    
    public UnitOfWork(DbContext context)
    {
        _context = context;
        _repositories = new Dictionary<Type, object>();
    }
    
    public IRepository<T> Repository<T>() where T : class
    {
        var type = typeof(T);
        
        if (!_repositories.ContainsKey(type))
        {
            _repositories[type] = new Repository<T>(_context);
        }
        
        return (IRepository<T>)_repositories[type];
    }
    
    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }
}
```

## Testing Strategies

### Unit Testing with xUnit

```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepository;
    private readonly Mock<IEmailService> _mockEmailService;
    private readonly UserService _userService;
    
    public UserServiceTests()
    {
        _mockRepository = new Mock<IUserRepository>();
        _mockEmailService = new Mock<IEmailService>();
        _userService = new UserService(_mockRepository.Object, _mockEmailService.Object);
    }
    
    [Fact]
    public async Task RegisterUser_WithValidData_ShouldCreateUserAndSendEmail()
    {
        // Arrange
        var command = new RegisterUserCommand("test@example.com", "password");
        var user = new User { Id = 1, Email = command.Email };
        
        _mockRepository.Setup(r => r.AddAsync(It.IsAny<User>()))
            .ReturnsAsync(user);
            
        // Act
        var result = await _userService.RegisterUserAsync(command);
        
        // Assert
        Assert.True(result.IsSuccess);
        _mockRepository.Verify(r => r.AddAsync(It.IsAny<User>()), Times.Once);
        _mockEmailService.Verify(e => e.SendWelcomeEmailAsync(user.Email), Times.Once);
    }
}
```

### Integration Testing

```csharp
public class UserControllerIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public UserControllerIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task CreateUser_ReturnsCreatedUser()
    {
        // Arrange
        var client = _factory.CreateClient();
        var command = new CreateUserCommand("test@example.com", "password");
        
        // Act
        var response = await client.PostAsJsonAsync("/api/users", command);
        
        // Assert
        response.EnsureSuccessStatusCode();
        var user = await response.Content.ReadFromJsonAsync<UserDto>();
        Assert.NotNull(user);
        Assert.Equal(command.Email, user.Email);
    }
}
```

## Performance Considerations

### Caching Strategy

```csharp
public class CachedUserService : IUserService
{
    private readonly IUserService _userService;
    private readonly IDistributedCache _cache;
    
    public async Task<User?> GetUserByIdAsync(int id)
    {
        var cacheKey = $"user:{id}";
        var cachedUser = await _cache.GetStringAsync(cacheKey);
        
        if (cachedUser != null)
        {
            return JsonSerializer.Deserialize<User>(cachedUser);
        }
        
        var user = await _userService.GetUserByIdAsync(id);
        
        if (user != null)
        {
            await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(user), 
                new DistributedCacheEntryOptions { SlidingExpiration = TimeSpan.FromMinutes(30) });
        }
        
        return user;
    }
}
```

### Async/Await Best Practices

```csharp
// Good: Proper async/await usage
public async Task<IEnumerable<User>> GetActiveUsersAsync()
{
    return await _context.Users
        .Where(u => u.IsActive)
        .ToListAsync();
}

// Avoid: Blocking calls in async methods
public async Task<IEnumerable<User>> GetActiveUsersAsync()
{
    return _context.Users  // Don't do this
        .Where(u => u.IsActive)
        .ToList();         // This blocks
}
```

## Conclusion

Building scalable .NET applications requires deliberate architectural decisions. The patterns and practices outlined here provide a solid foundation, but remember that architecture should evolve with your application's needs. Start with these fundamentals, measure performance and maintainability, and refine your approach based on real-world feedback.

The key is consistency - once you establish these patterns, maintain them across your codebase. Your future self (and team) will thank you when the application grows and requirements change.
