# Managing Production Configurations in ASP.NET Core WebAPI Using Environment Variables

- Canonical URL: https://imzihad21.github.io/articles/a/managing-production-configurations-in-aspnet-core-webapi-using-environment-variables-3nmf/
- Source URL: https://dev.to/imzihad21/managing-production-configurations-in-aspnet-core-webapi-using-environment-variables-3nmf
- Web View: https://imzihad21.github.io/articles/a/managing-production-configurations-in-aspnet-core-webapi-using-environment-variables-3nmf/
- Published: 2025-01-25T03:30:01.000Z
- Modified: 2025-01-25T03:30:01.000Z
- Reading time: 2 minutes
- Tags: dotnet, docker, devops, env

## Managing Production Configurations in ASP.NET Core Web API Using Environment Variables

Production configuration should be dynamic, secure, and deployment-friendly. ASP.NET Core already gives a strong configuration pipeline that lets you override settings cleanly without changing code.

This guide explains how to use environment variables with `appsettings` files for real production workflows.

### Why It Matters

- Keeps secrets out of source code.
- Allows runtime config updates per environment.
- Works naturally with Docker, Kubernetes, and cloud platforms.
- Reduces risky manual edits in production.

### Core Concepts

#### 1. ASP.NET Core Configuration Sources

ASP.NET Core merges config from multiple providers.

Typical order (low to high priority):

- Default values in code
- `appsettings.json`
- `appsettings.{Environment}.json`
- Environment variables
- Command-line arguments

#### 2. Environment-Specific appsettings Files

`ASPNETCORE_ENVIRONMENT` controls environment-specific file loading.

If:

```text
ASPNETCORE_ENVIRONMENT=Production
```

Then `appsettings.Production.json` is loaded automatically and merged.

#### 3. Environment Variable Override Pattern

Use `__` (double underscore) for nested keys.

- JSON key: `AppSettings:ApiUrl`
- Env var: `AppSettings__ApiUrl`

#### 4. Example Configuration Merge

Base config:

```json
{
  "AppSettings": {
    "ApiUrl": "https://api.default.com",
    "ApiKey": "default-api-key"
  }
}
```

Production override:

```json
{
  "AppSettings": {
    "ApiUrl": "https://api.production.com"
  }
}
```

Environment variables:

```bash
AppSettings__ApiUrl=https://api.override.com
AppSettings__ApiKey=override-api-key
```

Final result at runtime:

- `ApiUrl`: `https://api.override.com`
- `ApiKey`: `override-api-key`

#### 5. Strongly Typed Options

Bind config sections into typed options.

```csharp
public sealed class AppSettingsOptions
{
    public string ApiUrl { get; set; } = string.Empty;
    public string ApiKey { get; set; } = string.Empty;
}
```

```csharp
builder.Services
    .AddOptions<AppSettingsOptions>()
    .Bind(builder.Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

#### 6. Secure Secret Management

Use platform secret stores for sensitive values in production.

### Practical Example

#### Docker Compose

```yaml
services:
  webapi:
    image: yourapp:latest
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - AppSettings__ApiUrl=https://api.docker.com
      - AppSettings__ApiKey=docker-api-key
    ports:
      - "5000:80"
```

Using `.env` file:

```yaml
services:
  webapi:
    image: yourapp:latest
    env_file:
      - .env
    ports:
      - "5000:80"
```

#### Kubernetes

ConfigMap for non-sensitive values:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: appsettings-config
data:
  AppSettings__ApiUrl: https://api.k8s.com
```

Secret for sensitive values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: appsettings-secret
type: Opaque
data:
  AppSettings__ApiKey: a3BpLXZhbHVlCg==
```

Deployment injection:

```yaml
envFrom:
  - configMapRef:
      name: appsettings-config
  - secretRef:
      name: appsettings-secret
```

If configuration is correct, deployment changes become config operations, not emergency redeploy events.

### Common Mistakes

- Committing secrets into `appsettings.json`.
- Forgetting `__` for nested environment variable keys.
- Skipping startup validation for required config values.
- Mixing sensitive and non-sensitive values in same storage.
- Assuming local environment behavior matches production.

### Quick Recap

- Use `appsettings.json` for defaults.
- Use `appsettings.{Environment}.json` for environment-level overrides.
- Use environment variables for final runtime control.
- Validate critical settings at startup.
- Store secrets in proper secret managers.

### Next Steps

1. Add config health check endpoint for required keys.
2. Integrate cloud secret manager (Key Vault/Secrets Manager).
3. Add CI validation for missing environment variables.
4. Use tooling like `dotnet-appsettings-env` to generate env mappings faster.