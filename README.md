# restapi — Spring Boot Microservice

A Spring Boot 3 REST API microservice with a fully automated CI/CD pipeline that builds a Docker image, pushes it to AWS ECR, and deploys it to AWS EKS via GitHub Actions.

---

## Table of Contents

- [Project Structure](#project-structure)
- [Local Development](#local-development)
- [Docker](#docker)
- [Kubernetes Manifests](#kubernetes-manifests)
- [CI/CD Pipeline](#cicd-pipeline)
- [AWS Prerequisites](#aws-prerequisites)
- [GitHub Secrets](#github-secrets)
- [API Endpoints](#api-endpoints)

---

## Project Structure

```
restapi/
├── .github/
│   └── workflows/
│       └── ci-cd.yml              # GitHub Actions CI/CD pipeline
├── k8s/
│   ├── deployment.yaml            # EKS Deployment — 2 replicas with health probes
│   ├── service.yaml               # ClusterIP Service on port 80 → 8080
│   └── ingress.yaml               # AWS ALB Ingress for /api routes
├── src/
│   ├── main/
│   │   ├── java/org/example/sap/
│   │   │   ├── Main.java                         # Spring Boot entry point
│   │   │   └── controller/
│   │   │       └── HealthController.java         # REST controllers
│   │   └── resources/
│   │       └── application.properties            # App configuration
│   └── test/
│       └── java/                                 # Test classes
├── Dockerfile                     # Multi-stage Docker build
├── pom.xml                        # Maven — Spring Boot 3.3.4, Java 21
└── README.md
```

---

## Local Development

### Prerequisites

- Java 21
- Maven 3.9+

### Run locally

```bash
mvn spring-boot:run
```

App starts at `http://localhost:8080`.

### Build JAR

```bash
mvn clean package
java -jar target/restapi-1.0-SNAPSHOT.jar
```

### Run tests

```bash
mvn clean verify
```

---

## Docker

### Build image locally

```bash
docker build -t restapi:local .
```

### Run container locally

```bash
docker run -p 8080:8080 restapi:local
```

### Dockerfile overview

The Dockerfile uses a two-stage build:

| Stage | Base image | Purpose |
|---|---|---|
| `builder` | `maven:3.9.6-eclipse-temurin-21` | Compile and package the JAR |
| runtime | `eclipse-temurin:21-jre-alpine` | Lean production image |

A non-root user (`appuser`) is created and used at runtime for security.

---

## Kubernetes Manifests

All manifests are in the `k8s/` directory and are applied during the GitHub Actions deploy job.

### deployment.yaml

- **Replicas:** 2
- **Image:** resolved at deploy time from ECR
- **Liveness probe:** `GET /actuator/health` — starts after 30 s, every 10 s
- **Readiness probe:** `GET /actuator/health` — starts after 15 s, every 5 s
- **Resource limits:** 500m CPU / 512 Mi memory per pod

### service.yaml

- Type: `ClusterIP`
- Exposes port 80, forwards to container port 8080

### ingress.yaml

- Uses the AWS Load Balancer Controller (`alb` ingress class)
- Scheme: `internet-facing`
- Routes all `/api` traffic to the service

> **Note:** The AWS Load Balancer Controller must be installed in your EKS cluster for the Ingress to provision an ALB.

---

## CI/CD Pipeline

Pipeline file: `.github/workflows/ci-cd.yml`

### Triggers

| Event | Branch | Jobs that run |
|---|---|---|
| Pull Request opened/updated | `main` | `build` only |
| Push (PR merged) | `main` | `build` → `docker-build-push` → `deploy` |

### Jobs

```
PR raised
    └── [build] mvn clean verify

PR merged to main
    └── [build] mvn clean verify
            └── [docker-build-push]
                    ├── docker build (tag: <sha8> + latest)
                    └── docker push → AWS ECR
                            └── [deploy]
                                    ├── aws eks update-kubeconfig
                                    ├── inject image tag into deployment.yaml
                                    ├── kubectl apply -f k8s/
                                    └── kubectl rollout status deployment/restapi
```

### Job: `build`

1. Checks out source
2. Sets up Java 21 (Temurin) with Maven cache
3. Runs `mvn clean verify`
4. Uploads the JAR as a build artifact

### Job: `docker-build-push`

1. Downloads the JAR artifact from the `build` job
2. Authenticates to AWS and logs in to ECR
3. Generates an image tag from the first 8 characters of `GITHUB_SHA`
4. Builds the Docker image using the multi-stage `Dockerfile`
5. Pushes both `:<sha8>` and `:latest` tags to ECR

### Job: `deploy`

1. Authenticates to AWS
2. Runs `aws eks update-kubeconfig` to configure `kubectl`
3. Uses `sed` to substitute `ECR_REGISTRY`, `ECR_REPOSITORY`, and `IMAGE_TAG` placeholders in `k8s/deployment.yaml`
4. Applies all manifests (`deployment.yaml`, `service.yaml`, `ingress.yaml`)
5. Waits for rollout to complete with a 120-second timeout
6. Prints pod and service status

---

## AWS Prerequisites

### 1. ECR Repository

Create the repository before the first pipeline run:

```bash
aws ecr create-repository \
  --repository-name restapi \
  --region <your-region>
```

The ECR registry URL will be in the format:
```
<account-id>.dkr.ecr.<region>.amazonaws.com
```

### 2. EKS Cluster

The cluster must already exist. To create one with `eksctl`:

```bash
eksctl create cluster \
  --name my-eks-cluster \
  --region <your-region> \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2
```

### 3. AWS Load Balancer Controller

Required for the Ingress to provision an AWS ALB. Follow the [official installation guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/).

### 4. IAM Permissions

The IAM user or role whose credentials are stored in GitHub Secrets must have:

| Permission | Policy |
|---|---|
| Push images to ECR | `AmazonEC2ContainerRegistryPowerUser` |
| Authenticate to EKS | `eks:DescribeCluster` + RBAC entry in the cluster |

To grant `kubectl` access, add the IAM identity to the `aws-auth` ConfigMap:

```bash
eksctl create iamidentitymapping \
  --cluster my-eks-cluster \
  --region <your-region> \
  --arn arn:aws:iam::<account-id>:user/<username> \
  --group system:masters
```

---

## GitHub Secrets

Go to **GitHub repo → Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret name | Description | Example |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | IAM access key ID | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | IAM secret access key | `wJalrXUtnFEMI/K7MDENG/...` |
| `AWS_REGION` | AWS region | `us-east-1` |
| `ECR_REGISTRY` | Full ECR registry hostname | `123456789012.dkr.ecr.us-east-1.amazonaws.com` |
| `ECR_REPOSITORY` | ECR repository name | `restapi` |
| `EKS_CLUSTER_NAME` | EKS cluster name | `my-eks-cluster` |

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/health` | Returns service health status |
| `GET` | `/api/hello` | Returns a hello message |
| `GET` | `/actuator/health` | Spring Boot Actuator health (used by K8s probes) |

### Sample responses

**GET /api/health**
```json
{
  "status": "UP",
  "service": "restapi"
}
```

**GET /api/hello**
```json
{
  "message": "Hello from restapi!"
}
```

**GET /actuator/health**
```json
{
  "status": "UP"
}
```
