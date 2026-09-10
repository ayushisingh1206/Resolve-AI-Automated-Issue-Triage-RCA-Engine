# Resolve AI ⚡

Enterprise-grade event-driven AIOps platform built using Spring Boot microservices, Apache Kafka, Spring AI, and vector databases for automated support ticket triage and Root Cause Analysis (RCA).

![Java](https://img.shields.io/badge/Java-21-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.x-brightgreen)
![Spring AI](https://img.shields.io/badge/Spring%20AI-enabled-blue)
![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-event--driven-black)
![Docker](https://img.shields.io/badge/Docker-containerized-2496ED)
![MySQL](https://img.shields.io/badge/MySQL-relational%20store-4479A1)

![Resolve AI System Architecture](./architecture-diagram.png)

---

## 1. Project Overview

**Project Name**

Resolve AI

**Problem Statement**

Modern engineering and support teams suffer from alert fatigue. Traditional incident management systems struggle when alert ingestion, duplicate checking, log correlation, and root cause analysis are done manually, leading to high Mean Time to Resolution (MTTR).

**Why This Application Was Built**

This project was built to demonstrate production-style AI integration within a distributed backend architecture for a senior backend role, including:

* service decomposition
* event-driven messaging via Apache Kafka
* Retrieval-Augmented Generation (RAG) pipelines
* LLM-based classification and confidence routing
* containerized deployment and observability

**High-Level Objective**

Deliver an autonomous AI-driven platform that ingests high-volume system alerts, semantically checks for duplicate incidents, classifies severity, and generates actionable Root Cause Analysis (RCA) reports for engineers.

**Business Value**

* drastically reduced MTTR via automated initial triage
* elimination of redundant ticket tracking using vector embeddings
* scalable handling of alert spikes via Kafka event queues
* cleaner security boundary through API gateway routing

---

## 2. Project Description

**What the Application Does**

Resolve AI supports the end-to-end incident lifecycle:

* automated webhook ingestion of alerts from monitoring tools
* semantic deduplication against historical tickets using vector search
* LLM-driven classification (Severity, Domain, Blast Radius)
* context enrichment (RAG) by fetching related system logs
* automated RCA draft generation using Spring AI
* event-driven notification dispatch to engineering teams

**Main User Journey**

1. Monitoring tool (e.g., Datadog) triggers an alert webhook to the API Gateway.
2. Ingestion service validates the payload and publishes the raw event to Kafka.
3. Triage service consumes the event, converts it to a vector embedding, and queries the Vector DB for duplicates.
4. If unique, the AI Core service classifies the issue's severity and domain.
5. AI Core service uses Spring AI to generate a detailed RCA summary based on retrieved context.
6. Audit service persists the state, and the Notification service pushes the RCA draft to a Slack channel.

**Core Modules**

* Ingress routing and rate limiting (`api-gateway`)
* Alert webhook validation and publishing (`ingestion`)
* Semantic search and deduplication (`triage`)
* LLM reasoning and RAG orchestration (`ai-core`)
* Relational state persistence and auditing (`audit-storage`)
* Alert dispatching (`notification`)
* Infra: Apache Kafka, MySQL, Vector DB, Redis

---

## 3. Architecture Overview

**High-Level Architecture**

* **API Gateway:** Single ingress point, API key validation, and route-level rate limiting.
* **Domain Services:** 5 independent Spring Boot microservices (Ingestion, Triage, AI-Core, Audit, Notification).
* **Datastores:**
  * MySQL: Relational persistence for ticket states, RCAs, and audit trails.
  * Vector Database: Stores high-dimensional text embeddings for semantic deduplication.
  * Redis: In-memory caching for API Gateway rate-limiting.
* **Messaging:** Apache Kafka handles high-throughput event ingestion and decouples the AI processing pipeline.
* **AI Integration:** Spring AI / LangChain4j orchestrates LLM classification and RAG pipelines.
* **Observability:** Spring Boot Actuator, Micrometer, Prometheus, and Grafana for metrics and Kafka lag monitoring.

**Request Lifecycle**

1. Monitoring tool (Client) sends an alert payload to `api-gateway`.
2. Gateway validates API keys and forwards the request to the `ingestion-service`.
3. `ingestion-service` immediately publishes the raw event to the Kafka topic (`alert.received`) and returns a `202 Accepted` response to prevent timeouts.
4. `triage-service` consumes the event, generates embeddings, queries the Vector DB for duplicates, and publishes to `alert.triaged`.
5. `ai-core-service` consumes the triaged event, builds the prompt context, calls the LLM for RCA generation, and publishes to `rca.generated`.
6. `audit-storage` independently listens to all Kafka topics to maintain a real-time state machine in MySQL.

**Sync vs Async Communication**

* **Synchronous (OpenFeign / REST):** Dashboard read operations (e.g., fetching ticket history and final RCA summaries from the audit service).
* **Asynchronous (Apache Kafka):** The core event pipeline. Decouples fast webhook ingestion from potentially slow, rate-limited LLM API calls.

**Architectural Decisions**

* Event-driven ingestion ensures that spikes in monitoring alerts do not crash the system or cause webhook timeouts while waiting for LLM processing.
* Vector-based deduplication handles slight variations in error messages and stack traces much better than exact string matching.
* **CQRS Pattern:** The system separates the write/command path (Kafka event streams) from the read/query path (MySQL audit database).
* Gateway-centric security offloads API key validation, keeping internal microservices focused purely on business logic.

---

## 4. Tech Stack Documentation

**Backend**

| Technology | What it is | Why chosen / Problem solved |
|---|---|---|
| Java 21 | LTS JVM runtime | Modern language/runtime; uses Virtual Threads for high-throughput AI API calls. |
| Spring Boot 3.4.x | App framework | Rapid service bootstrap with production-ready actuator features. |
| Spring AI | AI integration framework | Standardized interface for LLM orchestration and Vector DB connectivity. |
| Spring Cloud Gateway | Reactive API gateway | Centralized routing, API key validation, and rate limiting. |
| Apache Kafka | Event streaming broker | Decouples webhook ingestion from slow LLM inference, ensuring no dropped alerts. |
| OpenFeign | Declarative HTTP client | Simple inter-service synchronous calls for dashboard reads. |
| Spring Data JPA | ORM | Relational domain persistence for audit trails. |
| Flyway | DB migration | Versioned schema changes for MySQL. |
| Redis | In-memory data store | Gateway rate limiting and short-term caching. |

**Database & Persistence**

* **MySQL:** Maintains strong transactional consistency for ticket lifecycle, audit logs, and final RCA reports.
* **Vector Database (PgVector / Milvus):** Stores high-dimensional textual embeddings of error logs to perform rapid cosine-similarity searches for deduplication.

**Containerization & Build**

* **Docker & Docker Compose:** Used for spinning up isolated dev, local, and observability stacks.
* **Maven:** Dependency management and build lifecycle.

**Observability**

* **Prometheus:** Metrics scraping (Kafka lag, LLM latency).
* **Grafana:** Dashboards for visualizing system health and alert ingestion rates.
* **Micrometer + Actuator:** Standardized telemetry and health endpoints.

---

## 5. Complete Microservice Breakdown

| Service | Port | Purpose | Database | Key Dependencies |
|---|---|---|---|---|
| api-gateway | 8080 | Ingress routing, API Key validation, Rate limiting | Redis | - |
| ingestion | 8081 | Alert webhook validation and Kafka publishing | - | Kafka Producer |
| triage | 8082 | Generates embeddings, semantic duplicate checking | Vector DB | Kafka, Spring AI |
| ai-core | 8083 | LLM classification, RAG context building, RCA generation | - | Kafka, Spring AI, LLM API |
| audit-storage | 8084 | Persists state changes and RCA results via JPA | MySQL | Kafka Consumer |
| notification | 8085 | Dispatches completed RCAs to Slack/Jira | - | Kafka Consumer |

---

## 6. API Documentation (Service-wise)

**Common Headers**

* `Authorization: Bearer <access_token>` or `X-API-Key: <key>` for protected dashboard or webhook ingress APIs.
* Gateway injects trusted internal headers: `X-Tenant-Id`, `X-Source-System`.

### Ingestion Service (`/api/v1/alerts`)

**Webhooks:**
* `POST /api/v1/alerts/ingest`
* `GET /api/v1/alerts/status/{jobId}`

**Internal:**
* `POST /api/internal/alerts/requeue`

**Sample request (`POST /api/v1/alerts/ingest`):**
```json
{
  "source": "DATADOG",
  "service": "payment-gateway",
  "error_message": "Connection timeout acquiring database lock",
  "stack_trace": "com.mysql.cj.jdbc.exceptions.CommunicationsException...",
  "timestamp": "2026-09-08T10:00:00Z"
}
```

**Sample response:**
```json
{
  "success": true,
  "status": 202,
  "message": "Alert queued successfully",
  "data": {
    "jobId": "job_01J7A29M8X9N3R",
    "fingerprint": "fp_pay_npe_charge"
  },
  "timestamp": "2026-09-08T10:00:01Z"
}
```

### Triage Service (`/api/triage`, `/api/internal`)

**Deduplication & Search:**
* `GET /api/v1/triage/match/{fingerprint}`
* `POST /api/v1/triage/evaluate-similarity`
* `GET /api/v1/triage/clusters?service={serviceName}`

**Embeddings:**
* `POST /api/v1/triage/embeddings/generate`
* `DELETE /api/v1/triage/embeddings/{ticketId}`

**Internal:**
* `GET /api/internal/triage/similarity-threshold`
* `POST /api/internal/triage/sync-vector-db`

### AI-Core Service (`/api/rca`, `/api/internal`)

**RCA Generation:**
* `POST /api/v1/rca/generate`
* `GET /api/v1/rca/{incidentId}`
* `PATCH /api/v1/rca/{incidentId}/feedback` (Thumbs up/down for LLM accuracy)

**Prompts & Config:**
* `GET /api/v1/rca/prompts/active`
* `PUT /api/v1/rca/prompts/{templateId}`

**Internal:**
* `POST /api/internal/rca/context-enrich`

**Sample request (`POST /api/v1/rca/generate`):**
```json
{
  "incidentId": "inc_90124",
  "errorSignature": "NullPointerException: Cannot read field 'balance' of null",
  "recentCommits": [
    {
      "hash": "b47f91c",
      "author": "dev@company.com"
    }
  ]
}
```

### Audit & Ticket Service (`/api/tickets`, `/api/internal`)

**Tickets (Read/Write):**
* `GET /api/v1/tickets?page=0&size=10&status=OPEN`
* `GET /api/v1/tickets/{ticketId}`
* `PATCH /api/v1/tickets/{ticketId}/status`
* `GET /api/v1/tickets/{ticketId}/rca`
* `GET /api/v1/tickets/{ticketId}/audit-trail`

**Analytics:**
* `GET /api/v1/tickets/analytics/mttr`
* `GET /api/v1/tickets/analytics/duplicate-ratio`

**Internal:**
* `POST /api/internal/tickets/create`
* `POST /api/internal/audit-trail/log`

### Notification Service (`/api/notifications`, `/api/internal`)

**Feed & Dispatch:**
* `GET /api/v1/notifications?page=0&size=10&isRead=false`
* `GET /api/v1/notifications/unread-count`
* `PATCH /api/v1/notifications/{notificationId}/read`
* `PATCH /api/v1/notifications/read-all`

**Integrations:**
* `POST /api/v1/notifications/slack/webhook`
* `POST /api/v1/notifications/jira/sync`

**Internal:**
* `POST /api/internal/notifications/dispatch`

### Error Response Pattern

All services use a consistent error envelope mapped via `@RestControllerAdvice`:

```json
{
  "success": false,
  "error": {
    "type": "VALIDATION|BUSINESS|AUTH|DOWNSTREAM_TIMEOUT",
    "code": "ERR_INVALID_PAYLOAD",
    "message": "Human readable message",
    "status": 400,
    "timestamp": "2026-09-08T10:00:01Z",
    "path": "/api/v1/alerts/ingest",
    "errors": [
      {
        "field": "source",
        "message": "must not be blank"
      }
    ]
  }
}
```

---

## 7. Inter-Service Communication Mapping

**Feign Communication (Synchronous)**

* `api-gateway` -> `ingestion-service`: route inbound webhook payloads (`POST /api/v1/alerts/ingest`)
* `triage-service` -> `audit-storage`: fetch historical ticket metadata for duplicate validation (`GET /api/internal/tickets/{ticketId}`)
* `ai-core-service` -> `audit-storage`: retrieve past RCA contexts for few-shot LLM prompting (`GET /api/internal/tickets/rca-history`)
* `api-gateway` -> `audit-storage`: dashboard read operations for active tickets and analytics (`GET /api/v1/tickets/...`)

**Kafka Communication (Asynchronous)**

| Producer | Topic | Events Produced |
|---|---|---|
| `ingestion-service` | `alert.received` | `alert.created` |
| `triage-service` | `alert.triaged` | `ticket.duplicate_found`, `ticket.requires_rca` |
| `ai-core-service` | `rca.generated` | `rca.completed`, `rca.failed` |

**Consumers:**
* `triage-service` (`alert.received`) -> generates vector embeddings and checks deduplication.
* `ai-core-service` (`alert.triaged`) -> triggers Spring AI prompt chain for unique issues.
* `notification-service` (`rca.generated`) -> dispatches Slack/Jira alerts with the final report.
* `audit-storage` (all topics) -> acts as a sink to update the MySQL state machine in real-time.

---

## 8. Apache Kafka Configuration

**Topics & Partitions**

* `alert.received`: 3 partitions. Keyed by `service_name` to ensure chronological ordering of alerts from the same source.
* `alert.triaged`: 3 partitions. Keyed by `incident_id`.
* `rca.generated`: 3 partitions. Keyed by `incident_id`.

**Consumer Groups**

* `triage-group`: subscribes to `alert.received`.
* `ai-core-group`: subscribes to `alert.triaged`.
* `notification-group`: subscribes to `rca.generated`.
* `audit-group`: wildcard regex subscription (`alert.*`, `rca.*`) for global state tracking.

**Retry / DLQ (Dead Letter Queue)**

* Configured using Spring Kafka `DefaultErrorHandler` with exponential backoff (e.g., retries at 2s, 4s, 8s) for transient failures like Vector DB timeouts.
* Non-retryable exceptions (e.g., malformed JSON parsing) or exhausted retries are published directly to `<topic_name>.DLT`.
* DLQ consumers log poison messages for manual intervention without blocking the main partition offset.

**Message Flow**

1. Monitoring webhook (e.g., Datadog) hits Gateway.
2. Ingestion service publishes raw JSON payload to `alert.received`.
3. Triage service consumes it, checks Vector DB for duplicates, and publishes result to `alert.triaged`.
4. AI Core service consumes the triaged event, builds context, calls the LLM, and publishes to `rca.generated`.
5. Audit Storage consumes from all topics sequentially to update the exact ticket status in MySQL.

---

## 9. Security Documentation

**Authentication Flow**

* **Webhooks (Machine-to-Machine):** Monitoring tools (Datadog, Sentry) authenticate via `X-API-Key` or HMAC signatures in the request headers.
* **Dashboard Users:** Engineers log in via auth service and receive JWT access + refresh tokens. Access tokens are signed with RSA private keys.
* API Gateway validates the API Keys (for webhooks) and parses JWTs (for dashboard requests).

**Authorization**

* Gateway injects trusted identity headers (`X-Tenant-Id`, `X-User-Role`) after validation.
* Downstream services use a `GatewayAuthFilter` to build the Spring `SecurityContext` from these headers.
* Endpoint-level `@PreAuthorize("hasRole('ROLE_ENGINEER')")` enforces strict access control for RCA reads/updates.

**API Key & Token Notes**

* Public routes: `/api/v1/alerts/ingest` (requires valid webhook API key), `/api/auth/login`.
* Internal endpoints: Any route matching `/api/internal/**` is strictly blocked externally at the Gateway level (Forbidden from outside).
* Tokens are stateless, but a Redis-based blacklist is maintained for immediate revocation on logout.

**Gateway-Level Security Controls**

* **Token & Key Validation:** Validates JWT claims or matching API keys against configured tenant secrets.
* **Anti-Header-Spoofing:** Actively removes client-supplied `X-Tenant-Id` or `X-User-*` headers to prevent malicious privilege escalation, replacing them with verified values.
* **Rate Limiting (Redis):** Strict route-specific policies (e.g., webhook ingestion is capped at 1000 req/sec per tenant to prevent DDoS during cascading system failures).

---

## 10. Database Design

**MySQL (Audit & State Schema)**

* `tickets`: id, tenant_id, source, service_name, severity, status (ENUM), raw_payload, created_at
* `ticket_rca`: id, ticket_id, summary, confidence_score, recommended_action, generated_at
* `audit_trail`: id, ticket_id, event_type (RECEIVED, TRIAGED, RCA_COMPLETED), kafka_topic, timestamp

**Vector Database (PgVector / Milvus - Semantic Schema)**

* `incident_embeddings`: id, ticket_id, service_name, text_chunk (concatenated stack trace & error message), embedding_vector (1536 dimensions)

**Redis (Cache & State)**

* `rate_limiter`: Tracks API Gateway request counts per tenant.
* `deduplication_locks`: Distributed locks to prevent race conditions when simultaneous duplicate alerts arrive.

**Relationship Overview (Text ER)**

* `tickets` 1 -> 1 `ticket_rca` (An incident has exactly one root cause analysis)
* `tickets` 1 -> many `audit_trail` (A ticket goes through multiple state transitions)
* `tickets` 1 -> 1 `incident_embeddings` (Each unique ticket gets vectorized for future duplicate checking)
* `ticket_rca` implicitly uses context from multiple past tickets via semantic proximity searches.

---

## 11. Configuration Management

* **Config Import:** Services import configuration via `spring.config.import=configserver:http://localhost:8888`.
* **Git Sourcing:** Config server sources properties dynamically from a dedicated Git repository (e.g., `resolve-ai-config`).
* **Profiles:** Primarily `dev` for local execution, with a `docker` overlay for the containerized runtime.
* **Dynamic Refresh:** Spring Cloud Bus refresh endpoint (`/actuator/busrefresh`) is available to propagate config updates across services without downtime (e.g., updating LLM prompt templates or similarity thresholds).
* **Encryption:** Encryption support is enabled in the config server (`encrypt.key`) to securely serve `{cipher}` values like database passwords.
* **Secret Handling Expectation:**
  * Keep `.env` files out of VCS (Git).
  * Inject sensitive keys (e.g., `OPENAI_API_KEY`, Webhook HMAC secrets) via environment variables or a secrets manager in production.
  * Keep API Gateway RSA/JWT keys under a mounted secret path in the container.

---

## 12. Docker & Deployment

**Docker Compose Layers**

* `infra.yaml`: Apache Kafka (Broker + Zookeeper/KRaft), Redis
* `databases.yaml`: MySQL, Vector DB (PgVector or ChromaDB)
* `services.yaml`: config-server, eureka-server, all core AI/domain services, api-gateway
* `observability-and-monitoring.yaml`: Prometheus, Grafana, Micrometer/Tempo (for tracing)

**Startup Order**

`infra` -> `db` -> `config-server` -> `eureka` -> `business services` -> `api-gateway`

**Networking**

* Shared bridge network: `resolve_ai_network`
* Inter-container DNS resolution using exact service names (e.g., `http://audit-storage:8084`).

**Volumes**

* Persistent data mounts for MySQL (`/var/lib/mysql`), Vector DB, Kafka logs, and Redis.
* Mounted `init-db.sql` for automated schema and user generation upon MySQL startup.
* Mounted configuration directories for Grafana dashboards and Prometheus scrape configs.

**Health Checks**

* Configured heavily across all containers to ensure the strict startup order.
* Uses Spring Boot Actuator (`/actuator/health`) for domain services.
* Uses native container commands for infrastructure (e.g., `mysqladmin ping` for MySQL, `redis-cli ping` for Redis, and native Kafka health checks).

---

## 13. Monitoring & Observability

**Metrics**

* All services expose `/actuator/prometheus`.
* Prometheus scrapes the API gateway, Kafka brokers, Redis, and all domain services.

**Logging**

* Structured service logs shipped by Promtail/Grafana Alloy from the Docker daemon.
* Loki stores centralized logs for easy querying.
* Grafana can query logs and correlate them with distributed traces.

**Tracing**

* Micrometer Tracing (with Zipkin/Tempo) is configured.
* `traceId` and `spanId` are propagated across synchronous Feign calls and asynchronous Kafka message headers to track an alert from ingestion to RCA generation.

**Dashboards & Alerts**

* Grafana datasources provisioned for Prometheus, Loki, and Tempo.
* Future scope: Explicit alerting rules for Kafka consumer lag and LLM API latency spikes.

---

## 14. Folder Structure

```text
infrastructure/
├── api-gateway/
├── config-server/
└── eureka-server/
services/
├── ingestion/
├── triage/
├── ai-core/
├── audit-storage/
└── notification/
common/
└── dto-library/
docker/
└── docker-compose/
    ├── dev/
    ├── local/
    └── observability/
secrets/
public/
web/
```

* `infrastructure/*`: platform-level services
* `services/*`: business microservices
* `common/dto-library`: shared event contracts and Kafka payload DTOs
* `docker/docker-compose/dev`: fully containerized environment
* `docker/docker-compose/local`: infra-only for running services from IDE
* `docker/docker-compose/observability`: monitoring/logging stack configs
* `web`: React-based incident triage and RCA dashboard

---

## 15. Design Patterns & Best Practices Used

* Layered architecture (controller -> service -> repository)
* DTO mapping (raw webhook payloads, internal events, HTTP responses)
* Retrieval-Augmented Generation (RAG) for LLM context enrichment
* Event-driven architecture with Kafka topic routing
* CQRS (Command Query Responsibility Segregation) separating fast Kafka ingestion from MySQL read/audit queries
* Idempotency guards (Vector DB similarity thresholds to prevent duplicate RCA processing)
* Circuit breaker + retry (Resilience4j) for external LLM API calls
* Standardized response/error wrapper
* Centralized exception handling per service (`@RestControllerAdvice`)
* Security boundary with gateway + internal API segregation

---

## 16. Setup Guide

**Prerequisites**

* Java 21
* Maven 3.9+
* Docker Desktop
* OpenAI API Key (or local LLM setup)

**Option A: Full Docker (recommended)**

```bash
cd docker/docker-compose/dev
cp .env.example .env
# Add your OPENAI_API_KEY to .env
docker-compose -f docker-compose.infra.yml -f docker-compose.services.yml up -d
```

Access:
* API Gateway: `http://localhost:8080`
* Eureka: `http://localhost:8761`
* Grafana: `http://localhost:3000`
* Prometheus: `http://localhost:9090`
* Kafka UI (Optional): `http://localhost:8081`

**Option B: Run infra in Docker, services from IDE**

```bash
cd docker/docker-compose/local
docker-compose up -d
```

Then start services in this exact order:
1. `config-server`
2. `eureka-server`
3. `ingestion`, `triage`, `ai-core`, `audit-storage`, `notification`
4. `api-gateway`

---

## 17. Testing

**Current Test Coverage in Repo**

* Basic Spring Boot context tests present in all services (`*ApplicationTests`).

**How to Run**

```bash
# service-wise
cd services/ai-core && mvn test
cd services/triage && mvn test
# repeat for others
```

**API Testing**

* Use Postman/Insomnia through the gateway (`localhost:8080`).
* Authenticate first by providing the `X-API-Key` header for protected webhook flows.
* Validate async side effects (vector embeddings creation, RCA generation, Slack notifications) by checking the `audit-storage` database states.

---

## 18. Future Improvements

* Add a dedicated DLQ processing UI for failed Kafka events.
* Complete OpenTelemetry rollout with distributed trace propagation across all Spring AI boundaries.
* Add Testcontainers-based integration tests for MySQL, PgVector, and Kafka.
* Introduce CI pipeline (GitHub Actions) for unit/integration/security checks.
* Externalize secrets to HashiCorp Vault or AWS Secrets Manager for production.
* Add Kubernetes manifests/Helm charts and HPA policy for scale-out.
* Implement local LLM support (Ollama/vLLM) for fully air-gapped, on-premise RCA generation.

---

## Appendix: Quick API Catalog

**Public/Client APIs by Service**

* Ingestion: `/api/v1/alerts/*`
* Triage: `/api/v1/triage/*`
* AI-Core: `/api/v1/rca/*`
* Audit-Storage: `/api/v1/tickets/*`
* Notification: `/api/v1/notifications/*`

**Internal APIs (service-to-service)**

* `/api/internal/alerts/*`
* `/api/internal/triage/*`
* `/api/internal/rca/*`
* `/api/internal/tickets/*`
* `/api/internal/notifications/*`

> **Note:** Gateway blocks external access to internal endpoints.
