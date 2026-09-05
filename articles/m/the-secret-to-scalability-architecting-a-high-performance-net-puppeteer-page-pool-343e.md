# The Secret to Scalability: Architecting a High-Performance .NET Puppeteer Page Pool

- Canonical URL: https://imzihad21.github.io/articles/a/the-secret-to-scalability-architecting-a-high-performance-net-puppeteer-page-pool-343e/
- Source URL: https://dev.to/imzihad21/the-secret-to-scalability-architecting-a-high-performance-net-puppeteer-page-pool-343e
- Web View: https://imzihad21.github.io/articles/a/the-secret-to-scalability-architecting-a-high-performance-net-puppeteer-page-pool-343e/
- Published: 2025-07-31T17:39:47.000Z
- Modified: 2025-07-31T17:39:47.000Z
- Reading time: 3 minutes
- Tags: dotnet, puppeteer, pdfgeneration, chromium

Launching a new Chromium process per request does not scale. Startup latency, renderer memory overhead, and unbounded concurrency will eventually collapse the service.

This guide shows a production-ready page-pool architecture in .NET using PuppeteerSharp.

### Why It Matters

- Reduces per-request rendering latency.
- Prevents memory spikes from unbounded browser/page creation.
- Adds backpressure under load instead of crashing.
- Improves reliability with health checks and page recycling.

### Core Concepts

#### 1. Split Responsibilities by File

Recommended structure:

1. `PooledChromiumPdfEngine.cs`
2. `ChromiumRenderingHealthCheck.cs`
3. `ChromiumRenderingServiceCollectionExtensions.cs`
4. `Program.cs`
5. `PdfRenderController.cs`
6. `appsettings.json`

#### 2. Use a Bounded Page Pool

Use `Channel<IPage>` as a reusable page pool and concurrency limiter.

```csharp
using System.Threading.Channels;
using PuppeteerSharp;

public sealed class PooledChromiumPdfEngine
{
    private IBrowser? _browserInstance;
    private readonly int _maxPooledPages;
    private readonly Channel<IPage> _availablePages;

    public PooledChromiumPdfEngine(IConfiguration configuration)
    {
        var configuredPageLimit = configuration.GetValue<int>("PdfRendering:MaxConcurrentPages");

        if (configuredPageLimit < 1 || configuredPageLimit > 64)
            throw new ArgumentOutOfRangeException(nameof(configuredPageLimit));

        _maxPooledPages = configuredPageLimit;
        _availablePages = Channel.CreateBounded<IPage>(new BoundedChannelOptions(_maxPooledPages)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false
        });
    }
}
```

#### 3. Warm Up Browser and Pages at Startup

Initialize Chromium once and pre-create pool pages.

```csharp
public async Task WarmUpAsync()
{
    var executablePath = Environment.GetEnvironmentVariable("PUPPETEER_EXECUTABLE_PATH");

    if (string.IsNullOrWhiteSpace(executablePath) || !File.Exists(executablePath))
    {
        var revision = await new BrowserFetcher().DownloadAsync();
        executablePath = revision.GetExecutablePath();
    }

    _browserInstance = await Puppeteer.LaunchAsync(new LaunchOptions
    {
        ExecutablePath = executablePath,
        Headless = true,
        Timeout = 300000,
        Args =
        [
            "--no-sandbox",
            "--disable-setuid-sandbox",
            "--disable-dev-shm-usage",
            "--disable-gpu",
            "--disable-extensions"
        ]
    });

    for (var i = 0; i < _maxPooledPages; i++)
    {
        var page = await _browserInstance.NewPageAsync();
        await page.SetJavaScriptEnabledAsync(false);
        await _availablePages.Writer.WriteAsync(page);
    }
}
```

#### 4. Render with Checkout/Return Pattern

Each request borrows one page, renders, then returns or replaces it.

```csharp
public async Task<byte[]> RenderAsync(string htmlMarkup)
{
    if (_browserInstance == null || !_browserInstance.IsConnected)
        throw new InvalidOperationException("Chromium backend unavailable.");

    var page = await _availablePages.Reader.ReadAsync();
    var shouldRecyclePage = true;

    try
    {
        await page.SetContentAsync(htmlMarkup, new NavigationOptions
        {
            WaitUntil = [WaitUntilNavigation.Load],
            Timeout = 300000
        });

        var pdfBytes = await page.PdfDataAsync(new PdfOptions
        {
            PrintBackground = true,
            Format = PaperFormat.A4
        });

        shouldRecyclePage = false;
        return pdfBytes;
    }
    finally
    {
        if (shouldRecyclePage || page.IsClosed)
        {
            if (!page.IsClosed)
                await page.DisposeAsync();

            page = await _browserInstance.NewPageAsync();
            await page.SetJavaScriptEnabledAsync(false);
        }

        await page.GoToAsync("about:blank");
        await _availablePages.Writer.WriteAsync(page);
    }
}
```

#### 5. Add Health Check Integration

```csharp
public sealed class ChromiumRenderingHealthCheck(
    PooledChromiumPdfEngine pdfEngine) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        return await pdfEngine.IsBrowserResponsiveAsync(cancellationToken)
            ? HealthCheckResult.Healthy()
            : HealthCheckResult.Unhealthy();
    }
}
```

#### 6. Register via DI Extensions

```csharp
public static class ChromiumRenderingServiceCollectionExtensions
{
    public static IServiceCollection AddChromiumPdfEngine(this IServiceCollection services)
    {
        services.AddSingleton<PooledChromiumPdfEngine>();
        services.AddSingleton<IHealthCheck, ChromiumRenderingHealthCheck>();
        return services;
    }

    public static async Task WarmUpChromiumPdfEngineAsync(this IServiceProvider provider)
    {
        var engine = provider.GetRequiredService<PooledChromiumPdfEngine>();
        await engine.WarmUpAsync();
    }
}
```

### Practical Example

#### Program Startup

```csharp
builder.Services
    .AddChromiumPdfEngine()
    .AddHealthChecks()
    .AddCheck<ChromiumRenderingHealthCheck>(nameof(ChromiumRenderingHealthCheck));

var app = builder.Build();

await app.Services.WarmUpChromiumPdfEngineAsync();

app.MapHealthChecks("/healthz");
app.Run();
```

#### API Endpoint

```csharp
[ApiController]
public sealed class PdfRenderController(PooledChromiumPdfEngine pdfEngine) : ControllerBase
{
    [HttpPost("/api/render/pdf")]
    public async Task<IActionResult> RenderPdfAsync([FromBody] string htmlMarkup)
    {
        var pdfBytes = await pdfEngine.RenderAsync(htmlMarkup);
        return File(pdfBytes, "application/pdf");
    }
}
```

This keeps controllers thin and rendering complexity isolated where it belongs.

### Common Mistakes

- Launching Chromium per request.
- No bounded concurrency control for render pages.
- Returning dirty/crashed pages to pool.
- Skipping startup warm-up and paying first-request penalty.
- No health probe for browser responsiveness.

### Quick Recap

- One browser instance, many pooled pages.
- Bounded channel enforces concurrency/backpressure.
- Warm-up pages at startup for predictable latency.
- Recycle unhealthy pages immediately.
- Wire health checks and DI for operational stability.

### Next Steps

1. Add load tests to tune `MaxConcurrentPages` against memory limits.
2. Add metrics for checkout wait time and render duration.
3. Add graceful shutdown disposal for browser/page resources.
4. Add HTML sanitization and template validation for input safety.