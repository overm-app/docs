# OVERM — OverEngineerMenu

> A deliberately overengineered microservices application built to learn distributed systems, cloud infrastructure, and DevOps practices end to end.

**Author:** Mario Irigoyen — [@Tar-Mairon24](https://github.com/Tar-Mairon24)  
**Status:** Active development — Phase 1  
**Stack:** Go · Python · Angular · Kubernetes · Kafka · MongoDB · PostgreSQL · Novu · AWS EKS · Terraform · ArgoCD · OpenTelemetry · Tempo

---

## What is this?

OVERM is a recipe and weekly menu planning app. You upload recipes, it calculates calories and macros, and generates a meal plan for the week. You can also upload a photo of a handwritten recipe and it transcribes it automatically.

The app is simple by design. The point is the process — microservices, event-driven async flows, proper service contracts, GitOps deployments, IaC, and production-grade observability. A practical exercise in planning, developing, and shipping a microservice architecture using DevOps and GitOps principles.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repositories](#repositories)
- [Services](#services)
- [Data Layer](#data-layer)
- [Async Messaging — Kafka Topics](#async-messaging--kafka-topics)
- [Service Contracts](#service-contracts)
- [Git Flow & Branching Strategy](#git-flow--branching-strategy)
- [CI/CD Pipeline](#cicd-pipeline)
- [Configuration Management](#configuration-management)
- [Infrastructure & IaC](#infrastructure--iac)
- [Observability](#observability)
- [Roadmap](#roadmap)
- [Key Design Decisions](#key-design-decisions)

---

## Architecture Overview

```
Clients (Web · Mobile)
        │
   CloudFront (CDN)
        │
   ALB Ingress Controller
        │
┌───────────────────────────────────────────────────────┐
│  Amazon EKS — namespace: overm                        │
│                                                       │
│  api-auth            api-recipe-catalog               │
│  api-menu-shuffle    api-nutrition                    │
│  api-image-processing    api-notifications            │
│  web-frontend                                         │
│                                                       │
│  ┌──────────────────────────────────────────────────┐ │
│  │  Novu (self-hosted · Helm)                       │ │
│  │  Templates · Retries · SendGrid · FCM · APNs     │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  ┌──────────────────────────────────────────────────┐ │
│  │  ArgoCD                                          │ │
│  │  One Application per service per environment     │ │
│  │  Syncs from overm-app/manifests                  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │  namespace: monitoring                          │  │
│  │  Prometheus · Grafana · Loki · Promtail · Tempo │  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
        │
   Data Layer
   PostgreSQL (users) · MongoDB (recipes · menus)
   AWS Secrets Manager · S3
```

Full architecture diagram: [`docs/diagrams/overm-architecture-v3.drawio`](./diagrams/overm-architecture-v3.drawio)

Git flow diagram: [`docs/diagrams/overm-gitflow.drawio`](./diagrams/overm-gitflow.drawio)

---

## Repositories

All repositories live under the [overm-app](https://github.com/overm-app) GitHub organization.

| Repository | Description | Language | Written by |
|---|---|---|---|
| **[docs](https://github.com/overm-app/docs)** | Central documentation — you are here | Markdown | Humans |
| **[contracts](https://github.com/overm-app/contracts)** | OpenAPI specs + Kafka schemas + Swagger UI | YAML | Humans |
| **[pipelines](https://github.com/overm-app/pipelines)** | Reusable CI workflows, K8s manifest templates, per-service YAML configs, render script, ArgoCD Application CRDs | YAML · Python | Humans |
| **[manifests](https://github.com/overm-app/manifests)** | Generated K8s manifests per service per environment — never edited by hand | YAML | CI pipeline only |
| **[IaC](https://github.com/overm-app/IaC)** | Terraform — AWS infrastructure | HCL | Humans |
| **[monitoring](https://github.com/overm-app/monitoring)** | Helm values, Grafana dashboards, ServiceMonitors | YAML | Humans |
| **[api-auth](https://github.com/overm-app/api-auth)** | User auth, JWT, Google OAuth | Go | Humans |
| **[api-recipe-catalog](https://github.com/overm-app/api-recipe-catalog)** | Recipe CRUD, image upload trigger | Go | Humans |
| **[api-menu-shuffle](https://github.com/overm-app/api-menu-shuffle)** | Menu generation and management | Go | Humans |
| **[api-nutrition](https://github.com/overm-app/api-nutrition)** | Macro calculator, external API integration | Go | Humans |
| **[api-image-processing](https://github.com/overm-app/api-image-processing)** | OCR, handwriting transcription | Python | Humans |
| **[api-notifications](https://github.com/overm-app/api-notifications)** | Kafka consumer → Novu API bridge | Go | Humans |
| **[web-frontend](https://github.com/overm-app/web-frontend)** | Angular app, served by nginx inside K8s | Angular | Humans |

**Rule:** `manifests` is owned by the CI pipeline. Every manifest in it is generated by `render.py` from the templates and configs in `pipelines`. Manual edits will be overwritten on the next deploy and will cause an ArgoCD drift alert.

---

## Services

### api-auth
Handles all authentication. Issues JWTs consumed and validated locally by every other service — no network call per request. Supports email/password and Google OAuth.

- **Stack:** Go, Gin, PostgreSQL
- **Endpoints:** `POST /auth/v1/login` · `POST /auth/v1/register` · `POST /auth/v1/refresh` · `POST /auth/v1/logout` · `GET /auth/v1/google` · `GET /auth/v1/google/callback` · `GET /users/v1/me`
- **Token delivery:** Bearer JWT for mobile/API clients · HttpOnly encrypted cookie for web browser clients (CSRF protected)
- **Contract:** [`contracts/user-auth-api.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/user-auth-api.yaml) · [`contracts/user-auth-web.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/user-auth-web.yaml)

### api-recipe-catalog
Manages the user's recipe library. Handles manual recipe creation and triggers async image processing via Kafka. Calls api-nutrition synchronously during recipe creation so macros are available immediately.

- **Stack:** Go, Gin, MongoDB
- **Endpoints:** `GET /recipe-catalog/v1/recipes` · `POST /recipe-catalog/v1/recipes` · `GET /recipe-catalog/v1/recipes/:id` · `PATCH /recipe-catalog/v1/recipes/:id` · `DELETE /recipe-catalog/v1/recipes/:id` · `POST /recipe-catalog/v1/upload` · `GET /recipe-catalog/v1/upload/:upload_id`
- **Produces:** `recipe.image.uploaded` · `recipe.created`
- **Contract:** [`contracts/recipe-catalog.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/recipe-catalog.yaml)

### api-menu-shuffle
Owns all menu logic. Generates weekly/monthly meal plans from the user's recipe catalog. MVP uses a random shuffle algorithm. Future versions will support pinned recipes, calorie targets, prep time filters, and dietary constraints.

- **Stack:** Go, Gin, MongoDB
- **Endpoints:** `POST /menu/v1/generate` · `GET /menu/v1/active` · `PATCH /menu/v1/:id/confirm` · `GET /menu/v1/history`
- **Produces:** `menu.activated`
- **Contract:** [`contracts/menu-shuffle.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/menu-shuffle.yaml)

**Menu generation request shape:**
```json
{
  "time_range": "week",
  "meals_per_day": 3,
  "strategy": "shuffle",
  "pinned_recipes": [],
  "targets": { "calories_per_day": null },
  "filters": { "max_prep_minutes": null, "tags_excluded": [] }
}
```
The `strategy` field selects the algorithm. `pinned_recipes`, `targets`, and `filters` are null in MVP and ignored by the shuffle strategy — no schema changes needed when constrained generation is added.

### api-nutrition
Calculates macros and calories for a given list of ingredients. Called synchronously by api-recipe-catalog during recipe creation and by api-image-processing after transcription. Looks up unknown ingredients against the USDA FoodData or Edamam external API and caches results.

- **Stack:** Go, Gin
- **Endpoints:** `POST /nutrition/v1/calculate`
- **Contract:** [`contracts/nutrition.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/nutrition.yaml)

### api-image-processing
Consumes `recipe.image.uploaded` from Kafka. Sends the image to an AI model for OCR and handwriting transcription, then calls api-nutrition to calculate macros, saves the completed recipe to MongoDB, and publishes `recipe.transcription.completed`. The AI backend is swappable — starts with an external API (OpenAI Vision / Google Vision), future goal is a self-trained model for specific handwriting recognition.

- **Stack:** Python
- **Consumes:** `recipe.image.uploaded`
- **Produces:** `recipe.transcription.completed`
- **Contract:** [`contracts/kafka-contracts.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/kafka-contracts.yaml)

### api-notifications
A minimal Kafka consumer that reads from the `notification.send` topic and calls Novu's HTTP API. It owns no delivery logic itself. All templating, retries, provider switching, and delivery tracking are handled by Novu.

- **Stack:** Go
- **Consumes:** `notification.send`
- **Calls:** Novu HTTP API (`POST /v1/events/trigger`)
- **Contract:** [`contracts/kafka-contracts.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/kafka-contracts.yaml)

### Novu (self-hosted)
Open source notification platform deployed via Helm chart inside EKS. Handles email, push, and SMS delivery with a provider abstraction layer — swap SendGrid for Mailgun without touching any service code. Alertmanager also routes infrastructure alerts through Novu so there is one single place for all notification templates and delivery logs.

- **Deployment:** `helm install novu novu/novu --namespace overm`
- **Providers:** SendGrid / Mailgun (email) · FCM / APNs (push) · Twilio (SMS, future)
- **Dashboard:** Internal Novu UI for managing templates and viewing delivery status

### web-frontend
Web application served by nginx as a container inside Kubernetes. Communicates with backend services via HttpOnly cookies — JWTs are never accessible to JavaScript. CSRF protected.

- **Stack:** Angular, nginx
- **Auth pattern:** HttpOnly cookie (not localStorage — XSS protection)

---

## Data Layer

| Store | Used by | Contents |
|---|---|---|
| **PostgreSQL** | api-auth | `users` table · `refresh_tokens` table |
| **MongoDB** | api-recipe-catalog | `recipes` collection (with macros, status: active/archived) |
| **MongoDB** | api-menu-shuffle | `menus` collection (status: draft/active/archived) |
| **S3** | api-image-processing | Uploaded recipe images |

**Why two databases:**  
PostgreSQL for users because auth data is relational and needs strong consistency — users reference refresh tokens, OAuth providers, and future roles.  
MongoDB for recipes and menus because the data is naturally document-shaped — ingredients, steps, and macros are nested and variable in structure.

**Soft deletes everywhere:**  
Recipes are never deleted, only set to `status: archived`. This preserves history for future statistics features. Menus follow the same pattern — generating a new menu archives the previous one rather than deleting it.

---

## Async Messaging — Kafka Topics

Only flows where the user does not need to block are async. Everything else is synchronous HTTP.

| Topic | Producer | Consumer | Trigger |
|---|---|---|---|
| `recipe.image.uploaded` | api-recipe-catalog | api-image-processing | User uploads a photo |
| `recipe.transcription.completed` | api-image-processing | api-recipe-catalog · api-notifications | Transcription finished |
| `recipe.created` | api-recipe-catalog | (future: stats, recommendations) | Any recipe created |
| `menu.activated` | api-menu-shuffle | api-notifications | User confirms a menu |
| `notification.send` | any service | api-notifications | Generic notification request |

**Design rule:**  
If the user is waiting for a response → HTTP.  
If it's a background job the user doesn't block on → Kafka.

Full message schemas: [`contracts/kafka-contracts.yaml`](https://github.com/overm-app/contracts/blob/main/contracts/kafka-contracts.yaml)

---

## Service Contracts

All service contracts live in [`contracts`](https://github.com/overm-app/contracts). They are written before any code is written for a service.

| Contract file | Description |
|---|---|
| `user-auth-api.yaml` | Auth service — mobile and API clients (Bearer JWT) |
| `user-auth-web.yaml` | Auth service — browser clients (HttpOnly cookie) |
| `recipe-catalog.yaml` | Recipe catalog CRUD and upload |
| `menu-shuffle.yaml` | Menu generation and management |
| `nutrition.yaml` | Macro calculation |
| `kafka-contracts.yaml` | All Kafka message schemas |

**Swagger UI** mounts all contracts and runs locally:

```bash
git clone https://github.com/overm-app/contracts
cd contracts
docker compose up
# open http://localhost:8090
```

**Standard error shape across all services:**
```json
{
  "code": "RECIPE_NOT_FOUND",
  "message": "No recipe with id 'xyz' was found",
  "details": {}
}
```

**JWT claims shape** (shared across all services for local validation):
```json
{
  "sub":   "user-uuid",
  "email": "user@example.com",
  "name":  "Mario Irigoyen",
  "iat":   1730000000,
  "exp":   1730086400
}
```

---

## Git Flow & Branching Strategy

Full diagram: [`docs/diagrams/overm-gitflow.drawio`](./diagrams/overm-gitflow.drawio)

```
feature/issue-NNN  →  develop  →  QA  →  main
                                          ↑
                              hotfix/issue-NNN (from main, merges back to main AND develop)
```

| Branch | Environment | Trigger |
|---|---|---|
| `feature/issue-NNN-short-description` | Local docker-compose | Created per GitHub Issue |
| `develop` | Local docker-compose | PR from feature branch |
| `QA` | Minikube | PR from develop |
| `main` | EKS (AWS) | PR from QA |
| `hotfix/issue-NNN` | Local → EKS | Production bug, branches from main |

**Rules:**
- One feature branch per GitHub Issue — branch name must reference the issue number
- Hotfix branches merge back to **both** `main` and `develop` — skipping this causes drift
- Every PR uses the template in `.github/pull_request_template.md`
- Hotfixes skip the QA branch — they go directly from hotfix → main

**Commit message format:**
```
feat: add login endpoint #001
fix: handle empty ingredient list #042
chore: update Go version to 1.22
```

---

## CI/CD Pipeline

CI and CD are split. GitHub Actions owns the build. ArgoCD owns the deploy. They meet at the `manifests` repo — GitHub Actions commits rendered manifests there, ArgoCD syncs them to the cluster.

### GitHub Actions — CI (per service repo)

Each service repo contains a single workflow file that calls the reusable workflow in `pipelines`. The service repo has no knowledge of Kubernetes, templates, or deployment.

```yaml
# Each service repo — entire CI file
jobs:
  pipeline:
    uses: overm-app/pipelines/.github/workflows/go-service.yml@main
    with:
      service_name: api-auth
      go_version: "1.22"
      environment: qa        # or prod
    secrets: inherit
```

**Pipeline stages:**

```
push to branch
     │
     ├── test (go test ./... + coverage threshold 80%)
     ├── lint (golangci-lint)
     ├── vulnerability scan (govulncheck)
     ├── build and push image → Docker Hub
     │     tags: sha-<commit>  ·  <semver on release tag>
     │
     └── render and commit manifests
           ├── checkout pipelines repo
           ├── run scripts/render.py
           │     reads: templates/go-deployment.yaml (and service, configmap, hpa...)
           │     reads: configs/<service>/<environment>.yaml
           │     injects: GitHub org + repo variables
           │     writes: rendered manifests to overm-app/manifests
           │
           ├── commit to overm-app/manifests
           │
           └── (QA only) apply K8s Secret via kubectl
                 reads values from GitHub Secrets
                 never written to disk or committed to Git
```

**Image tagging:**
```
overmapp/api-auth:sha-a1b2c3d   ← git SHA (immutable, used for rollbacks)
overmapp/api-auth:1.0.0         ← semver on release tag
```

**GitHub Variables:**

| Scope | Variable | Example |
|---|---|---|
| Org | `DOCKERHUB_ORG` | `overmapp` |
| Org | `K8S_NAMESPACE` | `overm` |
| Repo | `SERVICE_NAME` | `api-auth` |
| Repo | `SERVICE_PORT` | `8081` |
| Repo | `SERVICE_TYPE` | `go` |
| Repo | `REPLICAS_QA` | `1` |
| Repo | `REPLICAS_PROD` | `2` |

**GitHub Secrets** (never in manifests or config files):

| Scope | Secret | Used in |
|---|---|---|
| Repo | `MONGO_URI_QA` | Applied to Minikube via kubectl, not committed |
| Repo | `JWT_SECRET_QA` | Applied to Minikube via kubectl, not committed |
| Org | AWS OIDC credentials | EKS deploys — no static credentials |

### ArgoCD — CD

ArgoCD runs inside the cluster and watches `overm-app/manifests`. One Application per service per environment. When the CI pipeline commits new rendered manifests, ArgoCD detects the change and syncs automatically.

```
overm-app/manifests (commit by CI)
        │
   ArgoCD detects diff
        │
   ├── api-auth-qa      → syncs manifests/qa/api-auth/      → Minikube
   ├── api-auth-prod    → syncs manifests/prod/api-auth/     → EKS
   └── ...one app per service per env
```

**ArgoCD Application shape:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-auth-qa
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/overm-app/manifests
    targetRevision: main
    path: qa/api-auth
  destination:
    server: https://kubernetes.default.svc
    namespace: overm
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ArgoCD Application CRDs live in `pipelines/argocd/qa/` and `pipelines/argocd/prod/` — written once by hand, unchanged unless a new service is added.

**Rollback:** revert the relevant commit in `overm-app/manifests`. ArgoCD detects the revert and syncs back to the previous state. No pipeline rerun needed.

**Cluster uptime:** Neither Minikube nor EKS runs 24/7. ArgoCD lives inside the cluster — when the cluster is down, ArgoCD is down with it. PRs can be merged and manifests committed while the environment is offline. When the cluster comes back up, ArgoCD reconciles to the current HEAD of `manifests` in a single sync, regardless of how many commits accumulated while it was offline.

> **Note on sync windows:** ArgoCD supports sync windows to restrict deployments to specific time ranges — useful when a dedicated QA team needs to control when new versions land in their environment. OVERM does not use them, but the feature is relevant in any team context where environment stability windows matter.

**Secrets in QA vs prod:**

| | QA (Minikube) | prod (EKS) |
|---|---|---|
| Secret source | GitHub Secrets | AWS Secrets Manager |
| Secret delivery | `kubectl create secret` in CI pipeline | External Secrets Operator |
| Goes through ArgoCD | No — never committed | No — managed by ESO |
| Go service reads | `os.Getenv(...)` | `os.Getenv(...)` |

The Go service always reads secrets from environment variables. It does not know or care whether they were injected by the pipeline or by ESO. Org-level GitHub secrets are shared across all repos in `overm-app`. AWS credentials use OIDC — no static keys stored anywhere.

---

## Configuration Management

Application config is separated from secrets and from infrastructure. Each service has one YAML config file per environment, stored in `pipelines/configs/`. These are the only files to edit when changing application behavior — they go through a PR and the pipeline generates the ConfigMap automatically.

### pipelines repo structure

```
pipelines/
├── .github/workflows/
│   ├── go-service.yml          ← reusable CI + render workflow
│   └── python-service.yml
│
├── templates/                  ← K8s manifest templates, shared across all services
│   ├── go-deployment.yaml
│   ├── python-deployment.yaml
│   ├── configmap.yaml
│   ├── service.yaml
│   ├── backend-config.yaml
│   ├── hpa.yaml
│   └── replicaset.yaml
│
├── configs/                    ← one folder per service, one file per env
│   ├── api-auth/
│   │   ├── qa.yaml
│   │   └── prod.yaml
│   ├── api-recipe-catalog/
│   │   ├── qa.yaml
│   │   └── prod.yaml
│   └── ...
│
├── argocd/
│   ├── qa/
│   │   ├── api-auth.yaml
│   │   └── api-recipe-catalog.yaml
│   └── prod/
│       ├── api-auth.yaml
│       └── api-recipe-catalog.yaml
│
└── scripts/
    └── render.py
```

### Config file format

Config files are YAML, mounted as `/etc/config/config.yaml` inside the pod. They contain only non-secret values. Secrets are injected as environment variables by the pipeline (QA) or ESO (prod) and read via `os.Getenv`.

```yaml
# configs/api-recipe-catalog/prod.yaml
server:
  port: 8082
  timezone: America/Monterrey

log:
  level: info

env: prod

mongo:
  db_name: overm_recipes
  pool:
    max: 25
    min: 5
```

QA and prod files are structurally identical — only values differ. This means the config loader has one code path and diffs between environments are always explicit.

### What goes where

| Value type | Where it lives | How it reaches the pod |
|---|---|---|
| App config (ports, pool sizes, log level) | `pipelines/configs/<service>/<env>.yaml` | Mounted as ConfigMap at `/etc/config/config.yaml` |
| Infrastructure variables (image tag, replicas, namespace) | GitHub Variables (org or repo level) | Injected into manifest templates by `render.py` at pipeline time |
| Secrets (DB URIs, JWT secret) | GitHub Secrets (QA) · AWS Secrets Manager (prod) | Env vars injected by pipeline kubectl (QA) or ESO (prod) |

### render.py

A Python script in `pipelines/scripts/render.py`. Called by the reusable workflow after the image is pushed. It reads the manifest templates, substitutes `${VARIABLE}` placeholders with GitHub Variables, wraps the service config YAML into a ConfigMap manifest, and writes the final manifests to `overm-app/manifests`. It fails loudly on any unreplaced placeholder, preventing silent misconfiguration.

### manifests repo

`overm-app/manifests` is generated output. Never edit it by hand.

```
manifests/
├── qa/
│   ├── api-auth/
│   │   ├── deployment.yaml
│   │   ├── configmap.yaml
│   │   ├── service.yaml
│   │   ├── backend-config.yaml
│   │   ├── hpa.yaml
│   │   └── replicaset.yaml
│   └── api-recipe-catalog/
│       └── ...
└── prod/
    └── ...
```

> **Tradeoff note:** In a production environment, generated manifests would be stored in an artifact registry (Nexus, OCI) rather than Git — this avoids polluting commit history and decouples rollback from Git operations. For OVERM, Git storage is a documented tradeoff: the setup stays simple, history is auditable, and rollback is one `git revert` away. The separation between `pipelines` (source) and `manifests` (generated) keeps meaningful commit history clean.

---

## Infrastructure & IaC

All cloud infrastructure is defined in Terraform inside [`IaC`](https://github.com/overm-app/IaC).

**AWS resources managed by Terraform:**
- EKS cluster and node groups
- RDS PostgreSQL
- DocumentDB
- S3 buckets
- CloudFront distribution
- ALB and target groups
- IAM roles (including GitHub Actions OIDC role)
- AWS Secrets Manager secrets
- VPC, subnets, security groups

**Cost note:** EKS is not always running. The cluster is spun up on demand and torn down when not in use. `terraform destroy` is one command.

---

## Observability

**Stack:** Prometheus + Loki + Tempo + Grafana deployed via Helm into the `monitoring` namespace. The three observability pillars — metrics, logs, and traces — are all queryable from Grafana, with Loki and Tempo sharing `trace_id` so a trace can be correlated directly to the logs for that request.

Every service exposes two endpoints required before a PR is merged:
- `GET /metrics` — Prometheus scrape endpoint (no auth)
- `GET /healthz` — Kubernetes liveness probe (no auth)

**Metrics:** Prometheus scrapes every service every 15 seconds via ServiceMonitor CRDs.
- Go runtime metrics (automatic — zero code)
- Custom business metrics: `overm_recipes_created_total`, `overm_nutrition_calc_duration_seconds`, Kafka consumer lag
- Kubernetes cluster metrics (kube-prometheus-stack, out of the box)

**Logs:** Structured JSON via `go.uber.org/zap`. Every log line includes a `trace_id` field. Promtail DaemonSet collects from all pods automatically. Queryable in Grafana via LogQL.

**Traces:** Distributed tracing via OpenTelemetry SDK (`otel-go`), collected by Tempo.

Every Go service initializes a tracer in `main.go` that exports spans to Tempo via OTLP. Instrumentation layers:

- HTTP handlers — automatic via `otelgin` middleware (one span per request, zero code per handler)
- MongoDB calls — manual child spans wrapping repository operations
- Outbound HTTP calls — manual child spans for calls to api-nutrition and external APIs
- Kafka producer — trace context injected into message headers on publish
- Kafka consumer — trace context extracted from message headers on consume, child span created

The Kafka instrumentation is what connects traces across service boundaries. A trace started in api-recipe-catalog when a user uploads an image continues in api-image-processing when the message is consumed — producing one end-to-end trace showing the full upload → transcribe → notify flow.

Telemetry config is in the service YAML config file, not hardcoded:

```yaml
telemetry:
  endpoint: http://tempo.monitoring.svc.cluster.local:4317
  service_name: api-recipe-catalog
```

**Alertmanager** fires alerts into Novu — reusing the same notification platform as the application. One place for all delivery templates, whether triggered by a user action or an infrastructure alert.

---

### Port conventions

| Service | Port |
|---|---|
| api-auth | 8081 |
| api-recipe-catalog | 8082 |
| api-menu-shuffle | 8083 |
| api-nutrition | 8084 |
| api-image-processing | 8085 |
| api-notifications | 8086 |
| web-frontend | 3000 |
| Novu API | 3000 (internal) |
| Novu Dashboard | 4200 (internal) |
| contracts (Swagger UI) | 8090 |
| Grafana | 3001 |
| Prometheus | 9090 |
| Tempo | 3200 (internal) |

---

## Roadmap

### Phase 1 — Core services + observability foundation (Feb → May 2025)
- [x] Architecture design and service contracts
- [x] Git flow and CI/CD pipeline design
- [x] GitHub org, branch protection, org secrets
- [x] `api-auth` — JWT, Google OAuth, PostgreSQL
- [x] `api-recipe-catalog` — CRUD, sync nutrition call
- [ ] `api-menu-shuffle` — MVP shuffle algorithm
- [ ] Basic frontend — login, recipe list, generate menu
- [ ] `scripts/render.py` — template injection script with validation
- [ ] ArgoCD installed in Minikube — one Application CRD per service
- [ ] `pipelines` repo configured for Phase 1 services to Minikube (QA)
- [ ] All Phase 1 services running in Minikube end to end
- [ ] Prometheus + Loki + Tempo + Grafana deployed via Helm in Minikube (`monitoring` namespace)
- [ ] `monitoring` repo — `values-qa.yaml` per tool (Prometheus, Loki, Tempo, Grafana)
- [ ] otel-go tracer initialized in api-auth, api-recipe-catalog, api-menu-shuffle
- [ ] HTTP instrumentation via `otelgin` middleware — one span per request, zero code per handler
- [ ] MongoDB call instrumentation — manual child spans in repository layer
- [ ] Grafana data sources configured — Prometheus, Loki, Tempo
- [ ] Grafana data source correlation — jump from trace span to Loki logs by trace_id
- [ ] First custom dashboards — service RED metrics (rate, errors, duration) for Phase 1 services

### Phase 2 — Async, AI, and Kafka observability (Jun → Aug 2025)
- [ ] Kafka setup — topics and local broker
- [ ] `api-nutrition` — external API integration
- [ ] `api-image-processing` — external AI API (OpenAI Vision / Google Vision)
- [ ] Novu self-hosted + `api-notifications` — email and push delivery
- [ ] Full upload → transcribe → save flow working end to end
- [ ] `pipelines` repo extended for Phase 2 services to Minikube (QA)
- [ ] otel-go instrumentation in api-nutrition, api-image-processing, api-notifications
- [ ] Kafka producer instrumentation — trace context injected into message headers on publish
- [ ] Kafka consumer instrumentation — trace context extracted from headers, child span created
- [ ] End-to-end trace visible in Grafana — upload → Kafka → transcribe → notify in one timeline
- [ ] Kafka consumer lag dashboard — per topic, per consumer group

### Phase 3 — GitOps and Cloud (Sep → Oct 2025)
- [ ] `pipelines` repo — manifest templates for all K8s resource types
- [ ] `pipelines` repo — per-service YAML configs (qa + prod) for all services
- [ ] `manifests` repo — created, ArgoCD pointed at it
- [ ] ArgoCD installed in EKS — one Application CRD per service per environment
- [ ] GitHub Actions reusable workflows updated — CI only, render + commit replaces deploy step
- [ ] Terraform — EKS cluster, RDS, DocumentDB, S3
- [ ] AWS Secrets Manager + OIDC — zero static credentials
- [ ] External Secrets Operator in EKS — syncs prod secrets into K8s Secrets
- [ ] `monitoring` repo — `values-prod.yaml` per tool, observability stack deployed to EKS
- [ ] OVERM running on EKS end to end

### Phase 4 — Polish (Oct → Nov 2025)
- [ ] Additional Grafana dashboards — recipe pipeline flow, menu activity, image processing latency
- [ ] Alertmanager rules — service error rate, Kafka lag, pod restarts routed through Novu
- [ ] AWS Practitioner certification
- [ ] Frontend improvements

### Future / nice to have
- [ ] Self-trained OCR model for specific handwriting
- [ ] Constrained menu generation (pinned recipes, calorie targets, prep time filters)
- [ ] Recipe statistics and user insights
- [ ] React Native mobile app
- [ ] Menu archive analytics
- [ ] Service identities / mTLS for service-to-service authentication
- [ ] Server-side token blocklist
- [ ] Migrate manifest storage from Git to OCI artifact registry

---

## Key Design Decisions

**Why contracts before code?**  
Writing OpenAPI contracts before implementing handlers forces API design to be a first-class concern. When api-menu-shuffle needs to call api-recipe-catalog, the contract is the spec — no reading through someone else's handlers to figure out what fields come back. It also makes drift visible: if the code does not match the contract, something is wrong.

**Why Kafka only for two flows?**  
Kafka introduces operational complexity that is only justified when async is genuinely needed. Image processing is slow and the user should not block on it. Notifications are fire-and-forget. Recipe creation is synchronous because the user is waiting for a 201 with the created recipe including macros. The rule: if the user is staring at a spinner, use HTTP.

**Why PostgreSQL for users and MongoDB for recipes?**  
Users are relational. A user has refresh tokens, an auth provider, and future relationships to roles and preferences. Strong consistency and foreign keys matter here. Recipes are documents — ingredients, steps, and macros are nested, variable in structure, and do not benefit from normalization. MongoDB's document model fits naturally. Two databases, two schemas, one clear reason for each.

**Why JWT validated locally?**  
Every service shares `AUTH_JWT_SECRET` and validates the token locally. The alternative makes auth a synchronous dependency of every single service call — if auth is slow or down, everything is slow or down. Local validation means each service is independently resilient. The tradeoff is that revoking a token requires waiting for expiry (24h) unless a blocklist is maintained — an acceptable tradeoff for this project.

**Why HttpOnly cookies for the web frontend?**  
JWTs stored in localStorage are readable by any JavaScript on the page. HttpOnly cookies are set by the server and are completely inaccessible to JavaScript. The browser attaches them automatically on every request. CSRF protection (X-CSRF-Token header + cookie) covers the one new attack surface this introduces. Mobile apps use Bearer JWT in memory since they do not have the same XSS risk model.

**Why Novu instead of a custom notification service?**  
Novu is open source, self-hostable, and runs as a Helm chart inside the same cluster. It handles templating, retries, delivery tracking, and provider abstraction. Swap SendGrid for Mailgun without touching any service code. The `api-notifications` bridge is ~50 lines of Go that reads from Kafka and calls Novu's API.

**Why split CI (GitHub Actions) and CD (ArgoCD)?**  
GitHub Actions is good at building, testing, and pushing artifacts. It has no cluster state awareness, no drift detection, and no self-healing. ArgoCD is purpose-built for exactly that. The split gives a clear boundary: Actions produces a versioned artifact and commits it, ArgoCD takes responsibility for the cluster matching that artifact. Each tool does what it is designed for.

**Why centralize manifest templates and configs in the pipelines repo?**  
Service repos are application code. They have no business knowing how they are deployed. Centralizing templates means a change to the Deployment template propagates to every service on the next deploy — one fix, not twelve. Centralizing configs means every environment's configuration is visible and reviewable in one place, and changes go through a PR with a clear audit trail.

**Why YAML for service config files?**  
The entire infrastructure stack — Kubernetes manifests, ArgoCD apps, GitHub Actions workflows, Helm charts — already speaks YAML. Using YAML for application config keeps tooling consistent and cognitive overhead low. TOML is idiomatic for Go CLIs and dev tools but for runtime config mounted as a ConfigMap in Kubernetes, YAML is what the ecosystem expects.

**Why two secret management strategies (GitHub Secrets for QA, AWS Secrets Manager for prod)?**  
The environments have different trust boundaries. Minikube runs locally and has no access to AWS — GitHub Secrets injected at pipeline time are the practical solution. EKS runs in AWS and has IAM — the External Secrets Operator pulling from Secrets Manager is the production-grade approach that avoids any secret ever touching the pipeline. Both strategies deliver the same interface to the Go service: environment variables.

**Why Tempo over Jaeger for distributed tracing?**  
Both are valid tracing backends but Tempo integrates directly into the existing Grafana instance — traces, metrics, and logs are all queryable from one place. Grafana's trace viewer links spans to Loki logs via shared `trace_id`, which is the three pillars working together. Jaeger would add a second UI with no additional capability given the stack already in use.

**Why OpenTelemetry SDK over zero-code instrumentation (Beyla)?**  
Beyla uses eBPF to instrument services at the kernel level with no code changes, but it only sees network boundaries. It cannot trace what happens inside a function, a database call, or a Kafka message handler. The SDK requires explicit instrumentation — wrapping MongoDB calls and Kafka producers in child spans — but the result is traces that show exactly what the code is doing, not just that HTTP traffic occurred. The Kafka context propagation in particular, where a trace started in one service continues in another via message headers, is a distributed systems concept worth implementing directly.

**Why store generated manifests in a separate Git repo?**  
Mixing generated files into the pipelines repo pollutes commit history with automated noise — every deploy is a commit, which buries meaningful changes like template and config updates. A dedicated `manifests` repo keeps the signal clean: every commit in `pipelines` is a human decision, every commit in `manifests` is a pipeline action. The tradeoff versus a proper artifact registry (Nexus, OCI) is that Git is not designed for artifact storage and rollback is a Git operation rather than a registry pointer change — a known and documented limitation at this scale.

---

## About

Built by Mario Irigoyen as a learning project during the final semester of a Computer Systems Engineering degree while working as an SRE intern at Softtek.

The app is intentionally overengineered. A recipe app does not need Kafka, Kubernetes, or a separate nutrition microservice. Every architectural decision that would be justified at scale is practiced here at small scale so the patterns are familiar when they matter for real.

*"I designed and built a microservices system from scratch. The app helps me and my mom plan our meals. The architecture could handle a few more."*

---

*Last updated: March 2026*