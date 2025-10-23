# Rocket Platform — Event-Driven Microservices in Go

A small event-driven system built in **Go**:  
loosely coupled microservices communicating via **Kafka** and **Redis**.

The goal is to simulate a **rocket launch tracking platform** —  
where new launch events are ingested, processed, and queried in real time.

---

## 🧱 Architecture Overview

The platform consists of **two services** communicating through Kafka

```
[ HTTP Clients ]
|
v
+––––––––––+           Kafka topic: rocket.events       +------------------+
|      Gateway       | -------------------------------> |      Tracker     |
|  :8088 /messages   |                                  |  Kafka consumer  |
|  :8088 /rockets    | <------------------------------- |  (DLQ optional)  |
+––––––––––+        (future: compacted snapshots)       +——————+
|                                                       |
| REST (read)                                           | apply
v                                                                  v
+——————+
|      Redis       |
|  rocket states   |
+——————+
```
- **Gateway** — validates incoming HTTP messages and publishes normalized events to Kafka. Also exposes `GET /rockets` that reads current state from Redis.
- **Tracker** — consumes events from Kafka and atomically applies them to Redis (Lua script), producing an always-up-to-date read model.
- **Redis** — serves as the real-time state store (fast reads for `GET /rockets`).
- **(Optional)** DLQ processor, Prometheus, Grafana.

---

## Project Layout
```
rocket-platform/
go.mod
go.sum
Makefile
docker-compose.yml              # redpanda (Kafka), redis, prometheus, grafana
README.md

/cmd
/gateway                        # HTTP :8088 -> /messages + /rockets
    main.go
/tracker                        # Kafka consumer -> Redis apply
    main.go

/internal
/config                         # env + defaults
    config.go
/log                            # zap logger
    logger.go
/telemetry                      # (optional) OTel + Prometheus
    telemetry.go
/kafka                          # producer/consumer helpers
    producer.go
    consumer.go
/redis                          # redis client + Lua scripts
    client.go
    scripts.go
scripts/
    apply_and_drain.lua
/contracts                      # request/events schemas + validation
    message.go
    event.go
    validate.go
/model                          # domain models (rocket state, snapshot)
    rocket.go
/query                          # read models from Redis (for /rockets)
    repo.go
/apply                          # apply events -> Redis (used by tracker)
    apply.go
/httpmw                         # chi middlewares (request-id, recover, logging)
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

