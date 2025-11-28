# OpenTelemetry Workshop

This repository contains the sexercises for the Grafana Labs Introduction to OpenTelemetry workshop (forked for Java and C# users).

## Workshop Overview

This hands-on workshop teaches OpenTelemetry fundamentals through progressive labs using **Java and .NET backend services**. You'll learn:

- Zero-code instrumentation (Java agent)
- Manual SDK instrumentation (C# .NET)
- Custom spans and span attributes
- Custom metrics with business context
- Distributed tracing across services
- Integration with Grafana Cloud
- Error troubleshooting with telemetry

## Architecture

```
k6 Load Test → GameServer:8090 (C#/.NET) → RollDice:8080 (Java)
                         ↓ OTLP
                  Alloy:4317/4318 → Grafana Cloud
```

### Services

1. **RollDice** (Java Spring Boot) - Demonstrates zero-code instrumentation with Java agent
2. **GameServer** (C# .NET ASP.NET Core) - Demonstrates manual SDK instrumentation
3. **Grafana Alloy** - OpenTelemetry collector with production-ready configuration

## Workshop Structure

Participants follow a **language-specific track** (Java or .NET) with 5 progressive labs:

### Lab 1: Test Your Application (15 minutes)
- Understand the service architecture
- Test endpoints independently
- Verify services are working correctly

### Lab 2: Configure Alloy & Grafana Cloud (15 minutes)
- Configure Grafana Alloy with OTLP exporter
- Set up Grafana Cloud credentials
- Understand telemetry pipeline

### Lab 3: Add Basic Instrumentation (30 minutes)

**Java Track:**
- Attach OpenTelemetry Java agent (zero-code)
- Configure environment variables
- Send telemetry to Grafana Cloud via Alloy
- View automatic instrumentation

**C# .NET Track:**
- Configure OpenTelemetry SDK in Program.cs
- Add tracing and metrics instrumentation
- Send telemetry to Grafana Cloud via Alloy
- View distributed traces and service maps

### Lab 4: Add Custom Instrumentation (45 minutes)

**Java Track:**
- Create OpenTelemetryConfig for Tracer and Meter beans
- Add custom spans for business logic
- Add span attributes and events
- Create custom metrics

**C# .NET Track:**
- Create custom spans with `ActivitySource`
- Add business-specific metrics with `Meter`
- Add span attributes and events
- Troubleshoot with telemetry

### Lab 5: View Data in Grafana Cloud (15 minutes)
- Explore Application Observability
- View distributed traces
- Search by span attributes
- Find and debug errors
- Analyze service dependencies

## Getting Started

### Prerequisites

- Modern web browser with JavaScript enabled
- Access to Grafana Cloud instance (provided by workshop facilitator)
- Workshop credentials (sent via email)

### Workshop Environment

All services run in a single Eclipse Theia IDE container, accessible via web browser. No local installation required!

### Workshop Documentation

Track-specific guides are included in the repository:

- **Java Track**: [JAVA-WORKSHOP.md](JAVA-WORKSHOP.md) - Complete guide for RollDice service
- **.NET Track**: [DOTNET-WORKSHOP.md](DOTNET-WORKSHOP.md) - Complete guide for GameServer service

Each guide includes:
- Step-by-step lab instructions
- Code examples and snippets
- Error scenarios for troubleshooting practice
- Tips and troubleshooting sections

## Template Philosophy

**Source templates** (`source/gameserver`, `source/rolldice`):
- ✅ Working applications without OpenTelemetry code
- ✅ OpenTelemetry packages pre-installed (no `mvn install` or `dotnet add package` needed)
- ✅ Students add instrumentation code themselves during labs
- ✅ Simulates instrumenting existing applications

**Reference implementations** (`source/*-reference`):
- ✅ Complete working solutions
- ✅ Full OpenTelemetry instrumentation
- ✅ Available when students get stuck
- ✅ Show best practices

## Key Features

### Zero-Code vs. SDK Approach

**Java (RollDice):**
- ✅ Zero-code instrumentation with Java agent
- ✅ No code changes required for basic instrumentation
- ✅ Perfect for legacy applications
- ✅ Auto-instruments frameworks (Spring Boot, HTTP clients, JDBC)
- ✅ Add custom instrumentation via OpenTelemetry API

**C# (.NET) (GameServer):**
- ✅ Manual SDK instrumentation
- ✅ Full control over telemetry
- ✅ Custom metrics and spans via Activity API
- ✅ Ideal for greenfield projects
- ✅ Native .NET diagnostics integration

### Error Scenarios for Learning

Both services include intentional error scenarios to practice troubleshooting:

**RollDice Service:**
- Slow responses (every 15th request)
- Input validation errors (player name > 50 chars)
- Dice malfunction (every 200th request)
- Streak detection (telemetry feature)

**GameServer Service:**
- Distributed tracing with slow downstream calls
- Service unavailable scenarios
- Request timeouts
- Invalid dice value handling

### Production-Ready Patterns

- **Service-based architecture** with dependency injection
- **OTLP protocol** for telemetry export (HTTP/Protobuf)
- **Resource attributes** for service identification
- **Grafana Alloy** for telemetry collection and transformation
- **Custom instrumentation** for business metrics
- **Distributed tracing** with W3C trace context propagation
- **Error handling** with span status and exception recording

## Technology Stack

- **Java:** 17 with Spring Boot 3.3.1
- **.NET:** 8.0 with ASP.NET Core
- **OpenTelemetry:**
  - Java agent (v2.15.0) for automatic instrumentation
  - Java API (v1.44.1) for custom instrumentation
  - .NET SDK (v1.12.0) for manual instrumentation
- **Grafana Alloy:** 1.8.3
- **k6:** v0.59.0 for load testing
- **Eclipse Theia:** 1.60.100 for web-based IDE

## Load Testing

All load tests are in the `/opt/loadtests` directory with comprehensive documentation.

**Quick commands:**

```bash
# Java track
cd /opt/loadtests
k6 run --duration 2m --vus 2 rolldice-loadtest.js

# .NET track
cd /opt/loadtests
k6 run --duration 2m --vus 2 gameserver-loadtest.js

# Both tracks (comprehensive)
cd /opt/loadtests
k6 run --duration 5m --vus 3 workshop-loadtest.js
```

See [loadtests/README.md](loadtests/README.md) for detailed documentation.

## Development

### Template Structure

Templates are designed for workshop participants to add instrumentation:

**Java Template** (`source/rolldice`):
- Spring Boot application without OTel code
- OpenTelemetry API dependency in pom.xml
- Students create `config/OpenTelemetryConfig.java`
- Students add custom spans in `RolldiceController.java`

**.NET Template** (`source/gameserver`):
- ASP.NET Core application without OTel code
- OpenTelemetry NuGet packages in Gameserver.csproj
- Students configure SDK in `Program.cs`
- Students add custom spans in `GameController.cs`

## License

Copyright © Grafana Labs

## Support

For questions during the workshop:
- Ask your facilitator
- Check the troubleshooting sections in workshop guides
- Refer to reference implementations in `source/*-reference/`

## Additional Resources

- [OpenTelemetry Java Instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation)
- [OpenTelemetry .NET](https://github.com/open-telemetry/opentelemetry-dotnet)
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/)
- [Grafana Cloud Application Observability](https://grafana.com/docs/grafana-cloud/monitor-applications/)

---

**Workshop Focus:** Java + .NET Backend Monitoring with OpenTelemetry
**Format:** Language-specific tracks (Java or .NET)
**Approach:** Zero-code (Java) vs. Manual SDK (.NET)
**Deployment:** Single Eclipse Theia container
**Documentation:** Track-specific guides (JAVA-WORKSHOP.md, DOTNET-WORKSHOP.md)
