# Troubleshooting

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
