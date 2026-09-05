# Building a Linux Based Minimal and Efficient .NET 8 Application Docker Image

- Canonical URL: https://imzihad21.github.io/articles/a/building-a-linux-based-minimal-and-efficient-net-8-application-docker-image-50jp/
- Source URL: https://dev.to/imzihad21/building-a-linux-based-minimal-and-efficient-net-8-application-docker-image-50jp
- Web View: https://imzihad21.github.io/articles/a/building-a-linux-based-minimal-and-efficient-net-8-application-docker-image-50jp/
- Published: 2024-11-24T04:29:31.000Z
- Modified: 2024-11-24T04:29:31.000Z
- Reading time: 2 minutes
- Tags: dotnet, docker, linux, devops

## Building a Linux-Based Minimal and Efficient .NET 8 Application Docker Image

A good Docker image is not only "it runs". It should also be small, secure, and fast enough for real deployment pipelines. For .NET 8 on Linux, multi-stage build with Alpine is a practical baseline.

This guide uses `linux-musl-x64` targeting and explains why this pattern works well in containerized environments.

### Why It Matters

- Smaller image sizes improve pull and deploy speed.
- Multi-stage build separates compile and runtime concerns.
- Alpine + musl targeting works well for lightweight Linux containers.
- Runtime hardening reduces production risk.

### Core Concepts

#### 1. Build Stage with .NET SDK

Use SDK image to restore and publish the app.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
ARG BUILD_CONFIGURATION=Release
ARG RUNTIME=linux-musl-x64
WORKDIR /src
```

#### 2. Restore with Layer Caching

Copy project files first for better cache reuse.

```dockerfile
COPY ["MyApplication/MyApplication.csproj", "MyApplication/"]
RUN dotnet restore "./MyApplication/MyApplication.csproj" -r "$RUNTIME"
```

#### 3. Publish Optimized Artifacts

Copy source after restore, then publish release output.

```dockerfile
COPY . .
RUN dotnet publish "./MyApplication/MyApplication.csproj" \
    -c "$BUILD_CONFIGURATION" \
    -r "$RUNTIME" \
    --self-contained false \
    -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishReadyToRun=true
```

#### 4. Runtime Stage with ASP.NET Image

Use runtime image only in final stage to reduce size.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine
```

#### 5. Globalization and Timezone Support

Install required packages for culture/timezone behavior.

```dockerfile
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false
RUN apk add --no-cache icu-libs tzdata
```

#### 6. Runtime Hardening

Run as non-root user and expose only required port.

```dockerfile
WORKDIR /app
USER app
EXPOSE 8080
```

### Practical Example

Complete Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
ARG BUILD_CONFIGURATION=Release
ARG RUNTIME=linux-musl-x64
WORKDIR /src

COPY ["MyApplication/MyApplication.csproj", "MyApplication/"]
RUN dotnet restore "./MyApplication/MyApplication.csproj" -r "$RUNTIME"

COPY . .
RUN dotnet publish "./MyApplication/MyApplication.csproj" \
    -c "$BUILD_CONFIGURATION" \
    -r "$RUNTIME" \
    --self-contained false \
    -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishReadyToRun=true

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine

ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false
RUN apk add --no-cache icu-libs tzdata

WORKDIR /app
USER app
EXPOSE 8080

COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApplication.dll"]
```

Build and run:

```bash
docker build -t myapplication:net8 .
docker run -d -p 8080:8080 --name myapplication myapplication:net8
```

When image size drops and deploy time drops, DevOps team suddenly likes you more.

### Common Mistakes

- Using SDK image in production runtime stage.
- Copying full source before `dotnet restore` and losing cache efficiency.
- Forgetting `icu-libs` when globalization is required.
- Running container as root without reason.
- Mismatch between published runtime target and base image behavior.

### Quick Recap

- Multi-stage build is the baseline for .NET container optimization.
- `linux-musl-x64` aligns with Alpine-based runtime.
- Publish settings can improve startup and image quality.
- Runtime stage should stay minimal and non-root.
- Add globalization dependencies only when needed.

### Next Steps

1. Add `.dockerignore` to reduce build context.
2. Add image scanning in CI for vulnerabilities.
3. Benchmark startup with and without ReadyToRun.
4. Add health checks for orchestrator readiness.