# Manual Instructions: USS Enterprise Diagnostics (Aspire + Dapr Workflow)

Step-by-step instructions for building an Aspire workflow application that analyzes 3 USS Enterprise subsystems in parallel, prioritizes the results, and notifies the bridge if the priority is urgent. All activities use randomly generated mock data.

The project name throughout is **`EnterpriseDiagnostics`** — change it consistently if you'd like a different name.

---

# Module 1 — Project Creation

Open a terminal in the directory where you want the solution folder created, then run the commands below.

### 1.1 Scaffold the Aspire solution

```shell,run
aspire new aspire-starter -n EnterpriseDiagnostics -o EnterpriseDiagnostics --non-interactive --test-framework none
```

> The `--non-interactive` and `--test-framework none` flags are required, otherwise the CLI blocks waiting for user input.

Expected output:

```text,nocopy
Searching for available project template versions...
🧊 Getting templates...
📦 Using project templates version: 13.3.0
🚀 Creating new Aspire project...
📦 Created or updated NuGet.config in the project directory with required package sources.
✅ Project created successfully in <PATH>\EnterpriseDiagnostics.
```

### 1.2 About the generated Web project

The starter template also generates an `EnterpriseDiagnostics.Web` Blazor project. We won't use it in this walkthrough — you can ignore it and leave it in place.

Move into the solution folder for the remaining commands:

```shell
cd EnterpriseDiagnostics
```

### 1.3 Fix the AppHost launch URLs (HTTPS + fixed ports)

Open `EnterpriseDiagnostics.AppHost/Properties/launchSettings.json`. Two changes:

1. Replace every `http://` with `https://` for the `applicationUrl` and the `ASPIRE_*` environment variable URLs (`ASPIRE_DASHBOARD_OTLP_ENDPOINT_URL`, `ASPIRE_DASHBOARD_MCP_ENDPOINT_URL`, `ASPIRE_RESOURCE_SERVICE_ENDPOINT_URL`). Otherwise startup fails with an `ASPIRE_ALLOW_UNSECURED_TRANSPORT` error.
2. Replace the templated random ports with fixed ports so the dashboard URL is stable across runs.

The `https` profile should look like this:

```json
    "https": {
    "commandName": "Project",
    "dotnetRunMessages": true,
    "launchBrowser": true,
    "applicationUrl": "https://localhost:17000",
    "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "DOTNET_ENVIRONMENT": "Development",
        "ASPIRE_DASHBOARD_OTLP_ENDPOINT_URL": "https://localhost:17001",
        "ASPIRE_DASHBOARD_MCP_ENDPOINT_URL": "https://localhost:17002",
        "ASPIRE_RESOURCE_SERVICE_ENDPOINT_URL": "https://localhost:17003",
        "DOTNET_DASHBOARD_OTLP_HTTP_ENDPOINT_URL": "https://localhost:17004"
        }
    }
```

> If your template generates fewer `ASPIRE_*`/`DOTNET_DASHBOARD_*` keys than shown above, only update the ones that exist — don't add the missing ones. The dashboard itself is reached at the `applicationUrl` (here, `https://localhost:17000`).

### 1.4 Add the NuGet packages

Run from the **solution root** (`EnterpriseDiagnostics/`):

```shell,run,copy
dotnet add EnterpriseDiagnostics.ApiService/EnterpriseDiagnostics.ApiService.csproj package Dapr.Workflow --version 1.17.9
dotnet add EnterpriseDiagnostics.ApiService/EnterpriseDiagnostics.ApiService.csproj package Dapr.Workflow.Versioning --version 1.17.9
dotnet add EnterpriseDiagnostics.AppHost/EnterpriseDiagnostics.AppHost.csproj package CommunityToolkit.Aspire.Hosting.Dapr --version 13.0.0
dotnet add EnterpriseDiagnostics.AppHost/EnterpriseDiagnostics.AppHost.csproj package Aspire.Hosting.Valkey --version 13.3.0
```

The AppHost project  should have these packages:

```text,nocopy
<PackageReference Include="Aspire.Hosting.Valkey" Version="13.3.0" />
<PackageReference Include="CommunityToolkit.Aspire.Hosting.Dapr" Version="13.0.0" />
```

The ApiService project  should have these packages:

```text,nocopy
<PackageReference Include="Dapr.Workflow" Version="1.17.9" />
<PackageReference Include="Dapr.Workflow.Versioning" Version="1.17.9" />
<PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.0.7" />
```

Do a `dotnet build` to build the solution.

---

# Module 2 — Workflow & Activity Definition

All workflow related files will live under `EnterpriseDiagnostics.ApiService/`. Create the following folders:

```shell,run
mkdir EnterpriseDiagnostics.ApiService/Models
mkdir EnterpriseDiagnostics.ApiService/Workflows
mkdir EnterpriseDiagnostics.ApiService/Activities
```

### 2.1 Models — `Models/EnterpriseModels.cs`

Create the models first, they all record type models and are all placed in the same file. There are input and output records for the workflow, the subsystem analyses the prioritization, and the bridge notification activities.

First create the file:

```shell,run,copy
touch EnterpriseDiagnostics.Models/Models.cs
```

Then copy the code:

```csharp
namespace EnterpriseDiagnostics.Models;

public record EnterpriseAnalysisInput(string Id, string MissionName);

public record EnterpriseAnalysisOutput(
    string MissionName,
    string Priority,
    string Summary,
    bool BridgeNotified);

public record SubsystemAnalysisInput(string SubsystemName);

public record SubsystemAnalysisOutput(
    string SubsystemName,
    string Status,
    int AnomalyScore,
    double PowerLevel);

public record PrioritizationInput(SubsystemAnalysisOutput[] Analyses);

public record PrioritizationOutput(string Priority, string Summary);

public record BridgeNotificationInput(string Priority, string Summary);

public record BridgeNotificationOutput(bool Acknowledged, string AcknowledgedBy);
```

### 2.2 Workflow — `Workflows/EnterpriseAnalysisWorkflow.cs`

Now create the workflow, it will fan-out/fan-in over 3 subsystems to run diagnostics, then prioritize, then conditionally notify the bridge.

First create the file:

```shell,run,copy
touch EnterpriseDiagnostics.Workflows/EnterpriseAnalysisWorkflow.cs
```

Then copy the code:

```csharp,run,copy
using Microsoft.Extensions.Logging;
using Dapr.Workflow;
using EnterpriseDiagnostics.Activities;
using EnterpriseDiagnostics.Models;

namespace EnterpriseDiagnostics.Workflows;

internal sealed partial class EnterpriseAnalysisWorkflow
    : Workflow<EnterpriseAnalysisInput, EnterpriseAnalysisOutput>
{
    private static readonly string[] Subsystems =
    [
        "WarpDrive",
        "LifeSupport",
        "Shields",
    ];

    public override async Task<EnterpriseAnalysisOutput> RunAsync(
        WorkflowContext context,
        EnterpriseAnalysisInput input)
    {
        var logger = context.CreateReplaySafeLogger<EnterpriseAnalysisWorkflow>();
        LogStart(logger, context.InstanceId, input.MissionName);

        var analysisTasks = Subsystems.Select(name =>
            context.CallActivityAsync<SubsystemAnalysisOutput>(
                nameof(AnalyzeSubsystemActivity),
                new SubsystemAnalysisInput(name)));

        var analyses = await Task.WhenAll(analysisTasks);

        var prioritization = await context.CallActivityAsync<PrioritizationOutput>(
            nameof(PrioritizeAnalysesActivity),
            new PrioritizationInput(analyses));

        bool bridgeNotified = false;
        if (string.Equals(prioritization.Priority, "Urgent", StringComparison.OrdinalIgnoreCase))
        {
            LogUrgent(logger, context.InstanceId);
            var notification = await context.CallActivityAsync<BridgeNotificationOutput>(
                nameof(NotifyBridgeActivity),
                new BridgeNotificationInput(prioritization.Priority, prioritization.Summary));
            bridgeNotified = notification.Acknowledged;
        }

        return new EnterpriseAnalysisOutput(
            input.MissionName,
            prioritization.Priority,
            prioritization.Summary,
            bridgeNotified);
    }

    [LoggerMessage(LogLevel.Information, "Starting analysis workflow {Id} for mission {Mission}")]
    static partial void LogStart(ILogger logger, string Id, string Mission);

    [LoggerMessage(LogLevel.Warning, "Urgent priority detected on workflow {Id}; notifying bridge")]
    static partial void LogUrgent(ILogger logger, string Id);
}
```

### 2.3 Activity — `Activities/AnalyzeSubsystemActivity.cs`

Now create the AnalyzeSubsystemActivity, it returns randomly-generated mock telemetry for a single subsystem.

First create the file:

```shell,run,copy
touch EnterpriseDiagnostics.Activities/AnalyzeSubsystemActivity.cs
```

Then copy the code:

```csharp,run,copy
using Microsoft.Extensions.Logging;
using Dapr.Workflow;
using EnterpriseDiagnostics.Models;

namespace EnterpriseDiagnostics.Activities;

internal sealed partial class AnalyzeSubsystemActivity(ILogger<AnalyzeSubsystemActivity> logger)
    : WorkflowActivity<SubsystemAnalysisInput, SubsystemAnalysisOutput>
{
    private static readonly string[] Statuses = ["Nominal", "Degraded", "Critical"];

    public override Task<SubsystemAnalysisOutput> RunAsync(
        WorkflowActivityContext context,
        SubsystemAnalysisInput input)
    {
        LogAnalyzing(logger, input.SubsystemName);

        var random = Random.Shared;
        var status = Statuses[random.Next(Statuses.Length)];
        var anomalyScore = random.Next(0, 101);
        var powerLevel = Math.Round(random.NextDouble() * 100.0, 2);

        var result = new SubsystemAnalysisOutput(
            input.SubsystemName,
            status,
            anomalyScore,
            powerLevel);

        LogAnalyzed(logger, input.SubsystemName, status, anomalyScore);
        return Task.FromResult(result);
    }

    [LoggerMessage(LogLevel.Information, "Analyzing subsystem {Subsystem}")]
    static partial void LogAnalyzing(ILogger logger, string Subsystem);

    [LoggerMessage(LogLevel.Information, "Analyzed {Subsystem}: status={Status}, anomaly={Anomaly}")]
    static partial void LogAnalyzed(ILogger logger, string Subsystem, string Status, int Anomaly);
}
```

### 2.4 Activity — `Activities/PrioritizeAnalysesActivity.cs`

Create the PrioritizeAnalysesActivity which aggregates the three subsystem reports into a single prioritized report.

First create the file:

```shell,run,copy
touch EnterpriseDiagnostics.Activities/PrioritizeAnalysesActivity.cs
```

Then copy the code:

```csharp
using Microsoft.Extensions.Logging;
using Dapr.Workflow;
using EnterpriseDiagnostics.Models;

namespace EnterpriseDiagnostics.Activities;

internal sealed partial class PrioritizeAnalysesActivity(ILogger<PrioritizeAnalysesActivity> logger)
    : WorkflowActivity<PrioritizationInput, PrioritizationOutput>
{
    public override Task<PrioritizationOutput> RunAsync(
        WorkflowActivityContext context,
        PrioritizationInput input)
    {
        var maxAnomaly = input.Analyses.Max(a => a.AnomalyScore);
        var anyCritical = input.Analyses.Any(a => a.Status == "Critical");

        string priority = (anyCritical, maxAnomaly) switch
        {
            (true, _) => "Urgent",
            (false, >= 70) => "Urgent",
            (false, >= 40) => "Warning",
            _ => "Normal",
        };

        var summary = string.Join("; ",
            input.Analyses.Select(a =>
                $"{a.SubsystemName}={a.Status} (anomaly {a.AnomalyScore}, power {a.PowerLevel}%)"));

        LogPriority(logger, priority, maxAnomaly);
        return Task.FromResult(new PrioritizationOutput(priority, summary));
    }

    [LoggerMessage(LogLevel.Information, "Prioritized analyses: {Priority} (max anomaly={Max})")]
    static partial void LogPriority(ILogger logger, string Priority, int Max);
}
```

### 2.5 Activity — `Activities/NotifyBridgeActivity.cs`

Create the final activity, NotifyBridgeActivity, that mock-notifies the bridge with a randomly chosen acknowledging officer.

First create the file:

```shell,run,copy
touch EnterpriseDiagnostics.Activities/NotifyBridgeActivity.cs
```

Then copy the code:

```csharp
using Microsoft.Extensions.Logging;
using Dapr.Workflow;
using EnterpriseDiagnostics.Models;

namespace EnterpriseDiagnostics.Activities;

internal sealed partial class NotifyBridgeActivity(ILogger<NotifyBridgeActivity> logger)
    : WorkflowActivity<BridgeNotificationInput, BridgeNotificationOutput>
{
    private static readonly string[] Officers =
    [
        "Capt. Picard",
        "Cmdr. Riker",
        "Lt. Cmdr. Data",
        "Lt. Worf",
    ];

    public override Task<BridgeNotificationOutput> RunAsync(
        WorkflowActivityContext context,
        BridgeNotificationInput input)
    {
        var officer = Officers[Random.Shared.Next(Officers.Length)];
        LogNotify(logger, input.Priority, officer);

        return Task.FromResult(new BridgeNotificationOutput(
            Acknowledged: true,
            AcknowledgedBy: officer));
    }

    [LoggerMessage(LogLevel.Warning, "Bridge notified ({Priority}); acknowledged by {Officer}")]
    static partial void LogNotify(ILogger logger, string Priority, string Officer);
}
```

Run a `dotnet build` to verify to solution builds correctly.

```shell,run,copy
dotnet build
```

### 2.6 Update `EnterpriseDiagnostics.ApiService/Program.cs`

Replace the file contents with:

```csharp
using Microsoft.AspNetCore.Mvc;
using Dapr.Workflow;
using Dapr.Workflow.Versioning;
using EnterpriseDiagnostics.Activities;
using EnterpriseDiagnostics.Models;
using EnterpriseDiagnostics.Workflows;

var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.Services.AddDaprWorkflow(options =>
{
    options.RegisterActivity<AnalyzeSubsystemActivity>();
    options.RegisterActivity<PrioritizeAnalysesActivity>();
    options.RegisterActivity<NotifyBridgeActivity>();
});
builder.Services.AddDaprWorkflowVersioning();

var app = builder.Build();

app.MapPost("/start", async (
    [FromServices] DaprWorkflowClient workflowClient,
    [FromBody] EnterpriseAnalysisInput workflowInput) =>
{
    var instanceId = await workflowClient.ScheduleNewWorkflowAsync(
        name: nameof(EnterpriseAnalysisWorkflow),
        instanceId: workflowInput.Id,
        input: workflowInput);

    return Results.Ok(new { instanceId });
});

app.MapGet("/status/{instanceId}", async (
    [FromRoute] string instanceId,
    [FromServices] DaprWorkflowClient workflowClient) =>
{
    var state = await workflowClient.GetWorkflowStateAsync(instanceId);
    if (state is null || !state.Exists)
    {
        return Results.NotFound($"Workflow instance '{instanceId}' not found.");
    }

    var output = state.ReadOutputAs<EnterpriseAnalysisOutput>();
    return Results.Ok(new { state, output });
});

app.MapPost("pause/{instanceId}", async (
    [FromRoute] string instanceId,
    [FromServices] DaprWorkflowClient workflowClient) =>
{
    await workflowClient.SuspendWorkflowAsync(instanceId);
    return Results.Accepted();
});

app.MapPost("resume/{instanceId}", async (
    [FromRoute] string instanceId,
    [FromServices] DaprWorkflowClient workflowClient) =>
{
    await workflowClient.ResumeWorkflowAsync(instanceId);
    return Results.Accepted();
});

app.MapPost("terminate/{instanceId}", async (
    [FromRoute] string instanceId,
    [FromServices] DaprWorkflowClient workflowClient) =>
{
    await workflowClient.TerminateWorkflowAsync(instanceId);
    return Results.Accepted();
});

app.MapDefaultEndpoints();

app.Run();
```

> Workflow types are auto-registered by `AddDaprWorkflowVersioning()` — only activities need explicit `RegisterActivity<T>()` calls.

### 2.7 Verify

From the solution root:

```shell
dotnet build EnterpriseDiagnostics.sln
```

Fix any errors before continuing.

---

# Module 3 — AppHost Definition

All files in this module live under `EnterpriseDiagnostics.AppHost/`. Create a `Resources/` folder.

### 3.1 `Resources/statestore.yaml`

Used by the ApiService Dapr sidecar (runs on the host, so `localhost`).

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: workflow-store
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: "localhost:16379"
    - name: redisPassword
      value: "state-store-123"
    - name: actorStateStore
      value: true
```

### 3.2 `Resources/statestore-dashboard.yaml`

Used by the Diagrid Dashboard container (runs in Docker, so `host.docker.internal`).

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: workflow-store
scopes:
  - diagrid-dashboard
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: "host.docker.internal:16379"
    - name: redisPassword
      value: "state-store-123"
    - name: actorStateStore
      value: true
```

### 3.3 Update `EnterpriseDiagnostics.AppHost.csproj`

Add a `Content` item group so the YAML files are copied to the output directory. The full file should look like:

```xml
<Project Sdk="Aspire.AppHost.Sdk/13.2.2">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <UserSecretsId>auto-generated-id</UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\EnterpriseDiagnostics.ApiService\EnterpriseDiagnostics.ApiService.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.Valkey" Version="13.2.2" />
    <PackageReference Include="CommunityToolkit.Aspire.Hosting.Dapr" Version="13.0.0" />
  </ItemGroup>

  <ItemGroup>
    <Content Include="Resources\**\*.*">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      <Link>Resources\%(RecursiveDir)%(Filename)%(Extension)</Link>
    </Content>
  </ItemGroup>

</Project>
```

> Keep the existing `<UserSecretsId>` value generated by `aspire new` — don't replace it with `auto-generated-id`.

### 3.4 Replace `AppHost.cs`

```csharp
using System.Reflection;
using CommunityToolkit.Aspire.Hosting.Dapr;

var builder = DistributedApplication.CreateBuilder(args);

builder.AddDapr();

string executingPath = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location)
    ?? throw new("Where am I?");

var cachePassword = builder.AddParameter("cache-password", "state-store-123", secret: true);
var cache = builder
    .AddValkey("cache", 16379, cachePassword)
    .WithContainerName("workflow-state")
    .WithDataVolume("workflow-state-data");

var apiService = builder
    .AddProject<Projects.EnterpriseDiagnostics_ApiService>("apiservice")
    .WithDaprSidecar(new DaprSidecarOptions
    {
        LogLevel = "debug",
        ResourcesPaths =
        [
            Path.Join(executingPath, "Resources"),
        ],
    });

apiService.WaitFor(cache);

builder
    .AddContainer("diagrid-dashboard", "ghcr.io/diagridio/diagrid-dashboard:latest")
    .WithContainerName("diagrid-dashboard")
    .WithBindMount(Path.Join(executingPath, "Resources"), "/app/components")
    .WithEnvironment("COMPONENT_FILE", "/app/components/statestore-dashboard.yaml")
    .WithEnvironment("APP_ID", "diagrid-dashboard")
    .WithHttpEndpoint(targetPort: 8080)
    .WithReference(cache);

builder.Build().Run();
```

### 3.5 Verify

From the solution root:

```shell
dotnet build EnterpriseDiagnostics.sln
```

---

# Module 4 — Run Application

### 4.1 Start Aspire

From the solution root:

```shell
aspire run
```

The Aspire dashboard opens automatically in your browser. Wait until **`apiservice`**, **`cache`**, and **`diagrid-dashboard`** all show as Running.

### 4.2 Find the ApiService URL

In the Aspire dashboard, click the **`apiservice`** resource and copy its HTTPS endpoint (something like `https://localhost:7XXX`). Use this URL in the curl commands below as `<API_URL>`.

### 4.3 Start a workflow with curl

```shell
curl -k -X POST <API_URL>/start ^
  -H "Content-Type: application/json" ^
  -d "{\"id\":\"mission-001\",\"missionName\":\"Encounter at Farpoint\"}"
```

(PowerShell users: replace `^` line continuations with backticks `` ` ``, or put it all on one line.)

The response returns the `instanceId`, e.g.:

```json
{ "instanceId": "mission-001" }
```

### 4.4 Check the workflow status

```shell
curl -k <API_URL>/status/mission-001
```

The response contains the workflow `state` and the deserialized `output` with `Priority`, `Summary`, and `BridgeNotified` fields.

### 4.5 Open the Diagrid Dev Dashboard

In the Aspire dashboard, find the **`diagrid-dashboard`** resource and click its HTTP endpoint (port 8080). In the dashboard you can:

- Browse all workflow instances backed by the `workflow-store` component.
- See the timeline of activity executions (the three parallel `AnalyzeSubsystemActivity` calls, then `PrioritizeAnalysesActivity`, and conditionally `NotifyBridgeActivity`).
- Inspect inputs and outputs for each step.

Run the workflow several times with different `id` values — because the activities use random data, you'll see runs that escalate to **Urgent** (and trigger `NotifyBridgeActivity`) and others that resolve to **Normal** or **Warning**.
