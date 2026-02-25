# OVERM — OverEngineerMenu

> A deliberately overengineered microservices application built to learn distributed systems, cloud infrastructure, and DevOps practices end to end.

**Author:** Mario Irigoyen — [@Tar-Mairon24](https://github.com/Tar-Mairon24)  
**Status:** Active development — Phase 1  
**Stack:** Go · Python · Vue/Angular · Kubernetes · Kafka · MongoDB · PostgreSQL · AWS EKS

---

## What is this?

OVERM is a recipe and weekly menu planning app. You upload recipes, it calculates calories and macros, and generates a meal plan for the week. You can also upload a photo of a handwritten recipe and it transcribes it automatically.

The app itself is simple on purpose. The point is the architecture — polyrepo microservices, event-driven async flows, proper service contracts, CI/CD pipelines, IaC, and production-grade observability. Everything you would find in a real distributed system at a real company, built from scratch by one person.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repositories](#repositories)
- [Services](#services)
- [Data Layer](#data-layer)
- [Async Messaging — Kafka Topics](#async-messaging--kafka-topics)
- [Service Contracts](#service-contracts)
- [Git Flow & Branching Strategy](#git-flow--branching-strategy)
- [Versioning Scheme](#versioning-scheme)
- [CI/CD Pipeline](#cicd-pipeline)
- [Infrastructure & IaC](#infrastructure--iac)
- [Observability](#observability)
- [Local Development](#local-development)
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
│  overm-auth          overm-recipe-catalog             │
│  overm-menu-shuffle  overm-nutrition                  │
│  overm-image-processing  overm-notification           │
│  overm-frontend                                       │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │  namespace: monitoring                          │  │
│  │  Prometheus · Grafana · Loki · Promtail         │  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
        │
   Data Layer
   PostgreSQL (users) · MongoDB (recipes · menus)
   AWS Secrets Manager · S3
```

Full architecture diagram: [`overm-docs/diagrams/overm-architecture-v2.drawio`](./diagrams/overm-architecture-v2.drawio)

Git flow diagram: [`overm-docs/diagrams/overm-gitflow.drawio`](./diagrams/overm-gitflow.drawio)

---

## Repositories

All repositories live under the [@Tar-Mairon24](https://github.com/Tar-Mairon24) GitHub account with the `overm-` prefix so they sort together.

| Repository | Description | Language |
|---|---|---|
| **[overm-docs](https://github.com/Tar-Mairon24/overm-docs)** | Central documentation — you are here | Markdown |
| **[overm-contracts](https://github.com/Tar-Mairon24/overm-contracts)** | OpenAPI specs + Kafka message schemas | YAML |
| **[overm-pipelines](https://github.com/Tar-Mairon24/overm-pipelines)** | Reusable GitHub Actions workflows | YAML |
| **[overm-auth](https://github.com/Tar-Mairon24/overm-auth)** | User auth, JWT, Google OAuth | Go |
| **[overm-recipe-catalog](https://github.com/Tar-Mairon24/overm-recipe-catalog)** | Recipe CRUD, image upload trigger | Go |
| **[overm-menu-shuffle](https://github.com/Tar-Mairon24/overm-menu-shuffle)** | Menu generation and management | Go |
| **[overm-nutrition](https://github.com/Tar-Mairon24/overm-nutrition)** | Macro calculator, external API integration | Go |
| **[overm-image-processing](https://github.com/Tar-Mairon24/overm-image-processing)** | OCR, handwriting transcription | Python |
| **[overm-notification](https://github.com/Tar-Mairon24/overm-notification)** | Email and push notifications | Go |
| **[overm-frontend](https://github.com/Tar-Mairon24/overm-frontend)** | Web app, served by nginx inside K8s | Vue / Angular |
| **[overm-swagger](https://github.com/Tar-Mairon24/overm-swagger)** | Swagger UI mounting all service contracts | Docker |
| **[overm-monitoring](https://github.com/Tar-Mairon24/overm-monitoring)** | Helm values, Grafana dashboards, ServiceMonitors | YAML |

---

## Services

### overm-auth
Handles all authentication. Issues JWTs consumed and validated locally by every other service — no network call per request. Supports email/password and Google OAuth.

- **Stack:** Go, Gin, PostgreSQL
- **Endpoints:** `POST /auth/v1/login` · `POST /auth/v1/register` · `POST /auth/v1/refresh` · `POST /auth/v1/logout` · `GET /auth/google` · `GET /auth/google/callback` · `GET /users/me`
- **Token delivery:** Bearer JWT for mobile/API clients · HttpOnly encrypted cookie for web browser clients (CSRF protected)
- **Contract:** [`overm-contracts/user-auth-api.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/user-auth-api.yaml) · [`overm-contracts/user-auth-web.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/user-auth-web.yaml)

### overm-recipe-catalog
Manages the user's recipe library. Handles manual recipe creation and triggers async image processing via Kafka. Calls overm-nutrition synchronously during recipe creation so macros are available immediately.

- **Stack:** Go, Gin, MongoDB
- **Endpoints:** `GET /recipe-catalog/v1/recipes` · `POST /recipe-catalog/v1/recipes` · `GET /recipe-catalog/v1/recipes/:id` · `PATCH /recipe-catalog/v1/recipes/:id` · `DELETE /recipe-catalog/v1/recipes/:id` · `POST /recipe-catalog/v1/recipes/upload`
- **Produces:** `recipe.image.uploaded` · `recipe.created`
- **Contract:** [`overm-contracts/recipe-catalog.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/recipe-catalog.yaml)

### overm-menu-shuffle
Owns all menu logic. Generates weekly/monthly meal plans from the user's recipe catalog. MVP uses a random shuffle algorithm. Future versions will support pinned recipes, calorie targets, prep time filters, and dietary constraints.

- **Stack:** Go, Gin, MongoDB
- **Endpoints:** `POST /menu/v1/generate` · `GET /menu/v1/active` · `PATCH /menu/v1/:id/confirm` · `GET /menu/v1/history`
- **Produces:** `menu.activated`
- **Contract:** [`overm-contracts/menu-shuffle.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/menu-shuffle.yaml)

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

### overm-nutrition
Calculates macros and calories for a given list of ingredients. Called synchronously by overm-recipe-catalog during recipe creation and by overm-image-processing after transcription. Looks up unknown ingredients against the USDA FoodData or Edamam external API and caches results.

- **Stack:** Go, Gin
- **Endpoints:** `POST /nutrition/v1/calculate`
- **Contract:** [`overm-contracts/nutrition.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/nutrition.yaml)

### overm-image-processing
Consumes `recipe.image.uploaded` from Kafka. Sends the image to an AI model for OCR and handwriting transcription, then calls overm-nutrition to calculate macros, saves the completed recipe to MongoDB, and publishes `recipe.transcription.completed`. The AI backend is swappable — starts with an external API (OpenAI Vision / Google Vision), future goal is a self-trained model for specific handwriting recognition.

- **Stack:** Python
- **Consumes:** `recipe.image.uploaded`
- **Produces:** `recipe.transcription.completed`
- **Contract:** [`overm-contracts/kafka-contracts.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/kafka-contracts.yaml)

### overm-notification
Consumes notification events from Kafka and delivers them via email or push. Decoupled from all other services — nothing calls it directly, everything publishes to `notification.send` and this service handles delivery.

- **Stack:** Go
- **Consumes:** `notification.send`
- **Contract:** [`overm-contracts/kafka-contracts.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/kafka-contracts.yaml)

### overm-frontend
Web application served by nginx as a container inside Kubernetes. Communicates with backend services via HttpOnly cookies — JWTs are never accessible to JavaScript. CSRF protected.

- **Stack:** Vue or Angular, nginx
- **Auth pattern:** HttpOnly cookie (not localStorage — XSS protection)

---

## Data Layer

| Store | Used by | Contents |
|---|---|---|
| **PostgreSQL** | overm-auth | `users` table · `refresh_tokens` table |
| **MongoDB** | overm-recipe-catalog | `recipes` collection (with macros, status: active/archived) |
| **MongoDB** | overm-menu-shuffle | `menus` collection (status: draft/active/archived) |
| **S3** | overm-image-processing | Uploaded recipe images |

**Why two databases:**  
PostgreSQL for users because auth data is relational and needs strong consistency — users reference refresh tokens, OAuth providers, and future roles.  
MongoDB for recipes and menus because the data is naturally document-shaped — ingredients, steps, and macros are nested and variable in structure.

**Soft deletes everywhere:**  
Recipes are never deleted, only set to `status: archived`. This preserves history for future statistics features ("your most cooked recipe", "average weekly calories over 3 months").

Menus follow the same pattern — generating a new menu archives the previous one rather than deleting it.

---

## Async Messaging — Kafka Topics

Only flows where the user does not need to block are async. Everything else is synchronous HTTP.

| Topic | Producer | Consumer | Trigger |
|---|---|---|---|
| `recipe.image.uploaded` | overm-recipe-catalog | overm-image-processing | User uploads a photo |
| `recipe.transcription.completed` | overm-image-processing | overm-recipe-catalog · overm-notification | Transcription finished |
| `recipe.created` | overm-recipe-catalog | (future: stats, recommendations) | Any recipe created |
| `menu.activated` | overm-menu-shuffle | overm-notification | User confirms a menu |
| `notification.send` | any service | overm-notification | Generic notification request |

**Design rule:**  
If the user is waiting for a response → HTTP.  
If it's a background job the user doesn't block on → Kafka.

Full message schemas: [`overm-contracts/kafka-contracts.yaml`](https://github.com/Tar-Mairon24/overm-contracts/blob/main/kafka-contracts.yaml)

---

## Service Contracts

All service contracts live in [`overm-contracts`](https://github.com/Tar-Mairon24/overm-contracts). They are written before any code is written for a service.

| Contract file | Description |
|---|---|
| `user-auth-api.yaml` | Auth service — mobile and API clients (Bearer JWT) |
| `user-auth-web.yaml` | Auth service — browser clients (HttpOnly cookie) |
| `recipe-catalog.yaml` | Recipe catalog CRUD and upload |
| `menu-shuffle.yaml` | Menu generation and management |
| `nutrition.yaml` | Macro calculation |
| `kafka-contracts.yaml` | All Kafka message schemas |

**Swagger UI** mounts all contracts and runs locally via [`overm-swagger`](https://github.com/Tar-Mairon24/overm-swagger):

```bash
cd overm-swagger
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

Full diagram: [`overm-docs/diagrams/overm-gitflow.drawio`](./diagrams/overm-gitflow.drawio)

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
- Hotfixes skip the QA branch — they go directly from hotfix → main, then cherry-pick to develop

**Commit message format:**
```
feat(auth): add login endpoint #001
fix(recipe): handle empty ingredient list #042
chore(pipeline): update Go version to 1.22
```

---

## Versioning Scheme

Semantic versioning: `MAJOR.MINOR.PATCH`

| Bump | When |
|---|---|
| `PATCH` (0.0.x) | Bug fix, no new functionality |
| `MINOR` (0.x.0) | New endpoint or feature, backward compatible |
| `MAJOR` (x.0.0) | Breaking contract change — incompatible API change |

A breaking contract change is specifically when a change to a file in `overm-contracts` would break existing consumers. That is the trigger for a major version bump.

---

## CI/CD Pipeline

Pipelines are defined once in [`overm-pipelines`](https://github.com/Tar-Mairon24/overm-pipelines) as reusable GitHub Actions workflows. Each service repo calls the central pipeline with parameters — changing the pipeline once propagates to every service automatically.

```yaml
# Each service repo — entire CI file
jobs:
  pipeline:
    uses: Tar-Mairon24/overm-pipelines/.github/workflows/go-service.yml@main
    with:
      service_name: overm-auth
      go_version: "1.22"
    secrets: inherit
```

**Pipeline stages per service:**

```
push to branch
     │
     ├── test (go test ./... + coverage threshold 60%)
     ├── lint (golangci-lint)
     │
     └── build-and-push (Docker image → Docker Hub)
              │
              ├── if branch = QA   → deploy to Minikube
              └── if branch = main → deploy to EKS
```

**Image tagging:**
```
yourname/overm-auth:develop         ← branch (moving)
yourname/overm-auth:sha-a1b2c3d     ← git SHA (immutable, for rollbacks)
yourname/overm-auth:1.0.0           ← semver on release tag
```

**Secrets management:**  
GitHub Secrets during development. AWS Secrets Manager via OIDC (no static credentials) when deployed to EKS. The same secrets are consumed by running pods via the External Secrets Operator syncing into Kubernetes Secrets.

---

## Infrastructure & IaC

All cloud infrastructure is defined in Terraform inside [`overm-monitoring`](https://github.com/Tar-Mairon24/overm-monitoring) under `terraform/`.

**AWS resources managed by Terraform:**
- EKS cluster and node groups
- RDS PostgreSQL
- DocumentDB (MongoDB compatible)
- S3 buckets
- CloudFront distribution
- ALB and target groups
- IAM roles (including GitHub Actions OIDC role)
- AWS Secrets Manager secrets
- VPC, subnets, security groups

**Cost note:** EKS is not cheap. The cluster is spun up for development and torn down when not in use. `terraform destroy` is one command. Never leave it running overnight.

---

## Observability

**Stack:** Prometheus + Loki + Grafana (PLG) deployed via Helm into the `monitoring` namespace.

Every service exposes two endpoints required before a PR is merged:
- `GET /metrics` — Prometheus scrape endpoint (no auth)
- `GET /healthz` — Kubernetes liveness probe (no auth)

**Metrics collected:**
- Go runtime metrics (automatic — zero code)
- Custom business metrics: `overm_recipes_created_total`, `overm_nutrition_calc_duration_seconds`, Kafka consumer lag
- Kubernetes cluster metrics (kube-prometheus-stack, out of the box)

**Logs:** Structured JSON via `go.uber.org/zap`. Promtail DaemonSet collects from all pods automatically. Queryable in Grafana via LogQL.

**Alertmanager** fires alerts into `overm-notification` service — reusing your own infrastructure instead of a third-party alerting tool.

Install:
```bash
helm install overm-monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

helm install overm-loki grafana/loki-stack \
  --namespace monitoring --set promtail.enabled=true
```

---

## Local Development

### Prerequisites
- Docker
- docker-compose
- Go 1.22+
- minikube (for QA testing)

### First time setup
```bash
# Create the shared network — run once
docker network create overm-network
```

### Running a service locally
```bash
# Clone the service
git clone https://github.com/Tar-Mairon24/overm-auth
cd overm-auth

# Copy env template
cp .env.example .env
# Fill in values in .env

# Start service + its dependencies
docker compose up
```

### Running Swagger UI
```bash
git clone https://github.com/Tar-Mairon24/overm-swagger
cd overm-swagger
# Copy contracts from overm-contracts into ./contracts/
docker compose up
# open http://localhost:8090
```

### Port conventions
| Service | Port |
|---|---|
| overm-auth | 8081 |
| overm-recipe-catalog | 8082 |
| overm-menu-shuffle | 8083 |
| overm-nutrition | 8084 |
| overm-image-processing | 8085 |
| overm-notification | 8086 |
| overm-frontend | 3000 |
| overm-swagger | 8090 |
| Grafana | 3001 |
| Prometheus | 9090 |

---

## Roadmap

### Phase 1 — Core services (Feb → May 2025)
- [x] Architecture design and service contracts
- [x] Git flow and CI/CD pipeline design
- [ ] `overm-auth` — JWT, Google OAuth, PostgreSQL
- [ ] `overm-recipe-catalog` — CRUD, sync nutrition call
- [ ] `overm-menu-shuffle` — MVP shuffle algorithm
- [ ] Basic frontend — login, recipe list, generate menu
- [ ] All three services running in Minikube

### Phase 2 — Async and AI (Jun → Aug 2025)
- [ ] Kafka setup — topics and local broker
- [ ] `overm-nutrition` — external API integration
- [ ] `overm-image-processing` — external AI API (OpenAI Vision / Google Vision)
- [ ] `overm-notification` — email delivery
- [ ] Full upload → transcribe → save flow working end to end

### Phase 3 — Cloud (Sep → Oct 2025)
- [ ] Terraform — EKS cluster, RDS, DocumentDB, S3
- [ ] GitHub Actions pipelines — all services automated
- [ ] AWS Secrets Manager + OIDC — zero static credentials
- [ ] External Secrets Operator in EKS
- [ ] OVERM running on EKS end to end

### Phase 4 — Observability and polish (Oct → Nov 2025)
- [ ] Prometheus + Loki + Grafana via Helm
- [ ] Custom dashboards — recipe pipeline, menu activity, service RED metrics
- [ ] AWS Practitioner certification
- [ ] Frontend improvements
- [ ] Begin job search

### Future / nice to have
- [ ] Self-trained OCR model for specific handwriting
- [ ] Constrained menu generation (pinned recipes, calorie targets, prep time filters)
- [ ] Recipe statistics and user insights
- [ ] React Native mobile app
- [ ] Menu archive analytics

---

## Key Design Decisions

**Why polyrepo?**  
Each service is an independently deployable unit with its own pipeline, its own versioning, and its own Dockerfile. This mirrors how real teams work. One repo per service means one PR per change, one pipeline per service, and clean separation of concerns. The overhead of managing multiple repos solo is minimal.

**Why contracts before code?**  
Writing OpenAPI contracts before implementing handlers forces you to think about API design as a first-class concern. When overm-menu-shuffle needs to call overm-recipe-catalog, the contract is the spec — no reading through someone else's handlers to figure out what fields come back. It also makes drift visible: if the code doesn't match the contract, something is wrong.

**Why Kafka only for two flows?**  
Kafka introduces operational complexity that is only justified when you genuinely need async. Image processing is slow and the user should not block on it — correct use of Kafka. Notifications are fire-and-forget — correct use of Kafka. Recipe creation is synchronous because the user is waiting for a 201 with their created recipe including macros — putting that through Kafka would mean you can't return anything meaningful to the user. The rule: if the user is staring at a spinner, use HTTP.

**Why PostgreSQL for users and MongoDB for recipes?**  
Users are relational. A user has refresh tokens, an auth provider, and future relationships to roles and preferences. Strong consistency and foreign keys matter here. Recipes are documents — ingredients, steps, and macros are nested, variable in structure, and don't benefit from normalization. MongoDB's document model fits naturally. Two databases, two schemas, one clear reason for each.

**Why JWT validated locally?**  
Every service shares `AUTH_JWT_SECRET` and validates the token locally. The alternative — calling overm-auth on every request to validate — would make auth a synchronous dependency of every single service call. If auth is slow or down, everything is slow or down. Local validation means each service is independently resilient. The tradeoff is that revoking a token requires waiting for expiry (24h) unless you maintain a blocklist — an acceptable tradeoff for this project.

**Why HttpOnly cookies for the web frontend?**  
JWTs stored in localStorage are readable by any JavaScript on the page — including injected scripts from XSS attacks. HttpOnly cookies are set by the server and are completely inaccessible to JavaScript, even your own. The browser attaches them automatically on every request. CSRF protection (X-CSRF-Token header + cookie) covers the one new attack surface this introduces. Mobile apps use Bearer JWT in memory since they don't have the same XSS risk model.

---

## About

Built by Mario Irigoyen as a learning project during the final semester of a Computer Systems Engineering degree while working as an SRE intern at Softtek.

The app is intentionally overengineered. A recipe app does not need Kafka, Kubernetes, or a separate nutrition microservice. That is the point. Every architectural decision that would be justified at scale is practiced here at small scale so the patterns are familiar when they matter for real. But I dont want to be stuck developing an app ment for 10 people minimum for 5 years just to learn.

*"I designed and built a polyrepo microservices system from scratch. The app helps me and my mom plan our meals. The architecture could handle a few more."*

---

*Last updated: February 2026*
