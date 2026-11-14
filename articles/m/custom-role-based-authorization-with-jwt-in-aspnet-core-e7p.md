# Custom Role-Based Authorization with JWT in ASP.NET Core

- Canonical URL: https://imzihad21.github.io/articles/a/custom-role-based-authorization-with-jwt-in-aspnet-core-e7p/
- Source URL: https://dev.to/imzihad21/custom-role-based-authorization-with-jwt-in-aspnet-core-e7p
- Web View: https://imzihad21.github.io/articles/a/custom-role-based-authorization-with-jwt-in-aspnet-core-e7p/
- Published: 2024-11-03T16:39:58.000Z
- Modified: 2024-11-03T16:39:58.000Z
- Reading time: 6 minutes
- Tags: dotnet, aspnetcore, jwt, authorization

## Custom role-based authorization with JWT in ASP.NET Core

When an API grows, scattered authorization checks become hard to trust and hard to audit. In distributed or monolithic architectures, hardcoding permissions into individual controller actions creates brittle security boundaries, leads to permission drift, and requires code redeployments whenever role policies change.

Combining JWT authentication with a centralized route-aware authorization handler enforces fail-closed access control backed by dynamic database-driven permissions without requiring redeployment.

### The problem and production context

In many enterprise APIs, authorization logic begins as simple role attributes on controllers. As routes multiply and permission granularity increases, access rules become dispersed across dozens of files.

- **Failure scenario**: A newly introduced administrative endpoint lacks explicit role checks or relies on flawed claim extraction. An authenticated user possessing a low-privilege JWT accesses the route, triggering unauthorized state mutations or exfiltrating tenant data. Concurrently, unhandled exceptions during claim parsing allow execution to proceed or result in nondescript server errors that mask security violations.
- **Why default approaches fall short**: Built-in attribute-based checks such as `[Authorize(Roles = "Admin")]` couple route permissions directly to compiled source code. When business administrators update permissions dynamically in a database, compiled role attributes cannot adapt without recompiling and redeploying the entire service.
- **Production impact**: Hardcoded authorization models increase maintenance friction, elevate the risk of accidental privilege escalation, and create operational bottlenecks when enterprise role policies change frequently.

### Mental model and core concepts

Centralizing authorization through ASP.NET Core policies requires understanding requirement markers, claim extraction, route normalization, and fail-closed evaluation.

#### 1. Custom authorization requirement

An authorization requirement acts as a marker or rule contract evaluated by ASP.NET Core policy handlers:

```csharp
public sealed class RoutePermissionRequirement : IAuthorizationRequirement
{
}
```

The requirement plugs into the policy pipeline and triggers the associated authorization handler whenever an incoming HTTP request is evaluated.

#### 2. Public endpoint allowlist

Certain routes (such as authentication handshakes or public key exchange endpoints) must bypass authorization evaluation. Isolating public routes in a centralized allowlist prevents accidental public exposure:

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

#### 3. Route normalization and claim extraction

During request handling, the authorization pipeline extracts the route path and JWT claims from the current `HttpContext`. Route segments are normalized (such as extracting controller and action path components) to construct deterministic permission lookup keys. The user's role identifier is extracted from the `ClaimTypes.Role` claim and parsed safely. If parsing fails, the pipeline fails closed immediately.

#### 4. Database-driven permission verification

Instead of evaluating static enum values, the handler queries a dynamic permission store (such as a database repository or distributed cache) to verify whether the extracted role ID holds granted permissions for the normalized route.

#### 5. Policy pipeline registration

Registering the custom authorization handler and attaching the requirement to the default authorization policy guarantees that every authenticated endpoint enforces route permissions automatically.

#### 6. Security behavior and operational boundaries

This model guarantees deterministic security outcomes:
- Public endpoints are explicitly isolated.
- The role identifier is extracted from authenticated JWT claims.
- Permissions are validated dynamically against a backing data store.
- Missing claims, empty route segments, or parsing errors result in explicit authorization failure (`context.Fail()`).
- Unexpected exceptions are caught, logged with structured diagnostics, and terminated with explicit failure.

Operational considerations include:
- Keeping the claim schema stable across internal and external microservices.
- Keeping route normalization aligned with API routing conventions.
- Migrating from substring matching to exact route pattern matching as routing complexity increases.
- Implementing structured audit logs for all denied access attempts.
- Adding bounded caching to mitigate database query overhead on high-throughput routes.

### Production implementation

Implement the requirement, authorization handler, service registration extensions, and application pipeline setup.

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;

public interface IPermissionRepository
{
    Task<bool> CheckAccessAsync(long roleId, string normalizedRoute, CancellationToken cancellationToken);
}

public sealed class RoutePermissionRequirement : IAuthorizationRequirement
{
}

public static class PublicApiEndpoints
{
    public static readonly HashSet<string> Routes =
        new(StringComparer.OrdinalIgnoreCase)
        {
            "Authenticate",
            "FetchUserDataByKey"
        };
}

public sealed class RoutePermissionAuthorizationHandler
    : AuthorizationHandler<RoutePermissionRequirement>
{
    private readonly IPermissionRepository _permissionRepository;
    private readonly ILogger<RoutePermissionAuthorizationHandler> _logger;

    public RoutePermissionAuthorizationHandler(
        IPermissionRepository permissionRepository,
        ILogger<RoutePermissionAuthorizationHandler> logger)
    {
        _permissionRepository = permissionRepository;
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

            string? requestPath = httpContext.Request.Path.Value;

            if (IsPublicEndpoint(requestPath))
            {
                context.Succeed(requirement);
                return;
            }

            if (!TryExtractPermissionContext(httpContext, out string normalizedRoute, out long roleId))
            {
                context.Fail();
                return;
            }

            bool isAuthorized = await _permissionRepository
                .CheckAccessAsync(roleId, normalizedRoute, httpContext.RequestAborted);

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

        string? requestPath = httpContext.Request.Path.Value;

        if (string.IsNullOrWhiteSpace(requestPath))
        {
            return false;
        }

        string? roleClaimValue = httpContext.User.FindFirstValue(ClaimTypes.Role);

        if (!long.TryParse(roleClaimValue, out roleId))
        {
            return false;
        }

        string[] routeSegments = requestPath.Split('/', StringSplitOptions.RemoveEmptyEntries);

        if (routeSegments.Length < 2)
        {
            return false;
        }

        normalizedRoute = routeSegments.Length >= 3
            ? $"{routeSegments[1]}/{routeSegments[2]}"
            : $"{routeSegments[0]}/{routeSegments[1]}";

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

public static class AuthorizationConfigurationExtensions
{
    public static IServiceCollection AddRoutePermissionAuthorization(
        this IServiceCollection services)
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

Configure the dependency injection container and middleware pipeline in application startup:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();

builder.Services.AddRoutePermissionAuthorization();
builder.Services.AddControllers();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### Architectural trade-offs and edge cases

Adopting dynamic database-driven authorization shifts security management from source code to runtime infrastructure.

* **Latency versus consistency**: Querying database permissions on every request provides immediate revocation and updates when permissions change, but adds database roundtrips to API latency. To balance this, high-throughput systems introduce in-memory caching with bounded time-to-live (TTL) or cache invalidation events triggered on permission mutation.
* **Failure recovery**: The handler implements strict fail-closed architecture. If database connectivity is interrupted or token parsing fails, requests are explicitly denied (`context.Fail()`) rather than defaulting to permissive access.
* **Scale limitations**: When APIs expand to hundreds of microservices, centralizing authorization via direct database queries can bottleneck the database. Systems operating at high scale transition to distributed permission caches (such as Redis) or compiled authorization sidecars (such as Open Policy Agent).
* **Route normalization ambiguity**: Substring matching in endpoint allowlists risks false positives where a restricted route (such as `/api/admin/AuthenticateUser`) unintentionally bypasses authorization because it matches the substring `Authenticate`. Production systems require strict path matching or endpoint metadata attributes.

### Common anti-patterns and gotchas

* **Unvalidated claim parsing**: Developers parse role claims using direct string indexing without verifying claim presence or type validity. When tokens omit the claim, the handler throws unhandled exceptions. Always use `TryParse` patterns and fail closed.
* **Hardcoding permission logic in controllers**: Developers scatter `User.IsInRole()` checks across individual controller actions. Access rules become impossible to audit holistically and drift across versions. Centralize all checks in dedicated authorization handlers.
* **Unrestricted allowlist expansion**: Developers allow public route allowlists to grow without architectural review, accidentally exposing internal endpoints to anonymous access.
* **Ignoring route versioning in permission keys**: Changing an API route from `/api/v1/users` to `/api/v2/users` breaks permission matching if normalized route keys do not account for version prefixes.
* **Treating authorization rejections as exceptions**: Developers throw application exceptions to stop unauthorized access, causing 500 Internal Server Errors instead of returning clean 403 Forbidden responses. Use `context.Fail()` to let ASP.NET Core generate appropriate HTTP status codes.

### Implementation checklist

1. Create the `RoutePermissionRequirement` class implementing `IAuthorizationRequirement`.
2. Define a centralized `PublicApiEndpoints` registry for routes bypassing permission checks.
3. Implement `RoutePermissionAuthorizationHandler` with fail-closed claim extraction and database verification.
4. Register the authorization handler and set the default authorization policy using `SetDefaultPolicy`.
5. Ensure `app.UseAuthentication()` precedes `app.UseAuthorization()` in the middleware pipeline.
6. Add bounded caching with TTL for permission lookups to protect backing databases from request amplification.
7. Replace substring matching with exact route pattern matching to prevent false-positive public endpoint bypasses.
8. Implement structured audit logging for all authorization failures and denied access attempts.
9. Write integration tests validating public route bypass, authorized access, and 403 Forbidden enforcement on unauthorized calls.