# When to Mock

Mock at **system boundaries** only:

- External APIs (payment, email, etc.)
- Databases (sometimes - prefer test DB; in .NET, the real engine in Testcontainers)
- Time/randomness (in .NET, inject `TimeProvider` or the project's clock abstraction)
- File system (sometimes)

Don't mock:

- Your own classes/modules
- Internal collaborators
- Anything you control

## EF Core InMemory is not a test database

The InMemory provider enforces no unique indexes, foreign keys, cascades or transactions, and it translates queries differently from a relational provider. A test that passes on InMemory proves the code's logic, never what the database decides. Test database-dependent behaviour against the real engine, and follow the project's ADR on which kind of test owns what. Substitute the database, not EF Core's API: a mocked `DbContext` or `DbSet<T>` tests your mock setup.

## Designing for Mockability

At system boundaries, design interfaces that are easy to mock:

**1. Use dependency injection**

Pass external dependencies in rather than creating them internally:

```csharp
// Easy to mock
public sealed class PaymentProcessor(IPaymentClient paymentClient)
{
    public Task<ChargeResult> ProcessAsync(Order order) => paymentClient.ChargeAsync(order.Total);
}

// Hard to mock
public sealed class PaymentProcessor
{
    public Task<ChargeResult> ProcessAsync(Order order)
    {
        var client = new StripeClient(Environment.GetEnvironmentVariable("STRIPE_KEY"));
        return client.ChargeAsync(order.Total);
    }
}
```

**2. Prefer SDK-style interfaces over generic fetchers**

Create specific methods for each external operation instead of one generic method with conditional logic:

```csharp
// GOOD: Each method is independently mockable
public interface IShopApi
{
    Task<User> GetUserAsync(Guid id);
    Task<IReadOnlyList<Order>> GetOrdersAsync(Guid userId);
    Task<Order> CreateOrderAsync(CreateOrderRequest request);
}

// BAD: Mocking requires conditional logic inside the mock
public interface IShopApi
{
    Task<HttpResponseMessage> SendAsync(string endpoint, HttpMethod method, object? body = null);
}
```

The SDK approach means:
- Each mock returns one specific shape (`api.GetUserAsync(id).Returns(user)` in NSubstitute)
- No conditional logic in test setup
- Easier to see which endpoints a test exercises
- Type safety per endpoint
