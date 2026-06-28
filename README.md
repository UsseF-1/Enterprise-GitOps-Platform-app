# Enterprise GitOps Platform — App

> **Part of a 3-repository GitOps system.**
> This repo holds the application source code and the CI pipeline.
> The other two repos handle infrastructure and Helm/deployment.
>
> | Repo | Purpose |
> |------|---------|
> | **`Enterprise-GitOps-Platform-app`** ← you are here | Java source, Dockerfiles, CI pipeline |
> | [`Enterprise-GitOps-Platform-infra`](https://github.com/UsseF-1/Enterprise-GitOps-Platform-infra) | Terraform — EKS, ECR, VPC, SonarQube |
> | [`Enterprise-GitOps-Platform-helm`](https://github.com/UsseF-1/Enterprise-GitOps-Platform-helm) | Helm chart, ArgoCD manifests |

---

## What This Repo Does

Every push to `main` triggers a CI/CD pipeline that:

1. Builds the Java WAR with Maven
2. Packages it inside a Docker image (multi-stage build → Tomcat 10)
3. Runs a **Trivy** vulnerability scan — blocks the push on CRITICAL/HIGH findings
4. Pushes the image to **AWS ECR** tagged with the short commit SHA
5. Clones the Helm repo and updates `values.yaml` with the new image tag
6. ArgoCD picks up the change and deploys to EKS automatically

Every pull request to `main` triggers a separate quality gate:

1. Maven build + unit tests + Checkstyle report
2. SonarQube scan
3. SonarQube quality gate check (blocks merge on failure)

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── ci.yml              # Full CI/CD pipeline
├── docker/
│   ├── app/
│   │   └── Dockerfile          # Multi-stage: Maven build → Tomcat 10
│   ├── db/
│   │   └── Dockerfile          # MySQL 8 + schema seed
│   └── web/
│       ├── Dockerfile          # Nginx reverse proxy
│       └── nginvproapp.conf    # Nginx upstream config → Tomcat:8080
├── src/
│   └── main/
│       ├── java/com/visualpathit/account/   # Spring MVC source
│       └── resources/
│           ├── application.properties       # Runtime config (DB, MQ, Cache)
│           └── accountsdb.sql              # DB schema
├── sonar-project.properties    # SonarQube project config
└── pom.xml                     # Maven build
```

---

## Application Stack

The app is a Spring MVC web application with the following runtime dependencies — all resolved as Kubernetes Services when deployed:

| Service | Kubernetes DNS | Port |
|---------|---------------|------|
| MySQL database | `vprodb` | 3306 |
| Memcached | `vprocache01` | 11211 |
| RabbitMQ | `vpromq01` | 5672 |

> These hostnames are hardcoded in `src/main/resources/application.properties` and match the Helm chart service names exactly.

---

## CI/CD Pipeline

### Triggers

| Event | Job(s) triggered |
|-------|-----------------|
| Pull Request → `main` | `build-and-sonar` |
| Push → `main` | `docker-build-push` → `update-helm` |

### Pipeline Flow

```
PR → main                          Push → main
─────────────────                  ──────────────────────────────────────
Checkout                           Checkout
  │                                  │
Maven build + tests                Build Docker image
  │                                  │
Checkstyle report                  Trivy scan (CRITICAL/HIGH → fail)
  │                                  │
SonarQube scan                     Push to ECR  (:SHA + :latest)
  │                                  │
Quality gate check                 Clone Helm repo
                                     │
                                   yq → update values.yaml (image + tag)
                                     │
                                   Commit & push to Helm repo
                                     │
                                   ArgoCD detects change → deploys to EKS
```

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM user with ECR push permissions |
| `AWS_SECRET_ACCESS_KEY` | Corresponding secret key |
| `SONAR_TOKEN` | SonarQube authentication token |
| `GITOPS_PAT` | GitHub PAT with write access to the Helm repo |
| `HELM_REPO_USER` | GitHub username owning the Helm repo |

### Required GitHub Variables

| Variable | Description |
|----------|-------------|
| `AWS_REGION` | e.g. `us-east-1` |
| `ECR_REPOSITORY` | ECR repository name e.g. `vprofile_app_image` |
| `HELM_REPO_NAME` | Helm repo name e.g. `Enterprise-GitOps-Platform-helm` |
| `SONAR_HOST_URL` | SonarQube server URL |

---

## Dockerfiles

### `docker/app/Dockerfile` — Application Image

Multi-stage build to keep the final image lean:

```
Stage 1 (BUILD_IMAGE):  maven:3.9.9-eclipse-temurin-21-jammy
  └── mvn install -DskipTests → target/vprofile-v2.war

Stage 2 (RUNTIME):      tomcat:10-jdk21
  └── Copies WAR → /usr/local/tomcat/webapps/ROOT.war
  └── EXPOSE 8080
```

### `docker/db/Dockerfile` — Database Image

```
Base:  mysql:8.0.33
Seeds: db_backup.sql → /docker-entrypoint-initdb.d/
```

> **Note:** This Dockerfile is kept for local/docker-compose use.
> On Kubernetes the DB image is pulled from `vprocontainers/vprofiledb` as defined in Helm values.

### `docker/web/Dockerfile` — Nginx Reverse Proxy

```
Base:  nginx:latest
Conf:  nginvproapp.conf → proxies :80 to upstream vproapp:8080
```

> Used for local docker-compose setups. On Kubernetes, the ALB Ingress handles external traffic directly to the app service.

---

## Local Development

### Prerequisites

- Java 21
- Maven 3.9+
- Docker

### Build the WAR

```bash
mvn clean package -DskipTests
```

### Build Docker image locally

```bash
docker build -f docker/app/Dockerfile -t vprofileapp:local .
```

### Run tests

```bash
mvn test
```

### Run Checkstyle

```bash
mvn checkstyle:checkstyle
```

---

## SonarQube Integration

Configuration lives in `sonar-project.properties`. The pipeline uses the `sonarsource/sonarqube-scan-action` action and blocks the PR merge if the quality gate fails.

To run locally against a SonarQube instance:

```bash
mvn sonar:sonar \
  -Dsonar.host.url=http://<SONARQUBE_IP>:9000 \
  -Dsonar.login=<SONAR_TOKEN>
```

> The SonarQube server is provisioned by Terraform in the `infra` repo under `sonarqube-EC2/`.

---

## Deployment

This repo does **not** deploy directly. After a push to `main`:

1. The pipeline updates the image tag in the Helm repo (`Enterprise-GitOps-Platform-helm`)
2. ArgoCD syncs the Helm chart to EKS automatically

To follow the full deployment flow, see:
- [`Enterprise-GitOps-Platform-infra`](https://github.com/UsseF-1/Enterprise-GitOps-Platform-infra) — for cluster setup
- [`Enterprise-GitOps-Platform-helm`](https://github.com/UsseF-1/Enterprise-GitOps-Platform-helm) — for Helm chart and ArgoCD config
