# OpenTelemetry Workshop - .NET Track

**Service**: GameServer (ASP.NET Core)
**Instrumentation**: Manual SDK Configuration
**Duration**: ~2 hours

---

## Lab 1: Test Your Application

### Understanding GameServer

GameServer is an ASP.NET Core Web API that manages a dice game. It calls the RollDice service to get dice rolls for players.

**Important**: GameServer depends on RollDice service. You'll need to start RollDice first.

### Start RollDice Service

```bash
cd /opt/rolldice
./run.sh
```

### Start GameServer

```bash
cd /opt/gameserver
./run.sh
```

**Test the endpoints:**
```bash
# Play a game (POST with JSON body)
curl -X POST http://localhost:8090/game/play \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice"}'

# Get game statistics
curl http://localhost:8090/game/stats

# Health check
curl http://localhost:8090/game/health
```

**Expected response:**
```json
{"playerName":"Alice","playerRoll":4,"computerRoll":3,"result":"Player wins!"}
```

The service runs on **port 8090**.

---

## Lab 2: Configure Alloy & Grafana Cloud

### What is Grafana Alloy?

Grafana Alloy is an OpenTelemetry collector that receives telemetry from your applications and forwards it to Grafana Cloud.

### Configure Alloy

1. **Get your Grafana Cloud credentials** from your workshop facilitator:
   - OTLP Endpoint (e.g., `https://otlp-gateway-prod-XX-XXX.grafana.net/otlp`)
   - Instance ID
   - API Token

2. **Edit the Alloy run script with your credentials:**
   ```bash
   nano /opt/alloy/run.sh
   ```

3. **Update the environment variables** (around line 6-8):
   ```bash
   export GRAFANA_CLOUD_OTLP_ENDPOINT="https://otlp-gateway-prod-XX-XXX.grafana.net/otlp"
   export GRAFANA_CLOUD_OTLP_USERNAME="YOUR_INSTANCE_ID"
   export GRAFANA_CLOUD_OTLP_PASSWORD="YOUR_API_TOKEN"
   ```

4. **Save the file** and start Alloy:
   ```bash
   cd /opt/alloy
   ./run.sh &
   ```

5. **Verify Alloy is running:**
   ```bash
   curl http://localhost:4318/v1/traces
   ```

   You should see a response indicating the OTLP endpoint is ready.

---

## Lab 3: Add SDK Instrumentation

### 3.1 Understanding Manual SDK Instrumentation

Unlike Java's zero-code agent, .NET requires you to configure OpenTelemetry in your application code. The NuGet packages are already installed in the project.

**What we'll do:**
- Add OpenTelemetry using statements
- Configure tracing, metrics, and logging in Program.cs
- Set up OTLP exporters
- Configure service identification with environment variables

### 3.2 Stop GameServer and Add Using Statements

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

### 3.3 Configure OpenTelemetry in Program.cs

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

### 3.4 Configure Service Identification

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

   **Important**: The `SERVICE_NAMESPACE` must be unique for each user to avoid collisions in Grafana Cloud.

3. **Save and start GameServer:**
   ```bash
   cd /opt/gameserver
   ./run.sh
   ```

### 3.5 Generate Traffic and Verify

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

---

## Lab 4: Add Custom Instrumentation

### 4.1 Understanding Custom Instrumentation

Automatic instrumentation captures framework-level telemetry (HTTP requests, HttpClient calls). Custom instrumentation lets you add business-specific context like game outcomes, player names, and custom metrics.

**Why System.Diagnostics?** This namespace provides .NET's native distributed tracing API (`ActivitySource`, `Activity`, `ActivityKind`) that OpenTelemetry uses under the hood. For metrics, we use `System.Diagnostics.Metrics` which provides `Meter` and `Counter` types.

**What we'll add:**
- Custom spans for game play operations
- Span attributes for game details (player name, rolls, results)
- Custom metrics (games started, games completed)
- Error handling with span status

### 4.2 Setup: Add Using Statements to GameService

1. **Edit GameService.cs:**
   ```bash
   nano /opt/gameserver/Services/GameService.cs
   ```

2. **Add using statements at the top:**
   ```csharp
   using System.Diagnostics;
   using System.Diagnostics.Metrics;
   using OpenTelemetry.Trace;
   ```

3. **Add static instrumentation fields after the existing private fields:**
   ```csharp
   // OpenTelemetry instrumentation
   private static readonly ActivitySource ActivitySource = new("Gameserver");
   private static readonly Meter Meter = new("Gameserver");
   private static readonly Counter<long> GamesStartedCounter = Meter.CreateCounter<long>(
       "games.started",
       unit: "{call}",
       description: "Number of games started");
   private static readonly Counter<long> GamesCompletedCounter = Meter.CreateCounter<long>(
       "games.completed",
       unit: "{call}",
       description: "Number of games completed");
   ```

### 4.3 Add Custom Span and Attributes to PlayGameAsync

1. **Wrap PlayGameAsync method with a custom span.** Find the `try` block at the start of PlayGameAsync and add BEFORE it:
   ```csharp
   // Create a custom span for the game play operation
   using var activity = ActivitySource.StartActivity("play", ActivityKind.Internal);
   ```

2. **Add games started counter.** Find where it logs "Player {PlayerName} is playing" and add AFTER the try { line but BEFORE the logger line:
   ```csharp
   // Increment games started counter
   GamesStartedCounter.Add(1);
   ```

3. **Add span attribute for player name.** Find where it logs "Player {PlayerName} is playing" and add AFTER it:
   ```csharp
   // Add span attribute for player name
   activity?.SetTag("game.player_name", request.Name);
   ```

### 4.4 Add Game Result Attributes and Metrics

1. **Add span attributes for game details.** Find where `GetResult` is called and add AFTER it:
   ```csharp
   // Add custom span attributes for game details
   activity?.SetTag("game.result", resultCode);
   activity?.SetTag("game.player_roll", playerRoll);
   activity?.SetTag("game.computer_roll", computerRoll);
   ```

2. **Add games completed counter with dimension.** Find where it logs "Game result was {ResultCode}" and add AFTER it:
   ```csharp
   // Increment games completed counter with winner dimension
   GamesCompletedCounter.Add(1, new KeyValuePair<string, object?>("winner", resultCode));
   ```

### 4.5 Instrument GameController GetStats Endpoint

1. **Edit GameController.cs:**
   ```bash
   nano /opt/gameserver/Controllers/GameController.cs
   ```

2. **Add using statements at the top:**
   ```csharp
   using System.Diagnostics;
   using OpenTelemetry.Trace;
   ```

3. **Add ActivitySource as a static field after the existing private fields:**
   ```csharp
   // OpenTelemetry instrumentation
   private static readonly ActivitySource ActivitySource = new("Gameserver");
   ```

4. **Add custom span to GetStats method.** Find the `GetStats()` method and add at the start:
   ```csharp
   // Create custom span for statistics endpoint
   using var activity = ActivitySource.StartActivity("get_statistics", ActivityKind.Server);
   ```

5. **Add span attributes for statistics.** Find where stats are retrieved and add AFTER the variable declarations:
   ```csharp
   // Add span attributes for statistics
   activity?.SetTag("stats.total_games", stats.TotalGames);
   activity?.SetTag("stats.player_wins", stats.PlayerWins);
   activity?.SetTag("stats.computer_wins", stats.ComputerWins);
   activity?.SetTag("stats.active_sessions", activeSessions);
   activity?.SetTag("stats.active_players", activePlayers);
   ```

6. **Add error handling in catch block.** Find the catch block in GetStats and add BEFORE the `return StatusCode(500, ...)` line:
   ```csharp
   // Record exception in span
   activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
   activity?.AddException(ex);
   ```

### 4.6 Rebuild, Test, and View Results

1. **Rebuild and restart:**
   ```bash
   cd /opt/gameserver
   dotnet build
   pkill -f Gameserver
   ./run.sh
   ```

2. **Generate traffic:**
   ```bash
   cd /opt/loadtests
   k6 run --duration 2m --vus 2 gameserver-loadtest.js
   ```

3. **View Custom Instrumentation in Grafana Cloud:**

   You should see:

   **Custom spans:**
   - `play` span with attributes:
     - `game.player_name`
     - `game.result`
     - `game.player_roll`
     - `game.computer_roll`
   - `get_statistics` span with attributes:
     - `stats.total_games`
     - `stats.player_wins`
     - `stats.computer_wins`
     - `stats.active_sessions`
     - `stats.active_players`

   **Custom metrics** in Explore → Metrics:
   - Search for `games.started` to see total games initiated
   - Search for `games.completed` to see games by winner (player/computer/tie)
   - Note: Counters are displayed as rates (c/s) in Grafana Cloud by default - this is expected behavior

   **Distributed traces** showing: HTTP request → gameserver → rolldice with custom attributes

**Tip:** Use `/opt/gameserver-reference` for the complete implementation.

---

## Lab 5: View Data in Grafana Cloud

### 5.1 Explore Application Observability

1. **Navigate to Application Observability** in Grafana Cloud

2. **Service Inventory:**
   - Filter by `environment=workshop`
   - Filter by `service.namespace=your-name-here`
   - Find your `gameserver` service

3. **Service Overview:**
   - Request rate
   - Error rate
   - Latency (p50, p95, p99)

### 5.2 Explore Distributed Traces

1. **Click on your service**
2. **View Traces**
3. **Find a trace and expand it:**
   - See the automatic HTTP instrumentation (ASP.NET Core)
   - See the automatic HttpClient instrumentation (gameserver → rolldice)
   - See your custom `play_game` span
   - View span attributes and timing

### 5.3 View Service Dependencies

1. **Service Map:**
   - See gameserver calling rolldice
   - View request rates and latencies between services

### 5.4 Search by Attributes

```
service.name="gameserver" && player.name="Alice"
```

### 5.5 Find Errors

If you generated error scenarios in the load test:
```
service.name="gameserver" && status.code="ERROR"
```

---

## Summary - What You Learned

✅ **Manual SDK instrumentation** with OpenTelemetry .NET
✅ **OTLP export** to Grafana Alloy
✅ **Custom spans** with ActivitySource
✅ **Span attributes** for business context
✅ **Distributed tracing** across services (gameserver → rolldice)

---

## Error Scenarios for Testing

The GameServer service includes intentional error scenarios to help you practice troubleshooting with distributed tracing:

### Scenario 1: RollDice Service Slow Responses
**Trigger**: RollDice service has built-in delays (every 15th request)
**Behavior**: GameServer waits for slow RollDice response
**What to look for**:
- Distributed trace showing high latency in rolldice span
- GameServer span waiting for rolldice to complete
- Total request time > 2 seconds
- Both services' spans visible in the same trace

**Testing**:
```bash
# Generate traffic to hit the slow response scenario
for i in {1..20}; do
  curl -X POST http://localhost:8090/game/play \
    -H "Content-Type: application/json" \
    -d '{"name":"Alice"}';
done
```

### Scenario 2: RollDice Service Unavailable
**Trigger**: Stop the RollDice service
**Behavior**: GameServer fails to connect to RollDice
**What to look for**:
- Activity status set to ERROR
- Exception details in the span
- HTTP client error in distributed trace
- Error propagation from GameServer

**Testing**:
```bash
# Stop RollDice service
pkill -f "java.*rolldice"

# Try to play a game
curl -X POST http://localhost:8090/game/play \
  -H "Content-Type: application/json" \
  -d '{"name":"Bob"}'

# Restart RollDice
cd /opt/rolldice && ./run.sh
```

### Scenario 3: Request Timeout
**Trigger**: If RollDice is very slow or unresponsive
**Behavior**: HttpClient timeout in GameServer
**What to look for**:
- Timeout exception captured in span
- `activity.AddException()` recording the timeout
- Partial distributed trace (GameServer span complete, RollDice may be incomplete)

**Testing**:
```bash
# Use load test with high concurrency to potentially trigger timeouts
cd /opt/loadtests
k6 run --duration 1m --vus 10 gameserver-loadtest.js
```

### Scenario 4: Invalid Dice Values
**Trigger**: DiceService validates dice rolls (should be 1-6)
**Behavior**: If RollDice returns invalid value, error is logged
**What to look for**:
- Validation errors in DiceService
- Span attributes showing invalid roll values
- Error status propagated to parent spans

**Testing**:
```bash
# Generate enough traffic to potentially hit edge cases
cd /opt/loadtests
k6 run --duration 2m --vus 5 gameserver-loadtest.js
```

### Scenario 5: Distributed Trace Visualization
**Trigger**: Normal operations with both services running
**Behavior**: Complete distributed traces across services
**What to look for**:
- Full trace showing: HTTP request → GameServer → HttpClient → RollDice → Response
- Trace context propagation (W3C trace context headers)
- Timing breakdown showing where time is spent
- Custom `play_game` span with business attributes

**Testing**:
```bash
# Generate clean traffic with both services healthy
for i in {1..10}; do
  curl -X POST http://localhost:8090/game/play \
    -H "Content-Type: application/json" \
    -d "{\"name\":\"Player$i\"}";
done
```

### Using Traces to Debug

1. **Find slow distributed calls**: Look for traces where GameServer → RollDice latency is high
2. **Find connection failures**: Search for `status.code="ERROR"` on gameserver service
3. **Analyze service dependencies**: Use service map to see request rates and error rates between services
4. **Compare healthy vs unhealthy**: Look at span attributes to understand what changed when errors occur
5. **Follow the trace**: Click through from GameServer span to RollDice span to see the full request flow

---

## Next Steps

- Explore metrics in Grafana Cloud
- Try the Java track with RollDice
- Add custom metrics with `Meter`
- Experiment with Activity events and baggage

---

## Troubleshooting

**Service won't start:**
```bash
# Check if port is in use
lsof -i :8090

# Kill existing process
pkill -f Gameserver
# Or: lsof -ti:8090 | xargs kill -9
```

**No traces in Grafana Cloud:**
```bash
# 1. Check Alloy is running and reachable
curl http://localhost:4318/v1/traces

# 2. Check OTEL_SERVICE_NAME is set in run.sh
cd /opt/gameserver && cat run.sh | grep OTEL_SERVICE_NAME

# Should show: export OTEL_SERVICE_NAME="gameserver"

# 3. Verify Program.cs uses signal-specific endpoints
cd /opt/gameserver && grep "v1/traces" Program.cs

# Should show: options.Endpoint = new Uri("http://localhost:4318/v1/traces");

# 4. Verify Alloy is configured with Grafana Cloud credentials
cd /opt/alloy && cat run.sh | grep GRAFANA_CLOUD

# 5. Test the telemetry pipeline - send a request and check Alloy logs
curl -X POST http://localhost:8090/game/play -H "Content-Type: application/json" -d '{"name":"Test"}'
```

**Important Note**: The .NET OTLP exporter v1.12.0 requires **signal-specific endpoint paths** (`/v1/traces`, `/v1/metrics`, `/v1/logs`). Using just `http://localhost:4318` without the signal path will silently fail.

**Build errors:**
```bash
# Check .NET version
dotnet --version  # Should be 8.0

# Clean build
cd /opt/gameserver
dotnet clean
dotnet build
```

**RollDice not responding:**
```bash
# GameServer needs RollDice running
cd /opt/rolldice
./run.sh

# Verify RollDice is running
curl http://localhost:8080/health
```
