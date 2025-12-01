# Lab 4: Add Custom Instrumentation

## 4.1 Understanding Custom Instrumentation

Automatic instrumentation captures framework-level telemetry (HTTP requests, database calls). Custom instrumentation lets you add business-specific context.

**What we'll add:**
- Custom spans for dice roll operations
- Span attributes for player names and roll values
- Span events for significant occurrences (streaks, slow responses)
- Custom metrics for roll distribution
- Error handling with span status

**Why do we need OpenTelemetryConfig?** The Java agent automatically instruments the framework, but to add custom spans and metrics, we need direct access to OpenTelemetry's `Tracer` and `Meter` objects. This configuration class makes them available via Spring dependency injection.

## 4.2 Create OpenTelemetry Configuration

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

## 4.3 Add Imports and Setup Tracer/Meter in Controller

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

## 4.4 Add Custom Span and Basic Attributes

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

## 4.5 Add Error Handling and Validation

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

## 4.6 Add Metrics and Advanced Instrumentation

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

## 4.7 Rebuild, Test, and View Results

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
