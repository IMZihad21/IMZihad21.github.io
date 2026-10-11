# .NET 9 dropped Swashbuckle — but you can still keep Swagger UI backed by OpenAPI

- Canonical URL: https://imzihad21.github.io/articles/a/net-9-dropped-swashbuckle-but-you-can-still-keep-swagger-ui-backed-by-openapi-50j3/
- Source URL: https://dev.to/imzihad21/net-9-dropped-swashbuckle-but-you-can-still-keep-swagger-ui-backed-by-openapi-50j3
- Web View: https://imzihad21.github.io/articles/a/net-9-dropped-swashbuckle-but-you-can-still-keep-swagger-ui-backed-by-openapi-50j3/
- Published: 2025-10-24T03:23:25.000Z
- Modified: 2025-10-24T03:23:25.000Z
- Reading time: 6 minutes
- Tags: dotnet, openapi, swagger, authentication

## .NET 9 dropped Swashbuckle generator, but you can still keep Swagger UI backed by OpenAPI

In .NET 9, ASP.NET Core removed the default dependency on Swashbuckle.AspNetCore in favor of built-in, native OpenAPI document generation. Upgrading applications without adapting schema generation leads to broken documentation endpoints, missing security definitions, and deprecated package conflicts.

Developers can retain the familiar Swagger UI frontend while delegating specification generation entirely to the native `Microsoft.AspNetCore.OpenApi` engine. Combining native OpenAPI registration with custom document and operation transformers produces interactive API documentation supporting JWT Bearer authentication and RFC 7807 Problem Details schemas without obsolete third-party generators.

### The problem and production context

Prior to .NET 9, ASP.NET Core templates relied on Swashbuckle for OpenAPI document generation and UI hosting. As Swashbuckle experienced slower maintenance cycles and architectural drift from official OpenAPI specifications, Microsoft built first-party OpenAPI generation directly into the framework.

- **Failure scenario**: Upgrading an existing ASP.NET Core Web API to .NET 9 while retaining legacy `AddSwaggerGen` registrations creates package dependency friction, deprecated API warnings, and schema generation discrepancies with modern System.Text.Json polymorphs. Removing Swashbuckle entirely removes interactive UI documentation, leaving consumers without an in-browser request sandbox.
- **Why default approaches fall short**: Removing Swashbuckle generator packages (`Swashbuckle.AspNetCore.SwaggerGen`) leaves teams without an interactive UI if they do not decouple UI rendering from schema generation. Conversely, relying solely on `app.MapOpenApi()` serves raw JSON without an exploratory browser console.
- **Production impact**: API consumers lose interactive sandbox exploration, developers struggle to verify authorization headers against secured endpoints, and client code generators receive non-standardized error schemas across divergent controllers.

Adopting native OpenAPI document generation paired with standalone Swagger UI addresses these challenges:
- Uses native OpenAPI document generation in ASP.NET Core.
- Retains the familiar Swagger UI developer experience.
- Avoids third-party generator plumbing and obsolete dependencies.
- Handles JWT authentication and problem details response contracts cleanly.

### Mental model and core concepts

#### 1. Built-in OpenAPI document registration

Register a named OpenAPI document and attach transformers for custom configuration:

```csharp
public static IServiceCollection AddOpenApiDocumentation(this IServiceCollection services)
{
    services.AddOpenApi("v1", options =>
    {
        options.AddDocumentTransformer<JwtBearerSecurityDocumentTransformer>();
        options.AddOperationTransformer<ProblemDetailsOperationTransformer>();
    });

    return services;
}
```

#### 2. OpenAPI and Swagger UI pipeline

Map the OpenAPI JSON endpoint and configure Swagger UI to load `/openapi/v1.json`:

```csharp
public static void UseOpenApiDocumentation(this WebApplication app)
{
    app.MapOpenApi();

    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/openapi/v1.json", "v1");
        options.EnablePersistAuthorization();
        options.DisplayRequestDuration();
        options.EnableTryItOutByDefault();
        options.EnableFilter();
        options.DocExpansion(DocExpansion.List);
        options.DefaultModelsExpandDepth(0);
    });
}
```

#### 3. JWT security scheme transformer

Add the JWT Bearer scheme and a default security requirement to the OpenAPI document:

```csharp
public sealed class JwtBearerSecurityDocumentTransformer : IOpenApiDocumentTransformer
{
    private const string SecuritySchemeId = JwtBearerDefaults.AuthenticationScheme;

    public Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
    {
        document.Info = new OpenApiInfo
        {
            Title = "DevInsightForge.API",
            Version = "v1",
            Description = "The DevInsightForge API built with ASP.NET Core, it ensures secure and efficient communication through JSON Web Tokens (JWT) for authentication."
        };

        document.Components ??= new OpenApiComponents();
        document.Components.SecuritySchemes ??= new Dictionary<string, IOpenApiSecurityScheme>();
        document.Components.SecuritySchemes[SecuritySchemeId] = new OpenApiSecurityScheme
        {
            BearerFormat = "JWT",
            Name = "JWT Authentication",
            In = ParameterLocation.Header,
            Type = SecuritySchemeType.Http,
            Scheme = JwtBearerDefaults.AuthenticationScheme,
            Description = "Put **_ONLY_** your JWT Bearer token on the textbox below!"
        };

        document.Security ??= [];
        document.Security.Add(new OpenApiSecurityRequirement
        {
            [new OpenApiSecuritySchemeReference(SecuritySchemeId, document, null)] = []
        });

        return Task.CompletedTask;
    }
}
```

#### 4. Operation transformer for problem details

Inject a reusable schema for 4xx and 5xx problem details responses across endpoints:

```csharp
public sealed class ProblemDetailsOperationTransformer : IOpenApiOperationTransformer
{
    public async Task TransformAsync(OpenApiOperation operation, OpenApiOperationTransformerContext context, CancellationToken cancellationToken)
    {
        operation.Responses ??= [];

        var errorResponseSchema = await context.GetOrCreateSchemaAsync(typeof(ErrorResponse), null, cancellationToken);

        operation.Responses["4xx/5xx"] = new OpenApiResponse
        {
            Description = typeof(ErrorResponse).Name,
            Content = new Dictionary<string, OpenApiMediaType>
            {
                ["application/problem+json"] = new OpenApiMediaType
                {
                    Schema = errorResponseSchema
                }
            }
        };

        return;
    }
}
```

### Production implementation

The following complete extension module encapsulates service registration, pipeline middleware, document-level security schemas, and standard error response operation transformers.

```csharp
using DevInsightForge.WebAPI.Contracts;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.OpenApi;
using Microsoft.OpenApi;
using Swashbuckle.AspNetCore.SwaggerUI;

namespace DevInsightForge.WebAPI.Extensions;

public static class OpenApiExtensions
{
    public static IServiceCollection AddOpenApiDocumentation(this IServiceCollection services)
    {
        services.AddOpenApi("v1", options =>
        {
            options.AddDocumentTransformer<JwtBearerSecurityDocumentTransformer>();
            options.AddOperationTransformer<ProblemDetailsOperationTransformer>();
        });

        return services;
    }

    public static void UseOpenApiDocumentation(this WebApplication app)
    {
        app.MapOpenApi();

        app.UseSwaggerUI(options =>
        {
            options.SwaggerEndpoint("/openapi/v1.json", "v1");
            options.EnablePersistAuthorization();
            options.DisplayRequestDuration();
            options.EnableTryItOutByDefault();
            options.EnableFilter();
            options.DocExpansion(DocExpansion.List);
            options.DefaultModelsExpandDepth(0);
        });
    }
}

public sealed class JwtBearerSecurityDocumentTransformer : IOpenApiDocumentTransformer
{
    private const string SecuritySchemeId = JwtBearerDefaults.AuthenticationScheme;

    public Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
    {
        document.Info = new OpenApiInfo
        {
            Title = "DevInsightForge.API",
            Version = "v1",
            Description = "The DevInsightForge API built with ASP.NET Core, it ensures secure and efficient communication through JSON Web Tokens (JWT) for authentication."
        };

        document.Components ??= new OpenApiComponents();
        document.Components.SecuritySchemes ??= new Dictionary<string, IOpenApiSecurityScheme>();
        document.Components.SecuritySchemes[SecuritySchemeId] = new OpenApiSecurityScheme
        {
            BearerFormat = "JWT",
            Name = "JWT Authentication",
            In = ParameterLocation.Header,
            Type = SecuritySchemeType.Http,
            Scheme = JwtBearerDefaults.AuthenticationScheme,
            Description = "Put **_ONLY_** your JWT Bearer token on the textbox below!"
        };

        document.Security ??= [];
        document.Security.Add(new OpenApiSecurityRequirement
        {
            [new OpenApiSecuritySchemeReference(SecuritySchemeId, document, null)] = []
        });

        return Task.CompletedTask;
    }
}

public sealed class ProblemDetailsOperationTransformer : IOpenApiOperationTransformer
{
    public async Task TransformAsync(OpenApiOperation operation, OpenApiOperationTransformerContext context, CancellationToken cancellationToken)
    {
        operation.Responses ??= [];

        var errorResponseSchema = await context.GetOrCreateSchemaAsync(typeof(ErrorResponse), null, cancellationToken);

        operation.Responses["4xx/5xx"] = new OpenApiResponse
        {
            Description = typeof(ErrorResponse).Name,
            Content = new Dictionary<string, OpenApiMediaType>
            {
                ["application/problem+json"] = new OpenApiMediaType
                {
                    Schema = errorResponseSchema
                }
            }
        };

        return;
    }
}
```

The application entry point registers authentication, applies the OpenAPI extensions, and mounts HTTP routes:

```csharp
builder.Services.AddControllers();
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecurityKey"]!))
        };
    });

builder.Services.AddOpenApiDocumentation();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseOpenApiDocumentation();
app.MapControllers();

app.Run();
```

Once the application starts, check these endpoints in the browser:
- OpenAPI JSON: `/openapi/v1.json`
- Swagger UI: `/swagger`

The authorize dialog in Swagger UI will accept a JWT bearer token, and responses will reflect the standardized problem details contract.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Native OpenAPI documents are generated dynamically on first request and cached by default in memory. Schema transformations execute during generation; extensive reflection or deep schema traversal in transformers adds small startup latency to the initial `/openapi/v1.json` load.
* **Failure recovery**: If a transformer throws an unhandled exception during document generation, requests to `/openapi/v1.json` return an HTTP 500 error, disabling Swagger UI without crashing the host process. Validation during build pipelines prevents broken transformers from reaching production.
* **Scale limitations**: Swagger UI is a client-side single-page application rendering raw OpenAPI schemas. For APIs with hundreds of endpoints and complex model graphs, rendering large JSON documents inside the browser can cause client memory spikes and UI lag. In massive architectures, split schemas across versioned documents.

### Common anti-patterns and gotchas

* **Leaving obsolete AddSwaggerGen configurations in place**: Retaining legacy Swashbuckle generator configurations alongside the native .NET 9 OpenAPI pipeline creates conflicting middleware and duplicate schema definitions. Remove `AddSwaggerGen` and retain only `Swashbuckle.AspNetCore.SwaggerUI`.
* **Calling UseSwaggerUI without mapping OpenApi endpoint**: Invoking `app.UseSwaggerUI()` without calling `app.MapOpenApi()` causes Swagger UI to fail with an HTTP 404 error when fetching the specification.
* **Registering authentication middleware without declaring OpenAPI security scheme**: Configuring JWT authentication in ASP.NET Core without adding a document transformer leaves Swagger UI without an Authorize button, preventing developers from testing protected endpoints.
* **Applying mismatched problem details schemas across controllers**: Using custom error response contracts without registering a global operation transformer creates inconsistent documentation schemas between endpoints.
* **Pulling in incompatible versions of Swashbuckle UI**: Using outdated versions of `Swashbuckle.AspNetCore.SwaggerUI` that depend on older Microsoft schema types can cause type resolution conflicts. Use the standalone UI package compatible with .NET 9.

### Implementation checklist

1. Remove `Swashbuckle.AspNetCore.SwaggerGen` from project dependencies.
2. Add `Microsoft.AspNetCore.OpenApi` and `Swashbuckle.AspNetCore.SwaggerUI` package references.
3. Register native OpenAPI services using `services.AddOpenApi("v1", options => ...)`.
4. Implement `JwtBearerSecurityDocumentTransformer` to define the security scheme and requirement.
5. Implement `ProblemDetailsOperationTransformer` to standardize 4xx and 5xx error responses.
6. Map OpenAPI endpoints using `app.MapOpenApi()` and configure `app.UseSwaggerUI()`.
7. Verify `/openapi/v1.json` and `/swagger` endpoints in the browser.
8. Add separate named OpenAPI documents for versioned API endpoints.
9. Group operations using tags to organize larger controller surfaces.
10. Validate the generated OpenAPI specification during CI builds.
11. Restrict Swagger UI access based on hosting environments.