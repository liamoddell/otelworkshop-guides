# OpenTelemetry Workshop - Java Track

**Service**: RollDice (Spring Boot)
**Instrumentation**: Zero-code with Java Agent
**Duration**: ~2 hours

---

## Lab 1: Test Your Application

### Understanding RollDice

RollDice is a Spring Boot REST API that simulates dice rolls. It's a standalone service that other services can call.

**Start the service:**
```bash
cd /opt/rolldice
./run.sh
```

**Test the endpoints:**
```bash
# Roll dice for a player
curl http://localhost:8080/rolldice?player=Alice

# Get statistics
curl http://localhost:8080/stats

# Health check
curl http://localhost:8080/health
```

**Expected response:**
```json
{"player":"Alice","roll":4}
```

The service runs on **port 8080**.

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

## Lab 3: Add Automatic Instrumentation

### 3.1 Understanding Zero-Code Instrumentation

The Java agent automatically instruments your application without code changes. It intercepts your application at runtime and adds OpenTelemetry instrumentation.

**What we'll do:**
- Configure environment variables for OTLP export
- Set resource attributes for service identification
- Attach the Java agent to the application
- Verify telemetry in Grafana Cloud

**Benefits:**
- No code changes required for basic instrumentation
- Perfect for legacy applications
- Auto-instruments frameworks (Spring Boot, HTTP clients, JDBC)

### 3.2 Stop RollDice and Configure Environment Variables

1. **Stop RollDice if running:**
   ```bash
   pkill -f "java.*rolldice"
   ```

2. **Edit the run script:**
   ```bash
   nano /opt/rolldice/run.sh
   ```

3. **Add before the `java -jar` line:**
   ```bash
   # Configure OpenTelemetry
   export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"
   export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
   export OTEL_RESOURCE_ATTRIBUTES="service.name=rolldice,service.namespace=your-name-here,service.instance.id=${HOSTNAME},deployment.environment=workshop"
   ```

   **Important**: Replace `your-name-here` with your unique namespace (e.g., your name). The `service.namespace` must be unique for each user to avoid collisions in Grafana Cloud.

   **Example**:
   ```bash
   export OTEL_RESOURCE_ATTRIBUTES="service.name=rolldice,service.namespace=loddell,service.instance.id=${HOSTNAME},deployment.environment=workshop"
   ```

### 3.3 Attach the Java Agent

1. **Modify the `java -jar` command to attach the agent:**

   Find the line:
   ```bash
   java -jar ./target/rolldice-0.0.1-SNAPSHOT.jar
   ```

   Replace it with:
   ```bash
   java -javaagent:./opentelemetry-javaagent.jar -jar ./target/rolldice-0.0.1-SNAPSHOT.jar
   ```

2. **Save and start RollDice:**
   ```bash
   cd /opt/rolldice
   ./run.sh
   ```

### 3.4 Generate Traffic and Verify

1. **Generate traffic:**
```bash
# Run load test
cd /opt/loadtests
k6 run --duration 2m --vus 2 rolldice-loadtest.js
```

2. **Verify in Grafana Cloud:**
   - Open Grafana Cloud (URL provided by facilitator)
   - Navigate to **Application Observability**
   - Look for your service: `rolldice` with namespace `your-name-here`
   - You should see:
     - HTTP requests
     - Service map
     - Traces

---

## Lab 4: Add Custom Instrumentation

### 4.1 Understanding Custom Instrumentation

Automatic instrumentation captures framework-level telemetry (HTTP requests, database calls). Custom instrumentation lets you add business-specific context.

**What we'll add:**
- Custom spans for dice roll operations
- Span attributes for player names and roll values
- Span events for significant occurrences (streaks, slow responses)
- Custom metrics for roll distribution
- Error handling with span status

**Why do we need OpenTelemetryConfig?** The Java agent automatically instruments the framework, but to add custom spans and metrics, we need direct access to OpenTelemetry's `Tracer` and `Meter` objects. This configuration class makes them available via Spring dependency injection.

### 4.2 Create OpenTelemetry Configuration

1. **Create config directory:**
   ```bash
   mkdir -p /opt/rolldice/src/main/java/com/example/rolldice/config
   ```

2. **Create OpenTelemetryConfig.java:**
   ```bash
   nano /opt/rolldice/src/main/java/com/example/rolldice/config/OpenTelemetryConfig.java
   ```

3. **Add configuration:**
   ```java
   package com.example.rolldice.config;

   import io.opentelemetry.api.GlobalOpenTelemetry;
   import io.opentelemetry.api.metrics.Meter;
   import io.opentelemetry.api.trace.Tracer;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;

   @Configuration
   public class OpenTelemetryConfig {

       @Bean
       public Tracer tracer() {
           return GlobalOpenTelemetry.getTracer("rolldice", "1.0.0");
       }

       @Bean
       public Meter meter() {
           return GlobalOpenTelemetry.getMeter("rolldice");
       }
   }
   ```

### 4.3 Add Imports and Setup Tracer/Meter in Controller

1. **Edit RolldiceController.java:**
   ```bash
   nano /opt/rolldice/src/main/java/com/example/rolldice/RolldiceController.java
   ```

2. **Add imports at the top of the file:**
   ```java
   import io.opentelemetry.api.trace.Tracer;
   import io.opentelemetry.api.trace.Span;
   import io.opentelemetry.api.trace.StatusCode;
   import io.opentelemetry.context.Scope;
   import io.opentelemetry.api.metrics.Meter;
   import io.opentelemetry.api.metrics.LongCounter;
   import io.opentelemetry.api.common.Attributes;

   import static io.opentelemetry.api.common.AttributeKey.longKey;
   import static io.opentelemetry.api.common.AttributeKey.stringKey;
   ```

3. **Inject Tracer and Meter using constructor** (add after the existing @Autowired fields):
   ```java
   private final Tracer tracer;
   private final LongCounter rollsByValueCounter;

   public RolldiceController(Tracer tracer, Meter meter) {
       this.tracer = tracer;

       // Create custom counter for rolls by value
       this.rollsByValueCounter = meter
               .counterBuilder("dice.rolls.by_value")
               .setDescription("Number of rolls by dice value")
               .setUnit("{roll}")
               .build();
   }
   ```

### 4.4 Add Custom Span and Basic Attributes

1. **Add custom span at the start of rollDice method.** Find the line `String playerName = player.orElse("anonymous");` and add BEFORE it:
   ```java
   // Create custom span for the roll request
   Span span = tracer.spanBuilder("roll_dice_request").startSpan();
   try (Scope scope = span.makeCurrent()) {
   ```

2. **Add span attribute for player name.** Right after the `String playerName = player.orElse("anonymous");` line, add:
   ```java
       // Add span attribute for player name
       span.setAttribute("player.name", playerName);
   ```

### 4.5 Add Error Handling and Validation

1. **Add error handling for validation.** Find the validation error block (playerName.length() > 50) and add span status BEFORE the return statement:
   ```java
   // Record error in span
   span.setStatus(StatusCode.ERROR, "Player name too long");
   span.setAttribute("error.type", "validation_error");
   ```

2. **Add error handling for dice malfunction.** Find the dice malfunction error block (currentRequest % 200 == 0) and add BEFORE the return statement:
   ```java
   span.setStatus(StatusCode.ERROR, "Dice malfunction");
   span.setAttribute("error.type", "dice_malfunction");
   span.addEvent("dice_malfunction", Attributes.of(stringKey("severity"), "high"));
   ```

### 4.6 Add Metrics and Advanced Instrumentation

1. **Add span event for slow responses.** Find the slow response simulation (Thread.sleep) and add AFTER it:
   ```java
   // Add span attribute and event for slow request
   span.setAttribute("request.is_slow", true);
   span.addEvent("slow_response_simulated",
       Attributes.of(longKey("delay_ms"), 2000L));
   ```

2. **Add span attribute and custom metric for dice roll.** Find the line `int roll = getRandomNumber(1, 6);` and add AFTER it:
   ```java
   // Add span attribute for roll value
   span.setAttribute("dice.roll_value", roll);

   // Record roll in custom metric
   rollsByValueCounter.add(1,
       Attributes.of(
           longKey("roll_value"), (long) roll,
           stringKey("player.name"), playerName
       ));
   ```

3. **Add span attributes for streaks.** Find the streak detection block (`if (isStreak)`) and add INSIDE the if block:
   ```java
   // Add span attributes for streak
   span.setAttribute("dice.is_streak", true);
   span.setAttribute("dice.streak_count", streakCount);

   // Add significant event for notable streaks
   if (streakCount >= 3) {
       span.addEvent("significant_streak_detected",
           Attributes.of(
               longKey("streak_count"), (long) streakCount,
               longKey("roll_value"), (long) roll));
   }
   ```

4. **Close the span at the end of the method.** At the very end of the rollDice method, BEFORE the final closing brace, add:
    ```java
        } finally {
            span.end();
        }
    ```

### 4.7 Rebuild, Test, and View Results

1. **Rebuild and restart:**
    ```bash
    cd /opt/rolldice
    mvn clean package -DskipTests
    pkill -f "java.*rolldice"
    ./run.sh
    ```

2. **Generate traffic:**
   ```bash
   cd /opt/loadtests
   k6 run --duration 2m --vus 2 rolldice-loadtest.js
   ```

3. **View Custom Instrumentation in Grafana Cloud:**

   You should see:

   **Custom span** `roll_dice_request` with multiple attributes:
   - `player.name`
   - `dice.roll_value`
   - `request.is_slow` (on slow requests)
   - `dice.is_streak` (on streaks)
   - `dice.streak_count` (on streaks)
   - `error.type` (on errors)

   **Span events:**
   - `slow_response_simulated` with delay details
   - `significant_streak_detected` with streak info
   - `dice_malfunction` on errors

   **Custom metrics** in Explore → Metrics:
   - Search for `dice.rolls.by_value` to see roll distribution by value
   - Note: Counters are displayed as rates (c/s) in Grafana Cloud by default - this is expected behavior

**Tip:** Use `/opt/rolldice-reference` for the complete implementation.

---

## Lab 5: View Data in Grafana Cloud

### 5.1 Explore Application Observability

1. **Navigate to Application Observability** in Grafana Cloud

2. **Service Inventory:**
   - Filter by `environment=workshop`
   - Filter by `service.namespace=your-name-here`
   - Find your `rolldice` service

3. **Service Overview:**
   - Request rate
   - Error rate
   - Latency (p50, p95, p99)

### 5.2 Explore Traces

1. **Click on your service**
2. **View Traces**
3. **Find a trace and expand it:**
   - See the automatic HTTP instrumentation
   - See your custom `roll_dice_request` span
   - View span attributes

### 5.3 Search by Attributes

```
service.name="rolldice" && player.name="Alice"
```

### 5.4 Find Errors

If you generated error scenarios in the load test:
```
service.name="rolldice" && status.code="ERROR"
```

---

## Summary - What You Learned

✅ **Zero-code instrumentation** with Java agent
✅ **OTLP export** to Grafana Alloy
✅ **Custom spans** for business logic
✅ **Span attributes** for context
✅ **Distributed tracing** visualization

---

## Error Scenarios for Testing

The RollDice service includes intentional error scenarios to help you practice troubleshooting with OpenTelemetry:

### Scenario 1: Slow Responses
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

### Scenario 2: Input Validation Error
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

### Scenario 3: Dice Malfunction
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

### Scenario 4: Streak Detection
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

### Using Traces to Debug

1. **Find slow requests**: Filter by latency > 1s in Application Observability
2. **Find validation errors**: Search for `error.type="validation_error"`
3. **Find malfunctions**: Search for `status.code="ERROR" && error.type="dice_malfunction"`
4. **Analyze patterns**: Use service metrics to spot anomalies in request rate or error rate

---

## Next Steps

- Explore metrics in Grafana Cloud
- Try the .NET track with GameServer
- Add custom metrics with `Meter`
- Experiment with span events and links

---

## Troubleshooting

**Service won't start:**
```bash
# Check if port is in use
lsof -i :8080

# Kill existing process
pkill -f "java.*rolldice"
```

**No traces in Grafana Cloud:**
```bash
# Check Alloy is running
curl http://localhost:4318/v1/traces

# Check environment variables
cd /opt/rolldice && source run.sh && env | grep OTEL
```

**Build errors:**
```bash
# Check Java version
java -version  # Should be 17

# Clean build
cd /opt/rolldice
mvn clean
mvn package -DskipTests
```
