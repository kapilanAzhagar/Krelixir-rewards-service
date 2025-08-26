Rewards & Redeem Microservices — Architecture and Runbook

Author: Kapil A. — Solution Architect
Purpose: Complete technical README describing the end-to-end design, tech stack, runtime flow, dev/run instructions, deployment and operational guidance for the Rewards and Redeem services. This document is written as an architect-level design and runbook you can use in interviews and for implementation.

Table of contents

Summary

Architecture overview

Technology stack

Components and responsibilities

Data and event model

API surface (synchronous)

Event topics (asynchronous)

Security and authentication

Resiliency and reliability

Caching strategy

Observability and tracing

Local development (dev profile, in-memory cache)

Build, containerize and ECR push

Kubernetes deployment (ECS / EKS notes)

CI/CD and release strategy

Testing strategy

Troubleshooting & runbook snippets

Tradeoffs and alternatives

Next steps and extensions

1. Summary

This repository implements two microservices:

Rewards Service (Resource Service) — calculates awardable points by consuming transactional events (hotel stays, restaurant reservations, casino spend) and responds to direct queries for a customer’s current points. It publishes confirmation events.

Redeem Service — consumes redemption requests, validates balances, performs deduction, and publishes confirmation events for downstream systems.

System is event-driven (Kafka/MSK) and also supports synchronous query APIs via OAuth2-secured endpoints. Images are stored in AWS ECR and services run on AWS container platform. Redis is used for caching; RDS for persistent storage. The design prioritizes scalability, observability, and resilient integration with external partner applications.

2. Architecture overview

High-level architecture components and flows:

Partner applications (Hotel, Casino, Dining) authenticate with Auth Service (OAuth2 client credentials) and publish business events to Kafka (MSK).

Kafka topics:

reward-events — business events describing transactions that should generate points

reward-confirmed — acknowledgements and enriched events after rewards are applied

redeem-events — redemption requests

redeem-confirmed — final confirmation events for successful (or failed) redemptions

Rewards Service consumes reward-events, applies rules, updates RDS and Redis, and emits reward-confirmed.

Redeem Service consumes redeem-events, validates using Redis/RDS, updates stores, emits redeem-confirmed.

Auth Service issues JWT tokens; API Gateway validates and routes external requests.

Monitoring: CloudWatch + Prometheus + Grafana + distributed tracing (OpenTelemetry).

Diagrammatically:

Partner Apps -> API Gateway / direct Kafka producers -> Kafka (MSK)
                       |                                   |
                       v                                   v
                   Auth Service                      Rewards Service <-> RDS / Redis
                                                          |
                                                          v
                                                     reward-confirmed -> downstream
                                                          |
                                                     Redeem Service (consumes redeem-events)

3. Technology stack

Primary choices (production-ready):

Language: Java 17

Frameworks: Spring Boot 3.x, Spring Security (OAuth2 Resource Server)

Event streaming: Apache Kafka / Amazon MSK

Cache: Redis (Amazon ElastiCache) or in-memory for dev

Persistence: PostgreSQL (Amazon RDS)

Auth: OAuth2 / OpenID Connect (Keycloak / AWS Cognito / Okta)

Containerization: Docker

Container registry: AWS ECR

Orchestration: AWS ECS (Fargate) or EKS

Observability:

Metrics: Micrometer + Prometheus (scrape) → CloudWatch / Grafana

Tracing: OpenTelemetry (Jaeger compatible)

Logs: JSON structured logs → CloudWatch / ELK

Resilience: Resilience4j (Retry, Circuit Breaker, Bulkhead)

Secrets: AWS Secrets Manager

CI/CD: GitHub Actions / AWS CodePipeline (build → test → image → ECR → deploy)

Infra as code: Terraform / CloudFormation / Helm Charts for Kubernetes

4. Components and responsibilities

Auth Service

Issues access tokens for partner systems using client credentials.

Stores client metadata and secrets securely.

Publishes JWKS for token verification.

API Gateway (ALB / API Gateway)

Provides external entry point.

Enforces TLS, rate limiting, WAF rules, and route-level ACLs.

Rewards Service

Consumes reward-events from Kafka.

Calculates points using rules engine (plugin/config-driven).

Updates persistent ledger (RDS) and cache (Redis).

Publishes reward-confirmed events and metrics.

Exposes GET /api/v1/rewards/{customerId} for runtime calculation and POST /api/v1/rewards/query for bulk.

Redeem Service

Consumes redeem-events.

Authenticates & authorizes redemption requests.

Validates points, reserves/deducts atomically.

Publishes redeem-confirmed.

Supporting components

Kafka (MSK): durable event streaming, topic retention policies defined.

RDS: normalized schema for transactions and customer balances.

ElastiCache Redis: hot cache for balances and rate-limiting counters.

Monitoring, tracing, secrets - central.

5. Data and event model

Example event schema (JSON)

reward-event:

{
  "eventId":"uuid",
  "eventType":"HOTEL_STAY|DINING|CASINO_SPEND",
  "customerId":"string",
  "timestamp":"2025-08-26T11:00:00Z",
  "payload": { ... }   // type-dependent data
}


reward-confirmed:

{
  "eventId":"uuid",
  "customerId":"string",
  "pointsAwarded": 50,
  "balanceAfter": 1230,
  "sourceEventId":"uuid",
  "timestamp":"2025-08-26T11:00:01Z"
}


redeem-event:

{
  "redeemRequestId":"uuid",
  "customerId":"string",
  "requestedPoints": 100,
  "merchantContext": { "merchantId":"M1", "orderId":"O123" },
  "timestamp":"2025-08-26T12:00:00Z"
}


redeem-confirmed:

{
  "redeemRequestId":"uuid",
  "customerId":"string",
  "pointsDeducted": 100,
  "balanceAfter": 1130,
  "status":"SUCCESS|FAILED",
  "timestamp":"2025-08-26T12:00:01Z"
}


Schema design note: Keep payloads small; store full audit data in RDS; use event versioning and schema registry (Confluent / AWS Glue Schema Registry) for contract-safe evolution.

6. API surface (synchronous)

Rewards Service (secured by OAuth2):

GET /api/v1/rewards/{customerId}

Query params: from (ISO date), to (ISO date), breakdown (boolean, default true)

Auth: Bearer token with scope rewards.read

Responses:

200 JSON with totalPoints, breakdown, calculationTimestamp

400 invalid params

401 unauthorized

503 upstream unavailability (with partial results possible)

POST /api/v1/rewards/query

Bulk queries. Body: list of {customerId, from, to}.

Response: list of reward responses.

Redeem Service:

POST /api/v1/redeem

Body: redeem-event JSON.

Auth: Bearer token with scope rewards.redeem

Behavior: validates, pushes redeem-events to Kafka or processes synchronously with guaranteed idempotency. Returns 202 Accepted with redeemRequestId or 409 for conflicts.

Idempotency and correlation: require idempotency-key header for POST operations; persist processed keys with TTL.

7. Event topics (asynchronous)

reward-events — incoming events from partners. Retention: 7–30 days depending on replay needs.

reward-confirmed — positive acknowledgements; subscribed by reporting, notifications, and reconciliation jobs.

redeem-events — incoming redemption requests.

redeem-confirmed — final history for billing, settlements.

dead-letter — failed events for manual inspection (long retention).

Partitioning: partition by customerId (consistent hashing) to preserve ordering per customer. Producer partitioning must be deterministic.

8. Security and authentication

Use OAuth2 Authorization Server (Keycloak/Cognito) as IdP.

Partner apps use client credentials flow to obtain tokens.

Resource servers validate JWT locally using IdP’s JWKS endpoint (no network call on each request).

Use scopes and roles:

rewards.read, rewards.write, rewards.redeem

Protect Kafka: use mTLS or SASL/SCRAM with IAM policies for MSK.

Secrets: store DB connections and client secrets in AWS Secrets Manager; do not bake secrets into images.

Network segmentation: ECS tasks / EKS pods in private subnets with security groups restricting access to MSK, RDS, ElastiCache.

Rate limiting: API Gateway enforces quotas per client. Throttle redemption endpoints more strictly.

9. Resiliency and reliability

Resilience patterns:

Circuit Breaker & Retry (Resilience4j) for synchronous upstream calls (Auth introspection, if used).

Bulkheads for upstreams to protect CPUs.

Retries with exponential backoff on network errors, but not for 4xx errors.

Idempotency:

Store eventId / redeemRequestId in RDS (or Redis) as processed marker; reject duplicates.

Durability:

Kafka for durable event storage; use at-least-once semantics.

Consumers must be idempotent.

Back-pressure and throttling:

Producers (partners) have quotas enforced at API Gateway.

Consumers monitor backlog and scale using HPA based on consumer lag metrics.

Disaster recovery:

RDS snapshots, cross-region replication if required.

MSK cross-region replication or failover strategy.

10. Caching strategy

What to cache:

Customer balance (hot), recent transactions, partner metadata.

Cache layers:

Primary: ElastiCache (Redis) with TTL and optimistic locking patterns.

Fallback: in-memory cache for local dev (application-dev.yml uses spring.cache.type=simple).

Cache invalidation:

On updates (award or redeem), update Redis immediately after DB commit (transactional outbox recommended).

Use event-driven invalidation: store change in DB, publish event, and consumer updates Redis.

11. Observability and tracing

Metrics:

Expose Micrometer metrics via /actuator/prometheus.

Custom metrics: processed events/sec, consumer lag, points awarded histogram, redemption success/failure counters.

Tracing:

Instrument services with OpenTelemetry (propagate traceparent).

Export to Jaeger/CloudWatch X-Ray.

Logging:

Structured JSON logs with requestId, customerId, eventId.

Log levels controlled by environment.

Dashboards & alerts:

Grafana dashboards: consumer lag, p99 latency, error rates, cache hit ratio.

Alerts: high consumer lag, steady error rate > threshold, DB connection pool exhausted.

Audit:

Persistent audit records in RDS (transaction table). Events are stored in Kafka and can be rehydrated.

12. Local development (dev profile)

Use application-dev.yml for running without Redis or Kafka.

Dev profile features:

spring.cache.type: simple (in-memory cache).

Mocked Kafka producers/consumers (in-memory or embedded Kafka if needed).

Embedded H2 or local PostgreSQL via Docker (optional).

Dev OAuth token generator endpoint /dev/token (permit in SecurityConfig) to get a JWT for Postman tests.

Steps to run locally (Windows without Docker Redis):

Ensure Java 17 is installed. If you don’t have Maven, use the Maven Docker image to build.

Create application-dev.yml under src/main/resources:

spring:
  cache:
    type: simple
logging:
  level:
    com.example.rewards: DEBUG


Run:

mvn -Dspring-boot.run.profiles=dev spring-boot:run
# Or from IDE set VM arg: -Dspring.profiles.active=dev


Get the dev JWT printed on startup or call /dev/token if implemented:

GET http://localhost:8080/dev/token


Test:

GET http://localhost:8080/api/v1/rewards/{customerId}
POST http://localhost:8080/api/v1/redeem

13. Build, containerize and ECR push

Sample steps (CI will run these):

Build JAR:

mvn clean package -DskipTests


Build Docker image:

docker build -t rewards-service:latest .


Authenticate Docker to ECR:

aws ecr get-login-password --region <REGION> | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com


Tag & push:

docker tag rewards-service:latest <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/rewards-service:latest
docker push <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/rewards-service:latest


(Optional) Use multi-arch or multi-stage Dockerfile to build inside container in CI pipeline.

CI note: Build and push should be triggered on merge to main with semantic version tags. Keep images immutable and tagged by commit SHA.

14. Kubernetes / ECS deployment notes

Use Helm charts for EKS; use task definitions for ECS.

Secrets: mount from AWS Secrets Manager or SSM Parameter Store via CSI driver.

Health checks:

Liveness: /actuator/health/liveness

Readiness: /actuator/health/readiness

Auto-scaling:

Horizontal Pod Autoscaler (CPU/memory or custom metrics like consumer lag).

Networking:

Place services in private subnets; use NAT gateway for outbound access to S3 or IdP if needed.

Service Mesh:

Optional Istio/Linkerd for mTLS, traffic shifting, retries.

Rolling update strategy:

Use readiness probes; gradually update with canary if required.

15. CI/CD and release strategy

Pipelines:

build → compile, unit tests, static analysis (spotbugs/checkstyle)

integration → run contract tests with WireMock/embedded Kafka

image-build → Docker build, scan image, tag

deploy-staging → deploy to staging cluster

smoke-tests → run post-deploy tests

promote-prod → manual approval to deploy to production

Rollback strategy:

Keep previous image tag available; if failures exceed threshold in monitoring, rollback via deployment revision.

Canary deployments:

Use weighted traffic routing to test new version on a small percentage before full rollout.

16. Testing strategy

Unit tests: pure logic classes (calculator, validation).

Integration tests:

WireMock for upstream HTTP stubs (Auth Service).

Embedded Kafka or Testcontainers MSK for event flows.

Testcontainers PostgreSQL for DB interactions.

Contract tests:

Provider/consumer tests for Kafka topics (schema and semantics).

End-to-end tests:

Deploy to ephemeral environment and run scenario tests (award → redeem flow).

17. Troubleshooting & runbook snippets

Consumer lag high:

Check consumer application logs; inspect kafka-consumer-groups --describe --group <group> to see offsets; scale consumers based on lag.

Unexpected 401:

Confirm token iss and aud claims match expected values; validate JWKS keys.

Cache inconsistency after redemption:

Ensure DB update is committed before publishing event or implement transactional outbox pattern.

Dead-lettered events:

Inspect dead-letter topic; reprocess after fixing root cause with a replay utility or a manual replayer.

Database connection issues:

Check connection pool size; ensure RDS security group allows application traffic.

18. Tradeoffs and alternatives

Runtime calculation vs ledger:

Runtime: always current, simpler for fresh data, but higher compute per request.

Ledger: efficient read for frequent queries, more storage and complexity for reconciliation. A hybrid approach (runtime + cached snapshots or nightly precomputation) is often optimal.

Kafka vs direct synchronous:

Kafka decouples and scales; provides durable events for auditing and replay. It adds operational complexity (MSK/EKS).

Synchronous APIs are simpler but couple systems tightly and can block partner flows.

Sidecar vs OAuth2:

Sidecar offloads validation but increases operational footprint.

OAuth2 with local JWT validation scales better and follows standard practices; recommended for multi-team enterprises.

19. Next steps and extensions

Implement transactional outbox to ensure DB → Kafka consistency.

Add schema registry for event versioning.

Implement fine-grained role and scope model for partner-specific privileges.

Add reconciliation job to compare events vs RDS data and repair inconsistencies.
