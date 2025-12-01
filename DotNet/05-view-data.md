# Lab 5: View Data in Grafana Cloud

## 5.1 Explore Application Observability

1. **Navigate to Application Observability** in Grafana Cloud

2. **Service Inventory:**
   - Filter by `environment=workshop`
   - Filter by `service.namespace=your-name-here`
   - Find your `gameserver` service

3. **Service Overview:**
   - Request rate
   - Error rate
   - Latency (p50, p95, p99)

## 5.2 Explore Distributed Traces

1. **Click on your service**
2. **View Traces**
3. **Find a trace and expand it:**
   - See the automatic HTTP instrumentation (ASP.NET Core)
   - See the automatic HttpClient instrumentation (gameserver → rolldice)
   - See your custom `play_game` span
   - View span attributes and timing

## 5.3 View Service Dependencies

1. **Service Map:**
   - See gameserver calling rolldice
   - View request rates and latencies between services

## 5.4 Search by Attributes

```
service.name="gameserver" && player.name="Alice"
```

## 5.5 Find Errors

If you generated error scenarios in the load test:
```
service.name="gameserver" && status.code="ERROR"
```

---

## Summary - What You Learned

✅ **Manual SDK instrumentation** with OpenTelemetry .NET
✅ **OTLP export** to Grafana Alloy
✅ **Custom spans** with ActivitySource
✅ **Span attributes** for business context
✅ **Distributed tracing** across services (gameserver → rolldice)

---

## Next Steps

- Explore metrics in Grafana Cloud
- Try the Java track with RollDice
- Add custom metrics with `Meter`
- Experiment with Activity events and baggage
