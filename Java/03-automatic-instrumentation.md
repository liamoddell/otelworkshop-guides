# Lab 3: Add Automatic Instrumentation

## 3.1 Understanding Zero-Code Instrumentation

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

## 3.2 Stop RollDice and Configure Environment Variables

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

   **Important**: Replace `your-name-here` with your unique namespace (e.g., your name). The `service.namespace` must be unique for each student to avoid collisions in Grafana Cloud.

   **Example**:
   ```bash
   export OTEL_RESOURCE_ATTRIBUTES="service.name=rolldice,service.namespace=loddell,service.instance.id=${HOSTNAME},deployment.environment=workshop"
   ```

## 3.3 Attach the Java Agent

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

## 3.4 Generate Traffic and Verify

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
