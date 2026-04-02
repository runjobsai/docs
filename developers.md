# Developer API

Source: historical live content from `https://www.runjobs.ai/developers`

## Build with AI Agent Teams

Send tasks to AI agent teams and get structured results via REST API. Perfect for automation, trading bots, data pipelines, and any application that needs AI-powered analysis.

## How It Works

### 1. Open a Workspace

Use any workspace you own, including one created from the Project Library. Generate an API token from the workspace toolbar.

### 2. Send a Task

POST a message to your project's API endpoint. Optionally define a callback schema for structured results.

### 3. Get Results

Poll for results or use the synchronous endpoint. Callback results are returned as precise JSON matching your schema.

## Authentication

Each workspace can have a unique API token (starts with `rj_`). This includes standard workspaces and workspaces created from subscribed projects. Find it by clicking the `</>` button in the workspace toolbar.

Use the token directly in the URL path, no headers needed:

```text
POST https://www.runjobs.ai/v1/run/{your_token}/send
```

## Send Message (Async)

Send a task and get back a `message_id` immediately. The AI team processes it in the background.

```bash
curl -X POST https://www.runjobs.ai/v1/run/rj_abc123.../send \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Analyze BTC current trend and give trading advice"
  }'
```

```json
{
  "success": true,
  "data": {
    "message_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

## Get Result

Poll the result endpoint with your `message_id`. Status transitions: `pending -> processing -> done`.

```bash
curl https://www.runjobs.ai/v1/run/rj_abc123.../result/550e8400-...
```

```json
{
  "success": true,
  "data": {
    "status": "done",
    "message_id": "550e8400-...",
    "content": "Based on technical analysis, BTC shows..."
  }
}
```

## Callback (Structured Results)

This is the powerful part.

Define a callback schema and the AI agents will return results as precise JSON, no natural language parsing needed. The callback becomes a tool that any agent in the team can call.

```bash
curl -X POST https://www.runjobs.ai/v1/run/rj_abc123.../send \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Analyze BTC current trend, give a trading signal",
    "callback": {
      "description": "Return a trading signal",
      "properties": {
        "action": {
          "type": "string",
          "enum": ["buy", "sell", "hold"]
        },
        "symbol": { "type": "string" },
        "confidence": {
          "type": "number",
          "description": "0-1 confidence score"
        },
        "reason": { "type": "string" }
      },
      "required": ["action", "symbol", "confidence", "reason"]
    }
  }'
```

```json
{
  "success": true,
  "data": {
    "status": "done",
    "message_id": "550e8400-...",
    "content": "Based on RSI oversold bounce and on-chain signals...",
    "callback_result": {
      "action": "buy",
      "symbol": "BTC",
      "confidence": 0.82,
      "reason": "RSI oversold bounce + large on-chain buy signals"
    }
  }
}
```

How callbacks work:

- Your callback schema becomes a temporary tool (prefixed `callback_`)
- All agents in the workspace can see and call it
- Works with multi-agent teams: PM delegates -> analyst executes -> callback returns
- One-time: disappears after being called

## Synchronous Chat

If you don't want to poll, use the sync endpoint. It waits for the agent to finish and returns the result directly.

```bash
curl -X POST https://www.runjobs.ai/v1/run/rj_abc123.../chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What is the current BTC price trend?",
    "timeout": 300,
    "callback": {
      "description": "Return current price trend",
      "properties": {
        "trend": { "type": "string", "enum": ["bullish", "bearish", "neutral"] },
        "price_usd": { "type": "number" }
      },
      "required": ["trend", "price_usd"]
    }
  }'
```

Timeout notes:

- `timeout` is optional
- Default: `120`
- Max: `600` (10 minutes)
- If the agent finishes within the timeout, the full result is returned
- Otherwise you get `status: "timeout"` with a `message_id` to poll later via `/result/:id`

## Rate Limits

| Limit | Value |
| --- | --- |
| Requests per token | 1 per second (burst: 2) |
| Sync chat timeout | Default 120s, max 600s (10 min). Set via `timeout` param. |
| Billing | Pay-per-use, same as UI (AI tokens consumed) |

Exceeding the rate limit returns `429 Too Many Requests`. When polling results, wait at least 2-5 seconds between requests.

## Examples

### Python - Trading Bot

```python
import requests, time

TOKEN = "rj_your_token_here"
BASE = f"https://www.runjobs.ai/v1/run/{TOKEN}"

# Send analysis request with callback
resp = requests.post(f"{BASE}/send", json={
    "message": "Analyze BTC/USDT for short-term trading",
    "callback": {
        "description": "Return a trading signal",
        "properties": {
            "action": {"type": "string", "enum": ["buy", "sell", "hold"]},
            "amount_pct": {"type": "number", "description": "% of portfolio"},
            "confidence": {"type": "number"},
        },
        "required": ["action", "amount_pct", "confidence"]
    }
})
msg_id = resp.json()["data"]["message_id"]

# Poll for result
while True:
    result = requests.get(f"{BASE}/result/{msg_id}").json()["data"]
    if result["status"] == "done":
        signal = result["callback_result"]
        print(f"Action: {signal['action']}, Amount: {signal['amount_pct']}%")
        # Execute trade...
        break
    time.sleep(5)
```

### Python - Sync Mode

```python
import requests

TOKEN = "rj_your_token_here"

result = requests.post(
    f"https://www.runjobs.ai/v1/run/{TOKEN}/chat",
    json={"message": "Summarize today's crypto news"},
    timeout=130
).json()

print(result["data"]["content"])
```

## Ready to Build?

Open a workspace, generate a token, and start integrating in minutes.

---

GitHub: <https://github.com/runjobs/docs>
