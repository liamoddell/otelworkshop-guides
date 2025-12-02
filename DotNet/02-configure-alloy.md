# Lab 2: Configure Alloy & Grafana Cloud

## What is Grafana Alloy?

Grafana Alloy is an OpenTelemetry collector that receives telemetry from your applications and forwards it to Grafana Cloud.

## Configure Alloy

1. **Get your Grafana Cloud credentials** from your workshop facilitator:
   - OTLP Endpoint (e.g., `https://otlp-gateway-prod-XX-XXX.grafana.net/otlp`)
   - Instance ID
   - API Token
  
   - As part of this workshop, your facilitator will showcase how to find these credentials. The basic steps will be:
     1. Head to Application Observability
     2. Click 'Connect Data'
     3. OpenTelemetry (OTLP)
     4. OpenTelemetry SDK
     5. Other
     6. Kubernetes
     7. OTel Collector
     8. Add your token name
     9. Scroll down to find the Endpoint, ID and generated API Token.

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
