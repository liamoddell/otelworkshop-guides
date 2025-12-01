# Lab 1: Test Your Application

## Understanding RollDice

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
