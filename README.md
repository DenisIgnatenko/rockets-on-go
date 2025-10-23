# Rocket Platform — Event-Driven Microservices in Go

A small event-driven system built in **Go**:  
loosely coupled microservices communicating via **Kafka** and **Redis**.

The goal is to simulate a **rocket launch tracking platform** —  
where new launch events are ingested, processed, and queried in real time.

---

## Architecture Overview


The platform consists of **four Go services** communicating asynchronously via **Kafka** and persisting state in **Redis**.

```
[ HTTP Clients ]
|
v
+-----------------+   Kafka topic: rocket.events    +-----------------+
|   Ingest API    | -----------------------------–> |     Tracker     |
|  :8088/messages |                                 |  Kafka consumer |
+-----------------+                                 +-----------------+
|                                                            |
| (future compacted topic)                                   | apply
|                                                            v
|                                                  +----------------+
|                                                  |     Redis      |
|                                                  |  rocket states |
|                                                  +----------------+
|                                                            ^
|                                                            |
|                Kafka topic: rocket.dlq                     |
|––––––––––––––––––––––––––-----------------------------–––> |
|                                                            |
+-----------------+                                 +----------------+
|   DLQ Processor |                                 |   Query API    |
|  (optional)     |                                 | :8090/rockets  |
+-----------------+                                 +----------------+
```
### 🧩 Component roles

- **Ingest API** — accepts incoming HTTP messages, validates payloads, and publishes normalized events to Kafka.
- **Tracker** — consumes events from Kafka and atomically applies updates to Redis (via Lua scripts).
- **Query API** — exposes REST endpoints to read current rocket state directly from Redis.
- **DLQ Processor** *(optional)* — handles malformed messages from a dead-letter queue.
- **Redis** — acts as the real-time state store for rocket states.
- **Kafka (or Redpanda)** — serves as the event bus connecting all components.

---

## Project Layout
```
rocket-platform/                      # Go mono-repo (Go 1.22+)

  go.mod
  go.sum
  Makefile                            # build / lint / test commands
  docker-compose.yml                  # redpanda (Kafka), redis, prometheus, grafana
  README.md

  cmd/                                # 4 independent binaries (services)
    ingest-api/                       # HTTP -> Kafka (write path)
      main.go
    tracker/                          # Kafka -> Redis (apply state, Lua)
      main.go
    query-api/                        # HTTP -> Redis (read path)
      main.go
    dlq-processor/                    # Kafka DLQ handler (optional)
      main.go

  internal/                           # shared logic and libraries
    config/                           # env vars, defaults
      config.go
    log/                              # zap logger setup
      logger.go
    telemetry/                        # (optional) OpenTelemetry + Prometheus
      telemetry.go
    kafka/                            # producer / consumer helpers
      producer.go
      consumer.go
    redis/                            # redis client + atomic Lua scripts
      client.go
      scripts.go
      scripts/
        apply_and_drain.lua           # apply event + drain per channel
    contracts/                        # schemas + validation
      message.go                      # legacy {metadata, message}
      event.go                        # normalized Kafka event
      validate.go
    model/                            # domain models (rocket state, snapshot)
      rocket.go
    apply/                            # apply events -> Redis (used by tracker)
      apply.go
    query/                            # read models from Redis (for /rockets)
      repo.go
    httpmw/                           # middlewares (request-id, recover, logging)
      requestid.go
      recover.go
```

## Endpoints
POST /messages  - Accepts the legacy payload the generator sends:
```
{
  "metadata": {
    "channel": "uuid",
    "messageNumber": 123,
    "messageTime": "2025-09-22T06:22:14.660427Z",
    "messageType": "RocketLaunched"
  },
  "message": { ...event-specific fields... }
}
```

-> validate -> normalize to internal Event -> publish to Kafka topic rocket.events.

* GET /rockets?sortBy=channel / type / mission / speed / lastMessageTime&order=asc / desc - Reads current snapshots from Redis (or a compacted topic later).
* GET /rockets/{channel} - Reads one snapshot from Redis.

