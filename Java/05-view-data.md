# Lab 5: View Data in Grafana Cloud

## 5.1 Explore Application Observability

1. **Navigate to Application Observability** in Grafana Cloud

2. **Service Inventory:**
   - Filter by `environment=workshop`
   - Filter by `service.namespace=your-name-here`
   - Find your `rolldice` service

3. **Service Overview:**
   - Request rate
   - Error rate
   - Latency (p50, p95, p99)

## 5.2 Explore Traces

1. **Click on your service**
2. **View Traces**
3. **Find a trace and expand it:**
   - See the automatic HTTP instrumentation
   - See your custom `roll_dice_request` span
   - View span attributes

## 5.3 Search by Attributes

```
service.name="rolldice" && player.name="Alice"
```

## 5.4 Find Errors

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

## Next Steps

- Explore metrics in Grafana Cloud
- Try the .NET track with GameServer
- Add custom metrics with `Meter`
- Experiment with span events and links
