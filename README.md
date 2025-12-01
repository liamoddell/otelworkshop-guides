# OpenTelemetry Workshop

This repository contains the exercises for the Grafana Labs Introduction to OpenTelemetry workshop (forked for Java and C# users).

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

Participants follow a **language-specific track** (Java or .NET) with 5 progressive labs.

**Note**: Each lab has been broken down into smaller, more consumable subsections to make it easier to follow along and complete the workshop at your own pace. You can tackle each subsection independently and take breaks between them.

### Lab 1: Test Your Application (15 minutes)
- Understand the service architecture
- Test endpoints independently
- Verify services are working correctly

### Lab 2: Configure Alloy & Grafana Cloud (15 minutes)
- Configure Grafana Alloy with OTLP exporter
- Set up Grafana Cloud credentials
- Understand telemetry pipeline

### Lab 3: Add Basic Instrumentation (30 minutes)

**Java Track** (broken into 4 sub-sections):
- 3.1: Understanding zero-code instrumentation
- 3.2: Configure environment variables
- 3.3: Attach OpenTelemetry Java agent
- 3.4: Generate traffic and verify in Grafana Cloud

**C# .NET Track** (broken into 5 sub-sections):
- 3.1: Understanding manual SDK instrumentation
- 3.2: Stop GameServer and add using statements
- 3.3: Configure OpenTelemetry in Program.cs
- 3.4: Configure service identification
- 3.5: Generate traffic and verify in Grafana Cloud

### Lab 4: Add Custom Instrumentation (45 minutes)

**Java Track** (broken into 7 sub-sections):
- 4.1: Understanding custom instrumentation
- 4.2: Create OpenTelemetryConfig
- 4.3: Add imports and setup Tracer/Meter
- 4.4: Add custom span and basic attributes
- 4.5: Add error handling and validation
- 4.6: Add metrics and advanced instrumentation
- 4.7: Rebuild, test, and view results

**C# .NET Track** (broken into 6 sub-sections):
- 4.1: Understanding custom instrumentation
- 4.2: Setup - Add using statements to GameService
- 4.3: Add custom span and attributes to PlayGameAsync
- 4.4: Add game result attributes and metrics
- 4.5: Instrument GameController GetStats endpoint
- 4.6: Rebuild, test, and view results

### Lab 5: View Data in Grafana Cloud (15 minutes)

**Both Tracks** (broken into subsections for easier navigation):
- 5.1: Explore Application Observability (service inventory, overview)
- 5.2: Explore distributed traces
- 5.3: View service dependencies (for .NET track) / Explore traces (for Java track)
- 5.4: Search by span attributes
- 5.5: Find and debug errors

## Getting Started

### Prerequisites

- Modern web browser with JavaScript enabled
- Access to Grafana Cloud instance (provided by workshop facilitator)
- Workshop credentials (sent via email)

### Workshop Environment

All services run in a single Eclipse Theia IDE container, accessible via web browser. No local installation required!

### Workshop Documentation

Track-specific guides are organized into logical sub-files within each track directory:

#### **Java Track** (`Java/`)
Complete guide for RollDice service, broken down into:
- [00-overview.md](Java/00-overview.md) - Workshop introduction
- [01-test-application.md](Java/01-test-application.md) - Lab 1: Test Your Application
- [02-configure-alloy.md](Java/02-configure-alloy.md) - Lab 2: Configure Alloy & Grafana Cloud
- [03-automatic-instrumentation.md](Java/03-automatic-instrumentation.md) - Lab 3: Add Automatic Instrumentation
- [04-custom-instrumentation.md](Java/04-custom-instrumentation.md) - Lab 4: Add Custom Instrumentation
- [05-view-data.md](Java/05-view-data.md) - Lab 5: View Data in Grafana Cloud
- [06-error-scenarios.md](Java/06-error-scenarios.md) - Error Scenarios for Testing
- [07-troubleshooting.md](Java/07-troubleshooting.md) - Troubleshooting

#### **.NET Track** (`DotNet/`)
Complete guide for GameServer service, broken down into:
- [00-overview.md](DotNet/00-overview.md) - Workshop introduction
- [01-test-application.md](DotNet/01-test-application.md) - Lab 1: Test Your Application
- [02-configure-alloy.md](DotNet/02-configure-alloy.md) - Lab 2: Configure Alloy & Grafana Cloud
- [03-sdk-instrumentation.md](DotNet/03-sdk-instrumentation.md) - Lab 3: Add SDK Instrumentation
- [04-custom-instrumentation.md](DotNet/04-custom-instrumentation.md) - Lab 4: Add Custom Instrumentation
- [05-view-data.md](DotNet/05-view-data.md) - Lab 5: View Data in Grafana Cloud
- [06-error-scenarios.md](DotNet/06-error-scenarios.md) - Error Scenarios for Testing
- [07-troubleshooting.md](DotNet/07-troubleshooting.md) - Troubleshooting

Each file includes:
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
**Documentation:** Track-specific guides organized in `Java/` and `DotNet/` directories
