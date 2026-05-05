# Spring PetClinic — DevOps Team Assignment
## Docker → AWS EKS → Production in 1 Week

> **Project Key:** `OPS`  
> **Team:** 10 DevOps Engineers  
> **Timeline:** 5 Working Days (Monday – Friday)  
> **Stack:** Docker · AWS EKS · Terraform · GitHub Actions · Helm · Kubernetes  
> **Rule:** Source code is FROZEN. No development. Pure DevOps only.

---

## Table of Contents
1. [The Mission](#1-the-mission)
2. [10 Engineer Assignments](#2-10-engineer-assignments)
3. [AWS EKS Architecture](#3-aws-eks-architecture)
4. [Kubernetes Project Structure](#4-kubernetes-project-structure)
5. [Day-by-Day Schedule](#5-day-by-day-schedule)
6. [Jira Tickets (OPS Board)](#6-jira-tickets-ops-board)
7. [Prerequisites — Day 1 Morning Checklist](#7-prerequisites--day-1-morning-checklist)
8. [Critical Deployment Order](#8-critical-deployment-order)
9. [Daily Definition of Done](#9-daily-definition-of-done)
10. [Rollback Procedures](#10-rollback-procedures)

---

## 1. The Mission

Deploy **8 Spring Boot microservices** to **AWS EKS** with full observability, CI/CD, and production-grade reliability in **5 days**. The application is fully built — your job is end-to-end infrastructure.

### Services to Deploy

| Service | Image Port | K8s Service Name | Role |
|---|---|---|---|
| config-server | 8888 | `config-server` | Serves YAML config to all services — **MUST start first** |
| discovery-server | 8761 | `discovery-server` | Eureka registry — **MUST start second** |
| customers-service | 8081 | `customers-service` | Owners + Pets CRUD |
| visits-service | 8082 | `visits-service` | Visit records CRUD |
| vets-service | 8083 | `vets-service` | Veterinarians (cached) |
| genai-service | 8084 | `genai-service` | AI chatbot (optional — needs OpenAI key) |
| api-gateway | 8080 | `api-gateway` | **Single public entry point** + AngularJS UI |
| admin-server | 9090 | `admin-server` | Spring Boot Admin monitoring |

### Observability Stack (also on EKS)

| Tool | Port | Purpose |
|---|---|---|
| Zipkin | 9411 | Distributed tracing |
| Prometheus | 9090 | Metrics collection |
| Grafana | 3000 | Dashboards |

### How the App Works (Critical Knowledge)
```
All services read config from:   http://config-server:8888  (K8s DNS name)
All services register with:      http://discovery-server:8761 (Eureka)
All traffic enters through:      api-gateway:8080 → routed via Eureka to backends
Profile active in all containers: docker  (set in Dockerfile ENV SPRING_PROFILES_ACTIVE=docker)
```
> The `docker` Spring profile is already baked into every image. This means  
> every service auto-connects to `http://config-server:8888` by DNS — your  
> K8s Service names **must match exactly**: `config-server`, `discovery-server`.

### Known Project Quirks — Read Before You Deploy

> These are facts about the source code that will cause confusion if you don't know them upfront.

**1. Docker image names produced by Maven build:**
The command `./mvnw clean install -P buildDocker` produces images with this naming scheme:
```
springcommunity/spring-petclinic-config-server
springcommunity/spring-petclinic-discovery-server
springcommunity/spring-petclinic-customers-service
springcommunity/spring-petclinic-visits-service
springcommunity/spring-petclinic-vets-service
springcommunity/spring-petclinic-genai-service
springcommunity/spring-petclinic-api-gateway
springcommunity/spring-petclinic-admin-server
```
ECR repositories **must be named to match** this prefix. Use the existing scripts to retag and push:
```bash
export REPOSITORY_PREFIX=<aws-account-id>.dkr.ecr.us-east-1.amazonaws.com/springcommunity
export VERSION=1.0.0
./scripts/tagImages.sh    # retags all 8 images for ECR
./scripts/pushImages.sh   # pushes all 8 images to ECR
```

**2. pom.xml EXPOSE port bugs (4 services):**
Four services have incorrect `docker.image.exposed.port` in their `pom.xml`. This affects only the Docker `EXPOSE` metadata — **not the actual runtime port** (Spring Boot controls the real port via config server). Do NOT use `EXPOSE` values for K8s `containerPort`. Use the actual Spring Boot app ports below:

| Service | pom.xml EXPOSE (wrong) | Actual App Port (use this) |
|---|---|---|
| vets-service | 8081 | **8083** |
| visits-service | 9090 (typo in pom) | **8082** |
| genai-service | 8081 | **8084** |
| api-gateway | 8081 | **8080** |
| customers-service | 8081 | 8081 ✅ |
| config-server | 8888 | 8888 ✅ |
| discovery-server | 8761 | 8761 ✅ |
| admin-server | 9090 | 9090 ✅ |

**3. vets-service and genai-service have a permanent `production` profile:**
These two services have `spring.profiles.active: production` hardcoded in their `application.yml`. When running in K8s with the `docker` profile, both `production` + `docker` will be active simultaneously. This is expected behavior:
- `production` profile on vets-service: activates Spring Cache (the `vets` cache-name)
- `production` profile on genai-service: activates AI model configuration
Do NOT try to suppress this. Seeing `production` in startup logs is normal.

**4. genai-service is a reactive application:**
The genai-service uses Spring WebFlux (reactive), not Spring MVC. It starts on Netty, not Tomcat. You will see `Netty started on port 8084` in logs — this is correct. Liveness/readiness probes on `/actuator/health` work identically.

**5. MySQL credentials override strategy for K8s:**
The `docker` profile is already baked into the image. To add `mysql` profile AND inject RDS credentials, override at the K8s deployment level:
```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "docker,mysql"                  # overrides Dockerfile's ENV
  - name: SPRING_DATASOURCE_URL
    valueFrom:
      secretKeyRef:
        name: petclinic-db-secret
        key: url                           # e.g. jdbc:mysql://<RDS-endpoint>:3306/<db>
  - name: SPRING_DATASOURCE_USERNAME
    valueFrom:
      secretKeyRef:
        name: petclinic-db-secret
        key: username
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: petclinic-db-secret
        key: password
```
Spring Boot env vars always override config server values — so the RDS endpoint you inject here wins over whatever the config repo's `mysql` profile says.

---

## 2. Ten Engineer Assignments

| Engineer | Role | Owns |
|---|---|---|
| **DE-1** | EKS Cluster Lead | EKS architecture, kubectl setup, cluster add-ons, go-live coordination |
| **DE-2** | AWS Infrastructure (Terraform) | VPC, EKS cluster, node groups, IAM, cluster autoscaler |
| **DE-3** | AWS Services (Terraform) | ECR repos, RDS MySQL, Secrets Manager, ALB, Route53, ACM |
| **DE-4** | Docker & Image Pipeline | Build all 8 images, push to ECR, image scanning, docker-compose validation |
| **DE-5** | K8s — Infrastructure Services | config-server + discovery-server manifests, deployment, validation |
| **DE-6** | K8s — Business Services (Group A) | customers-service + vets-service manifests, deployment, validation |
| **DE-7** | K8s — Business Services (Group B) | visits-service + genai-service manifests, deployment, validation |
| **DE-8** | K8s — Gateway & Ingress | api-gateway + admin-server manifests, AWS ALB Ingress, TLS |
| **DE-9** | CI/CD Pipelines | GitHub Actions: build, test, push to ECR, rolling deploy to EKS |
| **DE-10** | Monitoring & Observability | Prometheus, Grafana, Zipkin, CloudWatch alarms, dashboards |

### Who Depends on Whom
```
DE-2 ──► provides EKS cluster ──► DE-1 (configure kubectl)
DE-3 ──► provides ECR URLs     ──► DE-4 (push images)
DE-3 ──► provides RDS endpoint ──► DE-5, DE-6, DE-7 (service configs)
DE-4 ──► provides image tags   ──► DE-5, DE-6, DE-7, DE-8 (manifests)
DE-1 ──► installs add-ons      ──► DE-5-10 (can deploy to cluster)
DE-5 ──► config+discovery UP   ──► DE-6, DE-7, DE-8 (services register)
DE-8 ──► gateway + ingress UP  ──► DE-10 (end-to-end smoke test)
```

---

## 3. AWS EKS Architecture

```
┌────────────────────── AWS us-east-1 ──────────────────────────────────────────┐
│                                                                                │
│  Route53 (petclinic.team.com) ──► ACM Cert ──► ALB (internet-facing)          │
│                                                      │ HTTPS:443              │
│  ┌──────────── VPC: 10.0.0.0/16 ─────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  Public Subnets (10.0.1.x/2.x/3.x — 3 AZs)                           │   │
│  │   └── ALB lives here                                                   │   │
│  │   └── NAT Gateways (1 per AZ)                                         │   │
│  │                                                                         │   │
│  │  Private Subnets (10.0.4.x/5.x/6.x — 3 AZs)                          │   │
│  │   └── EKS Managed Node Group                                           │   │
│  │        ├── Node type: t3.large (4 nodes, auto-scale 2–8)               │   │
│  │        └── Namespace: petclinic                                         │   │
│  │              ├── config-server    (1 pod, 256m/512Mi)                   │   │
│  │              ├── discovery-server (1 pod, 256m/512Mi)                   │   │
│  │              ├── customers-service(2 pods, 512m/1Gi) ─► HPA             │   │
│  │              ├── visits-service   (2 pods, 512m/1Gi) ─► HPA             │   │
│  │              ├── vets-service     (2 pods, 512m/1Gi) ─► HPA             │   │
│  │              ├── genai-service    (1 pod, 512m/1Gi)                     │   │
│  │              ├── api-gateway      (2 pods, 512m/1Gi) ─► HPA ─► Ingress │   │
│  │              └── admin-server     (1 pod, 256m/512Mi)                   │   │
│  │                                                                         │   │
│  │        └── Namespace: monitoring                                        │   │
│  │              ├── prometheus (kube-prometheus-stack)                     │   │
│  │              ├── grafana    (kube-prometheus-stack)                     │   │
│  │              └── zipkin     (custom deployment)                         │   │
│  │                                                                         │   │
│  │  Database Subnets (10.0.7.x/8.x/9.x)                                   │   │
│  │   └── RDS MySQL 8.x (db.t3.medium, Multi-AZ)                           │   │
│  │        ├── petclinic_customers                                          │   │
│  │        ├── petclinic_vets                                               │   │
│  │        └── petclinic_visits                                             │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                │
│  ECR  ── 8 repos (one per service)                                            │
│  Secrets Manager ── petclinic/db/credentials, petclinic/openai/key           │
│  CloudWatch Logs ── /eks/petclinic/<service-name>                            │
│  S3 ── petclinic-tf-state (Terraform remote backend)                         │
└────────────────────────────────────────────────────────────────────────────────┘
```

### EKS Cluster Add-ons Required

| Add-on | Installed By | Purpose |
|---|---|---|
| AWS Load Balancer Controller | DE-1 | Creates ALB from Ingress resources |
| EBS CSI Driver | DE-1 | Persistent volumes for Prometheus/Grafana |
| Cluster Autoscaler | DE-2 | Scale node group based on pod demand |
| Metrics Server | DE-1 | Required for HPA to work |
| External Secrets Operator | DE-1 | Sync AWS Secrets Manager → K8s Secrets |
| AWS CloudWatch Container Insights | DE-10 | Log forwarding from pods to CloudWatch |

---

## 4. Kubernetes Project Structure

Create this folder structure in the repo root. Each engineer creates their section.

```
k8s/
├── 00-namespace.yaml                        ← DE-1
├── 00-storage-class.yaml                    ← DE-1
│
├── cluster-addons/                          ← DE-1
│   ├── aws-load-balancer-controller/
│   │   ├── values.yaml
│   │   └── install.sh
│   ├── external-secrets-operator/
│   │   └── install.sh
│   ├── metrics-server/
│   │   └── install.sh
│   └── cloudwatch-container-insights/
│       └── install.sh
│
├── config-server/                           ← DE-5
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
│
├── discovery-server/                        ← DE-5
│   ├── deployment.yaml
│   └── service.yaml
│
├── customers-service/                       ← DE-6
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── external-secret.yaml
│
├── vets-service/                            ← DE-6
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── external-secret.yaml
│
├── visits-service/                          ← DE-7
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── external-secret.yaml
│
├── genai-service/                           ← DE-7
│   ├── deployment.yaml
│   ├── service.yaml
│   └── external-secret.yaml
│
├── api-gateway/                             ← DE-8
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
│
├── admin-server/                            ← DE-8
│   ├── deployment.yaml
│   └── service.yaml
│
└── monitoring/                              ← DE-10
    ├── namespace.yaml
    ├── prometheus-stack-values.yaml
    ├── zipkin-deployment.yaml
    ├── zipkin-service.yaml
    └── cloudwatch-alarm-policy.json

terraform/
├── backend.tf                               ← DE-2
├── main.tf                                  ← DE-2
├── variables.tf                             ← DE-2
├── terraform.tfvars.example                 ← DE-2
├── vpc.tf                                   ← DE-2
├── eks.tf                                   ← DE-2
├── iam.tf                                   ← DE-2
├── ecr.tf                                   ← DE-3
├── rds.tf                                   ← DE-3
├── alb.tf                                   ← DE-3
├── acm.tf                                   ← DE-3
├── route53.tf                               ← DE-3
├── secrets-manager.tf                       ← DE-3
└── outputs.tf                               ← DE-2 + DE-3

.github/
└── workflows/
    ├── ci.yml                               ← DE-9
    ├── cd-build-push.yml                    ← DE-9
    ├── cd-deploy-eks.yml                    ← DE-9
    └── rollback.yml                         ← DE-9

docs/
├── architecture-eks.png                     ← DE-1
├── runbooks/
│   ├── deploy.md                            ← DE-9
│   ├── rollback.md                          ← DE-9
│   └── monitoring.md                        ← DE-10
└── go-live-checklist.md                     ← DE-1
```

---

## 5. Day-by-Day Schedule

### DAY 1 — MONDAY: Foundation (Parallel Start)
> All 10 engineers work simultaneously from 9 AM

| Time | DE-1 | DE-2 | DE-3 | DE-4 | DE-5 | DE-6 | DE-7 | DE-8 | DE-9 | DE-10 |
|---|---|---|---|---|---|---|---|---|---|---|
| 9–10 AM | **ALL: Team kickoff** — Walk through architecture, assign AWS accounts, set up Slack channels, share repo access | | | | | | | | | |
| 10–12 PM | Create k8s/ folder structure. Write namespace.yaml + storage-class.yaml | **Start Terraform VPC + EKS** (takes 20 min to plan, 30 min to apply for VPC, 15 min for EKS) | Start Terraform ECR repos (8 repos) + Secrets Manager entries | Build all 8 images locally with `./mvnw clean install -P buildDocker`. Test with `docker compose up` | Read config-server + discovery-server source. Write deployment.yaml drafts | Read customers + vets source. Write deployment.yaml drafts | Read visits + genai source. Write deployment.yaml drafts | Read api-gateway source. Write deployment.yaml + ingress.yaml drafts | Create GitHub Actions folder. Write `ci.yml` skeleton | Create monitoring/namespace.yaml. Research kube-prometheus-stack values |
| 12–1 PM | **Lunch Break** | | | | | | | | | |
| 1–3 PM | Research + install AWS LB Controller Helm chart. Write `install.sh` scripts for all add-ons | Terraform apply EKS cluster. Configure `aws eks update-kubeconfig`. Verify `kubectl get nodes` | Terraform apply RDS MySQL. Store credentials in Secrets Manager. Get RDS endpoint. | Fix any image build issues. Log in to ECR. Push all 8 images. Verify in AWS console. | Complete config-server + discovery-server K8s manifests (deployment, service, configmap) | Complete customers + vets K8s manifests (deployment, service, hpa, external-secret) | Complete visits + genai K8s manifests (deployment, service, hpa, external-secret) | Complete api-gateway + admin-server K8s manifests. Write Ingress with ALB annotations | Write `cd-build-push.yml` — trigger on merge to main, build, tag with SHA, push to ECR | Write prometheus-stack-values.yaml. Write zipkin deployment + service YAML |
| 3–5 PM | Install all cluster add-ons on EKS. Verify: `kubectl get pods -A` shows all add-ons healthy | Configure cluster autoscaler. Set node group min/max. Test scale event. | Configure ALB + ACM cert + Route53 record via Terraform | Run Trivy scan on all images. Document any CVEs. Create image-versions.md | PR: k8s/config-server/ + k8s/discovery-server/ submitted for DE-1 review | PR: k8s/customers-service/ + k8s/vets-service/ submitted for DE-1 review | PR: k8s/visits-service/ + k8s/genai-service/ submitted for DE-1 review | PR: k8s/api-gateway/ + k8s/admin-server/ submitted for DE-1 review | Add GitHub Actions secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `ECR_REGISTRY`, `EKS_CLUSTER_NAME` | PR: k8s/monitoring/ submitted for DE-1 review |
| **5 PM** | **Day 1 Checkpoint:** VPC + EKS cluster running. All 8 images in ECR. All K8s YAML PRs submitted. | | | | | | | | | |

---

### DAY 2 — TUESDAY: EKS Ready + Deploy Infrastructure Services
> EKS cluster must be healthy by 9 AM (started yesterday)

| Time | DE-1 | DE-2 | DE-3 | DE-4 | DE-5 | DE-6 | DE-7 | DE-8 | DE-9 | DE-10 |
|---|---|---|---|---|---|---|---|---|---|---|
| 9–10 AM | **Daily Standup (15 min)** Review Day 1 blockers. Merge all K8s YAML PRs. Verify cluster health. | | | | | | | | | |
| 10 AM–12 PM | Review + merge all K8s manifest PRs. Test `kubectl apply --dry-run=client` for all manifests | Verify node group: `kubectl get nodes -o wide`. Confirm 4 nodes `Ready`. Set up cluster autoscaler policy. | Verify RDS reachable from EKS: run a test pod `kubectl run mysql-test --image=mysql:8 -- mysql -h <RDS_ENDPOINT> -u petclinic -p`. Verify ECR pull IAM policy on node role. | Update all image tags in K8s deployment.yaml files with ECR URIs. Commit to all service manifest folders. | **DEPLOY config-server:** `kubectl apply -f k8s/config-server/`. Watch pod logs. Verify `/actuator/health` returns `UP`. | WAIT for config-server to be `Running`. Then prep MySQL schemas: run migration test pod to create databases. | WAIT for config-server + discovery-server to be `Running`. Prep external secrets for DB creds. | WAIT for External Secrets Operator ready. Verify External Secrets sync working: `kubectl get externalsecrets -n petclinic` | Write `cd-deploy-eks.yml` — trigger on new ECR image push. Use `kubectl set image deployment/<name> <container>=<new-image>` or `kubectl rollout restart`. | **DEPLOY monitoring namespace.** Install kube-prometheus-stack: `helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring -f k8s/monitoring/prometheus-stack-values.yaml` |
| 12–1 PM | **Lunch Break** | | | | | | | | | |
| 1–3 PM | Monitor cluster events: `kubectl get events -n petclinic --sort-by='.lastTimestamp'`. Triage any issues. | Enable CloudWatch Container Insights on EKS cluster. Verify log groups created. | Verify Secrets Manager → External Secrets sync: K8s Secrets appear with correct values. | Verify image pull from ECR works on EKS: run `kubectl run test-pull --image=<ECR_URI>/config-server:latest --restart=Never`. Clean up. | **DEPLOY discovery-server.** Watch Eureka dashboard on port-forward: `kubectl port-forward svc/discovery-server 8761:8761 -n petclinic`. Confirm Eureka UI accessible. Config-server must appear registered. | Create the 3 MySQL databases via RDS: `petclinic_customers`, `petclinic_vets`, `petclinic_visits`. Verify with test pod. | Verify External Secrets created correctly for DB services. Port-forward to verify secrets values. | Verify AWS Load Balancer Controller is ready: `kubectl get pods -n kube-system | grep aws-load-balancer`. Confirm ACM cert is `ISSUED` in AWS console. | End-to-end test CI pipeline: push a branch, watch GitHub Actions run, confirm image pushed to ECR with correct SHA tag. | Deploy Zipkin: `kubectl apply -f k8s/monitoring/zipkin-deployment.yaml`. Port-forward and verify UI. |
| 3–5 PM | Validate: config-server + discovery-server both `Running`, healthy, Eureka shows 2 registered services | Validate node autoscaling with manual test: set replicas high, watch new nodes join | Verify all Terraform outputs documented in `terraform/outputs.tf`. Share RDS endpoint + ECR URIs with team. | Build final image versions table (all 8 services, ECR URI, SHA tag) — commit as `docs/image-versions.md` | Verify config-server serves config: `kubectl exec -n petclinic <config-server-pod> -- curl http://localhost:8888/customers-service/docker` shows customers config | Configure MySQL profiles in config repo: add `spring.datasource.*` pointing to RDS endpoint for all 3 services | Verify external secrets for DB creds populated: `kubectl get secret petclinic-db-secret -n petclinic -o yaml` | Configure Ingress annotations: `kubernetes.io/ingress.class: alb`, `alb.ingress.kubernetes.io/scheme: internet-facing`, `alb.ingress.kubernetes.io/certificate-arn: <ACM_ARN>` | Test CD pipeline: merge a dummy change, watch image build + EKS deploy trigger | Verify Prometheus targets: port-forward `kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring`. Confirm Prometheus UI up. |
| **5 PM** | **Day 2 Checkpoint:** EKS healthy, config-server + discovery-server running, Prometheus + Zipkin running, CI/CD pipeline working | | | | | | | | | |

---

### DAY 3 — WEDNESDAY: Deploy All Services
> Config + Discovery must be GREEN before anything else deploys

| Time | DE-1 | DE-2 | DE-3 | DE-4 | DE-5 | DE-6 | DE-7 | DE-8 | DE-9 | DE-10 |
|---|---|---|---|---|---|---|---|---|---|---|
| 9 AM | **Daily Standup (15 min).** Go/No-Go: config-server ✓, discovery-server ✓, RDS accessible ✓, images in ECR ✓ | | | | | | | | | |
| 9:15–11 AM | War room lead: monitor all deployments in real-time via `kubectl get pods -n petclinic -w` | Watch node scaling as pods come up. Ensure sufficient capacity. | Verify Secrets Manager values correct: DB credentials, OpenAI key. Verify `ExternalSecret` custom resources sync. | Support any image pull errors — verify IAM policies, ECR permissions | **Support DE-6, DE-7:** Help troubleshoot config/Eureka registration issues | **DEPLOY customers-service:** `kubectl apply -f k8s/customers-service/`. Watch until `Running`. Port-forward and verify: `curl http://localhost:8081/owners` returns JSON. | **DEPLOY visits-service:** `kubectl apply -f k8s/visits-service/`. Port-forward and verify: `curl http://localhost:8082/visits` works. | **DEPLOY api-gateway:** `kubectl apply -f k8s/api-gateway/`. Check Ingress created: `kubectl get ingress -n petclinic`. Note ALB DNS name. | Verify CD pipeline: manually trigger deploy workflow for customers-service. Confirm new pod rolls out. | Configure Prometheus scrape targets for all app services. Verify `/actuator/prometheus` endpoint reachable from Prometheus. |
| 11 AM–1 PM | Run cross-service health check: `kubectl get pods -n petclinic` — all pods must show `Running` (not `CrashLoopBackOff`) | Troubleshoot any node pressure issues. Cordon/drain nodes if needed. | Troubleshoot any Secrets Manager sync failures. | Patch image tags if any service uses wrong image. | Verify all 4 infra + business services registered in Eureka: `kubectl port-forward svc/discovery-server 8761:8761 -n petclinic` — count 5 services | **DEPLOY vets-service:** `kubectl apply -f k8s/vets-service/`. Verify cache working: call `/vets` twice, check logs for cache HIT. | **DEPLOY genai-service:** `kubectl apply -f k8s/genai-service/`. Verify OPENAI_API_KEY secret injected. Test: `curl http://localhost:8084/actuator/health`. | Verify ALB is created in AWS Console. Confirm target group health checks pass for api-gateway pods. Get the ALB DNS name. Test `curl http://<ALB-DNS>/actuator/health` | Document deployment steps in `docs/runbooks/deploy.md` | Import PetClinic Grafana dashboard JSON. Verify metrics flowing: request rate, JVM heap, cache hit ratio. |
| 1 PM | **Lunch** | | | | | | | | | |
| 1–3 PM | **Integration test:** Open browser → `http://<ALB-DNS>`. Test: view owners → add owner → add pet → add visit → view vets. All must work. | Confirm HPA working: `kubectl get hpa -n petclinic`. Manually scale test. | Fix any remaining secret/config issues. | Run final image scan report. Commit to `docs/security/trivy-report.md` | **DEPLOY admin-server:** `kubectl apply -f k8s/admin-server/`. Port-forward and verify all services appear in Admin UI. | Validate customers + vets: end-to-end through API Gateway. `curl http://<ALB-DNS>/api/customer/owners` and `curl http://<ALB-DNS>/api/vet/vets` must return data | Validate visits + genai: `curl http://<ALB-DNS>/api/visit/pets/1/visits`. Test GenAI chat through UI. | Test HTTPS: confirm `https://petclinic.team.com` loads UI (after Route53 propagation). Confirm HTTP→HTTPS redirect. | Test rollback pipeline: `kubectl rollout undo deployment/customers-service -n petclinic`. Confirm reverts to previous. Document in `docs/runbooks/rollback.md`. | Set up CloudWatch alarms: ECS task CPU > 80%, ALB 5xx > 1%, RDS CPU > 70%. Test each alarm. |
| 3–5 PM | Triage all pod failures. Run `kubectl describe pod` on any non-Running pod. Fix. | Ensure all services have correct resource requests set to prevent OOMKilled. | Verify TLS cert works: `curl -v https://petclinic.team.com` shows valid cert. | Update `docs/image-versions.md` with final deployed image SHAs | Monitor config-server and discovery-server stability. Verify no restarts: `kubectl get pods -n petclinic -o wide` shows 0 restarts. | Validate HPA for customers + vets: `kubectl get hpa -n petclinic`. Run quick load to trigger scale. | Validate HPA for visits: run load, watch pod count increase. | Configure sticky sessions on ALB if needed (for AngularJS session state). Add annotations. | CD pipeline smoke test: push change → watch image build → EKS auto-deploys → verify new pods running. | Zipkin: trigger a complete flow (add owner → add pet) and confirm trace spans visible in Zipkin UI. |
| **5 PM** | **Day 3 Checkpoint:** ALL 8 services running. UI accessible via ALB DNS. All services in Eureka. Prometheus scraping all. | | | | | | | | | |

---

### DAY 4 — THURSDAY: Hardening, Monitoring & CI/CD Polish
> All services must be running before Day 4 begins

| Task | Owner | Description |
|---|---|---|
| **OPS-D4-1** Resource limits tuning | DE-1 + DE-2 | Review OOMKilled or CPU throttling events. Adjust requests/limits. Ensure no pod is `Pending` due to resource pressure. |
| **OPS-D4-2** HPA validation | DE-1 | Run `kubectl top pods -n petclinic`. Simulate load. Verify HPA scales from 2→4 replicas and back. |
| **OPS-D4-3** Pod Disruption Budgets | DE-1 | Create PDB for customers, vets, visits, api-gateway: minimum 1 pod always available during rolling updates. |
| **OPS-D4-4** Rolling update test | DE-9 | Deploy a new image tag via CI/CD. Watch `kubectl rollout status deployment/customers-service -n petclinic`. Zero downtime. |
| **OPS-D4-5** Rollback test | DE-9 | Trigger rollback workflow. Confirm service returns to previous version within 3 minutes. |
| **OPS-D4-6** Full Grafana dashboards | DE-10 | Configure panels: request rate (rpm), error rate (%), p99 latency, JVM heap per service, cache hit ratio (vets), Eureka registered count. |
| **OPS-D4-7** CloudWatch dashboards | DE-10 | Create `PetClinic-Production` CloudWatch dashboard: ALB request count, 5xx rate, ECS CPU per service, RDS connections. |
| **OPS-D4-8** Zipkin trace validation | DE-10 | Trigger: `GET /api/customer/owners` through ALB. Find trace in Zipkin. Confirm spans: api-gateway → customers-service. |
| **OPS-D4-9** Security group audit | DE-2 | Verify: RDS only accepts connections from EKS node SG. No 0.0.0.0/0 on DB SG. EKS nodes not directly internet-accessible. |
| **OPS-D4-10** Secrets audit | DE-3 | Verify no secrets in any K8s manifest YAML. All secrets come from ExternalSecret resources. `kubectl get secret -n petclinic` shows secrets without base64 values in git. |
| **OPS-D4-11** Load test | DE-10 + all | Run JMeter (`scripts/` folder) with 50 concurrent users for 10 minutes against `https://petclinic.team.com`. Target: p99 < 3s, error rate < 0.1%. |
| **OPS-D4-12** Alerting validation | DE-10 | Deliberately spike CPU on one pod. Confirm CloudWatch alarm fires → SNS notification received. |
| **OPS-D4-13** Multi-AZ validation | DE-2 | Cordon one node in AZ-1. Verify pods reschedule to AZ-2/AZ-3. Uncordon node. Verify recovery. |
| **OPS-D4-14** Image tag policy | DE-4 | Enforce ECR lifecycle policy: keep last 10 images. Enable ECR image scanning on push. |
| **OPS-D4-15** DNS + TLS verification | DE-3 | Verify Route53 A record resolves. ACM cert not expiring. HTTP redirects to HTTPS. |
| **OPS-D4-16** OWASP ZAP scan | DE-4 | Run OWASP ZAP baseline scan against `https://petclinic.team.com`. Fix any HIGH findings. Commit report. |
| **OPS-D4-17** Runbook completion | DE-9 | Complete `docs/runbooks/deploy.md` and `docs/runbooks/rollback.md`. Any engineer must be able to deploy/rollback without help. |
| **OPS-D4-18** Monitoring runbook | DE-10 | Complete `docs/runbooks/monitoring.md`. Document: how to access Grafana, how to read Zipkin traces, how to find logs in CloudWatch. |

**Day 4 Exit Criteria:** Load test passes. Grafana dashboards live. Rollback tested. No security findings. All runbooks written.

---

### DAY 5 — FRIDAY: Production Go-Live
> Final checks, sign-offs, and go-live

| Time | Activity | Owner |
|---|---|---|
| 9–9:30 AM | **Go/No-Go Meeting:** All engineers confirm their Day 4 checklist items complete | DE-1 chairs |
| 9:30–11 AM | Final production smoke test: run all 5 golden paths manually against `https://petclinic.team.com` | DE-10 |
| 9:30–11 AM | Final resource review: `kubectl top nodes`, `kubectl top pods -n petclinic` — no hot nodes | DE-2 |
| 9:30–11 AM | Final secrets check: rotate any test credentials. Confirm prod Secrets Manager values are production keys. | DE-3 |
| 9:30–11 AM | Tag all ECR images with `v1.0.0` in addition to SHA tag. Update `docs/image-versions.md`. | DE-4 |
| 11 AM–12 PM | Final Grafana/Zipkin/CloudWatch dashboard review. Confirm all metrics showing. Set alert contacts. | DE-10 |
| 12–1 PM | **Lunch** | |
| 1–2 PM | **Go-Live:** Announce URL to stakeholders. Monitor dashboards actively. | All |
| 2–4 PM | **Live monitoring watch:** All engineers on standby. Watch CloudWatch + Grafana for anomalies. | All |
| 4–5 PM | **Post-launch review:** Document any issues. Commit final `docs/go-live-checklist.md`. Capture metrics baseline. | DE-1 |
| 5 PM | **Project Complete** | |

---

## 6. Jira Tickets (OPS Board)

> Use project key `OPS`. Tag each ticket with the day label: `day1`, `day2`, `day3`, `day4`, `day5`.

---

### DE-1 — EKS Cluster Lead

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-1 | 1 | Create k8s/ folder structure and namespace.yaml | 2 | `kubectl apply -f k8s/00-namespace.yaml` creates `petclinic` and `monitoring` namespaces |
| OPS-2 | 1 | Write and test all cluster add-on install scripts | 3 | Each `install.sh` script runs without errors. `kubectl get pods -n kube-system` shows all add-ons `Running` |
| OPS-3 | 2 | Install AWS Load Balancer Controller via Helm | 3 | `helm list -n kube-system` shows `aws-load-balancer-controller`. `kubectl get pods -n kube-system` shows controller pods `Running` |
| OPS-4 | 2 | Install External Secrets Operator | 2 | `kubectl get pods -n external-secrets` shows ESO pods `Running`. `ClusterSecretStore` created pointing to AWS Secrets Manager |
| OPS-5 | 2 | Install Metrics Server | 1 | `kubectl top nodes` returns CPU/memory data |
| OPS-6 | 2 | Install CloudWatch Container Insights | 2 | Log groups `/aws/containerinsights/petclinic-eks-cluster/application` visible in CloudWatch |
| OPS-7 | 2 | Review and merge all Day 1 K8s manifest PRs | 3 | All PRs from DE-5, DE-6, DE-7, DE-8, DE-10 merged. `kubectl apply --dry-run=client` passes for all |
| OPS-8 | 3 | Real-time deployment coordination | 2 | All 8 services `Running` by end of Day 3. Zero `CrashLoopBackOff` pods. |
| OPS-9 | 4 | Create Pod Disruption Budgets | 2 | PDB exists for customers, vets, visits, api-gateway. `kubectl get pdb -n petclinic` shows min 1 available |
| OPS-10 | 4 | Validate HPA scaling | 2 | HPA scales api-gateway from 2→4 replicas under load. Scales back to 2 after cool-down |
| OPS-11 | 5 | Chair Go/No-Go meeting and produce go-live checklist | 3 | `docs/go-live-checklist.md` committed with all checkboxes filled and signed |

---

### DE-2 — AWS Infrastructure (Terraform)

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-12 | 1 | Terraform S3 backend + DynamoDB lock | 2 | `terraform init` uses S3 remote backend. Lock table prevents concurrent applies |
| OPS-13 | 1 | Terraform VPC — 3 AZs, public + private + DB subnets | 5 | VPC created. 9 subnets. Internet Gateway. 3 NAT Gateways. Route tables correct. |
| OPS-14 | 1 | Terraform EKS cluster + managed node group | 5 | `aws eks describe-cluster` shows `ACTIVE`. `kubectl get nodes` shows 4 nodes `Ready` |
| OPS-15 | 1 | Terraform IAM — EKS node role, OIDC provider, add-on roles | 5 | Node role has ECR pull policy. OIDC provider created for service account annotations |
| OPS-16 | 2 | Configure cluster autoscaler | 3 | Cluster autoscaler deployment running. Node group scales 2–8. Test with `kubectl scale deployment/api-gateway --replicas=10` |
| OPS-17 | 2 | Enable CloudWatch Container Insights on EKS | 2 | DaemonSet `cloudwatch-agent` running on all nodes. Log groups visible in CloudWatch |
| OPS-18 | 4 | Security group audit and tighten rules | 3 | RDS SG: only port 3306 from EKS node SG. No 0.0.0.0/0 on RDS. EKS worker SGs reviewed. |
| OPS-19 | 4 | Multi-AZ resilience test | 3 | Cordon AZ-1 node. Pods reschedule. Uncordon. All pods return to Running. Documented. |
| OPS-20 | 5 | Final infrastructure validation and Terraform state clean | 2 | `terraform plan` shows no drift. All resources tagged: `Project=petclinic`, `Environment=production` |

---

### DE-3 — AWS Services (Terraform)

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-21 | 1 | Terraform ECR — 8 repositories with lifecycle policy | 3 | 8 ECR repos exist. Lifecycle policy: keep last 10 images. Image scanning on push enabled. |
| OPS-22 | 1 | Terraform Secrets Manager — DB creds + OpenAI key | 3 | Secrets created: `petclinic/db/credentials` (JSON: username, password, host, port), `petclinic/openai/key`. Values set manually (not in Terraform code) |
| OPS-23 | 1 | Terraform RDS MySQL 8.x (Multi-AZ, private subnet) | 5 | RDS endpoint reachable from EKS private subnet. Port 3306 open from EKS node SG only |
| OPS-24 | 2 | Create 3 MySQL databases on RDS | 2 | Databases `petclinic_customers`, `petclinic_vets`, `petclinic_visits` created. Petclinic user has full rights to each. |
| OPS-25 | 2 | Verify ExternalSecret syncs DB credentials into K8s | 3 | `kubectl get secret petclinic-db-secret -n petclinic` exists with `username`, `password`, `host`, `port` keys |
| OPS-26 | 2 | Terraform ACM — wildcard cert for team domain | 3 | ACM cert status: `ISSUED`. Covers `*.petclinic.team.com` |
| OPS-27 | 2 | Terraform Route53 — A record aliasing to ALB | 3 | `petclinic.team.com` resolves to ALB DNS after propagation. `nslookup petclinic.team.com` returns ALB IPs |
| OPS-28 | 4 | Secrets rotation policy | 2 | Secrets Manager rotation enabled for DB password (90-day rotation). Rotation Lambda connected. |
| OPS-29 | 5 | Final TLS and DNS verification | 2 | `curl -I https://petclinic.team.com` returns HTTP 200. Cert details show correct domain. No warnings. |

---

### DE-4 — Docker & Image Pipeline

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-30 | 1 | Build all 8 Spring Boot images locally | 3 | `./mvnw clean install -P buildDocker` completes with `BUILD SUCCESS`. All 8 images visible in `docker images` |
| OPS-31 | 1 | Validate app with docker compose | 3 | `docker compose up` starts all services. `http://localhost:8080` shows PetClinic UI. All golden paths work. |
| OPS-32 | 1 | Login to ECR and push all 8 images | 3 | All 8 images visible in ECR console with tag `latest` and `v1.0.0-beta` |
| OPS-33 | 1 | Run Trivy security scan on all images | 3 | No CRITICAL CVEs. HIGH CVEs documented in `docs/security/trivy-report.md`. Base image: `eclipse-temurin:17` |
| OPS-34 | 2 | Verify ECR pull works from EKS node | 2 | `kubectl run pull-test --image=<ECR_URI>/config-server:latest --restart=Never -n petclinic`. Pod status `Completed`. No `ErrImagePull`. |
| OPS-35 | 2 | Create docs/image-versions.md | 2 | File lists all 8 services with ECR URI, latest image tag, SHA256 digest, build date |
| OPS-36 | 3 | Support image troubleshooting during Day 3 deployments | 2 | Zero `ImagePullBackOff` or `ErrImagePull` events in `kubectl get events -n petclinic` |
| OPS-37 | 4 | Tag all images v1.0.0 and push | 2 | `v1.0.0` tags visible in ECR for all 8 services. `docs/image-versions.md` updated. |
| OPS-38 | 4 | Run OWASP ZAP baseline scan | 3 | ZAP scan against `https://petclinic.team.com`. No HIGH findings. Report at `docs/security/zap-report.html`. |
| OPS-39 | 5 | Final ECR audit — no dangling images | 1 | ECR shows only `latest`, `v1.0.0`, and last 3 SHA-tagged images per service. Old images purged by lifecycle policy. |

---

### DE-5 — K8s Infrastructure Services

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-40 | 1 | Write config-server K8s manifests | 3 | `deployment.yaml`: 1 replica, image from ECR, port 8888, liveness/readiness on `/actuator/health`, CPU 256m/512Mi mem. `service.yaml`: ClusterIP, port 8888, name `config-server` |
| OPS-41 | 1 | Write discovery-server K8s manifests | 3 | `deployment.yaml`: 1 replica, port 8761, `EUREKA_INSTANCE_PREFERIPADDRESS=true`, liveness on `/actuator/health`. `service.yaml`: ClusterIP + headless, port 8761, name `discovery-server` |
| OPS-42 | 1 | Write config-server ConfigMap for Git repo override | 2 | ConfigMap sets `GIT_REPO` env var pointing to team's forked config repo (or public repo if using default) |
| OPS-43 | 2 | Deploy config-server and verify | 3 | Pod `Running` (0 restarts). `kubectl exec ... -- curl http://localhost:8888/actuator/health` returns `{"status":"UP"}`. `kubectl exec ... -- curl http://localhost:8888/customers-service/docker` returns valid YAML |
| OPS-44 | 2 | Deploy discovery-server and verify | 3 | Pod `Running`. Eureka dashboard accessible via port-forward. config-server registered in Eureka. `DISCOVERY-SERVER` shows as registered instance. |
| OPS-45 | 3 | Monitor config + discovery stability throughout Day 3 | 2 | Zero pod restarts for config-server and discovery-server all day. All 8 services appear registered in Eureka. |
| OPS-46 | 4 | Write health-check script for infra services | 2 | Script `scripts/check-infra.sh` tests config-server `/actuator/health`, discovery-server Eureka API, and prints pass/fail for each |
| OPS-47 | 5 | Final infrastructure services validation | 1 | Both pods `Running`. Zero restarts since last deployment. Config-server serving all 8 service configs correctly. |

---

### DE-6 — K8s Business Services (Group A: Customers + Vets)

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-48 | 1 | Write customers-service K8s manifests | 3 | `deployment.yaml`: 2 replicas, port 8081, `SPRING_PROFILES_ACTIVE=docker,mysql`, envFrom DB secret, liveness/readiness probes. `service.yaml`: ClusterIP. `hpa.yaml`: min 2, max 6, CPU target 70% |
| OPS-49 | 1 | Write vets-service K8s manifests | 3 | Same pattern as customers. Port 8083. `hpa.yaml`: min 2, max 4 (vets data read-heavy, cached) |
| OPS-50 | 1 | Write ExternalSecret manifests for DB services | 3 | `external-secret.yaml` for both services. References `ClusterSecretStore`. Syncs `petclinic/db/credentials` to K8s Secret `petclinic-db-secret` |
| OPS-51 | 2 | Create MySQL databases and verify connectivity | 3 | Databases `petclinic_customers` and `petclinic_vets` exist on RDS. Test pod: `mysql -h <RDS> -u petclinic -p petclinic_customers -e "show tables"` returns empty (schema created on app start) |
| OPS-52 | 3 | Deploy customers-service and verify | 3 | 2 pods `Running`. Port-forward: `curl http://localhost:8081/owners` returns `[]` (empty list). Service registered in Eureka as `CUSTOMERS-SERVICE`. MySQL tables created on first start. |
| OPS-53 | 3 | Deploy vets-service and verify | 3 | 2 pods `Running`. Port-forward: `curl http://localhost:8083/vets` returns list of vets. Cache `vets` populated. Registered in Eureka as `VETS-SERVICE`. |
| OPS-54 | 3 | End-to-end test via API Gateway | 2 | `curl https://<ALB>/api/customer/owners` returns JSON. `curl https://<ALB>/api/vet/vets` returns vet list. |
| OPS-55 | 4 | HPA validation for customers + vets | 2 | Run `kubectl run load-test --image=busybox -- sh -c "while true; do wget -q -O- http://customers-service:8081/owners; done"`. HPA scales pods. |
| OPS-56 | 5 | Final validation — golden paths for owners/pets/vets | 2 | Through UI: Add owner, add pet, view vet list — all succeed. No errors in pod logs. |

---

### DE-7 — K8s Business Services (Group B: Visits + GenAI)

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-57 | 1 | Write visits-service K8s manifests | 3 | `deployment.yaml`: 2 replicas, port 8082, `SPRING_PROFILES_ACTIVE=docker,mysql`, DB secret env. `hpa.yaml`: min 2, max 6 |
| OPS-58 | 1 | Write genai-service K8s manifests | 3 | `deployment.yaml`: 1 replica, port 8084, `SPRING_PROFILES_ACTIVE=docker`. Env: `OPENAI_API_KEY` from ExternalSecret. If no key, set fallback to dummy — service starts but AI calls return 503. |
| OPS-59 | 1 | Write ExternalSecret for genai OpenAI key | 2 | `external-secret.yaml` references `petclinic/openai/key` from Secrets Manager → K8s Secret `petclinic-openai-secret` |
| OPS-60 | 2 | Create MySQL database for visits | 2 | Database `petclinic_visits` exists on RDS. Test connectivity from EKS. |
| OPS-61 | 3 | Deploy visits-service and verify | 3 | 2 pods `Running`. Port-forward: `curl http://localhost:8082/pets/1/visits` returns `[]`. Registered in Eureka as `VISITS-SERVICE`. MySQL schema created. |
| OPS-62 | 3 | Deploy genai-service and verify | 3 | 1 pod `Running`. Port-forward: `curl http://localhost:8084/actuator/health` returns `UP`. Registered in Eureka as `GENAI-SERVICE`. Test chat through UI if OpenAI key is valid. |
| OPS-63 | 3 | Verify genai circuit breaker fallback | 2 | Scale genai-service to 0: `kubectl scale deployment/genai-service --replicas=0 -n petclinic`. Open UI chat — shows friendly fallback message, not 500. Scale back to 1. |
| OPS-64 | 4 | HPA validation for visits-service | 2 | HPA scales visits-service pods under load. Scales back during cool-down. |
| OPS-65 | 5 | Final validation — add visit, verify GenAI | 2 | Add visit to pet through UI. Visit appears in list. GenAI chat responds (or shows fallback gracefully). |

---

### DE-8 — K8s Gateway & Ingress

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-66 | 1 | Write api-gateway K8s manifests | 3 | `deployment.yaml`: 2 replicas, port 8080, `SPRING_PROFILES_ACTIVE=docker`. `service.yaml`: ClusterIP, port 8080. `hpa.yaml`: min 2, max 6 |
| OPS-67 | 1 | Write Ingress manifest with ALB annotations | 5 | `ingress.yaml` with annotations: `kubernetes.io/ingress.class: alb`, `alb.ingress.kubernetes.io/scheme: internet-facing`, `alb.ingress.kubernetes.io/target-type: ip`, `alb.ingress.kubernetes.io/certificate-arn: <ACM_ARN>`, `alb.ingress.kubernetes.io/listen-ports: [{"HTTP":80},{"HTTPS":443}]`, `alb.ingress.kubernetes.io/ssl-redirect: '443'` |
| OPS-68 | 1 | Write admin-server K8s manifests | 2 | `deployment.yaml`: 1 replica, port 9090, `SPRING_PROFILES_ACTIVE=docker`. `service.yaml`: ClusterIP, port 9090. |
| OPS-69 | 2 | Verify ACM cert ARN and update Ingress | 2 | Get ACM ARN from DE-3: `aws acm list-certificates`. Update `ingress.yaml` with correct ARN. |
| OPS-70 | 3 | Deploy api-gateway and verify Ingress | 5 | 2 pods `Running`. `kubectl get ingress -n petclinic` shows ALB DNS. ALB visible in AWS console. Target group shows 2 healthy targets. `curl http://<ALB-DNS>/actuator/health` returns 200. |
| OPS-71 | 3 | Verify HTTPS and HTTP redirect | 3 | `curl -I http://<ALB-DNS>` returns 301 redirect to HTTPS. `curl -I https://petclinic.team.com` returns 200. Certificate valid. |
| OPS-72 | 3 | Deploy admin-server and verify | 2 | Port-forward: Spring Boot Admin UI shows all 8 services registered as UP. |
| OPS-73 | 4 | Configure ALB access logging | 2 | ALB access logs enabled, stored in S3 bucket `petclinic-alb-logs`. 30-day retention. |
| OPS-74 | 4 | Test gateway circuit breaker in EKS | 3 | Scale customers-service to 0. Hit `/api/customer/owners` through ALB. Receive 503 with fallback body, not 500 exception. Scale back to 2. |
| OPS-75 | 5 | Final gateway validation — all routes working | 2 | All 4 routes via ALB: `/api/customer/**`, `/api/vet/**`, `/api/visit/**`, `/api/genai/**` all return expected responses. |

---

### DE-9 — CI/CD Pipelines

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-76 | 1 | Create GitHub Actions CI pipeline — build and test | 3 | `.github/workflows/ci.yml`: triggers on PR, runs `./mvnw clean verify`. Reports pass/fail on PR. |
| OPS-77 | 1 | Set up GitHub Actions secrets | 2 | Secrets configured: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `ECR_REGISTRY`, `EKS_CLUSTER_NAME`. No secrets in YAML files. |
| OPS-78 | 2 | Create CD pipeline — build and push to ECR | 5 | `.github/workflows/cd-build-push.yml`: triggers on push to `main`. Builds all changed services. Tags with `${{ github.sha }}` and `latest`. Pushes to ECR. |
| OPS-79 | 2 | Create CD pipeline — rolling deploy to EKS | 5 | `.github/workflows/cd-deploy-eks.yml`: after push to ECR, updates deployment image tag in EKS. `kubectl rollout status` waits for deployment to stabilize. |
| OPS-80 | 3 | Create rollback workflow | 3 | `.github/workflows/rollback.yml`: manual trigger with input: service name + image tag. Runs `kubectl set image deployment/<service> <container>=<ECR>:<tag> -n petclinic`. Confirms rollout. |
| OPS-81 | 3 | End-to-end CI/CD test | 3 | Merge a dummy change to `main`. Watch: CI passes → image pushed to ECR → EKS deployment updated → new pods `Running`. Total time < 10 minutes. |
| OPS-82 | 4 | Test rollback pipeline | 3 | Trigger rollback workflow for customers-service to previous SHA. Confirm previous image running within 3 minutes. Document in `docs/runbooks/rollback.md`. |
| OPS-83 | 4 | Write deployment and rollback runbooks | 3 | `docs/runbooks/deploy.md` and `docs/runbooks/rollback.md`. Any team member can follow them independently. |
| OPS-84 | 5 | Final CI/CD validation | 2 | Full pipeline run: code change → CI → ECR → EKS. Zero manual steps. Confirmed by DE-1. |

---

### DE-10 — Monitoring & Observability

| Ticket | Day | Title | Points | Acceptance Criteria |
|---|---|---|---|---|
| OPS-85 | 1 | Write kube-prometheus-stack Helm values | 3 | `k8s/monitoring/prometheus-stack-values.yaml` with: persistence enabled (EBS 20Gi for Prometheus, 10Gi for Grafana), scrape interval 15s, admin password set, service type ClusterIP |
| OPS-86 | 1 | Write Zipkin K8s manifests | 2 | `zipkin-deployment.yaml`: 1 replica, image `openzipkin/zipkin`, port 9411, memory 512Mi. `zipkin-service.yaml`: ClusterIP port 9411. |
| OPS-87 | 2 | Install kube-prometheus-stack via Helm | 3 | `helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml`. All pods `Running`. Port-forward: Prometheus UI and Grafana UI accessible. |
| OPS-88 | 2 | Deploy Zipkin | 2 | `kubectl apply -f k8s/monitoring/`. Zipkin pod `Running`. Port-forward: Zipkin UI accessible at `http://localhost:9411/zipkin/` |
| OPS-89 | 3 | Configure Prometheus scrape targets for all app services | 5 | Add `ServiceMonitor` CRDs for all 8 services pointing to `/actuator/prometheus`. Verify in Prometheus UI: all 8 targets show `UP` state. |
| OPS-90 | 3 | Import PetClinic Grafana dashboards | 3 | Import dashboard JSONs from `docker/grafana/`. Dashboard shows: request rate, error rate, JVM heap, cache hit ratio (vets), DB connection pool, active visits. |
| OPS-91 | 4 | Run JMeter load test and record baseline | 5 | 50 concurrent users, 10 minutes, against `https://petclinic.team.com`. p99 < 3s. Error rate < 0.1%. Record baseline in `docs/performance-baseline.md`. |
| OPS-92 | 4 | Configure CloudWatch alarms + SNS | 3 | Alarms: ALB 5xx > 1% (5 min period), EKS node CPU > 80%, RDS CPU > 70%. SNS topic sends email to team. Test: verify alarm transitions to ALARM state on demand. |
| OPS-93 | 4 | Create CloudWatch dashboard | 3 | Dashboard `PetClinic-Production`: ALB request count, 5xx rate, EKS pod count per service, RDS connections, P99 latency widget. |
| OPS-94 | 4 | Validate Zipkin distributed traces | 3 | Trigger: add owner via UI. Find trace in Zipkin. Confirm: api-gateway span → customers-service span visible with duration. |
| OPS-95 | 5 | 24-hour post-launch monitoring report | 3 | At end of Day 5, post monitoring report: total requests, error rate, p99 latency, scaling events, any anomalies. |

---

## 7. Prerequisites — Day 1 Morning Checklist

Every engineer must complete this before 10 AM on Day 1:

```bash
# 1. Clone the repo
git clone <team-repo-url>
cd spring-petclinic-microservices

# 2. Verify Java + Maven
java -version   # Must be: openjdk 17.x
./mvnw -version # Must be: Apache Maven 3.x

# 3. Verify Docker
docker --version         # Must be: 24+
docker compose version   # Must be: v2.x
docker ps                # Must run without sudo

# 4. Verify AWS CLI
aws --version            # Must be: aws-cli/2.x
aws sts get-caller-identity  # Must return your IAM user ARN

# 5. Verify kubectl (for DE-1 through DE-10 after Day 2)
kubectl version --client  # Must be: v1.29+

# 6. Verify Terraform (DE-2, DE-3 only)
terraform --version       # Must be: 1.9+

# 7. Verify Helm (DE-1, DE-10)
helm version              # Must be: v3.x
```

### AWS Account Prerequisites (M1/DE-2 to set up Day 1 morning)
- AWS account with admin IAM user for Terraform
- IAM users created for each engineer with these policies:
  - `AmazonEKSClusterPolicy`, `AmazonEKSWorkerNodePolicy`
  - `AmazonEC2ContainerRegistryFullAccess`
  - `AmazonRDSFullAccess`, `SecretsManagerReadWrite`
  - `AmazonRoute53FullAccess`, `AWSCertificateManagerFullAccess`
- S3 bucket for Terraform state: created manually before `terraform init`
- Domain name delegated to Route53 (or use ALB DNS directly for testing)

---

## 8. Critical Deployment Order

**NEVER deploy out of this order. This is not optional.**

```
Phase 1 — Infrastructure Layer (Day 2)
  Step 1: kubectl apply -f k8s/00-namespace.yaml
  Step 2: (Install cluster add-ons — DE-1)
  Step 3: kubectl apply -f k8s/config-server/
          → WAIT: curl http://config-server:8888/actuator/health returns UP
  Step 4: kubectl apply -f k8s/discovery-server/
          → WAIT: Eureka UI shows DISCOVERY-SERVER registered

Phase 2 — Business Services (Day 3, after Phase 1 complete)
  Step 5: kubectl apply -f k8s/customers-service/
  Step 6: kubectl apply -f k8s/vets-service/
  Step 7: kubectl apply -f k8s/visits-service/
  Step 8: kubectl apply -f k8s/genai-service/   (optional)
          → WAIT: all 4 appear in Eureka dashboard

Phase 3 — Frontend & Monitoring (Day 3, after Phase 2 complete)
  Step 9:  kubectl apply -f k8s/admin-server/
  Step 10: kubectl apply -f k8s/api-gateway/
           → WAIT: ALB target group shows healthy targets
  Step 11: kubectl apply -f k8s/monitoring/
```

### Quick Health Check Commands

```bash
# Check all pods at once
kubectl get pods -n petclinic -o wide

# Watch pods in real-time
kubectl get pods -n petclinic -w

# Check any failing pod
kubectl describe pod <pod-name> -n petclinic
kubectl logs <pod-name> -n petclinic --previous

# Verify Eureka registrations (port-forward)
kubectl port-forward svc/discovery-server 8761:8761 -n petclinic
# Open: http://localhost:8761

# Verify config server is serving config
kubectl port-forward svc/config-server 8888:8888 -n petclinic
curl http://localhost:8888/customers-service/docker

# Test API Gateway directly
kubectl port-forward svc/api-gateway 8080:8080 -n petclinic
curl http://localhost:8080/actuator/health
```

### Common Failure Patterns and Fixes

| Symptom | Cause | Fix |
|---|---|---|
| `CrashLoopBackOff` on any service | Config server not reachable | Check config-server pod first: `kubectl logs -n petclinic config-server-<pod>` |
| `ErrImagePull` | ECR IAM policy missing on node role | `aws iam attach-role-policy --role-name <node-role> --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly` |
| Service not appearing in Eureka | Discovery server not ready | Check discovery-server logs for startup errors. Restart the failing service pod. |
| ALB not creating | AWS LB Controller not running | `kubectl get pods -n kube-system | grep aws-load-balancer` — reinstall if missing |
| `OOMKilled` | Memory limit too low | Increase `resources.limits.memory` in deployment.yaml. `kubectl apply -f` to update. |
| DB connection refused | Wrong DB host in secret | Verify ExternalSecret synced: `kubectl get secret petclinic-db-secret -n petclinic -o jsonpath='{.data.host}' | base64 -d` |
| `Pending` pods | Insufficient node capacity | Check: `kubectl describe pod <pending-pod>`. If `Insufficient cpu/memory`, cluster autoscaler should add nodes within 2 min. |

---

## 9. Daily Definition of Done

### End of Day 1 ✓
- [ ] VPC created with 3 AZs (9 subnets total)
- [ ] EKS cluster `ACTIVE` with 4 nodes `Ready`
- [ ] All 8 ECR repositories created
- [ ] All 8 Docker images built and pushed to ECR
- [ ] All K8s YAML manifests written and PRs submitted
- [ ] GitHub Actions CI skeleton created
- [ ] `docker compose up` works locally (sanity check)

### End of Day 2 ✓
- [ ] All cluster add-ons installed and running (LB Controller, ESO, Metrics Server, CWAgent)
- [ ] config-server pod `Running` — health check returns `UP`
- [ ] discovery-server pod `Running` — Eureka UI accessible
- [ ] config-server + discovery-server registered in Eureka
- [ ] RDS MySQL accessible from EKS — 3 databases created
- [ ] External Secrets syncing DB creds to K8s Secrets
- [ ] CI pipeline running on PRs
- [ ] Prometheus + Grafana + Zipkin pods `Running` in monitoring namespace

### End of Day 3 ✓
- [ ] All 8 application services `Running` in petclinic namespace
- [ ] All 8 services registered in Eureka
- [ ] ALB created and showing healthy targets
- [ ] `https://petclinic.team.com` (or ALB DNS) loads PetClinic UI
- [ ] All golden paths work: view owners, add owner, add pet, add visit, view vets
- [ ] CD pipeline: push to main triggers EKS rolling deploy
- [ ] Prometheus scraping all 8 services
- [ ] Grafana dashboards showing live metrics
- [ ] Zipkin showing distributed traces

### End of Day 4 ✓
- [ ] Load test passed: p99 < 3s, error rate < 0.1%
- [ ] HPA validated: pods scale under load
- [ ] Rollback tested and documented
- [ ] Pod Disruption Budgets configured
- [ ] CloudWatch alarms active and tested
- [ ] OWASP ZAP scan complete — no HIGH findings
- [ ] All runbooks written and reviewed
- [ ] Security group audit complete

### End of Day 5 (Go-Live) ✓
- [ ] All 8 services `Running` with 0 restarts in last 2 hours
- [ ] HTTPS working with valid certificate
- [ ] All 5 golden path smoke tests pass
- [ ] Grafana dashboards showing production metrics
- [ ] CloudWatch alarms monitoring
- [ ] All images tagged `v1.0.0`
- [ ] Runbooks committed
- [ ] Team announced go-live to stakeholders
- [ ] `docs/go-live-checklist.md` signed off by DE-1

---

## 10. Rollback Procedures

### Service-Level Rollback (< 3 minutes)
```bash
# Option A: Roll back to previous deployment
kubectl rollout undo deployment/<service-name> -n petclinic

# Option B: Roll back to specific image tag
kubectl set image deployment/<service-name> \
  <service-name>=<ECR_URI>/<service-name>:<previous-sha> \
  -n petclinic

# Watch rollback progress
kubectl rollout status deployment/<service-name> -n petclinic

# Verify
kubectl get pods -n petclinic | grep <service-name>
```

### Full Stack Rollback (nuclear option)
```bash
# Scale all business services to 0
kubectl scale deployment customers-service visits-service vets-service \
  genai-service api-gateway --replicas=0 -n petclinic

# Fix the issue
# Redeploy in order (Step 5–10 from Section 8)
```

### Infrastructure Rollback (Terraform)
```bash
# Revert to last known good state
cd terraform/
terraform apply -target=<resource> -var-file=terraform.tfvars

# In emergency: restore from Terraform state history in S3
aws s3 cp s3://petclinic-tf-state/<previous-state>.tfstate terraform.tfstate
```

---

## Quick Reference Card (Print This)

```
┌─────────────────────────────────────────────────────────────┐
│           PETCLINIC EKS — QUICK REFERENCE                   │
├──────────────┬──────────────────────────────────────────────┤
│ Production   │ https://petclinic.team.com                   │
│ Grafana      │ kubectl port-forward svc/grafana 3000:80 -n monitoring │
│ Zipkin       │ kubectl port-forward svc/zipkin 9411:9411 -n monitoring │
│ Eureka       │ kubectl port-forward svc/discovery-server 8761:8761 -n petclinic │
│ Admin        │ kubectl port-forward svc/admin-server 9090:9090 -n petclinic │
├──────────────┼──────────────────────────────────────────────┤
│ Check pods   │ kubectl get pods -n petclinic -o wide        │
│ Check logs   │ kubectl logs -f <pod> -n petclinic           │
│ Describe pod │ kubectl describe pod <pod> -n petclinic      │
│ Rollback svc │ kubectl rollout undo deployment/<svc> -n petclinic │
│ Scale svc    │ kubectl scale deployment/<svc> --replicas=3 -n petclinic │
├──────────────┼──────────────────────────────────────────────┤
│ DE-1 (Lead)  │ Cluster, add-ons, go-live                   │
│ DE-2 (Infra) │ VPC, EKS, IAM, autoscaling                  │
│ DE-3 (AWS)   │ ECR, RDS, ALB, Route53, Secrets             │
│ DE-4 (Docker)│ Images, ECR push, security scan             │
│ DE-5 (K8s)   │ config-server, discovery-server             │
│ DE-6 (K8s)   │ customers-service, vets-service             │
│ DE-7 (K8s)   │ visits-service, genai-service               │
│ DE-8 (K8s)   │ api-gateway, admin-server, Ingress           │
│ DE-9 (CI/CD) │ GitHub Actions, ECR pipeline, rollback      │
│ DE-10 (Ops)  │ Prometheus, Grafana, Zipkin, CloudWatch      │
└──────────────┴──────────────────────────────────────────────┘
```

---

*Last updated: Day 0 (before project start). All changes to this document via PR only.*
