# TicketHub – Phase 1 (Epics 01–13) + Phase 2 (Kafka reliability and sagas)

> **Phase 2 starts at [docs/PHASE2_FLOW.md](docs/PHASE2_FLOW.md)** (story-by-story status: [docs/PHASE2_COVERAGE.md](docs/PHASE2_COVERAGE.md)) (the flow, explained step by step) and is operated with `scripts/phase2-lab.sh`.

Event/movie ticket booking platform built as Spring Boot microservices
(Java 17, Spring Boot 3.3, Spring Cloud 2023.0, PostgreSQL, Kafka, Docker Compose) with a full
observability stack (Prometheus, Grafana, Jaeger, ELK), optional Keycloak login, S3-compatible media storage
and an optional Stripe sandbox payment provider.

> **Verification status.** The code was written and reviewed statically (Java syntax-checked), but **it could not be
> compiled or run in the environment where the latest additions were made**. The original services were run by the
> owner on Docker (13 containers healthy, `scripts/e2e.sh` passed). Everything added afterwards (marked *new* below)
> must be confirmed on your machine: see [First start checklist](#first-start-checklist).

```
                         ┌────────────────────────── docker compose network ───────────────────────────┐
 client ──JWT──► :8080 apigate-service ──lb://──► event-service ◄──REST──► booking-service ──Kafka──► notification-service ──SMTP──► MailHog
                  │  (JWT check, routing,           │ event-db                │ booking-db  ▲                │ notification-db
                  │   X-User-* headers)             │                         │             │ REST (payment result)
                  │                                 └─────────────────────────┼──────► payment-service ── payment-db
                  └── eureka-server :8761  (service discovery)   config-server :8888 (central config)
```

| Service | Port | Owns |
|---|---|---|
| `apigate-service` | 8080 (only published business port) | routing, JWT validation, dev token endpoint |
| `event-service` | 8081 | events, venues, seat layout, shows, ticket price (`event_db`) |
| `booking-service` | 8082 | bookings, seat reservations, expiry, publishes `BookingConfirmed` (`booking_db`) |
| `payment-service` | 8083 | payments, simulated provider, signed webhook (`payment_db`) |
| `notification-service` | 8084 | notification records, Kafka consumer, email (`notification_db`) |
| `eureka-server` | 8761 | service discovery |
| `config-server` | 8888 | centralized configuration (`config-server/src/main/resources/config-repo`) |

Platform and tooling containers (all in `docker-compose.yml`):

| Container | URL / port | Purpose | Starts |
|---|---|---|---|
| `mailhog` | http://localhost:8025 | catches all emails (fake SMTP) | always |
| `kafka` | localhost:29092 (host), `kafka:9092` (Docker) | event bus | always |
| `minio` + `minio-init` *(new)* | http://localhost:9000 (S3 API), console http://localhost:9001 | S3-compatible media storage, bucket `tickethub-media` | always |
| `prometheus` *(new)* | http://localhost:9090 | metrics collection | always |
| `grafana` *(new)* | http://localhost:3000 (`admin`/`admin`) | dashboard "TicketHub Overview" | always |
| `jaeger` *(new)* | http://localhost:16686 | distributed traces | always |
| `keycloak` + `keycloak-db` *(new)* | http://localhost:8180 | OAuth2/OIDC identity provider | `--profile iam` |
| `elasticsearch`, `logstash`, `kibana` *(new)* | http://localhost:5601 | centralised log search | `--profile elk` |
| `kafka-ui` | http://localhost:8090 | browse topics/messages | `--profile tools` |

Each service owns its database; services talk only through REST (`/internal/**`, never exposed by the gateway) and Kafka.

## Run everything (one command)

```bash
cp .env.example .env          # edit the placeholder secrets (an existing .env keeps working: new variables have defaults)
docker compose up -d --build  # first build takes several minutes
docker compose ps             # wait until every service shows "healthy"
bash scripts/smoke-test.sh    # checks routing/security
bash scripts/e2e.sh           # full journey through the gateway (needs bash, curl, jq, openssl)
bash scripts/concurrency-test.sh 20   # 20 customers race for ONE seat: exactly one must win (ST-08)
docker compose down           # stop (add -v to delete DB/Kafka/MinIO volumes)
```
Optional parts:
```bash
docker compose --profile iam up -d        # Keycloak            (also set KEYCLOAK_ENABLED=true in .env)  -> docs/KEYCLOAK.md
docker compose --profile elk up -d        # ELK log search      (also set TICKETHUB_PROFILES=docker,elk)  -> docs/OBSERVABILITY.md
docker compose --profile tools up -d      # Kafka UI
docker compose --profile full up -d       # everything (needs ~8 GB RAM for Docker)
```
Useful UIs: Eureka http://localhost:8761 · MailHog http://localhost:8025 · Grafana http://localhost:3000 ·
Jaeger http://localhost:16686 · Prometheus http://localhost:9090 · MinIO http://localhost:9001 ·
config `http://localhost:8888/event-service/docker`.
Two instances of a service (SC-14): `docker compose up -d --scale event-service=2`.

On Windows run the `.sh` scripts from **Git Bash** (or WSL). `jq` is needed: `winget install jqlang.jq`, then reopen the terminal.

### First start checklist
1. `docker compose up -d --build` finishes and `docker compose ps` shows all containers `healthy`
   (`minio-init` exits after creating the bucket, that is normal).
2. http://localhost:9090/targets shows 5 targets `UP`; http://localhost:3000 shows the dashboard with data after a few requests.
3. `bash scripts/e2e.sh` prints `E2E PASSED` (it now also uploads a poster; use `CHECK_POSTER=0` to skip that step).
4. In Jaeger, pick `booking-service`, open a trace: you should see the call chain across services.
5. `docker compose logs booking-service --tail 5` prints JSON lines (set `TICKETHUB_LOG_APPENDER=CONSOLE` for text).
If a *new* piece fails, it is isolated: see [Troubleshooting](#troubleshooting).

## Authentication (development)
`POST /auth/token` on the gateway issues a JWT when `DEV_TOKEN_ENABLED=true`:
```bash
curl -s -X POST localhost:8080/auth/token -H 'Content-Type: application/json' \
  -d '{"userId":"alice","email":"alice@example.com"}'                       # CUSTOMER
curl -s -X POST localhost:8080/auth/token -H 'Content-Type: application/json' \
  -d '{"userId":"admin-1","email":"a@x.com","roles":["ADMIN"],"adminKey":"<DEV_ADMIN_KEY>"}'
```
The gateway validates the JWT, strips any client-sent `X-User-*` headers and forwards the verified identity.
Customer identity is never taken from a request body.

**Keycloak (new, optional):** with `KEYCLOAK_ENABLED=true` and `docker compose --profile iam up -d` the gateway also accepts
RS256 access tokens issued by Keycloak (realm `tickethub`, users `admin`/`alice`/`bob`/`john`), verified against Keycloak's
public keys. Dev tokens keep working side by side. Details: [docs/KEYCLOAK.md](docs/KEYCLOAK.md);
run the full flow with real logins: `AUTH_MODE=keycloak bash scripts/e2e.sh`.

## API (through the gateway)
| Area | Endpoints |
|---|---|
| Events | `GET /events`, `GET /events/{id}` (public) · `POST/PUT /events`, `PATCH /events/{id}/status` (ADMIN) · `POST /events/{id}/poster` multipart field `file`, JPEG/PNG/WebP up to 5 MB (ADMIN, stored in S3/MinIO) *(new)* |
| Venues | `POST/PUT/GET /venues`, `POST/GET /venues/{id}/seats` (ADMIN) |
| Shows | `GET /shows`, `GET /shows/{id}`, `GET /shows/{showId}/seats` (public) · `POST/PUT /shows`, `PATCH /shows/{id}/status` (ADMIN) |
| Bookings | `POST /bookings` (+ `Idempotency-Key` header), `GET /bookings/{id}`, `GET /bookings`, `POST /bookings/{id}/cancel` |
| Payments | `POST /payments`, `GET /payments/{id}`, `POST /payments/webhook` (HMAC `X-Signature`) |
| Notifications | `GET /notifications`, `GET /notifications/{id}` (ADMIN, operations view) |
| *Phase 2* Dead letters | `GET /notifications/dead-letters?status=PARKED`, `POST /notifications/dead-letters/{id}/replay` (ADMIN) |
| *Phase 2* Saga view | `GET /bookings/{id}/saga` (owner or ADMIN): state, plain-language reason, timeline |
| *Phase 2* Reconciliation | `POST /payments/reconciliation/run`, `GET /payments/reconciliation/report` (ADMIN) |

OpenAPI per service: `http://<service>:<port>/swagger-ui.html` (run the service locally, internal ports are not published in Docker).

Standard error format returned by every service and the gateway:
```json
{"timestamp":"...","status":409,"error":"Conflict","code":"SEAT_UNAVAILABLE","message":"...","path":"/bookings","correlationId":"...","fieldErrors":[]}
```

## Enums
`EventStatus` (DRAFT, PUBLISHED, CANCELLED) · `ShowStatus` (SCHEDULED, OPEN, SOLD_OUT, CANCELLED, COMPLETED) · `EventType` · `VenueStatus` · `SeatCategory` · `CurrencyCode` ·
`SeatReservationState` (AVAILABLE, RESERVED, BOOKED) · `BookingStatus` (PENDING_PAYMENT, CONFIRMED, PAYMENT_FAILED, CANCELLED, EXPIRED) · `PaymentResult`, `PaymentResultOutcome` ·
`PaymentStatus` (INITIATED, PENDING, UNKNOWN, SUCCESS, FAILED, CANCELLED, REFUNDED) · `ProviderStatus`, `SimulatedOutcome`, `PaymentProviderName`, `WebhookOutcome` ·
`NotificationChannel` (EMAIL, PUSH) · `NotificationStatus` (PENDING, SENT, RETRYING, DEAD, REPLAYED, FAILED) · `SagaState` (PAYMENT_PENDING, CONFIRMED, COMPENSATING, COMPENSATED, FAILED) · `NotificationChannel`, `NotificationType` · `Role` · `ErrorCode`.
Lifecycle enums contain their own transition rules (`canTransitionTo`).

## Exception handling
Every service has the same `GlobalExceptionHandler` (`@RestControllerAdvice`) + a `BusinessException` hierarchy
(404 not found, 409 conflict/seat unavailable/invalid transition, 422 business rule, 401/403, 502-504 dependency errors, 400 validation/malformed).
Stack traces and internals are never returned; unexpected errors return a generic 500 with the `correlationId` (also in logs via MDC).
The gateway has its own handler so routing failures (no route 404, no instance 503, timeout 504) use the same format.

## Key design decisions (and the story each satisfies)
* **No double booking (ST-08):** partial unique index `uq_active_show_seat (show_id, seat_id) WHERE state <> 'AVAILABLE'` – the database decides races; the loser gets `409 SEAT_UNAVAILABLE`.
* **Price snapshot (ST-46):** booking amount = show price × seats at creation; payment takes the amount from the booking; clients can't send amounts.
* **Idempotent booking (ST-53):** `Idempotency-Key` + unique `(customer_id, idempotency_key)` → retry returns the original (HTTP 200).
* **Expiry (ST-09):** scheduler marks overdue `PENDING_PAYMENT` bookings `EXPIRED` and frees seats; an expired booking is never confirmed (payment success after expiry is recorded as `REJECTED_EXPIRED`).
* **Payment → Booking result (ST-02/55/56):** *Phase 1* used a REST call; **Phase 2 replaced it with events** (`PaymentSucceeded` / `PaymentFailed` on Kafka via an outbox). The old `/internal/bookings/{id}/payment-result` endpoint still exists but Payment no longer calls it.
* **Webhook (ST-23/24):** HMAC-SHA256 over the raw body, constant-time compare, processed-event table (duplicates → `DUPLICATE_IGNORED`), late/conflicting callbacks ignored.
* **Kafka (ST-19/20/58/60):** topic `booking-confirmed` (3 partitions, key = bookingId); published **after commit**; Notification consumes in group `notification-service-group`; duplicates blocked by unique `source_event_id`; a failed email never touches the booking (Phase 2: non-blocking retry topics, then a dead-letter topic, see below). Demo: `scripts/kafka-groups-demo.sh`.
* **Resilience (ST-16/17/18):** explicit timeouts on every remote call; bounded retry only for idempotent calls (payment initiation is never retried); circuit breakers with fail-fast controlled errors. Simulate latency/outage with `payment.simulator.*` or `simulatedOutcome: TIMEOUT`.
* **Media (ST-30) *(new)*:** `POST /events/{id}/poster` validates type/size, stores the image through the S3 API (MinIO locally, AWS S3 in the cloud: same code, different `tickethub.media.*` settings) and saves its public URL on the event. The remote upload never runs inside a database transaction.
* **Real provider sandbox (ST-22) *(new)*:** `StripeSandboxPaymentProvider` (Stripe TEST mode, test card token `pm_card_visa`, payment id used as Idempotency-Key). Off by default; enable with `TICKETHUB_PROFILES=docker,stripe` and `STRIPE_SECRET_KEY=sk_test_...` (a live key is refused at start-up). Limitation: Stripe's own webhook signature format is not implemented; the sandbox flow confirms synchronously.
* **Config (ST-13):** config repo contains no secrets; passwords/keys come from environment variables.

## Epic 13 mapping
| Story | Where |
|---|---|
| ST-61 local infrastructure | `docker-compose.yml` (4 Postgres + volumes, Kafka, MailHog), `.env.example` |
| ST-32 Dockerize | `*/Dockerfile` (pinned tags, non-root, HEALTHCHECK), compose health-gated startup, CI image build |
| ST-33 repeatable deployment | `docker compose up -d --build` + `scripts/smoke-test.sh` |
| ST-34 release & rollback | `scripts/release.sh v1.0.0` / `scripts/rollback.sh v0.9.0` (image tag via `TICKETHUB_TAG`) |
| ST-62 end-to-end | `scripts/e2e.sh` |
| CI | `.github/workflows/ci.yml`: `mvn verify`, image build, and an *(new)* integration job that starts the stack and runs smoke, e2e and concurrency tests |

## Observability, security, cloud: story coverage *(new)*
| Story | Where |
|---|---|
| ST-15 Keycloak | `keycloak/realm/tickethub-realm.json`, compose profile `iam`, `KeycloakTokenVerifier` + `TokenVerifier` in the gateway, [docs/KEYCLOAK.md](docs/KEYCLOAK.md) |
| ST-25 structured logs | `*/src/main/resources/logback-spring.xml` (JSON in Docker), MDC `traceId`/`correlationId` |
| ST-26 ELK | compose profile `elk`, `observability/logstash/pipeline/logstash.conf`, Kibana data view auto-created |
| ST-27 metrics | `/actuator/prometheus` on every service, `observability/prometheus`, `observability/grafana` dashboard |
| ST-28 tracing | Micrometer Tracing + OpenTelemetry -> Jaeger; trace context also crosses Kafka headers |
| ST-29 incident lab | `scripts/incident-lab.sh`, [docs/INCIDENT_LAB.md](docs/INCIDENT_LAB.md) |
| ST-30 S3 media | `MediaStorageService`, `PosterService`, MinIO container |
| ST-31 RDS + secrets | `config-repo/*-aws.yml`, `docker-compose.aws.yml`, [docs/AWS_RDS_AND_SECRETS.md](docs/AWS_RDS_AND_SECRETS.md) (guide only, needs your AWS account) |
| ST-22 provider sandbox | `StripeSandboxPaymentProvider`, profile `stripe` |
| ST-08 concurrency proof | `scripts/concurrency-test.sh` (+ CI job) |

The gateway's actuator now listens on the unpublished management port **8089** (health check and Prometheus use it),
so `/actuator/*` is no longer reachable through port 8080.

## Phase 2 in one page (details: [docs/PHASE2_FLOW.md](docs/PHASE2_FLOW.md))
Phase 2 makes the Kafka messaging reliable and turns "pay for a booking" into a **saga** (a chain of local steps with an undo).

| # | Problem | What was added | Where |
|---|---|---|---|
| S0 | can't see inside Kafka | Kafka UI always on at http://localhost:8090 | `docker-compose.yml` |
| S1 | a bad message blocks the consumer | retry topics + dead-letter topic + dead-letter API (list, fix, replay) | `notification-service/.../consumer`, `DeadLetterController` |
| S2 | the same event arrives twice | **inbox** table, unique event id, written in the same transaction as the notification | `InboxRepository`, `NotificationRecorder` |
| S3 | event lost between DB commit and Kafka | **outbox** table + relay job | `booking-service/.../outbox`, `payment-service/.../outbox` |
| S4 | money taken, booking not confirmed | Payment publishes `PaymentSucceeded/Failed`; Booking consumes them (no REST callback) | `PaymentEventsConsumer`, `PaymentEventHandler` |
| S5 | late payment / cancel after confirm | `BookingRejected/Cancelled` -> refund -> `PaymentRefunded` (idempotent) | `PaymentRefundService`, `BookingService` |
| S6 | out-of-order events | key = bookingId, per-booking version numbers, stale events ignored | `aggregate_version` columns |
| S7 | where is booking X stuck? | saga state + timeline: `GET /bookings/{id}/saga`; choreography vs orchestration note | `SagaLog`, [docs/SAGA_DECISION.md](docs/SAGA_DECISION.md) |
| S8 | payment and booking disagree | reconciliation job + report | `payment-service/.../reconciliation` |
| S9 | duplicate / late / lost provider callbacks | forward-only status machine, `UNKNOWN` state, provider lookup | `PaymentStatus`, `ProviderInvoker.lookup` |
| S10 | email provider down | circuit breaker + rate limiter + retry topics | `EmailDeliveryGuard` |
| S11 | consumer falls behind | lag metrics + alert rules + scaling lab | `observability/prometheus/alerts.yml`, `phase2-lab.sh s11-lag` |
| S12 | rebuild state from history | replay consumer (new group, own DB, never sends) | compose profile `replay`, `notification-service-replay.yml` |
| S13 | event contract must change | v2 adds fields, keeps old one; consumers ignore unknown fields | `BookingConfirmedEvent` (both services) |
| ST-71 | service crashes after payment | kill-and-restart drill: `phase2-lab.sh s15-crash` | `scripts/` |
| ST-76 | service-to-service auth | shared `X-Internal-Key` on `/internal/**` (set `INTERNAL_API_KEY` in `.env`; off by default) | `InternalApiKeyFilter` |
| ST-77 | advanced incident analysis | `tickethub_*` business metrics, Grafana dashboard *TicketHub Phase 2*, more alerts | `observability/` |
| ST-78 | S3/RDS operations | SSE, versioning/lifecycle, pool limits, `scripts/db-restore-drill.sh`, [docs/HARDENING.md](docs/HARDENING.md) | |
| ST-79 | second channel | PUSH notifications next to EMAIL (simulated provider), independent status and circuit breaker | `PushNotificationSender` |
| S14 | everything breaks | `scripts/phase2-lab.sh s14-chaos` + [incident template](docs/INCIDENT_TEMPLATE.md) | `scripts/` |

Try it: `docker compose up -d --build`, then `bash scripts/e2e.sh` (now also tests cancel + refund), then
`bash scripts/phase2-lab.sh` (lists one command per scenario).

## Run a service locally (without Docker)
Start infra only: `docker compose up -d event-db booking-db payment-db notification-db kafka mailhog`, then run `eureka-server`, `config-server`, the services and the gateway with `mvn spring-boot:run` (each in its own terminal).
Export `SPRING_DATASOURCE_PASSWORD` (= `POSTGRES_PASSWORD`), `PAYMENT_WEBHOOK_SECRET` (payment) and `JWT_SECRET`, `DEV_TOKEN_ENABLED`, `DEV_ADMIN_KEY` (gateway).

## Tests
* Unit tests: status transition rules, webhook signature, JWT + public endpoints, message building, token verification
  (dev + Keycloak paths), poster validation, Stripe adapter against a local fake of Stripe's API. Run: `mvn verify`.
* System tests (need the running stack): `scripts/smoke-test.sh`, `scripts/e2e.sh`, `scripts/concurrency-test.sh`,
  `scripts/incident-lab.sh`. CI runs them in the `integration` job.
* Docker image builds skip test compilation (`-Dmaven.test.skip=true`), so a test problem can never block a deployment;
  CI is where tests run.

## Troubleshooting
| Symptom | Fix |
|---|---|
| `bash: jq: command not found` | `winget install jqlang.jq`, reopen Git Bash |
| `exec /bin/bash failed` in PowerShell | you used the WSL shim without a distro; open **Git Bash** instead |
| Whitelabel page on :8888 | normal: open `http://localhost:8888/event-service/docker` |
| Kafka UI refused on :8090 | it is part of the normal stack now (Phase 2): `docker compose up -d kafka-ui` |
| Booking stays `PENDING_PAYMENT` after paying | confirmation travels over Kafka now; wait a few seconds, then check `docker compose logs booking-service payment-service` and the outbox tables (docs/PHASE2_RUNBOOKS.md) |
| Gateway healthy check fails after upgrading | rebuild: `docker compose up -d --build apigate-service` (health moved to port 8089) |
| Keycloak tokens give 401 | `KEYCLOAK_ENABLED=true`, recreate the gateway, request the token from `http://localhost:8180` |
| Poster upload gives 503 | `docker compose logs minio minio-init`; the bucket must exist |
| No traces in Jaeger | `docker compose logs jaeger`; services must run with the `docker` profile |
| Elasticsearch exits | give Docker Desktop more memory (4 GB+), or do not use the `elk` profile |
| Windows line-ending errors in `.sh` | `sed -i 's/\r$//' scripts/*.sh .env` (`.gitattributes` prevents this in Git) |

## Not included
* **Razorpay.** The Phase 2 guide talks about Razorpay; this project has the *simulated* provider and the *Stripe test-mode* provider behind one `PaymentProviderGateway` interface, and refunds/lookup are implemented for those two. A Razorpay adapter would be a third class implementing the same interface.
* **Kafka topic names** follow the Phase 1 style (`booking-confirmed`, dead-letter topic `booking-confirmed-dlt`), not the guide's dotted style (`booking.confirmed.DLT`).
* **Nothing in this delivery was compiled or run by the author** (no Maven/Docker access where it was written). See *Phase 2: what is verified* in docs/PHASE2_FLOW.md.
* **Stripe's own webhook signature format** (the simulated provider's HMAC webhook is implemented).
* **A real AWS deployment** (guide and profile files only) and Kubernetes/production hardening.
* Services trust the identity headers added by the gateway and `/internal/**` is protected only by network isolation
  (service ports are not published). Add mutual TLS or an internal API key before exposing services directly.
* Keycloak realm users and the MinIO/Grafana default passwords are for local learning only.
