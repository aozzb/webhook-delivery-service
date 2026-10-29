# Demo runbook — show the webhook delivery service live

A step-by-step guide to run the whole project on a laptop and demonstrate it to
someone (e.g. in an interview). Everything here is runtime/config only — no code
changes. Commands are given for **Git Bash** (recommended) and **PowerShell**.

The demo tells one story: *the receiver is deliberately unreliable; the system
retries with backoff, recovers most deliveries automatically, parks the hopeless
ones in a dead-letter queue, and lets you replay them — nothing is lost or
duplicated.*

---

## 0. Prerequisites

- **Docker Desktop** installed and **running** (start the app, wait until it says
  "running"). This is the only thing that isn't a terminal command.
- A `.env` file in the project root with `POSTGRES_PASSWORD=<something>` (already
  present). Postgres locks this password in the first time its data volume is
  created; changing it later won't match an existing volume (see Troubleshooting).
- A terminal open in the project folder:
  ```bash
  cd /c/Users/BIT/Desktop/proj1
  ```

---

## 1. Start the whole stack

```bash
# Fresh, clean slate for the demo (wipes any old data):
docker compose down -v

# Build images (first run ~2-3 min) and start all services:
docker compose up -d --build

# Create the database schema:
docker compose run --rm migrate

# Sanity checks:
docker compose ps                   # 6 services "Up"; postgres/redis "(healthy)"
curl http://localhost:3000/health   # -> {"status":"ok","checks":{...ok...}}
```

The six services are: `api` (HTTP API + dashboard + metrics), `worker` (delivers
webhooks), `scheduler` (reconciling sweeper), `postgres`, `redis`, and
`chaos-receiver` (a deliberately flaky fake customer endpoint for the demo).

On a normal day (data already created) you can skip `down -v` and the migrate
step — just `docker compose up -d`.

---

## 2. Seed the demo (register app → endpoint → publish events)

### Git Bash

```bash
# Register an app and capture its API key into a variable:
KEY=$(curl -s -XPOST http://localhost:3000/applications \
  -H "content-type: application/json" -d '{"name":"demo"}' \
  | sed -n 's/.*"api_key":"\([^"]*\)".*/\1/p')

echo "$KEY"      # <-- your whsk_... key; copy it for the dashboard

# Create an endpoint pointing at the built-in flaky receiver,
# subscribed to events of type "demo.event":
curl -s -XPOST http://localhost:3000/endpoints \
  -H "authorization: Bearer $KEY" -H "content-type: application/json" \
  -d '{"url":"http://chaos-receiver:4000/hook","event_types":["demo.event"]}'

# Publish 15 events:
for i in $(seq 1 15); do
  curl -s -o /dev/null -XPOST http://localhost:3000/events \
    -H "authorization: Bearer $KEY" -H "content-type: application/json" \
    -d "{\"type\":\"demo.event\",\"payload\":{\"orderId\":$i}}"
done
echo "published 15 events"

echo "$KEY"      # show the key again for easy copying
```

### PowerShell

```powershell
$key = (Invoke-RestMethod -Method Post -Uri http://localhost:3000/applications `
  -ContentType 'application/json' -Body '{"name":"demo"}').api_key
$key   # <-- your whsk_... key

Invoke-RestMethod -Method Post -Uri http://localhost:3000/endpoints `
  -Headers @{authorization="Bearer $key"} -ContentType 'application/json' `
  -Body '{"url":"http://chaos-receiver:4000/hook","event_types":["demo.event"]}'

1..15 | ForEach-Object {
  Invoke-RestMethod -Method Post -Uri http://localhost:3000/events `
    -Headers @{authorization="Bearer $key"} -ContentType 'application/json' `
    -Body (@{type='demo.event'; payload=@{orderId=$_}} | ConvertTo-Json) | Out-Null
}
"published 15 events"; $key
```

Concepts in one line each:
- **app** — a customer account; owns endpoints; identified by an API key.
- **API key** (`whsk_...`) — shown once at creation; stored only as a hash.
- **endpoint** — a destination URL + which event types it wants.
- **event** — something that happened; publishing one fans out to every
  subscribed endpoint, creating a **delivery** per endpoint.

---

## 3. The dashboard (the actual demo)

1. Open **http://localhost:3000/dashboard**
2. Paste the `whsk_...` key and click **Load**.
3. You'll see 15 deliveries. Colors:
   - **green = succeeded**
   - **amber = pending / retrying** (will be retried automatically)
   - **red = dead** (exhausted all attempts) or **failed** (permanent error)
4. **Click Load again every few seconds.** Watch amber rows flip to green on
   their own — that's the worker retrying with exponential backoff. Click a row's
   short **id** to expand its attempt history and show e.g. `500 → 500 → 200`
   (recovered in 1–6 attempts, no human action).
5. For any **red (dead)** row, click **Replay**, then **Load** again → it turns
   green. That's the manual safety net once the receiver is healthy.

**Tip:** Load the dashboard *quickly* after publishing — the worker retries fast,
so if you wait too long everything will already be green and you'll miss the
"watch them recover" moment.

**Narration:** "The receiver is deliberately unreliable. The system signs every
request, retries failures with exponential backoff and jitter, recovers most
deliveries automatically, and parks the hopeless ones in a dead-letter queue
where I can replay them. A `UNIQUE(event_id, endpoint_id)` constraint plus an
`X-Webhook-Id` header mean it's at-least-once with no duplicates."

---

## 4. Bonus views (optional)

- **http://localhost:3000/metrics** — Prometheus metrics; show
  `webhook_deliveries{status="succeeded"}` and friends tick up.
- **http://localhost:3000/health** — the heartbeat (both checks `ok`).
- Live logs in a spare terminal:
  ```bash
  docker compose logs -f worker
  ```
  Watch `delivery succeeded` / `delivery failed; will retry` / `delivery dead`
  stream as events are processed. `Ctrl+C` to stop tailing.

---

## 5. Guarantee a dead-letter + replay moment (optional)

With random chaos, usually only 0–2 events go fully dead. To *guarantee* a
DLQ-and-replay demo, temporarily force the receiver to always fail, then fix it.
Still no code changes — just environment overrides on the chaos container.

### Git Bash

```bash
# 1. Make the receiver always fail, then publish:
CHAOS_FORCE_STATUS=500 docker compose up -d chaos-receiver
curl -s -o /dev/null -XPOST http://localhost:3000/events \
  -H "authorization: Bearer $KEY" -H "content-type: application/json" \
  -d '{"type":"demo.event","payload":{"willDie":true}}'

# 2. Wait ~35s; in the dashboard that delivery reaches "dead" (attempts=6).

# 3. Fix the receiver to always succeed, and wait ~30s for the per-endpoint
#    circuit breaker to cool down (it opened during the failures):
CHAOS_FORCE_STATUS=200 docker compose up -d chaos-receiver
#    ...wait ~30s...

# 4. In the dashboard, click Replay on the dead row -> Load -> it turns green.

# 5. Restore the receiver to random behaviour when done:
docker compose up -d --force-recreate chaos-receiver
```

### PowerShell

```powershell
$env:CHAOS_FORCE_STATUS='500'; docker compose up -d chaos-receiver
Invoke-RestMethod -Method Post -Uri http://localhost:3000/events `
  -Headers @{authorization="Bearer $key"} -ContentType 'application/json' `
  -Body '{"type":"demo.event","payload":{"willDie":true}}'
# wait ~35s for it to reach "dead", then:
$env:CHAOS_FORCE_STATUS='200'; docker compose up -d chaos-receiver
# wait ~30s (breaker cooldown), click Replay in the dashboard, then:
Remove-Item Env:CHAOS_FORCE_STATUS; docker compose up -d --force-recreate chaos-receiver
```

Why the ~30s wait before replay: sustained failures trip the endpoint's circuit
breaker open; replaying while open just fast-fails until the cooldown elapses.
This is real, intended behaviour — worth calling out in the demo.

---

## 6. Stop / reset

```bash
docker compose down        # stop everything, KEEP the data
docker compose down -v     # stop everything and WIPE the data (fresh next time)

# reset only the flaky receiver to its default random behaviour:
docker compose up -d --force-recreate chaos-receiver
```

---

## 7. Troubleshooting

- **`curl` can't connect / dashboard won't load:** Docker Desktop isn't running,
  or the stack isn't up. Run `docker compose ps`; if empty, `docker compose up -d`.
- **`/health` shows postgres not ok, logs say `password authentication failed`:**
  the `.env` password doesn't match the existing database volume. Either restore
  the old password in `.env`, or wipe and recreate: `docker compose down -v` then
  `docker compose up -d` + `docker compose run --rm migrate` (destroys data).
- **Dashboard says "Error 401":** wrong/empty API key in the box — re-copy the
  `whsk_...` value.
- **Everything is already green when you Load:** you waited too long; the worker
  recovered them automatically. Publish a fresh batch and Load immediately, or use
  section 5 to force a dead row.
- **Ports in use (5433 / 6379 / 3000 / 4000):** another process is using them;
  stop it or change the host port mapping in `docker-compose.yml`.
- **A delivery stays stuck after failures to one endpoint:** sustained failures
  can open that endpoint's circuit breaker; it recovers after the cooldown
  (~30s) via a half-open trial.

---

## Quick reference (the minimal happy path)

```bash
cd /c/Users/BIT/Desktop/proj1
docker compose up -d --build
docker compose run --rm migrate
KEY=$(curl -s -XPOST http://localhost:3000/applications -H "content-type: application/json" -d '{"name":"demo"}' | sed -n 's/.*"api_key":"\([^"]*\)".*/\1/p'); echo "$KEY"
curl -s -XPOST http://localhost:3000/endpoints -H "authorization: Bearer $KEY" -H "content-type: application/json" -d '{"url":"http://chaos-receiver:4000/hook","event_types":["demo.event"]}'
for i in $(seq 1 15); do curl -s -o /dev/null -XPOST http://localhost:3000/events -H "authorization: Bearer $KEY" -H "content-type: application/json" -d "{\"type\":\"demo.event\",\"payload\":{\"orderId\":$i}}"; done
echo "$KEY"   # paste into http://localhost:3000/dashboard
```
