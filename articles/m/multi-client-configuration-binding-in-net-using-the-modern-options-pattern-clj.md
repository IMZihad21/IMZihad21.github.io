# Named Configuration Binding in .NET Using the Modern Options Pattern

- Canonical URL: https://imzihad21.github.io/articles/a/multi-client-configuration-binding-in-net-using-the-modern-options-pattern-clj/
- Source URL: https://dev.to/imzihad21/multi-client-configuration-binding-in-net-using-the-modern-options-pattern-clj
- Web View: https://imzihad21.github.io/articles/a/multi-client-configuration-binding-in-net-using-the-modern-options-pattern-clj/
- Published: 2025-11-29T18:51:53.000Z
- Modified: 2025-11-29T18:51:53.000Z
- Reading time: 2 minutes
- Tags: dotnet, configuration, csharp, aspnetcore

When tenant/client identities are fixed in code, named options are a clean way to bind per-tenant settings from configuration. This avoids custom config loaders and keeps everything in the built-in .NET options system.

This guide shows a practical global + tenant configuration structure using modern options registration.

### Why It Matters

- Keeps tenant configuration strongly typed and predictable.
- Uses built-in dependency injection and options features.
- Avoids manual dictionary parsing logic.
- Scales cleanly when fixed tenant list grows.

### Core Concepts

#### 1. Configuration Shape

Use separate sections for global settings and tenant-specific settings.

```json
{
  "GlobalSettings": {
    "FeatureXEnabled": true,
    "ApiEndpoint": "https://example.com"
  },
  "Clients": {
    "ClientA": {
      "ConnectionString": "Server=A",
      "Region": "EU"
    },
    "ClientB": {
      "ConnectionString": "Server=B",
      "Region": "US"
    }
  }
}
```

#### 2. Fixed Tenant Identifiers

Represent tenants as enum values for stable names.

```csharp
public enum Client
{
    ClientA,
    ClientB
}
```

#### 3. Strongly Typed Options Models

```csharp
public sealed class GlobalSettings
{
    public bool FeatureXEnabled { get; init; }
    public string ApiEndpoint { get; init; } = string.Empty;
}

public sealed class ClientSettings
{
    public string ConnectionString { get; init; } = string.Empty;
    public string Region { get; init; } = string.Empty;
}
```

#### 4. Global Options Registration

Bind and validate global section once.

```csharp
builder.Services
    .AddOptions<GlobalSettings>()
    .Bind(builder.Configuration.GetSection("GlobalSettings"))
    .Validate(settings => !string.IsNullOrWhiteSpace(settings.ApiEndpoint), "ApiEndpoint is required")
    .ValidateOnStart();
```

#### 5. Named Options per Tenant

Bind one named options instance per enum value.

```csharp
foreach (Client client in Enum.GetValues<Client>())
{
    var clientName = client.ToString();

    builder.Services
        .AddOptions<ClientSettings>(clientName)
        .Bind(builder.Configuration.GetSection($"Clients:{clientName}"))
        .Validate(settings => !string.IsNullOrWhiteSpace(settings.ConnectionString), $"{clientName} ConnectionString is required")
        .Validate(settings => !string.IsNullOrWhiteSpace(settings.Region), $"{clientName} Region is required")
        .ValidateOnStart();
}
```

#### 6. Tenant Options Consumption

Resolve tenant settings through `IOptionsSnapshot<T>.Get(name)`.

```csharp
public sealed class ClientConsumer
{
    private readonly IOptionsSnapshot<ClientSettings> _clientSettings;

    public ClientConsumer(IOptionsSnapshot<ClientSettings> clientSettings)
    {
        _clientSettings = clientSettings;
    }

    public ClientSettings Get(Client client)
    {
        return _clientSettings.Get(client.ToString());
    }
}
```

### Practical Example

Minimal startup wiring:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddOptions<GlobalSettings>()
    .Bind(builder.Configuration.GetSection("GlobalSettings"))
    .ValidateOnStart();

foreach (Client client in Enum.GetValues<Client>())
{
    var name = client.ToString();
    builder.Services
        .AddOptions<ClientSettings>(name)
        .Bind(builder.Configuration.GetSection($"Clients:{name}"))
        .ValidateOnStart();
}

var app = builder.Build();
app.Run();
```

This keeps tenant config deterministic and explicit. No magic, no hidden fallback surprises.

### Common Mistakes

- Using free-form tenant names that drift from code identifiers.
- Skipping validation and discovering missing settings only at runtime.
- Mixing global and tenant settings in one model.
- Injecting `IConfiguration` everywhere instead of typed options.
- Forgetting named options lookup (`Get(name)`) for tenant config.

### Quick Recap

- Fixed tenant list maps naturally to named options.
- Global config and tenant config should be split clearly.
- Use `ValidateOnStart()` to fail fast on bad config.
- Keep consumption through typed options, not manual parsing.
- This pattern is clean, testable, and production-friendly.

### Next Steps

1. Add per-tenant secret loading from secure vault providers.
2. Add health checks that verify required tenant settings.
3. Add integration tests for each configured tenant binding.
4. Add migration path if tenants become dynamic in future.