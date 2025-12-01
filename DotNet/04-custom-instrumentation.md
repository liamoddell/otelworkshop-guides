# Lab 4: Add Custom Instrumentation

## 4.1 Understanding Custom Instrumentation

Automatic instrumentation captures framework-level telemetry (HTTP requests, HttpClient calls). Custom instrumentation lets you add business-specific context like game outcomes, player names, and custom metrics.

**Why System.Diagnostics?** This namespace provides .NET's native distributed tracing API (`ActivitySource`, `Activity`, `ActivityKind`) that OpenTelemetry uses under the hood. For metrics, we use `System.Diagnostics.Metrics` which provides `Meter` and `Counter` types.

**What we'll add:**
- Custom spans for game play operations
- Span attributes for game details (player name, rolls, results)
- Custom metrics (games started, games completed)
- Error handling with span status

## 4.2 Setup: Add Using Statements to GameService

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

## 4.3 Add Custom Span and Attributes to PlayGameAsync

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

## 4.4 Add Game Result Attributes and Metrics

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

## 4.5 Instrument GameController GetStats Endpoint

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

## 4.6 Rebuild, Test, and View Results

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
