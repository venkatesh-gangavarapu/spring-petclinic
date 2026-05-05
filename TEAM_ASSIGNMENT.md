# Spring PetClinic Microservices — Team Assignment
## From Local Development to AWS Production

> **Project:** PetClinic Cloud Migration  
> **Jira Project Key:** `PC`  
> **Team Size:** 10 Members  
> **Duration:** 10 Weeks (5 Sprints × 2 Weeks)  
> **Goal:** Build, containerize, and deploy all microservices to AWS ECS Fargate in production

---

## Table of Contents
1. [Project Summary](#1-project-summary)
2. [Team Roles & Ownership](#2-team-roles--ownership)
3. [AWS Target Architecture](#3-aws-target-architecture)
4. [Jira Epics & Stories](#4-jira-epics--stories)
5. [Sprint Plan](#5-sprint-plan)
6. [Definition of Done](#6-definition-of-done)
7. [Technical Prerequisites](#7-technical-prerequisites)
8. [Service Ports & Access URLs](#8-service-ports--access-urls)
9. [Branch & PR Strategy](#9-branch--pr-strategy)

---

## 1. Project Summary

This project takes the **Spring PetClinic Microservices** application and tasks a 10-person team with building every service from scratch, writing tests, containerizing with Docker, setting up CI/CD, and deploying to AWS ECS Fargate with full observability.

### What You Are Building

A veterinary clinic management system with 8 microservices:

```
                        ┌─────────────────────────────────────────────┐
                        │              AWS Cloud (us-east-1)           │
                        │                                              │
Browser ──► Route53 ──► │  ALB (HTTPS:443)                            │
                        │   └──► api-gateway:8080 (ECS Fargate)        │
                        │         ├──► customers-service:8081           │
                        │         ├──► visits-service:8082              │
                        │         ├──► vets-service:8083                │
                        │         └──► genai-service:8084               │
                        │                                              │
                        │  Infrastructure:                              │
                        │   config-server:8888  (pulls from GitHub)    │
                        │   discovery-server:8761 (Eureka)             │
                        │                                              │
                        │  Observability:                              │
                        │   zipkin:9411 | grafana:3030                 │
                        │   prometheus:9091 | admin-server:9090        │
                        │                                              │
                        │  Data:                                       │
                        │   RDS MySQL 8.x (Multi-AZ)                   │
                        └─────────────────────────────────────────────┘
```

### Technology Stack
| Layer | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 4.0.1, Spring Cloud 2025.1.0 |
| Frontend | AngularJS 1.8.3 (served by API Gateway) |
| Service Discovery | Netflix Eureka |
| Config Management | Spring Cloud Config Server |
| API Gateway | Spring Cloud Gateway + Resilience4j |
| AI Chatbot | Spring AI + OpenAI GPT-4o-mini |
| Database | HSQLDB (local) → RDS MySQL 8.x (AWS) |
| Containers | Docker + AWS ECR |
| Orchestration | AWS ECS Fargate |
| CI/CD | GitHub Actions |
| Infrastructure | Terraform |
| Monitoring | Prometheus + Grafana + Zipkin + CloudWatch |

---

## 2. Team Roles & Ownership

### Member Assignments

| Member | Role | Primary Service(s) | Jira Stories |
|---|---|---|---|
| **M1** | Project Lead / Cloud Architect | All services (review), AWS account, architecture | PC-1 to PC-5, PC-69 to PC-74 |
| **M2** | DevOps Engineer — Infrastructure | AWS VPC, ECS, RDS, ALB, Route53, ACM, Terraform | PC-40 to PC-56 |
| **M3** | DevOps Engineer — CI/CD | GitHub Actions pipelines, ECR, Docker builds | PC-30 to PC-39 |
| **M4** | Backend Developer | Config Server + Discovery Server | PC-6 to PC-10 |
| **M5** | Backend Developer | Customers Service | PC-11 to PC-14 |
| **M6** | Backend Developer | Vets Service | PC-15 to PC-18 |
| **M7** | Backend Developer | Visits Service | PC-19 to PC-21 |
| **M8** | Backend Developer | API Gateway + Frontend | PC-22 to PC-25 |
| **M9** | Backend Developer | GenAI Service | PC-26 to PC-29 |
| **M10** | QA / Observability Engineer | Testing, Grafana, CloudWatch, Alerts | PC-57 to PC-68 |

### Collaboration Rules
- All code must be reviewed by at least **1 other team member** before merge
- M1 must approve all infrastructure and architecture changes
- M10 must sign off on every service before it moves to production
- Daily standup: What did you do? What will you do? Any blockers?
- Sprint demo every 2 weeks — every member presents their service

---

## 3. AWS Target Architecture

### Infrastructure Diagram
```
┌──────────────────────────────── AWS VPC (10.0.0.0/16) ────────────────────────────────┐
│                                                                                         │
│  ┌─── Public Subnets (10.0.1.x, 10.0.2.x, 10.0.3.x) ───────────────────────────────┐ │
│  │  Application Load Balancer (internet-facing)                                       │ │
│  │  NAT Gateways (one per AZ)                                                        │ │
│  └───────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                         │
│  ┌─── Private Subnets (10.0.4.x, 10.0.5.x, 10.0.6.x) ──────────────────────────────┐ │
│  │  ECS Fargate Cluster                                                               │ │
│  │   ├── config-server     (512 CPU, 1024 MB)                                        │ │
│  │   ├── discovery-server  (512 CPU, 1024 MB)                                        │ │
│  │   ├── api-gateway        (512 CPU, 1024 MB)  ◄── ALB Target Group                │ │
│  │   ├── customers-service  (512 CPU, 1024 MB)                                       │ │
│  │   ├── visits-service     (512 CPU, 1024 MB)                                       │ │
│  │   ├── vets-service       (512 CPU, 1024 MB)                                       │ │
│  │   ├── genai-service      (512 CPU, 1024 MB)                                       │ │
│  │   ├── admin-server       (512 CPU, 1024 MB)                                       │ │
│  │   ├── tracing-server     (512 CPU, 1024 MB)                                       │ │
│  │   ├── prometheus-server  (512 CPU, 1024 MB)                                       │ │
│  │   └── grafana-server     (256 CPU, 512 MB)                                        │ │
│  │                                                                                    │ │
│  │  ┌── Database Subnet ─────────────────────────────────────────────────────────┐   │ │
│  │  │  RDS MySQL 8.x (db.t3.medium, Multi-AZ)                                    │   │ │
│  │  │  Databases: petclinic_customers, petclinic_vets, petclinic_visits           │   │ │
│  │  └────────────────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                         │
│  Supporting Services:                                                                   │
│   AWS ECR        — Docker image registry (one repo per service)                        │
│   Secrets Manager — OPENAI_API_KEY, DB_PASSWORD, DB_USERNAME                          │
│   S3              — Terraform state, Config Server backup                              │
│   CloudWatch      — Logs, metrics, alarms for all ECS tasks                           │
│   Route 53        — petclinic.yourdomain.com → ALB                                    │
│   ACM             — SSL certificate for *.yourdomain.com                               │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### Service Discovery in AWS
Eureka (discovery-server) runs inside ECS. All services register using their **ECS task private IP**. The Config Server environment variable `CONFIG_SERVER_URL` points to the internal discovery-server ECS service DNS name.

---

## 4. Jira Epics & Stories

> **Story Point Scale:** 1 (trivial) · 2 (small) · 3 (medium) · 5 (large) · 8 (very large) · 13 (epic-level)

---

### EPIC 1 — Project Setup & Repository Management
**Owner:** M1 | **Sprint:** 1 | **Total Points:** 18

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-1 | Create GitHub repository and project structure | M1 | 2 | Fork the base repo. Create feature branches per service. Set up `.github/` folder. | Repo exists, all 10 members have access, base code compiles with `./mvnw clean install` |
| PC-2 | Configure branch protection and PR templates | M1 | 2 | Protect `main` and `develop` branches. Require 1 reviewer. Add PR template. | Direct push to main blocked. PR template auto-fills on new PRs |
| PC-3 | AWS account setup and IAM roles | M1 + M2 | 5 | Create AWS account (or use existing). Create IAM users for each team member with least-privilege. Create service roles for ECS, ECR, RDS. | All members can login. ECS task execution role exists. No root credentials in use |
| PC-4 | Local development environment guide | M1 | 3 | Document how to set up local dev: JDK 17, Maven, Docker Desktop, AWS CLI, Terraform. | Every team member can run `./mvnw spring-boot:run` on their assigned service locally |
| PC-5 | Jira project creation and backlog population | M1 | 2 | Create Jira project `PC`. Import all stories from this document. Link GitHub integration. Set up 5 sprints. | All stories in Jira. GitHub commits auto-link to Jira tickets when message contains `PC-XX` |
| PC-6 | Architecture review and sign-off | M1 | 4 | M1 reviews the AWS architecture with team. Confirm ECS vs EKS, MySQL vs Aurora, etc. | Written ADR (Architecture Decision Record) committed to `/docs/adr/` folder |

---

### EPIC 2 — Infrastructure Services (Config Server + Discovery Server)
**Owner:** M4 | **Sprint:** 1-2 | **Total Points:** 21

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-7 | Config Server — local setup and smoke test | M4 | 3 | Run `spring-petclinic-config-server` locally on port 8888. Verify it serves config from GitHub | `curl http://localhost:8888/application/default` returns valid YAML |
| PC-8 | Config Server — point to team's own config repo | M4 | 3 | Fork `spring-petclinic-microservices-config` repo. Update config-server to pull from team fork. | Config changes in forked repo reflect in running services within 30s (refresh) |
| PC-9 | Config Server — add MySQL profile configs | M4 | 3 | Add `*-mysql.yml` config files for customers, vets, visits services in the config repo | Services can start with `--spring.profiles.active=docker,mysql` using RDS credentials |
| PC-10 | Discovery Server — local setup and smoke test | M4 | 3 | Run `spring-petclinic-discovery-server` locally on port 8761 | Eureka dashboard at http://localhost:8761 is accessible |
| PC-11 | Discovery Server — verify service registration | M4 | 3 | Start config-server + discovery-server + customers-service. Verify customers-service appears in Eureka. | Eureka shows `CUSTOMERS-SERVICE` registered with instance IP and port |
| PC-12 | Config + Discovery — integration test | M4 | 3 | Write a simple integration test: service starts, fetches config, registers with Eureka, health is UP | Test passes in CI with `./mvnw test -pl spring-petclinic-config-server,spring-petclinic-discovery-server` |
| PC-13 | Config Server — AWS readiness (env var config) | M4 | 3 | Ensure config server accepts `GIT_REPO` and `GIT_USERNAME`/`GIT_PASSWORD` as env vars for private repos | Config server starts in Docker with env vars passed via `docker run -e GIT_REPO=...` |

---

### EPIC 3 — Customers Service
**Owner:** M5 | **Sprint:** 1-2 | **Total Points:** 21

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-14 | Customers Service — local setup with HSQLDB | M5 | 3 | Run customers-service locally. Test all REST endpoints via Postman/curl | All CRUD endpoints work: GET /owners, POST /owners, GET /owners/{id}/pets, POST /owners/{id}/pets |
| PC-15 | Customers Service — Postman collection | M5 | 2 | Create Postman collection for all customers-service endpoints. Export and commit to `/docs/postman/` | Collection covers: list owners, get owner, create owner, list pets, add pet, update pet |
| PC-16 | Customers Service — unit tests | M5 | 5 | Write JUnit 5 tests for `OwnerResource` and `PetResource` REST controllers. Use MockMvc. | Test coverage ≥ 80% on web layer. All tests pass in CI |
| PC-17 | Customers Service — MySQL migration | M5 | 5 | Add MySQL datasource config. Test against local MySQL Docker container. Validate schema creation. | Service starts with `--spring.profiles.active=mysql` and all Postman tests pass against MySQL |
| PC-18 | Customers Service — actuator endpoints | M5 | 3 | Verify `/actuator/health`, `/actuator/metrics`, `/actuator/prometheus` all return valid data | All three actuator endpoints return HTTP 200 with expected JSON/text content |
| PC-19 | Customers Service — Micrometer custom metrics | M5 | 3 | Validate `@Timed("petclinic.owner")` and `@Timed("petclinic.pet")` metrics appear in Prometheus endpoint | `curl .../actuator/prometheus` shows `petclinic_owner_` and `petclinic_pet_` metric families |

---

### EPIC 4 — Vets Service
**Owner:** M6 | **Sprint:** 1-2 | **Total Points:** 18

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-20 | Vets Service — local setup with HSQLDB | M6 | 3 | Run vets-service locally. Test `/vets` endpoint. | `GET http://localhost:8083/vets` returns list of vets with specialties |
| PC-21 | Vets Service — validate Spring Cache | M6 | 3 | Call `/vets` twice. Verify second call is served from cache (no DB hit). Test cache eviction. | Logs show cache HIT on second request. Cache `vets` name appears in actuator/caches |
| PC-22 | Vets Service — Postman collection | M6 | 2 | Create Postman collection. Commit to `/docs/postman/` | Collection covers list vets, get vet by ID, get specialties |
| PC-23 | Vets Service — unit tests | M6 | 5 | Write JUnit 5 tests for `VetResource`. Test cache behavior with `@SpringBootTest`. | ≥ 80% coverage on web layer. All tests pass in CI |
| PC-24 | Vets Service — MySQL migration | M6 | 5 | Same pattern as customers-service. MySQL profile + validation. | Service starts with mysql profile. All vets data loads from `data.sql` seed correctly |

---

### EPIC 5 — Visits Service
**Owner:** M7 | **Sprint:** 1-2 | **Total Points:** 16

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-25 | Visits Service — local setup with HSQLDB | M7 | 3 | Run visits-service locally. Test endpoints. | `GET /pets/{petId}/visits` and `POST /visits` work correctly |
| PC-26 | Visits Service — Postman collection | M7 | 2 | Create collection for visits endpoints. | Collection covers: get visits by pet, create visit, get visit by ID |
| PC-27 | Visits Service — unit tests | M7 | 5 | Write JUnit 5 tests for `VisitResource`. Use MockMvc + H2. | ≥ 80% coverage on web layer. CI passes. |
| PC-28 | Visits Service — MySQL migration | M7 | 5 | MySQL profile + local Docker MySQL validation. | `POST /visits` persists to MySQL and `GET /pets/{petId}/visits` retrieves correctly |
| PC-29 | Visits Service — custom Micrometer metrics | M7 | 1 | Verify `@Timed("petclinic.visit")` metrics appear in prometheus endpoint | Metric `petclinic_visit_` appears in actuator/prometheus output |

---

### EPIC 6 — API Gateway & Frontend
**Owner:** M8 | **Sprint:** 2 | **Total Points:** 20

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-30 | API Gateway — local setup and routing test | M8 | 3 | Start all services. Access http://localhost:8080. Verify all routes work. | Browser shows PetClinic UI. Owner list, vet list, add visit all work end-to-end |
| PC-31 | API Gateway — validate all route definitions | M8 | 3 | Review `application.yml` routes. Test each route via Postman. Ensure path rewriting works. | Each route (`/api/customer/**`, `/api/visit/**`, `/api/vet/**`, `/api/genai/**`) correctly proxies requests |
| PC-32 | API Gateway — circuit breaker test | M8 | 5 | Stop customers-service. Hit `/api/customer/**` via gateway. Verify fallback fires. | `/fallback` endpoint returns graceful error message. UI shows friendly error, not stack trace |
| PC-33 | API Gateway — retry policy validation | M8 | 3 | Use Chaos Monkey or manual stop/start to simulate service flakiness. Verify retries happen. | Logs show retry attempts before fallback. No errors surfaced to frontend on transient failures |
| PC-34 | Frontend — AngularJS smoke test | M8 | 3 | Open UI in Chrome + Firefox. Test: view owners, add owner, add pet, add visit, view vets. | All golden paths work in both browsers without console errors |
| PC-35 | API Gateway — ALB header forwarding config | M8 | 3 | Ensure `X-Forwarded-For`, `X-Forwarded-Proto` headers pass correctly through ALB → Gateway. | Actuator health endpoint reflects correct base URL behind ALB. No redirect loops. |

---

### EPIC 7 — GenAI Service
**Owner:** M9 | **Sprint:** 2 | **Total Points:** 21

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-36 | GenAI Service — local setup with OpenAI key | M9 | 3 | Start genai-service with a valid `OPENAI_API_KEY`. Test chatbot via API Gateway UI. | Chat messages like "List all owners" return valid responses from AI |
| PC-37 | GenAI Service — test all AI tools | M9 | 5 | Verify each AI tool: list owners, list vets, get pet info, create owner, add pet. | Each tool invocation by the AI returns correct data from downstream services |
| PC-38 | GenAI Service — fallback when no API key | M9 | 3 | Start without OPENAI_API_KEY. Verify gateway circuit breaker handles GenAI failures gracefully. | Rest of app works normally. GenAI endpoint returns 503 with helpful message, not 500 |
| PC-39 | GenAI Service — Secrets Manager integration | M9 | 5 | Replace hardcoded env var `OPENAI_API_KEY` with AWS Secrets Manager lookup at startup. | Service retrieves key from Secrets Manager on ECS startup. No plaintext keys in task definitions. |
| PC-40 | GenAI Service — unit tests | M9 | 5 | Write tests for `PetclinicChatClient` and `PetclinicTools` using Spring AI mock. | Tests pass without a real OpenAI key. CI does not require `OPENAI_API_KEY`. |

---

### EPIC 8 — Containerization
**Owner:** M3 | **Sprint:** 3 | **Total Points:** 26

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-41 | Validate existing Dockerfiles for all services | M3 | 3 | Review `docker/Dockerfile`. Verify multi-stage build works for each service. | `docker build` succeeds for all 8 services. Image sizes < 400 MB each |
| PC-42 | Build all images locally and smoke test | M3 | 5 | Build all images. Run via docker compose. Verify app works end-to-end. | `docker compose up` results in working app at http://localhost:8080 |
| PC-43 | Create AWS ECR repositories | M3 + M2 | 3 | Create one ECR repo per service in us-east-1. Set lifecycle policy (keep last 10 images). | 8 ECR repos exist: `petclinic/config-server`, `petclinic/discovery-server`, etc. |
| PC-44 | Push all images to ECR | M3 | 3 | Tag images with ECR URI and push. Document push process. | All 8 images visible in ECR with tag `latest` and `v1.0.0` |
| PC-45 | Update docker-compose for team use | M3 | 3 | Add `docker` profile configs. Parameterize image tags. Add `.env.example` file. | Team can run `docker compose up` after copying `.env.example` to `.env` and filling values |
| PC-46 | Multi-platform image builds (amd64 + arm64) | M3 | 5 | Update Maven build to produce multi-platform images via Docker buildx. | `docker manifest inspect` shows both `linux/amd64` and `linux/arm64` platforms |
| PC-47 | Docker image security scan | M3 | 2 | Run Trivy or AWS ECR scan on all images. Fix any CRITICAL vulnerabilities. | No CRITICAL CVEs in any image. Scan results committed to `/docs/security/` |
| PC-48 | Document image build and push runbook | M3 | 2 | Write `/docs/runbooks/docker-build.md` with step-by-step build, tag, push commands. | Any team member can build and push any service image following the runbook without help |

---

### EPIC 9 — CI/CD Pipelines (GitHub Actions)
**Owner:** M3 | **Sprint:** 3 | **Total Points:** 29

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-49 | CI pipeline — build and unit test all services | M3 | 5 | Create `.github/workflows/ci.yml`. Trigger on PR. Run `mvn test` for changed service. | PRs cannot be merged if CI fails. Test results posted as PR comment |
| PC-50 | CI pipeline — Docker build validation | M3 | 3 | Add Docker build step to CI. Build image but do not push (validate only). | Every PR triggers a Docker build. Broken Dockerfiles block merge |
| PC-51 | CD pipeline — build and push to ECR on merge to develop | M3 | 5 | On merge to `develop`, build all changed services, tag with commit SHA, push to ECR. | ECR repos contain new images within 10 minutes of merge to develop |
| PC-52 | CD pipeline — deploy to staging ECS | M3 | 5 | On push to `develop`, trigger ECS service update for changed services. Wait for stability. | New task versions run in staging ECS within 15 minutes of merge. Old tasks drain gracefully |
| PC-53 | CD pipeline — production deploy (manual approval) | M3 | 5 | On push to `main`, create GitHub Actions workflow with `environment: production` requiring manual approval. | Production deploy only runs after M1 clicks "Approve" in GitHub Actions |
| PC-54 | CD pipeline — rollback workflow | M3 | 3 | Create manual workflow to roll back any service to a previous image tag. | Team can trigger rollback from GitHub Actions UI. ECS updates to previous task version within 5 minutes |
| PC-55 | Secrets management in GitHub Actions | M3 | 3 | Store AWS credentials, ECR URLs, and OPENAI_API_KEY as GitHub Actions secrets. | No secrets in `.yml` files. Workflows read from `${{ secrets.* }}`. |

---

### EPIC 10 — AWS Infrastructure (Terraform)
**Owner:** M2 | **Sprint:** 3-4 | **Total Points:** 55

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-56 | Terraform — S3 backend + DynamoDB state lock | M2 | 3 | Create S3 bucket `petclinic-tf-state-<account-id>` and DynamoDB table for lock. | `terraform init` uses S3 backend. No local `.tfstate` files committed |
| PC-57 | Terraform — VPC with public/private subnets | M2 | 5 | Create VPC (10.0.0.0/16) with 3 public + 3 private subnets across 3 AZs. NAT Gateways per AZ. | `terraform apply` creates VPC. Public subnets have internet gateway. Private subnets route via NAT |
| PC-58 | Terraform — Security Groups | M2 | 3 | Create SGs: ALB (443/80 from internet), ECS tasks (8080-8888 from ALB SG), RDS (3306 from ECS SG) | SGs created with least-privilege rules. No 0.0.0.0/0 on database SG |
| PC-59 | Terraform — IAM roles for ECS | M2 | 3 | Create: ECS task execution role (ECR pull, CloudWatch logs), ECS task role (Secrets Manager read, S3 read) | ECS tasks can pull images from ECR and write logs to CloudWatch without AWS credentials in code |
| PC-60 | Terraform — ECR repositories | M2 | 2 | Create 8 ECR repos via Terraform. Match repos already created in PC-43. | Terraform manages ECR repos. `terraform plan` shows no drift after PC-43 manual creation |
| PC-61 | Terraform — RDS MySQL (Multi-AZ) | M2 | 8 | Create RDS MySQL 8.x, db.t3.medium, Multi-AZ, in private DB subnet group. Store password in Secrets Manager. | RDS accessible from ECS private subnets only. `mysql -h <endpoint>` works from a bastion/ECS task |
| PC-62 | Terraform — ECS Fargate Cluster | M2 | 3 | Create ECS cluster `petclinic-prod`. Enable Container Insights. | Cluster exists. CloudWatch Container Insights dashboard auto-populates |
| PC-63 | Terraform — CloudWatch Log Groups | M2 | 2 | Create log groups: `/ecs/petclinic/<service-name>` for all 11 services. 30-day retention. | Log groups exist before ECS task deployment. No "log group doesn't exist" errors |
| PC-64 | Terraform — Secrets Manager entries | M2 | 3 | Create secrets: `petclinic/db/credentials` (username+password), `petclinic/openai/api-key` | Secrets created. ECS task role can read them. No plaintext values in Terraform code (use variables) |
| PC-65 | Terraform — ECS Task Definitions | M2 | 8 | Write task definitions for all 11 services. Include: image URI, CPU/memory, env vars, secrets from Secrets Manager, log config. | `aws ecs describe-task-definition --task-definition petclinic-<service>` returns valid definition |
| PC-66 | Terraform — Application Load Balancer | M2 | 5 | Create internet-facing ALB in public subnets. Target group pointing to api-gateway on port 8080. Health check: `/actuator/health`. | ALB accessible from internet. Routes to api-gateway ECS service. Health checks pass |
| PC-67 | Terraform — Route 53 + ACM Certificate | M2 | 5 | Create ACM cert for `petclinic.yourdomain.com`. Add Route 53 A record (alias to ALB). HTTPS listener on ALB. | `https://petclinic.yourdomain.com` loads PetClinic UI. Certificate valid. HTTP redirects to HTTPS |
| PC-68 | Terraform — ECS Services | M2 | 8 | Create ECS services for all 11 containers. Correct `depends_on` order: config → discovery → rest. Desired count: 1 (staging), 2 (prod). | All ECS services reach RUNNING state. Zero task restarts in first 5 minutes |
| PC-69 | Terraform — Auto Scaling for business services | M2 | 3 | Add Application Auto Scaling for customers, vets, visits, api-gateway. Target: 70% CPU. Min: 1, Max: 4. | Scaling policies attached. Load test triggers scale-out within 3 minutes |

---

### EPIC 11 — Monitoring & Observability
**Owner:** M10 | **Sprint:** 4 | **Total Points:** 34

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-70 | Zipkin — deploy to ECS and validate traces | M10 | 3 | Zipkin runs on ECS. All services configured with `MANAGEMENT_ZIPKIN_TRACING_ENDPOINT` env var. | Zipkin UI at internal URL shows traces for cross-service calls (e.g., api-gateway → customers-service) |
| PC-71 | Prometheus — deploy to ECS and scrape all services | M10 | 5 | Deploy Prometheus to ECS. Configure scrape targets for all services' `/actuator/prometheus` endpoints. | Prometheus UI shows all 8 services as UP targets. Metrics data visible. |
| PC-72 | Grafana — deploy to ECS and connect to Prometheus | M10 | 3 | Deploy Grafana to ECS. Add Prometheus as data source. | Grafana UI accessible. Can run a PromQL query against PetClinic metrics |
| PC-73 | Grafana — import custom PetClinic dashboards | M10 | 5 | Import existing dashboards from `docker/grafana/`. Create panels for: request rate, error rate, JVM heap, cache hit ratio, active visits. | Dashboard shows all panels with live data during a manual test session |
| PC-74 | Spring Boot Admin — deploy and verify all services | M10 | 3 | Admin server on ECS. All services show as UP with metrics, env, loggers accessible. | All 8 application services visible in Admin UI. Can change log level at runtime. |
| PC-75 | CloudWatch — custom metric alarms | M10 | 5 | Create alarms: ECS task CPU > 80%, ECS task memory > 85%, ALB 5xx > 1%, RDS CPU > 70%. | Alarms visible in CloudWatch. SNS notification triggers on alarm state (test with manual spike) |
| PC-76 | CloudWatch — log insights queries | M10 | 3 | Write and save CloudWatch Insights queries: errors by service, slow requests > 2s, exception frequency. | Queries saved and runnable. Results appear within 2 minutes of log generation |
| PC-77 | CloudWatch — application dashboard | M10 | 5 | Build CloudWatch dashboard: `PetClinic-Production`. Widgets: ALB request count, 5xx rate, ECS CPU per service, RDS connections, Zipkin trace count. | Dashboard readable at a glance. Auto-refreshes every 1 minute. |
| PC-78 | Distributed tracing — end-to-end validation | M10 | 2 | Manually perform: add owner → add pet → add visit. Capture trace ID. Find full trace in Zipkin. | Single trace ID visible across api-gateway → customers-service → visits-service spans |

---

### EPIC 12 — Testing & QA
**Owner:** M10 | **Sprint:** 4-5 | **Total Points:** 33

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-79 | Integration test — full service stack test | M10 | 8 | Write integration test suite that: starts all services via TestContainers or against staging, runs all API flows. | All integration tests pass in CI against staging environment |
| PC-80 | End-to-end test — golden path scenarios | M10 | 5 | Test manually (and document): Add Owner → Add Pet → Add Visit → View Vet List → Use AI Chat | All 5 golden paths succeed without errors in staging and production |
| PC-81 | Load test — JMeter test plan | M10 | 5 | Adapt existing JMeter test plan (`scripts/`). Run 50 concurrent users for 5 minutes against staging. | p99 response time < 3s. Error rate < 0.1%. No OOM errors in services during test. |
| PC-82 | Security test — OWASP ZAP scan | M10 | 5 | Run OWASP ZAP baseline scan against staging URL. Fix any HIGH severity findings. | No HIGH or CRITICAL findings in ZAP report. Report committed to `/docs/security/` |
| PC-83 | Chaos test — Chaos Monkey validation | M10 | 5 | Enable Chaos Monkey on customers-service in staging. Verify circuit breaker in API Gateway handles failures. | With Chaos Monkey active, UI shows fallback (not 500). Zipkin traces show circuit breaker trips. |
| PC-84 | Regression test — cross-service contract test | M10 | 5 | Write Spring Cloud Contract tests for api-gateway ↔ customers/vets/visits service contracts. | Contract tests run in CI. Breaking API change fails CI before merge. |

---

### EPIC 13 — Production Go-Live
**Owner:** M1 | **Sprint:** 5 | **Total Points:** 21

| Ticket | Title | Owner | Points | Description | Acceptance Criteria |
|---|---|---|---|---|---|
| PC-85 | Production deployment — config server first | M1 + M2 | 3 | Deploy config-server to production ECS. Verify it serves config to other services. | `curl https://internal-config-server/application/docker` returns valid YAML |
| PC-86 | Production deployment — discovery server | M1 + M2 | 2 | Deploy discovery-server to production ECS after config-server. | Eureka dashboard shows config-server registered. Port 8761 accessible internally |
| PC-87 | Production deployment — all business services | M1 + All | 5 | Deploy customers, vets, visits, genai, admin-server, tracing-server, prometheus, grafana. | All services register in Eureka. All /actuator/health return UP. |
| PC-88 | Production deployment — api-gateway + ALB wiring | M1 + M8 | 3 | Deploy api-gateway. Wire ALB target group. Test HTTPS end-to-end. | `https://petclinic.yourdomain.com` loads PetClinic UI fully |
| PC-89 | Production smoke test | M10 | 3 | Run golden path test suite against production URL. | All 5 golden paths pass in production |
| PC-90 | Production go-live checklist sign-off | M1 | 3 | Complete checklist: all health checks green, no CRITICAL alarms active, rollback plan documented, all logs flowing to CloudWatch. | Signed checklist committed to `/docs/go-live-checklist.md`. All 10 members confirm ready |
| PC-91 | Post-launch — 24-hour monitoring watch | M10 | 2 | Monitor CloudWatch dashboards for 24 hours post-launch. Document any anomalies. | Daily report posted to team Slack/channel. Zero P1 incidents in first 24 hours |

---

## 5. Sprint Plan

### Sprint 1 — Foundation & Local Dev (Weeks 1-2)

| Member | Stories | Goal |
|---|---|---|
| M1 | PC-1, PC-2, PC-3, PC-4, PC-5, PC-6 | Repo, AWS account, Jira board live |
| M2 | PC-3 (collab), PC-56, PC-57 | Terraform backend + VPC started |
| M3 | PC-41, PC-42 | Docker images build and run locally |
| M4 | PC-7, PC-8, PC-10, PC-11 | Config + Discovery servers running locally |
| M5 | PC-14, PC-15 | Customers service running locally with Postman collection |
| M6 | PC-20, PC-21, PC-22 | Vets service running locally |
| M7 | PC-25, PC-26 | Visits service running locally |
| M8 | PC-30, PC-31 | API Gateway routing all services |
| M9 | PC-36, PC-37 | GenAI service running locally |
| M10 | PC-79 (plan) | Integration test plan drafted |

**Sprint 1 Exit Criteria:** Every team member's service runs locally and connects to config/discovery. Full stack accessible at http://localhost:8080.

---

### Sprint 2 — Business Logic, Tests & MySQL (Weeks 3-4)

| Member | Stories | Goal |
|---|---|---|
| M1 | Code reviews, architecture support | All PRs reviewed |
| M2 | PC-58, PC-59, PC-61 | Security groups + RDS provisioned |
| M3 | PC-43, PC-44, PC-49, PC-50 | ECR repos + CI pipeline live |
| M4 | PC-9, PC-12, PC-13 | Config MySQL profiles + integration tests |
| M5 | PC-16, PC-17, PC-18, PC-19 | Customers unit tests + MySQL working |
| M6 | PC-23, PC-24 | Vets unit tests + MySQL working |
| M7 | PC-27, PC-28, PC-29 | Visits unit tests + MySQL working |
| M8 | PC-32, PC-33, PC-34, PC-35 | Circuit breaker + frontend tests |
| M9 | PC-38, PC-39, PC-40 | GenAI fallback + Secrets Manager integration |
| M10 | PC-79, PC-80 | Integration tests running against local stack |

**Sprint 2 Exit Criteria:** All services have unit tests passing in CI. MySQL migration validated. CI blocks broken builds.

---

### Sprint 3 — Containerization, CI/CD & AWS Infra Start (Weeks 5-6)

| Member | Stories | Goal |
|---|---|---|
| M1 | Architecture reviews, AWS account governance | IAM hardened |
| M2 | PC-60, PC-62, PC-63, PC-64, PC-65 | ECS cluster + task definitions in Terraform |
| M3 | PC-45, PC-46, PC-47, PC-48, PC-51, PC-52 | CD pipeline deploying to ECR + staging ECS |
| M4 | Collab on ECS task defs for infra services | Config/discovery task defs reviewed |
| M5 | Collab on ECS task def for customers | Customers task def reviewed |
| M6 | Collab on ECS task def for vets | Vets task def reviewed |
| M7 | Collab on ECS task def for visits | Visits task def reviewed |
| M8 | PC-35 (AWS ALB headers), collab on gateway task def | Gateway task def reviewed |
| M9 | Collab on genai task def + Secrets Manager | GenAI task def reviewed |
| M10 | PC-81, PC-82 | JMeter + OWASP ZAP against staging |

**Sprint 3 Exit Criteria:** All services have ECS task definitions. CD pipeline pushes to ECR on merge to develop. At least 3 services running in staging ECS.

---

### Sprint 4 — Full AWS Deployment & Monitoring (Weeks 7-8)

| Member | Stories | Goal |
|---|---|---|
| M1 | PC-85, PC-86 oversight | Infra services in production |
| M2 | PC-66, PC-67, PC-68, PC-69 | ALB + Route53 + ECS services + auto-scaling |
| M3 | PC-53, PC-54, PC-55 | Prod deploy pipeline + rollback workflow |
| M4 | Support M2 on config/discovery ECS troubleshooting | Config + discovery stable in ECS |
| M5 | Support staging validation for customers-service | Customers working in staging |
| M6 | Support staging validation for vets-service | Vets working in staging |
| M7 | Support staging validation for visits-service | Visits working in staging |
| M8 | PC-35, support API gateway in staging | Gateway + ALB working |
| M9 | Support genai-service in staging | GenAI working in staging with Secrets Manager |
| M10 | PC-70 to PC-78 | All monitoring/observability deployed and dashboards live |

**Sprint 4 Exit Criteria:** All services running in staging ECS. Grafana dashboards live. CloudWatch alarms active. Full app accessible at staging URL over HTTPS.

---

### Sprint 5 — QA, Hardening & Production Go-Live (Weeks 9-10)

| Member | Stories | Goal |
|---|---|---|
| M1 | PC-89, PC-90, PC-91 | Go-live sign-off |
| M2 | Final infra hardening, PC-69 auto-scaling validation | Production infra stable |
| M3 | PC-54 rollback test, pipeline docs | Rollback verified to work |
| M4 | Support production deployment of config/discovery | Infra services in production |
| M5 | Production customers-service deployment | In production |
| M6 | Production vets-service deployment | In production |
| M7 | Production visits-service deployment | In production |
| M8 | Production api-gateway deployment | In production |
| M9 | Production genai-service deployment | In production |
| M10 | PC-83, PC-84, PC-85, PC-86, PC-87, PC-88 | Load test, chaos test, post-launch watch |

**Sprint 5 Exit Criteria:** All services in production. HTTPS working. All smoke tests pass. 24-hour monitoring watch complete.

---

## 6. Definition of Done

A story is **Done** when ALL of the following are true:

### For every service story:
- [ ] Code compiles with `./mvnw clean install` with no warnings
- [ ] Unit tests written and passing (≥ 80% coverage on web layer)
- [ ] Service starts locally and all API endpoints respond correctly
- [ ] `/actuator/health` returns `{"status":"UP"}`
- [ ] `/actuator/prometheus` returns metrics data
- [ ] Service registers with Eureka when discovery-server is running
- [ ] Feature branch merged to `develop` via approved PR (min. 1 reviewer)
- [ ] CI pipeline passes (build + test + docker build)
- [ ] No TODO or debug logs left in code

### For AWS/infrastructure stories:
- [ ] Terraform code reviewed and applied to staging before production
- [ ] `terraform plan` shows no unexpected changes after apply
- [ ] Resources tagged with: `Project=petclinic`, `Environment=production`, `Owner=<member-name>`
- [ ] No secrets or credentials committed to git
- [ ] CloudWatch log group receiving logs
- [ ] Security group rules documented

### For CI/CD stories:
- [ ] Pipeline documented in `/docs/runbooks/`
- [ ] Secrets stored in GitHub Actions secrets, not in YAML
- [ ] Pipeline runs in < 15 minutes end-to-end
- [ ] Failing pipeline blocks merge

### For go-live:
- [ ] All 11 services (8 app + 3 monitoring) healthy in production ECS
- [ ] HTTPS working with valid ACM certificate
- [ ] All golden path smoke tests passing
- [ ] Grafana dashboard showing live data
- [ ] CloudWatch alarms configured and tested
- [ ] Rollback procedure documented and tested
- [ ] Go-live checklist signed off by M1 + M10

---

## 7. Technical Prerequisites

### Every Team Member Needs:
```
Software to install:
├── JDK 17+              → https://adoptium.net/
├── Maven 3.9+           → bundled via ./mvnw wrapper
├── Docker Desktop       → https://docker.com/products/docker-desktop
├── AWS CLI v2           → https://aws.amazon.com/cli/
├── Terraform 1.9+       → https://terraform.io/downloads
├── Git 2.x              → https://git-scm.com/
├── IntelliJ IDEA or VS Code
└── Postman (API testing)

AWS Setup (M1 provides credentials):
├── AWS CLI configured   → aws configure (Access Key + us-east-1)
├── ECR login command    → aws ecr get-login-password | docker login ...
└── kubectl (optional, if team chooses EKS)
```

### Verify Your Setup:
```bash
java -version          # Should show: openjdk 17.x
docker --version       # Should show: Docker 24+
aws --version          # Should show: aws-cli/2.x
terraform --version    # Should show: Terraform v1.9+

# Clone and build the project
git clone <team-repo-url>
cd spring-petclinic-microservices
./mvnw clean install -DskipTests
# Expected: BUILD SUCCESS for all 8 modules
```

### Environment Variables to Configure Locally:
```bash
# Create a .env file in project root (DO NOT COMMIT)
export OPENAI_API_KEY=sk-...          # Required for GenAI service only
export AWS_ACCOUNT_ID=123456789012    # Your AWS account ID
export AWS_REGION=us-east-1
export ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

# For local MySQL testing
export DB_HOST=localhost
export DB_PORT=3306
export DB_USERNAME=petclinic
export DB_PASSWORD=petclinic
```

---

## 8. Service Ports & Access URLs

### Local Development

| Service | Local URL | Notes |
|---|---|---|
| API Gateway (UI) | http://localhost:8080 | Start this last |
| Config Server | http://localhost:8888 | Start this first |
| Discovery Server (Eureka) | http://localhost:8761 | Start this second |
| Customers Service | http://localhost:8081 | |
| Visits Service | http://localhost:8082 | |
| Vets Service | http://localhost:8083 | |
| GenAI Service | http://localhost:8084 | Requires OPENAI_API_KEY |
| Admin Server | http://localhost:9090 | |
| Zipkin | http://localhost:9411/zipkin/ | |
| Prometheus | http://localhost:9091 | |
| Grafana | http://localhost:3030 | admin/admin |

### Staging (AWS ECS)

| Service | Internal URL | External |
|---|---|---|
| API Gateway | Internal ECS DNS | https://staging.petclinic.yourdomain.com |
| All others | Internal only (private subnets) | Via VPN or bastion only |

### Production (AWS ECS)

| Service | Internal URL | External |
|---|---|---|
| API Gateway | Internal ECS DNS | https://petclinic.yourdomain.com |
| Grafana | Internal only | https://monitoring.petclinic.yourdomain.com |
| Admin Server | Internal only | https://admin.petclinic.yourdomain.com |

---

## 9. Branch & PR Strategy

### Branch Naming Convention
```
main              ← production-only, protected
develop           ← integration branch, CI deploys to staging
feature/PC-XX-short-description    ← your work branch
hotfix/PC-XX-short-description     ← urgent production fixes
```

### Workflow
```
1. Pull latest develop
   git checkout develop && git pull

2. Create feature branch
   git checkout -b feature/PC-14-customers-service-local-setup

3. Commit with Jira ticket reference
   git commit -m "PC-14: Add customers service local setup and HSQLDB config"

4. Push and create PR → develop
   - Fill PR template
   - Assign reviewer
   - Link Jira ticket

5. CI must pass. At least 1 review approval required.

6. Merge to develop → auto-deploys to staging

7. Sprint end: develop → main via release PR
   - M1 approves
   - Manual trigger in GitHub Actions deploys to production
```

### Commit Message Format
```
PC-XX: Short imperative description (max 72 chars)

Optional body explaining WHY, not what.
Reference: PC-XX
```

---

## Quick Reference: Service Startup Order

When running locally **without** Docker Compose, always start in this order:

```
1. config-server        (port 8888) — must be HEALTHY first
2. discovery-server     (port 8761) — must be HEALTHY second
3. customers-service    (port 8081) ┐
4. visits-service       (port 8082) ├── any order, in parallel
5. vets-service         (port 8083) │
6. genai-service        (port 8084) ┘ (optional — needs OPENAI_API_KEY)
7. admin-server         (port 9090)
8. api-gateway          (port 8080) — start LAST
```

**Tip:** Wait until you see `Registered instance ... with Eureka` in each service's logs before starting the next tier.

---

*This document is the single source of truth for team assignments. All Jira tickets map 1:1 to stories in this document. Update this file via PR if requirements change.*
