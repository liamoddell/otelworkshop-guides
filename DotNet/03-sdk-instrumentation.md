# Lab 3: Add SDK Instrumentation

## 3.1 Understanding Manual SDK Instrumentation

Unlike Java's zero-code agent, .NET requires you to configure OpenTelemetry in your application code. The NuGet packages are already installed in the project.

**What we'll do:**
- Add OpenTelemetry using statements
- Configure tracing, metrics, and logging in Program.cs
- Set up OTLP exporters
- Configure service identification with environment variables

## 3.2 Stop GameServer and Add Using Statements

1. **Stop GameServer if running:**
   ```bash
   pkill -f Gameserver
   # Or use: lsof -ti:8090 | xargs kill -9
   ```

2. **Edit Program.cs:**
   ```bash
   nano /opt/gameserver/Program.cs
   ```

3. **Add OpenTelemetry using statements** at the top:
   ```csharp
   using OpenTelemetry.Logs;
   using OpenTelemetry.Metrics;
   using OpenTelemetry.Resources;
   using OpenTelemetry.Trace;
   ```

## 3.3 Configure OpenTelemetry in Program.cs

1. **Add OpenTelemetry configuration before `var app = builder.Build();`:**
   ```csharp
   // Configure OpenTelemetry - read from environment variables
   var serviceName = Environment.GetEnvironmentVariable("OTEL_SERVICE_NAME") ?? "gameserver";
   var serviceNamespace = Environment.GetEnvironmentVariable("SERVICE_NAMESPACE") ?? "default";
   var serviceInstanceId = Environment.GetEnvironmentVariable("HOSTNAME") ?? "localhost";

   // Add OpenTelemetry with resource attributes
   builder.Services.AddOpenTelemetry()
       .ConfigureResource(resource => resource
           .AddService(serviceName: serviceName, serviceVersion: "1.0.0")
           .AddAttributes(new Dictionary<string, object>
           {
               ["service.namespace"] = serviceNamespace,
               ["service.instance.id"] = serviceInstanceId,
               ["deployment.environment"] = "workshop"
           }))
       .WithTracing(tracing => tracing
           .AddAspNetCoreInstrumentation()
           .AddHttpClientInstrumentation()
           .AddSource("Gameserver")
           .AddOtlpExporter(options => {
               options.Endpoint = new Uri("http://localhost:4318/v1/traces");
               options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
           }))
       .WithMetrics(metrics => metrics
           .AddAspNetCoreInstrumentation()
           .AddHttpClientInstrumentation()
           .AddMeter("Gameserver")
           .AddOtlpExporter(options => {
               options.Endpoint = new Uri("http://localhost:4318/v1/metrics");
               options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
           }));

   // Configure logging to use OpenTelemetry
   builder.Logging.AddOpenTelemetry(options => {
       options.SetResourceBuilder(ResourceBuilder.CreateDefault()
           .AddService(serviceName: serviceName, serviceVersion: "1.0.0")
           .AddAttributes(new Dictionary<string, object>
           {
               ["service.namespace"] = serviceNamespace,
               ["service.instance.id"] = serviceInstanceId,
               ["deployment.environment"] = "workshop"
           }));
       options.AddOtlpExporter(exporterOptions => {
           exporterOptions.Endpoint = new Uri("http://localhost:4318/v1/logs");
           exporterOptions.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
       });
   });
   ```

   **Important**: Notice we use signal-specific endpoints (`/v1/traces`, `/v1/metrics`, `/v1/logs`). The .NET OTLP exporter requires these full paths.

## 3.4 Configure Service Identification

1. **Edit run.sh to configure service identification:**

   ```bash
   nano /opt/gameserver/run.sh
   ```

2. **Uncomment and set your unique namespace:**

   Find these lines:
   ```bash
   # export OTEL_SERVICE_NAME="gameserver"
   # export SERVICE_NAMESPACE="your-name"
   ```

   Uncomment them and **set your unique namespace**:
   ```bash
   export OTEL_SERVICE_NAME="gameserver"
   export SERVICE_NAMESPACE="loddell"  # Replace 'loddell' with your name
   ```

   **Important**: The `SERVICE_NAMESPACE` must be unique for each student to avoid collisions in Grafana Cloud.

3. **Save and start GameServer:**
   ```bash
   cd /opt/gameserver
   ./run.sh
   ```

## 3.5 Generate Traffic and Verify

1. **Generate traffic:**
```bash
# Run load test
cd /opt/loadtests
k6 run --duration 2m --vus 2 gameserver-loadtest.js
```

2. **Verify in Grafana Cloud:**
   - Open Grafana Cloud (URL provided by facilitator)
   - Navigate to **Application Observability**
   - Look for your service: `gameserver` with namespace `your-name-here`
   - You should see:
     - HTTP requests
     - Service map showing gameserver → rolldice
     - Traces with distributed context
