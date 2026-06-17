# Enablement Overview

> **Sources**:
> - https://learn.microsoft.com/en-us/azure/azure-monitor/optimization-insights/code-optimizations-profiler-overview
> - https://github.com/Azure/azuremonitor-opentelemetry-profiler-net
> **Last synced**: 2026-03-25
> **Note**: This is a local fallback copy. The SKILL.md instructs the agent to fetch the latest platform-specific page online first.

## Enablement methods by platform

### App Service (Windows) — ETW, no code change

1. Verify **Always on** is enabled (Settings → Configuration → General settings)
2. Must be on **Basic tier or higher**
3. Go to Monitoring → Application Insights → Turn on / Enable
4. Under .NET or .NET Core tab, set Collection level to **Recommended**
5. Set **Profiler and Code Optimizations** to **On**
6. Click **Apply**

**Cross-subscription setup** (manual app settings):

| App Setting | Value |
|---|---|
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | From App Insights Overview page |
| `APPINSIGHTS_PROFILERFEATURE_VERSION` | `1.0.0` |
| `DiagnosticServices_EXTENSION_VERSION` | `~3` |

### App Service (Linux) — EventPipe, code change required

Add the profiler NuGet package and enable it in code. See the EventPipe setup section below.

### Containers (AKS, Container Apps, ACI) — EventPipe, code change required

1. Add NuGet package (see EventPipe setup below)
2. Enable profiler in `Program.cs`
3. Set `APPLICATIONINSIGHTS_CONNECTION_STRING` environment variable
4. Build and deploy container

### Azure Functions (in-process, classic AI SDK) — ETW, no code change

Enable via Azure Portal (same as App Service). Requires **App Service plan** (not Consumption plan). This applies only to the **in-process** hosting model with the classic Application Insights SDK.

### Azure Functions (isolated worker / OpenTelemetry) — EventPipe, code change

Isolated-worker Functions apps (and any Functions app using `Azure.Monitor.OpenTelemetry.Exporter`) require the **EventPipe** profiler. See the EventPipe section below for setup instructions.

### VMs / VMSS — ETW, ARM template

Enable via ARM template with diagnostics extension. Application Insights SDK must be enabled in application code first.

### Service Fabric — ETW, ARM template

Enable via ARM template with diagnostics extension.

## EventPipe setup (code change path)

### With Azure Monitor OpenTelemetry distribution

```bash
dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore --prerelease
dotnet add package Azure.Monitor.OpenTelemetry.Profiler --prerelease
```

```csharp
using Azure.Monitor.OpenTelemetry.AspNetCore;
using Azure.Monitor.OpenTelemetry.Profiler;

builder.Services.AddOpenTelemetry()
    .UseAzureMonitor()
    .AddAzureMonitorProfiler();
```

### With Azure Monitor OpenTelemetry exporter (Azure Functions isolated worker)

> **Note:** This path uses the same underlying EventPipe profiler mechanism as the OTel distro path above, but the combination with `UseAzureMonitorExporter()` (rather than `UseAzureMonitor()`) is not yet documented in the [official profiler repo](https://github.com/Azure/azuremonitor-opentelemetry-profiler-net). It is expected to work because `AddAzureMonitorProfiler()` hooks into the OpenTelemetry pipeline independent of the exporter choice. If profiler data does not appear after enablement, fall back to the OTel distro path (`UseAzureMonitor()`) as a verified alternative.

This assumes the app already has `Microsoft.Azure.Functions.Worker.OpenTelemetry` and `Azure.Monitor.OpenTelemetry.Exporter` configured. If not, install them first:

```bash
dotnet add package Microsoft.Azure.Functions.Worker.OpenTelemetry
dotnet add package Azure.Monitor.OpenTelemetry.Exporter
dotnet add package Azure.Monitor.OpenTelemetry.Profiler --prerelease
```

```csharp
// In Program.cs for an isolated worker Functions app:
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(services =>
    {
        services.AddOpenTelemetry()
            .UseFunctionsWorkerDefaults()
            .UseAzureMonitorExporter()
            .AddAzureMonitorProfiler();
    })
    .Build();

host.Run();
```

> **Note:** `UseFunctionsWorkerDefaults()` is an `OpenTelemetryBuilder` extension from `Microsoft.Azure.Functions.Worker.OpenTelemetry` (distinct from the `IHostBuilder.ConfigureFunctionsWorkerDefaults()` call above). It registers Functions-specific instrumentation so that function invocations are properly traced. `UseAzureMonitorExporter()` exports telemetry directly to Azure Monitor.

The profiler reads the connection string from `APPLICATIONINSIGHTS_CONNECTION_STRING`, so no extra configuration is needed when that environment variable is already set.

### With classic Application Insights SDK

```bash
dotnet add package Microsoft.ApplicationInsights.Profiler.AspNetCore
```

```csharp
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddServiceProfiler();
```

## Verifying the profiler is running

Look for these log messages after starting the application:

```
Starting application insights profiler with connection string: InstrumentationKey=...
Service Profiler session started.
Finished calling trace uploader. Exit code: 0
Service Profiler session finished.
```

Profiler traces typically appear in Application Insights within **2–5 minutes**.

## Troubleshooting

- Enable debug logging: set log level for `Microsoft.ServiceProfiler` and `Microsoft.ApplicationInsights.Profiler` to `Debug`
- Verify the Application Insights connection string is correct
- Check that trigger thresholds are below observed usage levels
- For ETW on App Service: check that the `ApplicationInsightsProfiler3` WebJob is running

## Key constraints

- **5–15% CPU/memory overhead** when actively profiling
- Default: profiles for **30 seconds every hour** (random sampling)
- Also triggers on **>80% CPU** or **>80% memory** usage
- Only **one profiler** per web app
- Profiler data deleted after **15 days**
- Code Optimizations only works with default storage (not BYOS)
