# Resolve AI ⚡

**Enterprise-grade event-driven AIOps platform built using Spring Boot microservices, Apache Kafka, Spring AI, and vector databases for automated support ticket triage and Root Cause Analysis (RCA).**

![Java](https://img.shields.io/badge/Java-21-ED8B00?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.4-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![Spring AI](https://img.shields.io/badge/Spring_AI-LangChain4j-6DB33F?style=flat-square)
![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-Event_Driven-231F20?style=flat-square&logo=apachekafka&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Microservices-2496ED?style=flat-square&logo=docker&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-Persistence-4479A1?style=flat-square&logo=mysql&logoColor=white)

<br>

![Resolve AI System Architecture](./architecture-diagram.png)

---

## 1. Project Overview

### Problem Statement
Engineering and support teams spend hours manually triaging incoming alerts, searching for duplicate issues, and digging through logs to find the root cause of an incident. 

### Why This Application Was Built
This project was built to demonstrate production-style AI integration within a distributed backend architecture, showcasing:
* Service decomposition and event-driven messaging (Apache Kafka).
* Retrieval-Augmented Generation (RAG) pipelines for context enrichment.
* LLM-based classification and confidence routing.
* Resilience, observability, and containerized deployment.

### Business Value
* **Faster Resolution:** Reduces MTTR (Mean Time to Resolution) via automated initial triage.
* **Deduplication:** Eliminates redundant ticket tracking using vector embeddings.
* **Scalability:** Event-streaming handles high-velocity alert ingestion without dropping requests.
* **Clean Security Boundary:** API Gateway secures internal services while exposing a unified ingress.

---

## 2. Architecture Overview

### Request Lifecycle
1. Client/Monitoring Tool sends a POST webhook to `api-gateway`.
2. Gateway routes to `ingestion-service` which validates the payload.
3. `ingestion-service` publishes the raw event to Kafka (`alert.received`) and returns 202 Accepted.
4. `triage-service` consumes the event, converts it to a vector embedding, and queries the Vector DB for duplicates.
5. If unique, `ai-core-service` classifies the severity and generates an RCA using Spring AI.
6. The final output is persisted in MySQL and dispatched via the `notification-service`.

### Sync vs Async Communication
* **Synchronous (OpenFeign):** Dashboard UI fetching ticket status and RCA reports from the audit service.
* **Asynchronous (RabbitMQ/Kafka):** The AI processing pipeline (Ingestion -> Triage -> RCA -> Notify) uses event queues to prevent HTTP blocking during slow LLM API calls.

---

## 3. Tech Stack Documentation

### Backend & Infrastructure

| Technology | What it is | Why chosen / Problem solved |
| :--- | :--- | :--- |
| **Java 21** | LTS JVM runtime | Modern language features (Virtual Threads) for I/O heavy AI network calls. |
| **Spring Boot 3.4.x** | App framework | Rapid microservice bootstrap with production-ready actuator features. |
| **Spring AI & LangChain4j** | AI Frameworks | Standardized interfaces for LLMs and Vector Store integrations. |
| **Apache Kafka** | Message Broker | Decouples services and handles high-throughput webhook ingestion reliably. |
| **Spring Cloud Gateway** | API Gateway | Centralized routing, token validation, and rate limiting. |
| **OpenFeign** | HTTP Client | Declarative synchronous inter-service calls. |
| **MySQL** | Relational DB | Highly structured persistence with robust audit trails for ticket lifecycle. |
| **Vector DB** | Semantic Store | Stores high-dimensional text embeddings for fast cosine-similarity deduplication. |
| **Docker** | Containerization | Isolated, reproducible multi-container deployment environments. |

---

## 4. Microservice Breakdown

| Service | Port | Database | Key Responsibilities & Dependencies |
| :--- | :--- | :--- | :--- |
| **api-gateway** | 8080 | Redis | Ingress routing, Auth validation, Rate limiting. |
| **ingestion** | 8081 | None | Validates payloads, produces to Kafka. |
| **triage** | 8082 | Vector DB | Kafka consumer/producer, creates embeddings, checks duplicates. |
| **ai-core** | 8083 | None | LLM integration, generates RCA summaries and confidence scores. |
| **audit-storage**| 8084 | MySQL | Persists state changes and RCA results via JPA. |
| **notification** | 8085 | None | Consumes completed RCAs and dispatches to Slack/Email. |

---

## 5. API Documentation

### Ingestion Service (Public Ingress)
`POST /api/v1/alerts/ingest`

**Sample Request:**
```json
{
  "source": "DATADOG",
  "service": "payment-gateway",
  "error_message": "Connection timeout acquiring database lock",
  "stack_trace": "...",
  "timestamp": "2026-09-08T10:00:00Z"
}
```
### Audit Service (Internal Read Path)
`GET /api/v1/tickets/{ticketId}/rca`

**Sample Response:**
```json
{
  "ticketId": "TKT-101",
  "status": "RCA_GENERATED",
  "confidenceScore": 0.92,
  "isDuplicate": false,
  "rca_summary": "Database lock timeout caused by long-running reporting query in payment-gateway.",
  "recommended_action": "Kill PID 4592 and isolate reporting queries to the read-replica."
}

