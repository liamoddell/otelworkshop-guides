# Error Scenarios for Testing

The RollDice service includes intentional error scenarios to help you practice troubleshooting with OpenTelemetry:

## Scenario 1: Slow Responses
**Trigger**: Every 15th request
**Behavior**: 2-second delay before responding
**What to look for**:
- High latency in traces
- `request.is_slow` span attribute set to `true`
- `slow_response_simulated` span event with delay details

**Testing**:
```bash
for i in {1..20}; do curl "http://localhost:8080/rolldice?player=Alice"; done
```

## Scenario 2: Input Validation Error
**Trigger**: Player name longer than 50 characters
**Behavior**: Returns 400 Bad Request
**What to look for**:
- Span status set to ERROR
- `error.type=validation_error` attribute
- HTTP 400 response code in traces

**Testing**:
```bash
curl "http://localhost:8080/rolldice?player=ThisIsAVeryLongPlayerNameThatExceedsFiftyCharactersInLength"
```

## Scenario 3: Dice Malfunction
**Trigger**: Every 200th request
**Behavior**: Returns 500 Internal Server Error
**What to look for**:
- Span status set to ERROR with "Dice malfunction" message
- `error.type=dice_malfunction` attribute
- `dice_malfunction` span event with severity "high"
- HTTP 500 response code

**Testing**:
```bash
# Use the load test to generate enough traffic
cd /opt/loadtests
k6 run --duration 1m --vus 5 rolldice-loadtest.js
```

## Scenario 4: Streak Detection
**Trigger**: Rolling the same value multiple times in a row
**Behavior**: Normal operation, but adds telemetry
**What to look for**:
- `dice.is_streak` attribute set to true
- `dice.streak_count` attribute showing count
- `significant_streak_detected` span event (for streaks >= 3)

**Testing**:
```bash
# Roll multiple times and watch for streaks
for i in {1..10}; do curl "http://localhost:8080/rolldice?player=Bob"; done
```

## Using Traces to Debug

1. **Find slow requests**: Filter by latency > 1s in Application Observability
2. **Find validation errors**: Search for `error.type="validation_error"`
3. **Find malfunctions**: Search for `status.code="ERROR" && error.type="dice_malfunction"`
4. **Analyze patterns**: Use service metrics to spot anomalies in request rate or error rate
