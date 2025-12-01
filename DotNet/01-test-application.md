# Lab 1: Test Your Application

## Understanding GameServer

GameServer is an ASP.NET Core Web API that manages a dice game. It calls the RollDice service to get dice rolls for players.

**Important**: GameServer depends on RollDice service. You'll need to start RollDice first.

## Start RollDice Service

```bash
cd /opt/rolldice
./run.sh
```

## Start GameServer

```bash
cd /opt/gameserver
./run.sh
```

**Test the endpoints:**
```bash
# Play a game (POST with JSON body)
curl -X POST http://localhost:8090/game/play \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice"}'

# Get game statistics
curl http://localhost:8090/game/stats

# Health check
curl http://localhost:8090/game/health
```

**Expected response:**
```json
{"playerName":"Alice","playerRoll":4,"computerRoll":3,"result":"Player wins!"}
```

The service runs on **port 8090**.
