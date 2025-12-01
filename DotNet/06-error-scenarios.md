# Error Scenarios for Testing

The GameServer service includes intentional error scenarios to help you practice troubleshooting with distributed tracing:

## Scenario 1: RollDice Service Slow Responses
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

## Scenario 2: RollDice Service Unavailable
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

## Scenario 3: Request Timeout
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

## Scenario 4: Invalid Dice Values
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

## Scenario 5: Distributed Trace Visualization
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

## Using Traces to Debug

1. **Find slow distributed calls**: Look for traces where GameServer → RollDice latency is high
2. **Find connection failures**: Search for `status.code="ERROR"` on gameserver service
3. **Analyze service dependencies**: Use service map to see request rates and error rates between services
4. **Compare healthy vs unhealthy**: Look at span attributes to understand what changed when errors occur
5. **Follow the trace**: Click through from GameServer span to RollDice span to see the full request flow
