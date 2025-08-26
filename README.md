README — Rewards & Redeem Microservices (Enterprise Architecture)

Author: Kapil A.
Role: Solution Architect
Repository: rewards-platform
Last updated: 2025-08-26

Overview

This repository contains a production-ready skeleton and deployment guidance for an event-driven Rewards and Redeem platform built for the hospitality domain. The platform computes reward points in real-time from Hotel, Casino and Dining transactional events and supports redemption workflows. The system is designed for AWS with containerized services, an event backbone (Kafka / MSK), secure OAuth2 authentication, resilient communication, and observability.



<img width="1671" height="905" alt="image" src="https://github.com/user-attachments/assets/61f5ddba-ab5a-4cb2-8038-2f045ca4fa5e" />


This document describes:

high-level architecture and design decisions

components and responsibilities

tech stack and rationale

local development and run instructions (dev/profile)

build, containerize and push to ECR

deployment guidance (ECS / Fargate + MSK)

operational concerns: security, resilience, scaling, monitoring

testing, CI/CD and next steps

Read this as a solution architect briefing to present in interviews and to use as a checklist for implementation.

High-level architecture
[Hotel App]   [Casino App]   [Dining App]
     \            |               /
      \           |              /
       +--------> API Gateway <--------- (optional)
             (synchronous APIs)             (external clients authenticate via OAuth2)
                     |
                     v
               [Auth Service]
                 (OAuth2 Server)
                     |
                     v
Clients publish events -> [Kafka / MSK] <- Services consume and produce events
                          /    |    \
                         /     |     \
             [Rewards Service] [Redeem Service] [Notification / Reporting]
                 (consumes reward-events)   (consumes redeem-events)
                     |                          |
                     v                          v
                  RDS (Postgres)           RDS (Postgres)
                  Redis (ElastiCache)      Redis (ElastiCache)
                  CloudWatch               CloudWatch


All services run as containers (images stored in AWS ECR) and execute in AWS ECS (Fargate) or Kubernetes.

Auth Service issues JWTs (Client Credentials flow) for service-to-service and partner authentication.

Kafka (MSK) is the event backbone. Topics: reward-events, reward-confirmed, redeem-events, redeem-confirmed.

Rewards Service calculates points at runtime (no ledger), persists events to RDS, caches hot balances in Redis, emits confirmation events.

Redeem Service consumes redemption requests, validates and adjusts balances, emits confirmation events consumed by Notification/Reporting.

Key design decisions & rationale

Event-driven core (Kafka/MSK): decouples producers and consumers, supports high-throughput ingestion (hotel/casino/dining spikes), and enables eventual consistency for non-critical reads.

Runtime calculation, no ledger: points are computed on-demand from canonical transactional events. This guarantees data freshness, simplifies reconciliation, and avoids dual writes. If ingestion volume grows, we can optionally maintain precomputed aggregates (snapshots) for hot customers.

OAuth2 (OpenID Connect) for auth: industry standard, stateless JWT validation, scope-based access controls, easy third-party integration.

Stateless microservices: scale horizontally; services are lightweight, maintain no in-process state except cache.

Caching layer (Redis/ElastiCache): reduces repeated computation and upstream calls; short TTL for freshness.

Resilience patterns: retries, circuit-breakers, bulkheads (Resilience4j), and bounded thread pools to protect from cascading failures.

Observability: structured logs, correlation IDs, OpenTelemetry/Prometheus metrics, and CloudWatch integration.

Technology stack

Languages & Frameworks

Java 17+

Spring Boot 3.x (Web, Security, Data Redis, Actuator)

Resilience4j (retry, circuit-breaker, bulkhead)

Spring Kafka (consumer/producer)

Spring Security (OAuth2 resource server)

Jackson (JSON serialization)

Infrastructure / Platform

AWS ECR — container registry

AWS ECS (Fargate) or EKS — container runtime

Amazon MSK — managed Kafka

Amazon RDS (Postgres) — canonical storage

Amazon ElastiCache (Redis) — caching

Amazon API Gateway / ALB — routing & edge security

AWS Secrets Manager — secure secrets

Amazon CloudWatch — logs/metrics

Amazon S3 — artifacts / archival

Optional: Service Mesh (Istio / AWS App Mesh) for mTLS and policy

CI/CD / Tooling

GitHub Actions (or CodeBuild) pipelines

Docker / Dockerfile multi-stage builds

Terraform / CloudFormation for infra as code

WireMock for contract tests

JUnit + Mockito for unit tests

Repo structure (recommended)
rewards-platform/
├── README.md
├── rewards-service/
│   ├── src/
│   ├── Dockerfile
│   ├── pom.xml
│   └── ...
├── redeem-service/
│   ├── src/
│   ├── Dockerfile
│   ├── pom.xml
│   └── ...
├── auth-service/
│   ├── src/
│   ├── Dockerfile
│   ├── pom.xml
│   └── ...
├── infra/
│   ├── terraform/ or cloudformation/
│   └── k8s/ or ecs-manifests/
├── docs/
│   ├── architecture-diagrams/   (mermaid / png / plantuml)
│   └── runbooks/
└── .github/workflows/

Local development (no Docker / Windows dev profile)

Use application-dev.yml for local runs that employ an in-memory cache and embedded connectors. This is intended for developer speed when real infra (Redis, MSK) is not available.

Create file: src/main/resources/application-dev.yml

spring:
  cache:
    type: simple
logging:
  level:
    com.example.rewards: DEBUG


Run locally (with dev profile)

With Maven (if installed):

mvn -Dspring-boot.run.profiles=dev spring-boot:run


Or, from your IDE: set VM arg -Dspring.profiles.active=dev and run RewardsApplication.

Notes

@Cacheable will use ConcurrentMapCacheManager (in-memory).

Use mocked clients or WireMock endpoints for upstream Hotel/Casino/Dining data.

Use an embedded Kafka (testcontainers) or local Kafka for integration testing.

Building, Containerizing and pushing to ECR

Dockerfile (example pattern)

Multi-stage build for projects that produce a WAR/JAR.

Example steps:

Build stage: use Maven container to build artifact.

Runtime stage: use JRE / Tomcat or run java -jar.

Copy artifact into final image.

Build locally via Docker (no Maven installed)

# Build using Maven image to produce JAR
docker run --rm -v "$PWD":/app -w /app maven:3.9.4-eclipse-temurin-17 mvn -DskipTests clean package

# Build image
docker build -t rewards-service:latest .

# Authenticate to ECR
aws ecr get-login-password --region <AWS_REGION> | docker login --username AWS --password-stdin <ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com

# Tag & push
docker tag rewards-service:latest <ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com/rewards-service:latest
docker push <ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com/rewards-service:latest


CI/CD pipeline (high level)

GitHub Actions / CodePipeline:

Checkout

Maven build -> run tests -> package

Build Docker image -> tag

Push image to ECR

Deploy to ECS / update task definition (or push manifest to EKS)

Deployment guide — AWS ECS (Fargate) + MSK

Prerequisites

ECR repository for each service

VPC with subnets and security groups (private subnets for ECS tasks & MSK)

MSK cluster provisioned with required topics

RDS instance (Postgres) and ElastiCache (Redis)

Secrets stored in AWS Secrets Manager (DB creds, OAuth client secrets)

CloudWatch log groups and IAM roles for ECS tasks

Core steps

Push images to ECR (see above).

Create IAM roles: ECS task role and execution role.

Create task definitions (container definitions, env vars, secrets).

Create ECS service (Fargate) with desired count and autoscaling policies.

Configure MSK consumer groups in your services; wire brokers via VPC endpoints.

Configure ALB or API Gateway with target groups pointing to ECS services.

Setup health checks, liveness & readiness endpoints via Spring Actuator.

Configure CloudWatch alarms (error rate, CPU, consumer lag).

Deploy and verify topic consumption and event flow.

Security

Authentication & Authorization

Use an OAuth2 Authorization Server (Keycloak / Cognito / custom Auth Service).

For machine-to-machine, prefer Client Credentials flow.

Use JWTs signed by IdP; services validate JWT locally against JWKS endpoint.

Secrets

Use AWS Secrets Manager to retrieve DB credentials, OAuth client secrets, and any private keys.

Grant least-privilege IAM roles to ECS tasks for secrets retrieval.

Network

Place MSK, RDS and ElastiCache in private subnets.

Use security groups for least-privileged access (ECS tasks only to MSK brokers and RDS).

Optionally enable mTLS between services using Service Mesh.

Data protection

Encrypt RDS and EBS volumes at rest.

Use TLS for all service-to-service and external communications.

Resilience & Observability

Resilience

Use Resilience4j pattern for retries, circuit-breakers, and bulkheads at adapter boundaries.

Use bounded thread pools for Kafka consumer processing to avoid resource exhaustion.

Configure graceful shutdown and idempotent consumers to handle reprocessing.

Observability

Use Spring Boot Actuator: /actuator/health, /actuator/metrics, /actuator/prometheus.

Emit and scrape Prometheus metrics (prometheus micrometer) and forward to CloudWatch (or use Prometheus + Grafana).

Correlate logs using a request id (MDC) and include trace ids in logs (OpenTelemetry).

Monitor Kafka consumer lag, processing error rate, and downstream service latencies.

Testing strategy

Unit tests (JUnit 5, Mockito) for controllers, service logic and the rewards calculator rules.

Contract tests (Pact / WireMock) for upstream integrations (hotel/casino/dining).

Integration tests using Testcontainers:

local Kafka container

embedded Postgres or containerized Postgres

local Redis / embedded cache for verification

End-to-end tests:

Deploy to a staging environment wired to MSK and run full flow: publish event → Rewards consumes → confirm DB & emitted event.

Monitoring & runbooks

Key dashboards:

Request latency (API Gateway / ALB)

Rewards processing latency and throughput

Kafka consumer lag per topic / partition

Error rates and exceptions per service

Redis hit ratio and RDS query latency

Runbook essentials:

How to recover from consumer lag: check MSK health, scale consumer tasks, inspect errors.

How to reprocess events: use offsets to reconsume or create consumer tool to read historical topic.

Rolling upgrade steps: drain connections, update task definition, validate health checks.

Tradeoffs and alternatives

Runtime calculation vs ledger:

Runtime pros: always up-to-date, simpler data model, easier reconciliation.

Runtime cons: computation cost and latency on heavy requests. Alternative: maintain hybrid approach — snapshot aggregates for high-volume customers.

MSK vs SNS/SQS:

Kafka fits high-throughput, ordered streaming with exactly-once or at-least-once semantics.

SNS/SQS is simpler for fanout/queueing but lacks the rich stream processing semantics.

OAuth2 vs sidecar:

OAuth2 (IdP + JWT) is the recommended, standardized approach for internal and partner security.

Sidecar can be used to enforce policy at the host level or with a service mesh.

Operation checklist (pre-deploy)

 Infra defined and tested via Terraform (VPC, MSK, RDS, ElastiCache, ECR, ECS)

 ECR repositories created and images pushed

 IAM roles and policies validated (least privilege)

 Secrets added to Secrets Manager

 Health checks and readiness endpoints implemented

 Actuator endpoints enabled and secured (not publicly exposed)

 Logging and monitoring configured (CloudWatch + Prometheus)

 CI pipeline configured to run tests and push images

 Run smoke tests in staging (end-to-end)
