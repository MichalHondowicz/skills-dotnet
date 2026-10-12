# Good and Bad Tests

Examples use xUnit and Shouldly; follow the project's own test naming convention over the names here.

## Good Tests

**Integration-style**: Test through real interfaces, not mocks of internal parts.

```csharp
// GOOD: Tests observable behavior
[Fact]
public async Task UserCanCheckoutWithValidCart()
{
    Cart cart = CreateCart();
    cart.Add(product);

    CheckoutResult result = await checkout.CheckoutAsync(cart, paymentMethod);

    result.Status.ShouldBe(CheckoutStatus.Confirmed);
}
```

Characteristics:

- Tests behavior users/callers care about
- Uses public API only
- Survives internal refactors
- Describes WHAT, not HOW
- One logical assertion per test

## Bad Tests

**Implementation-detail tests**: Coupled to internal structure.

```csharp
// BAD: Tests implementation details
[Fact]
public async Task CheckoutCallsPaymentServiceProcess()
{
    IPaymentService payment = Substitute.For<IPaymentService>();
    var checkout = new CheckoutService(payment);

    await checkout.CheckoutAsync(cart, paymentMethod);

    await payment.Received(1).ProcessAsync(cart.Total);
}
```

Red flags:

- Mocking internal collaborators
- Testing private methods
- Asserting on call counts/order
- Test breaks when refactoring without behavior change
- Test name describes HOW not WHAT
- Verifying through external means instead of interface

```csharp
// BAD: Bypasses interface to verify
[Fact]
public async Task CreateUserSavesToDatabase()
{
    await client.PostAsJsonAsync("users", new { name = "Alice" });

    User? row = await dbContext.Users.SingleOrDefaultAsync(u => u.Name == "Alice");
    row.ShouldNotBeNull();
}

// GOOD: Verifies through interface
[Fact]
public async Task CreatedUserIsRetrievable()
{
    HttpResponseMessage created = await client.PostAsJsonAsync("users", new { name = "Alice" });
    Guid id = await created.Content.ReadFromJsonAsync<Guid>();

    UserResponse? retrieved = await client.GetFromJsonAsync<UserResponse>($"users/{id}");
    retrieved!.Name.ShouldBe("Alice");
}
```

In an ASP.NET Core app the HTTP API is usually the seam: `WebApplicationFactory<Program>` hosts it in-process, and Testcontainers supplies the real database behind it.

**Tautological tests**: Expected value restates the implementation, so the test passes by construction.

```csharp
// BAD: Expected value is recomputed the way the code computes it
[Fact]
public void CalculateTotalSumsLineItems()
{
    LineItem[] items = [new(Price: 10m), new(Price: 5m)];
    decimal expected = items.Sum(i => i.Price);
    Pricing.CalculateTotal(items).ShouldBe(expected);
}

// GOOD: Expected value is an independent, known literal
[Fact]
public void CalculateTotalSumsLineItems()
{
    Pricing.CalculateTotal([new(Price: 10m), new(Price: 5m)]).ShouldBe(15m);
}
```

## Tests that pass for the wrong reason

A test that passes while asserting nothing looks identical to a real one in every report. Traps that recur in ASP.NET Core suites:

- **The shared host neutralises the thing under test.** A shared `WebApplicationFactory` relaxes a limit (rate limiting, a quota) so ordinary tests run, and the test of that limit inherits the relaxed host. A test of a limit sets the limit itself (`factory.WithWebHostBuilder(b => b.UseSetting(...))`).
- **The request is rejected earlier than you think.** A test expects 401 for an expired token and gets 401 because the hand-made token's signature was invalid. Derive keys, hashes and ids exactly as production does, or obtain them through the real flow.
- **The fixture is malformed.** A fixture invalid in two ways fails for the wrong one. Make it valid in every respect except the one under test.
- **The seam under test isn't the seam exercised.** Integration tests that call `Database.MigrateAsync()` replay migration files, so editing an EF Core configuration without a migration changes nothing. Break the artifact the test actually loads, and use `MigrateAsync` rather than `EnsureCreated` so the migrations themselves are verified.

The red in red before green catches all four: break the one line the test defends and watch it fail for that reason.
