# Managing Production Configurations in ASP.NET Core WebAPI Using Environment Variables

- Canonical URL: https://imzihad21.github.io/articles/a/managing-production-configurations-in-aspnet-core-webapi-using-environment-variables-3nmf/
- Source URL: https://dev.to/imzihad21/managing-production-configurations-in-aspnet-core-webapi-using-environment-variables-3nmf
- Web View: https://imzihad21.github.io/articles/a/managing-production-configurations-in-aspnet-core-webapi-using-environment-variables-3nmf/
- Published: 2025-01-25T03:30:01.000Z
- Modified: 2025-01-25T03:30:01.000Z
- Reading time: 5 minutes
- Tags: dotnet, docker, devops, env

## Managing production configurations in ASP.NET Core Web API using environment variables

Managing production configuration requires flexibility, secret isolation, and compatibility with modern container runtimes. Hardcoding configuration values or baking environment-specific settings files directly into container images causes credential leaks in source control and forces expensive container rebuilds for simple configuration adjustments.

This architecture leverages ASP.NET Core's layered configuration provider pipeline to override base settings with environment variables at runtime without modifying application code or deployment artifacts. The pattern guarantees that secrets remain isolated in secure platform stores, hierarchical JSON configurations map predictably to container environment variables, and missing required parameters fail fast at application startup.

### The problem and production context

In containerized microservices running on Docker and Kubernetes, configuration management often breaks when teams treat container images as environment-specific artifacts. Baking environment files like `appsettings.Production.json` directly into Docker images exposes production secrets to developers and CI/CD logging systems. Furthermore, attempting to modify files manually inside running production containers violates immutability and leads to configuration drift across replica pods.

- **Failure scenario**: A database password or external API secret is committed to `appsettings.json` in source control. When the repository is cloned by developers or contractors, production credentials leak. In other cases, a deployment team uses single `_` characters instead of double `__` delimiters in container environment variables, causing ASP.NET Core to silently ignore the overrides and fallback to default or invalid connection strings.
- **Why default approaches fall short**: Static configuration files inside container images cannot be altered without rebuilding the entire container. When applications lack startup validation, pods boot up successfully, enter service discovery, and fail only when a customer request exercises an unconfigured service dependency.
- **Production impact**: Unvalidated missing configurations cause cascading HTTP 500 runtime exceptions during peak traffic. Credential leakage across source control risks catastrophic data breaches, and rebuilding entire container images for configuration tweaks creates deployment delays during incident response.

Operational requirements satisfied by this design:
- Keeps API keys, credentials, and connection strings out of source control.
- Enables runtime setting adjustments across development, staging, and production environments.
- Integrates with container orchestrators such as Docker and Kubernetes.
- Eliminates manual file modifications inside production containers or servers.

### Mental model and core concepts

#### 1. Layered configuration provider pipeline

ASP.NET Core resolves settings through an ordered pipeline of providers, where subsequent sources override earlier ones:
- Default values defined in code
- `appsettings.json`
- `appsettings.{Environment}.json`
- Environment variables
- Command-line arguments

#### 2. Environment-specific configuration activation

The `ASPNETCORE_ENVIRONMENT` environment variable dictates which environment-specific file loads at startup:

```text
ASPNETCORE_ENVIRONMENT=Production
```

When set to `Production`, the runtime automatically loads and merges `appsettings.Production.json` on top of `appsettings.json`.

#### 3. Hierarchical key mapping via double `__` delimiters

Hierarchical configuration keys in JSON map to environment variables using double `__` delimiters:
- JSON key: `AppSettings:ApiUrl`
- Environment variable: `AppSettings__ApiUrl`

#### 4. Deterministic configuration merge sequence

Base configuration in `appsettings.json`:

```json
{
  "AppSettings": {
    "ApiUrl": "https://api.default.com",
    "ApiKey": "default-api-key"
  }
}
```

Environment override in `appsettings.Production.json`:

```json
{
  "AppSettings": {
    "ApiUrl": "https://api.production.com"
  }
}
```

Environment variables provided at runtime:

```bash
AppSettings__ApiUrl=https://api.override.com
AppSettings__ApiKey=override-api-key
```

Final values resolved at application startup:
- `ApiUrl`: `https://api.override.com`
- `ApiKey`: `override-api-key`

#### 5. Strongly typed options and startup validation

Bind configuration sections to strongly typed classes and validate required fields at startup to guarantee configuration integrity:

```csharp
public sealed class AppSettingsOptions
{
    public string ApiUrl { get; set; } = string.Empty;
    public string ApiKey { get; set; } = string.Empty;
}
```

Registering options with startup validation:

```csharp
builder.Services
    .AddOptions<AppSettingsOptions>()
    .Bind(builder.Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

#### 6. Secure secret management boundaries

Store production credentials and connection strings in platform secret stores, such as Azure Key Vault or AWS Secrets Manager, rather than plain text configuration files.

### Production implementation

The following complete deployment manifests demonstrate environment variable injection across Docker Compose and Kubernetes orchestration environments.

#### Docker Compose deployment

Pass environment variables directly or reference a `.env` file within the compose configuration.

Direct environment configuration:

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

Using a `.env` file:

```yaml
services:
  webapi:
    image: yourapp:latest
    env_file:
      - .env
    ports:
      - "5000:80"
```

#### Kubernetes deployment manifests

Separate non-sensitive application settings into ConfigMaps while storing secrets and keys in Secret resources.

ConfigMap for non-sensitive settings:

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

Kubernetes deployment injection via `envFrom`:

```yaml
envFrom:
  - configMapRef:
      name: appsettings-config
  - secretRef:
      name: appsettings-secret
```

Structured configuration ensures environment-specific changes happen during deployment without modifying application artifacts.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Resolving configuration from environment variables at startup adds negligible initialization overhead while keeping runtime reads purely in-memory. However, updating an environment variable requires restarting the container process or triggering a rolling deployment pod recreation.
* **Failure recovery**: Using `.ValidateOnStart()` causes the host process to fail immediately if required configuration settings are absent or malformed. While this prevents unconfigured pods from serving traffic, orchestrators will mark the container as failing (`CrashLoopBackOff`), providing fast feedback to deployment pipelines.
* **Scale limitations**: When applications manage hundreds of microservice configuration flags, environment variable lists in deployment manifests become difficult to audit. Moving to a centralized distributed configuration provider (such as Azure App Configuration or HashiCorp Consul) with dynamic change subscriptions becomes necessary at massive scale.
* **Operating system delimiter variance**: Linux shells and container runtimes prohibit colons in environment variable names, necessitating the double `__` convention. Windows environments support colons natively, but using double `__` delimiters universally guarantees cross-platform consistency.

### Common anti-patterns and gotchas

* **Committing sensitive connection strings and secrets to appsettings.json**: Embedding database credentials in git repositories exposes credentials to unauthorized personnel and history scraping tools. Always inject production credentials through environment variables or secret vaults.
* **Using single colons or single `_` characters instead of double `__` delimiters**: Linux shells do not support colons in variable names, and single `_` characters (`AppSettings_ApiUrl`) are treated by ASP.NET Core as flat property names rather than hierarchical section separators. Always use double `__` delimiters (`AppSettings__ApiUrl`).
* **Skipping startup validation**: Without `.ValidateOnStart()`, missing keys are parsed as null or empty strings. Applications start successfully and crash hours later when dependent business logic executes.
* **Storing secrets alongside non-sensitive values in unencrypted configuration repositories**: Keeping plain text passwords in public or unencrypted GitOps repositories bypasses security controls. Decouple non-sensitive configuration into ConfigMaps and sensitive values into encrypted Secrets.
* **Assuming local configuration keys match production environments**: Discrepancies in casing or hierarchy between development `appsettings.json` and production environment definitions cause unexpected runtime fallbacks. Verify mappings through automated schema tests.

### Implementation checklist

1. Ensure all sensitive credentials, connection strings, and tokens are removed from `appsettings.json` and committed repository files.
2. Bind configuration sections to strongly typed classes and configure `.ValidateDataAnnotations()` and `.ValidateOnStart()`.
3. Set `ASPNETCORE_ENVIRONMENT=Production` on production container runtimes.
4. Format nested environment variables using double `__` delimiters to map hierarchical JSON structures cleanly.
5. Create Kubernetes ConfigMaps for non-sensitive operational parameters and Kubernetes Secrets for credentials.
6. Inject configuration into container pods using `envFrom` references in deployment manifests.
7. Implement a startup health check that verifies all required configuration keys and downstream dependencies are present.
8. Integrate cloud secret managers such as Azure Key Vault or AWS Secrets Manager.
9. Add CI pipeline checks to detect missing required environment variables before deployment.
10. Utilize CLI tools to generate environment variable mappings directly from `appsettings.json`.