# Troubleshooting

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
