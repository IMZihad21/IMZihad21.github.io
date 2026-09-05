# .NET 9 dropped Swashbuckle — but you can still keep Swagger UI backed by OpenAPI

- Canonical URL: https://imzihad21.github.io/articles/a/net-9-dropped-swashbuckle-but-you-can-still-keep-swagger-ui-backed-by-openapi-50j3/
- Source URL: https://dev.to/imzihad21/net-9-dropped-swashbuckle-but-you-can-still-keep-swagger-ui-backed-by-openapi-50j3
- Web View: https://imzihad21.github.io/articles/a/net-9-dropped-swashbuckle-but-you-can-still-keep-swagger-ui-backed-by-openapi-50j3/
- Published: 2025-10-24T03:23:25.000Z
- Modified: 2025-10-24T03:23:25.000Z
- Reading time: 3 minutes
- Tags: dotnet, openapi, swagger, authentication

## .NET 9 Dropped Swashbuckle Generator, but You Can Still Keep Swagger UI Backed by OpenAPI

In .NET 9, the default approach moved toward built-in OpenAPI generation. That does not mean interactive API docs are gone. You can still use Swagger UI as a frontend while keeping document generation native.

This setup uses `AddOpenApi` + transformers for schema/security behavior, then serves docs in Swagger UI.

### Why It Matters

- Uses native OpenAPI generation in ASP.NET Core.
- Keeps Swagger UI developer experience.
- Avoids legacy generator-specific plumbing.
- Supports JWT auth and problem-details response shaping.

### Core Concepts

#### 1. Built-In OpenAPI Document Registration

Register named OpenAPI document and plug in transformers.

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

#### 2. OpenAPI + Swagger UI Pipeline

Map OpenAPI endpoint and configure Swagger UI against `/openapi/v1.json`.

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

#### 3. JWT Security Scheme Transformer

Add JWT bearer scheme and default security requirement in document.

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

#### 4. Operation Transformer for Problem Details

Add a reusable 4xx/5xx response contract.

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

#### 5. Extension Class (Full)

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

#### 6. Program Wiring

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

### Practical Example

With this setup running:

- OpenAPI JSON: `/openapi/v1.json`
- Swagger UI: `/swagger`

JWT auth appears in Swagger UI authorize dialog, and problem-details response schema is injected consistently.

### Common Mistakes

- Keeping old `AddSwaggerGen` assumptions in .NET 9-native OpenAPI setup.
- Forgetting `app.MapOpenApi()` and only configuring Swagger UI.
- Adding JWT scheme in auth middleware but not in OpenAPI document.
- Inconsistent problem-details schema across endpoints.
- Mixing incompatible package versions.

### Quick Recap

- Generate docs with built-in OpenAPI.
- Display docs with Swagger UI package.
- Use document transformer for JWT security contract.
- Use operation transformer for standard error contract.
- Keep startup wiring minimal and explicit.

### Next Steps

1. Add multiple named OpenAPI docs for versioned APIs.
2. Add operation tags/grouping conventions for large APIs.
3. Add CI check to validate generated OpenAPI spec.
4. Add environment-based toggle for Swagger UI exposure.