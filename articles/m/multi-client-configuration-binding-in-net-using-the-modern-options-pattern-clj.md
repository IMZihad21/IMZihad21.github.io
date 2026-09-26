# Named Configuration Binding in .NET Using the Modern Options Pattern

- Canonical URL: https://imzihad21.github.io/articles/a/multi-client-configuration-binding-in-net-using-the-modern-options-pattern-clj/
- Source URL: https://dev.to/imzihad21/multi-client-configuration-binding-in-net-using-the-modern-options-pattern-clj
- Web View: https://imzihad21.github.io/articles/a/multi-client-configuration-binding-in-net-using-the-modern-options-pattern-clj/
- Published: 2025-11-29T18:51:53.000Z
- Modified: 2025-11-29T18:51:53.000Z
- Reading time: 4 minutes
- Tags: dotnet, configuration, csharp, aspnetcore

## Named configuration binding in .NET using the modern options pattern

When multi-tenant or multi-client applications rely on fixed tenant identities defined in code, managing heterogeneous endpoints, credentials, and regional settings through raw dictionaries or string lookups introduces configuration drift and runtime failures.

Named options provide a structured approach to binding per-tenant settings. This pattern separates shared global infrastructure settings from tenant-specific configurations while using native dependency injection, compile-time enum keys, and startup validation.

### The problem and production context

Applications servicing multiple known clients or tenants frequently maintain separate credentials, endpoints, and regional parameters. Developers often implement configuration access by querying `IConfiguration` directly across services using raw string paths or untyped dictionary representations.

- **Failure scenario**: A developer adds a new tenant key in `appsettings.json` with a typographical error in the connection string key name. The application starts without warnings. Later, an inbound user request targets the client, triggering a null reference exception deep within the persistence layer during production traffic.
- **Why default approaches fall short**: Direct `IConfiguration` injection provides no compile-time verification, no automatic schema validation, and no structured lifecycle integration. Standard `IOptions<T>` registrations only map a single instance per type, forcing teams to create separate wrapper classes for each tenant or resort to unvalidated nested dictionary mappings.
- **Production impact**: Missing or invalid tenant configurations escape into runtime execution, causing partial outages, untyped runtime errors, cross-tenant configuration contamination, and tedious debugging across distributed microservices.

Using strongly typed named options resolves these problems by:
- Keeping per-tenant configuration strongly typed, validated, and predictable.
- Using built-in Microsoft dependency injection and options abstractions.
- Eliminating manual string-based configuration parsing across services.
- Maintaining clarity as new clients are added to the fixed tenant registry.

### Mental model and core concepts

#### 1. Hierarchical configuration structure

Organize settings into distinct top-level sections for global configuration and tenant-specific entries in `appsettings.json`.

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

#### 2. Fixed tenant enumeration

Define tenant identifiers as enum members to ensure compile-time safety and consistent naming across configuration files.

```csharp
public enum Client
{
    ClientA,
    ClientB
}
```

#### 3. Strongly typed options models

Create dedicated, immutable options classes for global settings and per-tenant settings.

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

#### 4. Global options registration and validation

Bind and validate the global options section during application startup using startup validation.

```csharp
builder.Services
    .AddOptions<GlobalSettings>()
    .Bind(builder.Configuration.GetSection("GlobalSettings"))
    .Validate(settings => !string.IsNullOrWhiteSpace(settings.ApiEndpoint), "ApiEndpoint is required")
    .ValidateOnStart();
```

#### 5. Named options per tenant iteration

Iterate through the tenant enum members to bind and validate individual named option instances for each tenant section.

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

#### 6. Tenant options resolution

Inject `IOptionsSnapshot<T>` and retrieve tenant-specific configurations via `Get(name)` during request execution.

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

### Production implementation

The following host initialization binds global options, registers named options for all configured tenants, and verifies validity at host startup.

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

This pattern keeps tenant configuration deterministic, explicit, and integrated with the host lifecycle.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: `IOptionsSnapshot<T>` recomputes option instances once per request scope, enabling immediate reloading when configuration providers reload without restarting the process. This introduces a slight per-request object allocation overhead compared to singleton `IOptions<T>`.
* **Failure recovery**: Using `ValidateOnStart()` causes configuration errors to fail fast during the host startup phase. If a tenant entry is missing or fails validation predicates, the container aborts before opening HTTP listener sockets, preventing misconfigured pods from accepting traffic.
* **Scale limitations**: The enum-driven named options pattern suits fixed, known tenant registries (such as dedicated partner integrations or regional deployments). It is not suited for multi-tenant architectures requiring dynamic, unbounded tenant registration at runtime without application redeployment.

### Common anti-patterns and gotchas

* **Using arbitrary string literals for tenant names**: Hardcoding string values instead of strongly typed enum identifiers introduces typo-induced configuration lookup mismatches. Always drive registration and retrieval via compile-time constants or enums.
* **Skipping startup validation**: Omitting `ValidateOnStart()` allows misconfigured tenant sections to start cleanly, deferring missing configuration errors until production requests hit the specific tenant code path.
* **Merging global settings and client-specific settings**: Combining global infrastructure flags and tenant-specific connection details into a single monolithic options model breaks single-responsibility principles and complicates tenant isolation.
* **Injecting IConfiguration directly into business logic**: Reading raw keys directly from `IConfiguration` bypasses typed validation, hides dependencies, and tightly couples domain services to configuration schemas.
* **Omitting the named options parameter on resolution**: Calling `_clientSettings.Value` instead of `_clientSettings.Get(name)` returns the default un-named instance, which will contain uninitialized default values.

### Implementation checklist

1. Separate configuration sections into `GlobalSettings` and client-specific `Clients` objects.
2. Define a fixed enum containing all supported client or tenant identifiers.
3. Create immutable, strongly typed POCO classes for global and per-tenant settings.
4. Register global options with `.Bind()` and `.ValidateOnStart()`.
5. Iterate enum members to register named client options bound to their respective configuration sections.
6. Inject `IOptionsSnapshot<ClientSettings>` into client consumers and access options using `.Get(client.ToString())`.
7. Integrate external vault providers to load sensitive per-tenant connection strings.
8. Implement health checks verifying connectivity for all configured tenants at startup.
9. Write unit and integration tests confirming proper option binding for every registered enum value.
10. Establish a migration strategy if tenant definitions need to become dynamic in the future.