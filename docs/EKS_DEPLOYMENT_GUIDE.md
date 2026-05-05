# Spring PetClinic Microservices — Complete EKS Deployment Guide
## From Scratch to Production

> Built from actual source code. Every port, image name, and profile is verified against the real project files.

---

## Table of Contents

1. [Verified Project Facts](#1-verified-project-facts)
2. [Phase 0 — Prerequisites](#2-phase-0--prerequisites)
3. [Phase 1 — Day 1: Foundation](#3-phase-1--day-1-foundation)
4. [Phase 2 — Day 2: Add-ons + Infrastructure Services](#4-phase-2--day-2-add-ons--infrastructure-services)
5. [Phase 3 — Day 3: Deploy All Business Services](#5-phase-3--day-3-deploy-all-business-services)
6. [Phase 4 — Day 4: CI/CD Pipelines](#6-phase-4--day-4-cicd-pipelines)
7. [Phase 5 — Day 5: Hardening + Go-Live](#7-phase-5--day-5-hardening--go-live)
8. [Quick Reference](#8-quick-reference)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Verified Project Facts

Read this section before issuing any command. These are facts taken directly from the source code.

### Service Ports (use these in ALL K8s manifests)

| Service | Real Port | pom.xml EXPOSE | Image Name |
|---|---|---|---|
| config-server | **8888** | 8888 ✅ | `springcommunity/spring-petclinic-config-server` |
| discovery-server | **8761** | 8761 ✅ | `springcommunity/spring-petclinic-discovery-server` |
| customers-service | **8081** | 8081 ✅ | `springcommunity/spring-petclinic-customers-service` |
| visits-service | **8082** | 8081 ⚠️ wrong | `springcommunity/spring-petclinic-visits-service` |
| vets-service | **8083** | 8081 ⚠️ wrong | `springcommunity/spring-petclinic-vets-service` |
| genai-service | **8084** | 8081 ⚠️ wrong | `springcommunity/spring-petclinic-genai-service` |
| api-gateway | **8080** | 8081 ⚠️ wrong | `springcommunity/spring-petclinic-api-gateway` |
| admin-server | **9090** | 9090 ✅ | `springcommunity/spring-petclinic-admin-server` |

> The `docker.image.exposed.port` values in pom.xml for visits, vets, genai, and api-gateway are all wrong (say 8081).
> The real port is controlled by Spring Boot config. Use the **Real Port** column for all K8s `containerPort`, `targetPort`, and probe `port` values.

### Critical Behavior Facts

| Fact | Detail |
|---|---|
| Spring Boot version | 4.0.1 |
| Spring Cloud version | 2025.1.0 |
| Java version | 17 (eclipse-temurin:17 base image) |
| Docker profile | Baked into every image via `ENV SPRING_PROFILES_ACTIVE=docker` in Dockerfile |
| Config server DNS | When `docker` profile is active, every service hard-wires to `http://config-server:8888`. K8s Service name **must** be exactly `config-server`. |
| Discovery server DNS | K8s Service name **must** be exactly `discovery-server`. |
| vets-service profile | Has `spring.profiles.active: production` hardcoded in `application.yml`. Running with `docker` makes both `docker` + `production` active. This is expected — `production` activates Spring Cache. |
| genai-service profile | Also has `spring.profiles.active: production` hardcoded. Expected behavior. |
| genai-service runtime | Uses Spring WebFlux (reactive). Starts on Netty. Logs say `Netty started on port 8084` — this is correct. |
| Config server git repo | `https://github.com/spring-petclinic/spring-petclinic-microservices-config` |
| Image prefix | `springcommunity` |

### Docker Compose Port Map (local only — do NOT use for K8s)

```
api-gateway:       http://localhost:8080
config-server:     http://localhost:8888
discovery-server:  http://localhost:8761
customers-service: http://localhost:8081
visits-service:    http://localhost:8082
vets-service:      http://localhost:8083
genai-service:     http://localhost:8084
admin-server:      http://localhost:9090
zipkin:            http://localhost:9411
prometheus:        http://localhost:9091   ← host port 9091, not 9090
grafana:           http://localhost:3030   ← host port 3030, not 3000
```

---

## 2. Phase 0 — Prerequisites

### Tool Installation Checklist (Every Engineer)

```bash
# Java 17
java -version
# Expected: openjdk 17.x

# Maven wrapper (no install needed — bundled in repo)
./mvnw -version
# Expected: Apache Maven 3.x

# Docker
docker --version          # Expected: 24+
docker compose version    # Expected: v2.x
docker ps                 # Must run without sudo or error

# AWS CLI v2
aws --version             # Expected: aws-cli/2.x
aws sts get-caller-identity   # Must return your IAM user ARN

# kubectl v1.29+
kubectl version --client

# Terraform 1.9+
terraform --version

# Helm v3
helm version
```

### AWS Account Bootstrap (DE-2 — do once before Day 1)

```bash
# 1. Create S3 bucket for Terraform remote state
aws s3 mb s3://petclinic-tf-state --region us-east-1

# 2. Enable versioning on state bucket
aws s3api put-bucket-versioning \
  --bucket petclinic-tf-state \
  --versioning-configuration Status=Enabled

# 3. Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name petclinic-tf-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

---

## 3. Phase 1 — Day 1: Foundation

### Step 1.1 — Build All 8 Docker Images (DE-4)

```bash
cd spring-petclinic-microservices

# Build all 8 images — takes 10–20 min on first run
./mvnw clean install -P buildDocker -DskipTests

# Verify all 8 images exist
docker images | grep springcommunity
```

Expected — exactly these 8 images:
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

### Step 1.2 — Validate Locally with Docker Compose (DE-4)

Run the full stack locally to confirm the app works before touching AWS.

```bash
docker compose up -d

# Wait 60–90 seconds, then verify
curl http://localhost:8080/actuator/health

# Open browser: http://localhost:8080 — should show PetClinic UI
# Test golden paths:
#   - View owner list
#   - Add an owner
#   - Add a pet
#   - Add a visit
#   - View vet list

docker compose down
```

### Step 1.3 — Terraform: VPC + EKS (DE-2)

Create the `terraform/` directory in the repo root.

**`terraform/backend.tf`**
```hcl
terraform {
  backend "s3" {
    bucket         = "petclinic-tf-state"
    key            = "eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "petclinic-tf-lock"
    encrypt        = true
  }
}
```

**`terraform/variables.tf`**
```hcl
variable "aws_region"   { default = "us-east-1" }
variable "cluster_name" { default = "petclinic-eks-cluster" }
variable "vpc_cidr"     { default = "10.0.0.0/16" }
variable "db_password"  { sensitive = true }
```

**`terraform/vpc.tf`**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "petclinic-vpc"
  cidr = var.vpc_cidr

  azs              = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_subnets   = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets  = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
  database_subnets = ["10.0.7.0/24", "10.0.8.0/24", "10.0.9.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false
  one_nat_gateway_per_az = true

  enable_dns_hostnames = true
  enable_dns_support   = true

  # Required tags for EKS to discover subnets
  public_subnet_tags = {
    "kubernetes.io/role/elb"                    = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"           = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }

  tags = { Project = "petclinic", Environment = "production" }
}
```

**`terraform/eks.tf`**
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.31"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    petclinic = {
      instance_types = ["t3.large"]
      min_size       = 2
      max_size       = 8
      desired_size   = 4
    }
  }

  iam_role_additional_policies = {
    ECR = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  }

  tags = { Project = "petclinic", Environment = "production" }
}
```

**Apply:**
```bash
cd terraform/
terraform init
terraform plan
terraform apply        # VPC ~5 min, EKS cluster ~15 min

# Configure kubectl
aws eks update-kubeconfig \
  --name petclinic-eks-cluster \
  --region us-east-1

# Verify 4 nodes are Ready
kubectl get nodes
```

### Step 1.4 — Terraform: ECR + RDS + Secrets Manager (DE-3)

**`terraform/ecr.tf`**
```hcl
locals {
  services = [
    "spring-petclinic-config-server",
    "spring-petclinic-discovery-server",
    "spring-petclinic-customers-service",
    "spring-petclinic-visits-service",
    "spring-petclinic-vets-service",
    "spring-petclinic-genai-service",
    "spring-petclinic-api-gateway",
    "spring-petclinic-admin-server",
  ]
}

resource "aws_ecr_repository" "petclinic" {
  for_each             = toset(local.services)
  name                 = "springcommunity/${each.key}"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration { scan_on_push = true }

  tags = { Project = "petclinic" }
}

resource "aws_ecr_lifecycle_policy" "petclinic" {
  for_each   = aws_ecr_repository.petclinic
  repository = each.value.name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 10 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 10
      }
      action = { type = "expire" }
    }]
  })
}
```

**`terraform/rds.tf`**
```hcl
resource "aws_db_subnet_group" "petclinic" {
  name       = "petclinic-db-subnet-group"
  subnet_ids = module.vpc.database_subnets
  tags       = { Project = "petclinic" }
}

resource "aws_security_group" "rds" {
  name   = "petclinic-rds-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [module.eks.node_security_group_id]
  }
  tags = { Project = "petclinic" }
}

resource "aws_db_instance" "petclinic" {
  identifier                = "petclinic-mysql"
  engine                    = "mysql"
  engine_version            = "8.0"
  instance_class            = "db.t3.medium"
  allocated_storage         = 20
  db_name                   = "petclinic"
  username                  = "petclinic"
  password                  = var.db_password
  db_subnet_group_name      = aws_db_subnet_group.petclinic.name
  vpc_security_group_ids    = [aws_security_group.rds.id]
  multi_az                  = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "petclinic-final"
  tags = { Project = "petclinic" }
}
```

**`terraform/secrets-manager.tf`**
```hcl
resource "aws_secretsmanager_secret" "db" {
  name = "petclinic/db/credentials"
}

resource "aws_secretsmanager_secret" "openai" {
  name = "petclinic/openai/key"
}
```

After applying, populate secrets via CLI (never put values in .tf files):
```bash
RDS_ENDPOINT=$(terraform output -raw rds_endpoint)

# DB credentials
aws secretsmanager put-secret-value \
  --secret-id petclinic/db/credentials \
  --secret-string "{\"username\":\"petclinic\",\"password\":\"<your-password>\",\"host\":\"${RDS_ENDPOINT}\",\"port\":\"3306\"}"

# OpenAI key
aws secretsmanager put-secret-value \
  --secret-id petclinic/openai/key \
  --secret-string '{"api-key":"sk-..."}'
```

### Step 1.5 — Push Images to ECR (DE-4)

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=us-east-1
export REPOSITORY_PREFIX=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/springcommunity
export VERSION=1.0.0

# Authenticate to ECR
aws ecr get-login-password --region ${AWS_REGION} | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

# Tag and push using the existing project scripts
./scripts/tagImages.sh
./scripts/pushImages.sh

# Verify one repo as a sanity check
aws ecr list-images \
  --repository-name springcommunity/spring-petclinic-config-server \
  --region ${AWS_REGION}
```

### Step 1.6 — Create K8s Folder Structure (DE-1)

```bash
mkdir -p k8s/{cluster-addons/{aws-load-balancer-controller,external-secrets-operator,metrics-server,cloudwatch-container-insights},config-server,discovery-server,customers-service,vets-service,visits-service,genai-service,api-gateway,admin-server,monitoring}
```

**`k8s/00-namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: petclinic
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl get namespaces | grep -E "petclinic|monitoring"
```

---

## 4. Phase 2 — Day 2: Add-ons + Infrastructure Services

### Step 2.1 — Install Cluster Add-ons (DE-1)

**AWS Load Balancer Controller:**
```bash
# Create IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.0/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create IRSA service account
eksctl create iamserviceaccount \
  --cluster=petclinic-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts && helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=petclinic-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

**External Secrets Operator:**
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace

kubectl get pods -n external-secrets
```

**Metrics Server:**
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system

kubectl top nodes   # should return data within ~60s
```

### Step 2.2 — Create ClusterSecretStore (DE-1 + DE-3)

**`k8s/cluster-addons/external-secrets-operator/cluster-secret-store.yaml`**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

```bash
kubectl apply -f k8s/cluster-addons/external-secrets-operator/cluster-secret-store.yaml
```

### Step 2.3 — Create MySQL Databases on RDS (DE-3)

```bash
RDS_ENDPOINT=$(terraform -chdir=terraform output -raw rds_endpoint)

kubectl run mysql-setup \
  --image=mysql:8 \
  --restart=Never \
  -n petclinic \
  --env="MYSQL_PWD=<your-password>" \
  -- mysql -h ${RDS_ENDPOINT} -u petclinic -e "
    CREATE DATABASE IF NOT EXISTS petclinic_customers;
    CREATE DATABASE IF NOT EXISTS petclinic_vets;
    CREATE DATABASE IF NOT EXISTS petclinic_visits;
    GRANT ALL PRIVILEGES ON petclinic_customers.* TO 'petclinic'@'%';
    GRANT ALL PRIVILEGES ON petclinic_vets.* TO 'petclinic'@'%';
    GRANT ALL PRIVILEGES ON petclinic_visits.* TO 'petclinic'@'%';
    FLUSH PRIVILEGES;
  "

kubectl delete pod mysql-setup -n petclinic
```

### Step 2.4 — Deploy config-server (DE-5)

> **Deploy this first. Nothing else can start without it.**

**`k8s/config-server/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-server
  namespace: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-server
  template:
    metadata:
      labels:
        app: config-server
    spec:
      containers:
      - name: config-server
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-config-server:1.0.0
        ports:
        - containerPort: 8888
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker"
        resources:
          requests:
            cpu: 256m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8888
          initialDelaySeconds: 60
          periodSeconds: 15
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8888
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 10
```

**`k8s/config-server/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: config-server    # must be exactly "config-server"
  namespace: petclinic
spec:
  selector:
    app: config-server
  ports:
  - port: 8888
    targetPort: 8888
  type: ClusterIP
```

```bash
kubectl apply -f k8s/config-server/

# WAIT for Running before proceeding
kubectl get pods -n petclinic -w

# Verify health
kubectl exec -n petclinic deploy/config-server -- \
  curl -s http://localhost:8888/actuator/health
# Must return: {"status":"UP"...}

# Verify it serves app configs
kubectl exec -n petclinic deploy/config-server -- \
  curl -s http://localhost:8888/customers-service/docker
# Must return YAML config
```

### Step 2.5 — Deploy discovery-server (DE-5)

> **Deploy this second. Business services won't register without it.**

**`k8s/discovery-server/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: discovery-server
  namespace: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: discovery-server
  template:
    metadata:
      labels:
        app: discovery-server
    spec:
      containers:
      - name: discovery-server
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-discovery-server:1.0.0
        ports:
        - containerPort: 8761
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker"
        - name: EUREKA_INSTANCE_PREFERIPADDRESS
          value: "true"
        resources:
          requests:
            cpu: 256m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8761
          initialDelaySeconds: 60
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8761
          initialDelaySeconds: 30
          periodSeconds: 10
```

**`k8s/discovery-server/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: discovery-server    # must be exactly "discovery-server"
  namespace: petclinic
spec:
  selector:
    app: discovery-server
  ports:
  - port: 8761
    targetPort: 8761
  type: ClusterIP
```

```bash
kubectl apply -f k8s/discovery-server/

# Verify Eureka UI
kubectl port-forward svc/discovery-server 8761:8761 -n petclinic
# Open: http://localhost:8761 — Eureka dashboard should be accessible
# Ctrl+C to stop port-forward
```

### Step 2.6 — Deploy Monitoring Stack (DE-10)

**`k8s/monitoring/prometheus-stack-values.yaml`**
```yaml
prometheus:
  prometheusSpec:
    scrapeInterval: 15s
    retention: 7d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi

grafana:
  adminPassword: "petclinic-admin"   # change for production
  persistence:
    enabled: true
    size: 10Gi
  service:
    type: ClusterIP

alertmanager:
  enabled: true
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f k8s/monitoring/prometheus-stack-values.yaml

kubectl get pods -n monitoring
```

**`k8s/monitoring/zipkin-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zipkin
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zipkin
  template:
    metadata:
      labels:
        app: zipkin
    spec:
      containers:
      - name: zipkin
        image: openzipkin/zipkin
        ports:
        - containerPort: 9411
        resources:
          limits:
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: zipkin
  namespace: monitoring
spec:
  selector:
    app: zipkin
  ports:
  - port: 9411
    targetPort: 9411
  type: ClusterIP
```

```bash
kubectl apply -f k8s/monitoring/zipkin-deployment.yaml

# Verify
kubectl port-forward svc/zipkin 9411:9411 -n monitoring
# Open: http://localhost:9411/zipkin/
```

---

## 5. Phase 3 — Day 3: Deploy All Business Services

**Go/No-Go check before starting:**
```bash
# Both must be Running
kubectl get pods -n petclinic

# Monitoring must be Running
kubectl get pods -n monitoring
```

### Step 3.1 — ExternalSecret for DB Credentials (apply once — shared by customers, vets, visits)

**`k8s/customers-service/external-secret.yaml`**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: petclinic-db-secret
  namespace: petclinic
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: petclinic-db-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: petclinic/db/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: petclinic/db/credentials
      property: password
  - secretKey: host
    remoteRef:
      key: petclinic/db/credentials
      property: host
  - secretKey: port
    remoteRef:
      key: petclinic/db/credentials
      property: port
```

```bash
kubectl apply -f k8s/customers-service/external-secret.yaml

# Verify sync succeeded
kubectl get externalsecret -n petclinic
# STATUS column must show: SecretSynced

kubectl get secret petclinic-db-secret -n petclinic
```

### Step 3.2 — Deploy customers-service (DE-6)

**`k8s/customers-service/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customers-service
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: customers-service
  template:
    metadata:
      labels:
        app: customers-service
    spec:
      containers:
      - name: customers-service
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-customers-service:1.0.0
        ports:
        - containerPort: 8081
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker,mysql"
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:mysql://$(DB_HOST):$(DB_PORT)/petclinic_customers?useSSL=false&allowPublicKeyRetrieval=true"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: petclinic-db-secret
              key: host
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: petclinic-db-secret
              key: port
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
        resources:
          requests:
            cpu: 256m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8081
          initialDelaySeconds: 90
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8081
          initialDelaySeconds: 60
          periodSeconds: 10
```

**`k8s/customers-service/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: customers-service
  namespace: petclinic
spec:
  selector:
    app: customers-service
  ports:
  - port: 8081
    targetPort: 8081
  type: ClusterIP
```

**`k8s/customers-service/hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: customers-service
  namespace: petclinic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: customers-service
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```bash
kubectl apply -f k8s/customers-service/

kubectl port-forward svc/customers-service 8081:8081 -n petclinic
curl http://localhost:8081/owners   # returns []
```

### Step 3.3 — Deploy vets-service (DE-6)

> Both `docker` and `production` profiles will be active. This is normal — `production` activates Spring Cache. You will see both in the startup logs.

**`k8s/vets-service/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vets-service
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vets-service
  template:
    metadata:
      labels:
        app: vets-service
    spec:
      containers:
      - name: vets-service
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-vets-service:1.0.0
        ports:
        - containerPort: 8083    # correct port — pom.xml EXPOSE of 8081 is wrong
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker,mysql"
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:mysql://$(DB_HOST):$(DB_PORT)/petclinic_vets?useSSL=false&allowPublicKeyRetrieval=true"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: petclinic-db-secret
              key: host
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: petclinic-db-secret
              key: port
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
        resources:
          requests:
            cpu: 256m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8083    # correct port
          initialDelaySeconds: 90
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8083    # correct port
          initialDelaySeconds: 60
          periodSeconds: 10
```

**`k8s/vets-service/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: vets-service
  namespace: petclinic
spec:
  selector:
    app: vets-service
  ports:
  - port: 8083
    targetPort: 8083
  type: ClusterIP
```

**`k8s/vets-service/hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vets-service
  namespace: petclinic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vets-service
  minReplicas: 2
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```bash
kubectl apply -f k8s/vets-service/

# Confirm both "docker" and "production" profiles appear — this is expected
kubectl logs deploy/vets-service -n petclinic | grep -i "profiles"

kubectl port-forward svc/vets-service 8083:8083 -n petclinic
curl http://localhost:8083/vets   # returns vet list
```

### Step 3.4 — Deploy visits-service (DE-7)

**`k8s/visits-service/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: visits-service
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: visits-service
  template:
    metadata:
      labels:
        app: visits-service
    spec:
      containers:
      - name: visits-service
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-visits-service:1.0.0
        ports:
        - containerPort: 8082    # correct port — pom.xml EXPOSE of 8081 is wrong
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker,mysql"
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:mysql://$(DB_HOST):$(DB_PORT)/petclinic_visits?useSSL=false&allowPublicKeyRetrieval=true"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: petclinic-db-secret
              key: host
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: petclinic-db-secret
              key: port
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
        resources:
          requests:
            cpu: 256m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8082
          initialDelaySeconds: 90
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8082
          initialDelaySeconds: 60
          periodSeconds: 10
```

**`k8s/visits-service/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: visits-service
  namespace: petclinic
spec:
  selector:
    app: visits-service
  ports:
  - port: 8082
    targetPort: 8082
  type: ClusterIP
```

**`k8s/visits-service/hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: visits-service
  namespace: petclinic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: visits-service
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```bash
kubectl apply -f k8s/visits-service/
```

### Step 3.5 — Deploy genai-service (DE-7)

> genai-service is **reactive (WebFlux/Netty)**. Logs say `Netty started on port 8084` — this is correct, not an error. The `production` profile is hardcoded in `application.yml` — do not try to suppress it.

**`k8s/genai-service/external-secret.yaml`**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: petclinic-openai-secret
  namespace: petclinic
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: petclinic-openai-secret
    creationPolicy: Owner
  data:
  - secretKey: api-key
    remoteRef:
      key: petclinic/openai/key
      property: api-key
```

**`k8s/genai-service/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-service
  namespace: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: genai-service
  template:
    metadata:
      labels:
        app: genai-service
    spec:
      containers:
      - name: genai-service
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-genai-service:1.0.0
        ports:
        - containerPort: 8084    # correct port — pom.xml EXPOSE of 8081 is wrong
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker"    # "production" is already hardcoded in application.yml
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: petclinic-openai-secret
              key: api-key
              optional: true    # service starts without a key; only chat calls fail
        resources:
          requests:
            cpu: 256m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8084    # correct port
          initialDelaySeconds: 60
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8084    # correct port
          initialDelaySeconds: 30
          periodSeconds: 10
```

**`k8s/genai-service/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: genai-service
  namespace: petclinic
spec:
  selector:
    app: genai-service
  ports:
  - port: 8084
    targetPort: 8084
  type: ClusterIP
```

```bash
kubectl apply -f k8s/genai-service/

# Expected log line — confirms WebFlux/Netty is correct
kubectl logs deploy/genai-service -n petclinic | grep -i "netty\|started on port"
```

### Step 3.6 — Deploy api-gateway (DE-8)

> api-gateway port is **8080**. The pom.xml EXPOSE says 8081 — that is wrong. All probes and service ports must use 8080.

**`k8s/api-gateway/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-api-gateway:1.0.0
        ports:
        - containerPort: 8080    # correct port — pom.xml EXPOSE of 8081 is wrong
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker"
        resources:
          requests:
            cpu: 256m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080    # correct port
          initialDelaySeconds: 90
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080    # correct port
          initialDelaySeconds: 60
          periodSeconds: 10
```

**`k8s/api-gateway/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: petclinic
spec:
  selector:
    app: api-gateway
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

**`k8s/api-gateway/ingress.yaml`**

Replace `<ACM_CERT_ARN>` with the ARN from DE-3's Terraform output.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway-ingress
  namespace: petclinic
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: "<ACM_CERT_ARN>"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health
    alb.ingress.kubernetes.io/healthcheck-port: "8080"
spec:
  rules:
  - host: petclinic.team.com   # replace with your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8080
```

**`k8s/api-gateway/hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
  namespace: petclinic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```bash
kubectl apply -f k8s/api-gateway/

# Watch for ALB creation — takes 2–5 minutes
kubectl get ingress -n petclinic -w

# Get ALB DNS name
ALB_DNS=$(kubectl get ingress api-gateway-ingress -n petclinic \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB DNS: ${ALB_DNS}"

# Test
curl http://${ALB_DNS}/actuator/health
```

### Step 3.7 — Deploy admin-server (DE-8)

**`k8s/admin-server/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-server
  namespace: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin-server
  template:
    metadata:
      labels:
        app: admin-server
    spec:
      containers:
      - name: admin-server
        image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/springcommunity/spring-petclinic-admin-server:1.0.0
        ports:
        - containerPort: 9090
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker"
        resources:
          requests:
            cpu: 256m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 9090
          initialDelaySeconds: 90
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 9090
          initialDelaySeconds: 60
          periodSeconds: 10
```

**`k8s/admin-server/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: admin-server
  namespace: petclinic
spec:
  selector:
    app: admin-server
  ports:
  - port: 9090
    targetPort: 9090
  type: ClusterIP
```

```bash
kubectl apply -f k8s/admin-server/

# Access admin UI
kubectl port-forward svc/admin-server 9090:9090 -n petclinic
# Open: http://localhost:9090 — all 8 services should appear as UP
```

### Step 3.8 — Configure Prometheus Scraping (DE-10)

The project's `docker/prometheus/prometheus.yml` scrapes api-gateway, customers, visits, and vets. In K8s, use a ServiceMonitor to cover all 8 services.

**`k8s/monitoring/service-monitors.yaml`**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: petclinic-apps
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
    - petclinic
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - api-gateway
      - customers-service
      - visits-service
      - vets-service
      - genai-service
      - admin-server
      - config-server
      - discovery-server
  endpoints:
  - path: /actuator/prometheus
    interval: 15s
```

```bash
kubectl apply -f k8s/monitoring/service-monitors.yaml

# Verify in Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Open: http://localhost:9090 → Status → Targets
# All 8 services should show State=UP
```

### Step 3.9 — Import Grafana Dashboard (DE-10)

The project ships a ready-made dashboard at `docker/grafana/dashboards/grafana-petclinic-dashboard.json`.

```bash
# Access Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# Open: http://localhost:3000
# Login: admin / petclinic-admin

# Import: Dashboards → Import → Upload JSON file
# Select: docker/grafana/dashboards/grafana-petclinic-dashboard.json
```

### Step 3.10 — Verify All 8 Services in Eureka

```bash
kubectl port-forward svc/discovery-server 8761:8761 -n petclinic
# Open: http://localhost:8761
# Confirm all 8 appear:
# CONFIG-SERVER, DISCOVERY-SERVER, CUSTOMERS-SERVICE,
# VISITS-SERVICE, VETS-SERVICE, GENAI-SERVICE, API-GATEWAY, ADMIN-SERVER
```

---

## 6. Phase 4 — Day 4: CI/CD Pipelines (DE-9)

### Step 4.1 — GitHub Actions Secrets

Configure these in your GitHub repository under Settings → Secrets → Actions:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION              → us-east-1
ECR_REGISTRY            → <account-id>.dkr.ecr.us-east-1.amazonaws.com
EKS_CLUSTER_NAME        → petclinic-eks-cluster
```

### Step 4.2 — CI Pipeline

**`.github/workflows/ci.yml`**
```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven

    - name: Build and test
      run: ./mvnw clean verify

    - name: Build Docker images (validation only — no push)
      run: ./mvnw clean install -P buildDocker -DskipTests
```

### Step 4.3 — CD Build + Push Pipeline

**`.github/workflows/cd-build-push.yml`**
```yaml
name: CD — Build and Push to ECR

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build all 8 images
      run: ./mvnw clean install -P buildDocker -DskipTests

    - name: Tag and push to ECR
      env:
        REPOSITORY_PREFIX: ${{ secrets.ECR_REGISTRY }}/springcommunity
        VERSION: ${{ github.sha }}
      run: |
        ./scripts/tagImages.sh
        ./scripts/pushImages.sh
        VERSION=latest ./scripts/tagImages.sh
        VERSION=latest ./scripts/pushImages.sh
```

### Step 4.4 — CD Deploy to EKS Pipeline

**`.github/workflows/cd-deploy-eks.yml`**
```yaml
name: CD — Deploy to EKS

on:
  workflow_run:
    workflows: ["CD — Build and Push to ECR"]
    types: [completed]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig \
          --name ${{ secrets.EKS_CLUSTER_NAME }} \
          --region us-east-1

    - name: Rolling deploy all services
      env:
        IMAGE_TAG: ${{ github.event.workflow_run.head_sha }}
        ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
      run: |
        SERVICES=(config-server discovery-server customers-service \
                  visits-service vets-service genai-service api-gateway admin-server)
        for SERVICE in "${SERVICES[@]}"; do
          kubectl set image deployment/${SERVICE} \
            ${SERVICE}=${ECR_REGISTRY}/springcommunity/spring-petclinic-${SERVICE}:${IMAGE_TAG} \
            -n petclinic
          kubectl rollout status deployment/${SERVICE} -n petclinic --timeout=5m
        done
```

### Step 4.5 — Rollback Pipeline

**`.github/workflows/rollback.yml`**
```yaml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      service:
        description: "Service name (e.g. customers-service)"
        required: true
      image_tag:
        description: "Image tag to roll back to (SHA or version)"
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig \
          --name ${{ secrets.EKS_CLUSTER_NAME }} \
          --region us-east-1

    - name: Roll back service
      env:
        ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
      run: |
        SERVICE=${{ github.event.inputs.service }}
        TAG=${{ github.event.inputs.image_tag }}
        kubectl set image deployment/${SERVICE} \
          ${SERVICE}=${ECR_REGISTRY}/springcommunity/spring-petclinic-${SERVICE}:${TAG} \
          -n petclinic
        kubectl rollout status deployment/${SERVICE} -n petclinic --timeout=3m
```

---

## 7. Phase 5 — Day 5: Hardening + Go-Live

### Step 5.1 — Pod Disruption Budgets

**`k8s/pdb.yaml`**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: customers-service-pdb
  namespace: petclinic
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: customers-service
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: visits-service-pdb
  namespace: petclinic
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: visits-service
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: vets-service-pdb
  namespace: petclinic
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: vets-service
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
  namespace: petclinic
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: api-gateway
```

```bash
kubectl apply -f k8s/pdb.yaml
kubectl get pdb -n petclinic
```

### Step 5.2 — Go-Live Health Checks

Run all of these before announcing go-live:

```bash
# 1. All pods Running, zero restarts
kubectl get pods -n petclinic -o wide
kubectl get pods -n monitoring -o wide

# 2. Eureka shows all 8 services registered
kubectl port-forward svc/discovery-server 8761:8761 -n petclinic &
# Open: http://localhost:8761
# Kill port-forward after check: kill %1

# 3. Config server is serving configs
kubectl exec -n petclinic deploy/config-server -- \
  curl -s http://localhost:8888/customers-service/docker | head -5

# 4. All 4 API routes work via ALB
ALB_DNS=$(kubectl get ingress api-gateway-ingress -n petclinic \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl https://${ALB_DNS}/api/customer/owners       # returns []
curl https://${ALB_DNS}/api/vet/vets               # returns vet list
curl https://${ALB_DNS}/api/visit/pets/1/visits    # returns []
curl https://${ALB_DNS}/actuator/health            # returns UP

# 5. Prometheus targets all UP
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring &
# Open: http://localhost:9090 → Status → Targets

# 6. Grafana dashboard showing live data
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring &
# Open: http://localhost:3000  (admin / petclinic-admin)

# 7. Zipkin receiving traces
kubectl port-forward svc/zipkin 9411:9411 -n monitoring &
# Open: http://localhost:9411/zipkin/
```

### Step 5.3 — Tag Final Images as v1.0.0

```bash
export REPOSITORY_PREFIX=<account-id>.dkr.ecr.us-east-1.amazonaws.com/springcommunity
export VERSION=v1.0.0
./scripts/tagImages.sh
./scripts/pushImages.sh
```

---

## 8. Quick Reference

### Corrected Port Map for K8s

```
Namespace: petclinic
  config-server:     ClusterIP :8888   (probes: :8888)
  discovery-server:  ClusterIP :8761   (probes: :8761)
  customers-service: ClusterIP :8081   (probes: :8081)
  visits-service:    ClusterIP :8082   (probes: :8082)  ← NOT 8081
  vets-service:      ClusterIP :8083   (probes: :8083)  ← NOT 8081
  genai-service:     ClusterIP :8084   (probes: :8084)  ← NOT 8081
  api-gateway:       ClusterIP :8080   (probes: :8080)  ← NOT 8081
  admin-server:      ClusterIP :9090   (probes: :9090)

Namespace: monitoring
  prometheus: ClusterIP :9090  → port-forward: kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
  grafana:    ClusterIP :80    → port-forward: kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
  zipkin:     ClusterIP :9411  → port-forward: kubectl port-forward svc/zipkin 9411:9411 -n monitoring
```

### Access URLs (K8s port-forward)

| Tool | Command | URL |
|---|---|---|
| Eureka | `kubectl port-forward svc/discovery-server 8761:8761 -n petclinic` | http://localhost:8761 |
| Admin Server | `kubectl port-forward svc/admin-server 9090:9090 -n petclinic` | http://localhost:9090 |
| Grafana | `kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring` | http://localhost:3000 |
| Prometheus | `kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring` | http://localhost:9090 |
| Zipkin | `kubectl port-forward svc/zipkin 9411:9411 -n monitoring` | http://localhost:9411/zipkin/ |

### Rollback (any service, under 3 minutes)

```bash
# Roll back to previous version
kubectl rollout undo deployment/<service-name> -n petclinic

# Roll back to a specific image tag
kubectl set image deployment/<service-name> \
  <service-name>=<ECR_URI>/springcommunity/spring-petclinic-<service-name>:<tag> \
  -n petclinic

# Watch progress
kubectl rollout status deployment/<service-name> -n petclinic
```

### Common kubectl Commands

```bash
# All pods status
kubectl get pods -n petclinic -o wide

# Watch pods in real-time
kubectl get pods -n petclinic -w

# Describe a failing pod
kubectl describe pod <pod-name> -n petclinic

# View logs
kubectl logs <pod-name> -n petclinic
kubectl logs <pod-name> -n petclinic --previous   # logs from crashed container

# Resource usage
kubectl top pods -n petclinic
kubectl top nodes

# HPA status
kubectl get hpa -n petclinic
```

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `CrashLoopBackOff` on any service | config-server not reachable | Check config-server first: `kubectl logs -n petclinic deploy/config-server` |
| Service not appearing in Eureka | discovery-server not ready | Check discovery-server logs. Ensure K8s Service name is exactly `discovery-server` |
| Probes failing on vets/visits/genai/api-gateway | Wrong port in probe config | Verify you used the Real Port column from Section 1, not the pom.xml EXPOSE value |
| `ErrImagePull` / `ImagePullBackOff` | ECR IAM policy missing on node role | `aws iam attach-role-policy --role-name <node-role> --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly` |
| ALB not being created | AWS Load Balancer Controller not running | `kubectl get pods -n kube-system \| grep aws-load-balancer` — reinstall if absent |
| `OOMKilled` | Memory limit too low | Increase `resources.limits.memory` in deployment.yaml and re-apply |
| DB connection refused | Wrong RDS host in secret | `kubectl get secret petclinic-db-secret -n petclinic -o jsonpath='{.data.host}' \| base64 -d` |
| ExternalSecret not syncing | ClusterSecretStore misconfigured | `kubectl describe externalsecret petclinic-db-secret -n petclinic` — check Events section |
| `Pending` pods | Insufficient cluster capacity | Cluster autoscaler adds nodes within ~2 min. Check: `kubectl describe pod <pending-pod>` |
| genai-service logs show Netty | Expected — this is WebFlux | Not an error. genai-service is reactive. Port 8084 is correct. |
| vets/genai logs show `production` profile | Expected — hardcoded in application.yml | Not an error. Both `docker` + `production` active simultaneously is correct. |

---

*This guide is built from the actual source code. Verified files: `docker/Dockerfile`, `docker-compose.yml`, `docker/prometheus/prometheus.yml`, `scripts/tagImages.sh`, `scripts/pushImages.sh`, all `pom.xml` files, all service `application.yml` files.*
