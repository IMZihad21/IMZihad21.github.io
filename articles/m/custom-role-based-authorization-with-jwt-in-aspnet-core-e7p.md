# Custom Role-Based Authorization with JWT in ASP.NET Core

- Canonical URL: https://imzihad21.github.io/articles/a/custom-role-based-authorization-with-jwt-in-aspnet-core-e7p/
- Source URL: https://dev.to/imzihad21/custom-role-based-authorization-with-jwt-in-aspnet-core-e7p
- Web View: https://imzihad21.github.io/articles/a/custom-role-based-authorization-with-jwt-in-aspnet-core-e7p/
- Published: 2024-11-03T16:39:58.000Z
- Modified: 2024-11-03T16:39:58.000Z
- Reading time: 3 minutes
- Tags: dotnet, aspnetcore, jwt, authorization

When an API grows, scattered authorization checks become hard to trust and hard to audit. A centralized route-aware authorization layer keeps rules consistent and easier to maintain.

This pattern combines JWT authentication with dynamic role-based access control (RBAC) using a custom authorization handler.

### Why It Matters

- Keeps permission logic in one place instead of many controllers.
- Supports database-driven access rules without redeploying code.
- Enforces fail-closed behavior for missing claims or parsing issues.
- Improves traceability and auditing for protected endpoints.

### Core Concepts

#### 1. Custom Authorization Requirement

Create a marker requirement that plugs into the policy pipeline.

```csharp
public sealed class RoutePermissionRequirement : IAuthorizationRequirement
{
}
```

#### 2. Public Endpoint Allowlist

Centralize endpoints that bypass permission checks.

```csharp
public static class PublicApiEndpoints
{
    public static readonly HashSet<string> Routes =
        new(StringComparer.OrdinalIgnoreCase)
        {
            "Authenticate",
            "FetchUserDataByKey"
        };
}
```

#### 3. Core Authorization Handler

Resolve the route, extract the role ID from claims, and check permission store.

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;

public sealed class RoutePermissionAuthorizationHandler
    : AuthorizationHandler<RoutePermissionRequirement>
{
    private readonly IDataRepository _dataRepository;
    private readonly ILogger<RoutePermissionAuthorizationHandler> _logger;

    public RoutePermissionAuthorizationHandler(
        IDataRepository dataRepository,
        ILogger<RoutePermissionAuthorizationHandler> logger)
    {
        _dataRepository = dataRepository;
        _logger = logger;
    }

    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        RoutePermissionRequirement requirement)
    {
        try
        {
            if (context.Resource is not HttpContext httpContext)
            {
                context.Fail();
                return;
            }

            var requestPath = httpContext.Request.Path.Value;

            if (IsPublicEndpoint(requestPath))
            {
                context.Succeed(requirement);
                return;
            }

            if (!TryExtractPermissionContext(httpContext, out var normalizedRoute, out var roleId))
            {
                context.Fail();
                return;
            }

            var isAuthorized = await _dataRepository.Permissions
                .CheckAccess(roleId, normalizedRoute);

            if (!isAuthorized)
            {
                context.Fail();
                return;
            }

            context.Succeed(requirement);
        }
        catch (Exception exception)
        {
            _logger.LogError(exception, "Route permission authorization failed unexpectedly.");
            context.Fail();
        }
    }

    private static bool TryExtractPermissionContext(
        HttpContext httpContext,
        out string normalizedRoute,
        out long roleId)
    {
        normalizedRoute = string.Empty;
        roleId = 0;

        var requestPath = httpContext.Request.Path.Value;

        if (string.IsNullOrWhiteSpace(requestPath))
        {
            return false;
        }

        var roleClaimValue = httpContext.User.FindFirstValue(ClaimTypes.Role);

        if (!long.TryParse(roleClaimValue, out roleId))
        {
            return false;
        }

        var routeSegments = requestPath.Split('/', StringSplitOptions.RemoveEmptyEntries);

        if (routeSegments.Length < 3)
        {
            return false;
        }

        normalizedRoute = $"{routeSegments[1]}/{routeSegments[2]}";
        return !string.IsNullOrWhiteSpace(normalizedRoute);
    }

    private static bool IsPublicEndpoint(string? requestPath)
    {
        if (string.IsNullOrWhiteSpace(requestPath))
        {
            return true;
        }

        return PublicApiEndpoints.Routes.Any(route =>
            requestPath.Contains(route, StringComparison.OrdinalIgnoreCase));
    }
}
```

#### 4. Policy Registration

Register the handler and apply the requirement in the default authorization policy.

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;

public static class AuthorizationConfigurationExtensions
{
    public static IServiceCollection AddRoutePermissionAuthorization(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddScoped<IAuthorizationHandler, RoutePermissionAuthorizationHandler>();

        services.AddAuthorizationBuilder()
            .SetDefaultPolicy(new AuthorizationPolicyBuilder()
                .AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
                .RequireAuthenticatedUser()
                .AddRequirements(new RoutePermissionRequirement())
                .Build());

        return services;
    }
}
```

#### 5. Security Behavior

This model enforces route-sensitive authorization with strict outcomes:

- Public endpoints are explicitly isolated.
- Role ID is extracted from authenticated JWT claims.
- Permissions are validated against a data source.
- Missing claims or route parsing failures deny access.
- Unexpected exceptions are logged and denied.

#### 6. Operational Considerations

- Keep claim schema stable across services.
- Keep route normalization aligned with routing conventions.
- Consider exact route matching when substring checks become risky.
- Add structured audit logs for denied access events.
- Add permission caching if database load grows.

### Practical Example

Use the extension during service registration:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();

builder.Services.AddRoutePermissionAuthorization(builder.Configuration);

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

At this point, your controllers can focus on business logic while the authorization pipeline does its job without emotional support.

### Common Mistakes

- Parsing role claims without validating token claim format.
- Hardcoding permission checks inside controllers.
- Letting public endpoints grow without centralized review.
- Ignoring route versioning impact on normalized permission keys.
- Treating authorization denials as exceptions instead of expected outcomes.

### Quick Recap

- A custom requirement + handler centralizes RBAC decisions.
- JWT claims provide identity context for permission checks.
- Database-backed permissions keep access rules dynamic.
- Default policy wiring enforces authorization consistently.
- Fail-closed behavior protects against missing or malformed inputs.

### Next Steps

1. Add exact route-pattern matching to reduce false positives.
2. Add cached permission lookups with bounded TTL.
3. Add structured audit events for authorization denials.
4. Add integration tests for public route bypass and forbidden route access.