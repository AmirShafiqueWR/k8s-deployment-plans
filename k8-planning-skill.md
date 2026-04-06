# K8 Planning Skill
## Reusable Agent Skill — Generate Kubernetes Deployment Plans for Any Project

**Skill Name:** k8-planning-skill  
**Version:** 1.0  
**Author:** Class Project Submission  
**Purpose:** A reusable AI agent skill that generates complete, production-grade Kubernetes deployment plans for any application described to it.

---

## What This Skill Does

When given a description of any application, this skill produces a complete Kubernetes deployment plan covering all eight required topics:

1. Pods and Deployments / StatefulSets
2. Services and their types (ClusterIP, NodePort, LoadBalancer)
3. Resource requests and limits
4. ConfigMaps for configuration
5. Secrets management and expiry handling
6. Namespaces for isolation
7. RBAC Roles and RoleBindings
8. Inter-service communication

---

## Skill Format (SKILL.md)

Copy this into your OpenClaw skills directory as `k8-planning.md` or register it in your ClawHub skill registry.

---

```markdown
# Skill: Kubernetes Deployment Planner

## Description
Generate a complete, production-grade Kubernetes deployment plan for any application.
Given an app description, produce a detailed markdown plan covering all K8s components
needed to deploy it safely in a real cluster.

## Trigger phrases
- "Create a K8s plan for..."
- "Plan the Kubernetes deployment of..."
- "How would I deploy [app] on Kubernetes?"
- "Generate a K8s deployment plan for..."
- "Write the K8s architecture for..."

## Instructions

When the user provides an application description, follow these steps exactly:

### Step 1 — Analyse the application

Before writing anything, identify:
- How many services/components does the app have?
- Which components are STATELESS (safe to scale horizontally)?
- Which components are STATEFUL (need persistent storage)?
- What external APIs or secrets does it use?
- What is the expected traffic volume?
- Who needs to access it — public users, internal admins, or both?

### Step 2 — Decide Deployment vs StatefulSet for each component

Apply this rule:
- Stateless (FastAPI, Node.js, React frontend, notification service) → Deployment
- Stateful (database, cache, AI agent with memory, message queue) → StatefulSet

### Step 3 — Decide Service type for each component

Apply this rule:
- Needs public internet access → LoadBalancer
- Only needs access from within cluster → ClusterIP
- Needs restricted external access (VPN/admin only) → NodePort
- StatefulSet needing stable DNS → ClusterIP headless (clusterIP: None)

### Step 4 — Generate the plan

Produce a markdown document with these exact sections:

1. **Application overview** — describe each component and map it to K8s primitives
2. **Environments** — dev (Docker Desktop K8s), staging, production
3. **Namespaces** — one per environment, one per isolated tenant if multi-tenant
4. **Deployments and StatefulSets** — full YAML for each component
5. **Services** — full YAML with correct type for each component
6. **Resource requests and limits** — table + YAML, justify the numbers
7. **ConfigMaps** — all non-secret config values
8. **Secrets** — inventory table with rotation periods + creation commands
9. **RBAC** — ServiceAccounts, Roles, RoleBindings per component
10. **Inter-service communication** — ASCII diagram + NetworkPolicy YAML

### Step 5 — Security checklist

Before finalising, verify the plan includes:
- [ ] No secret values in ConfigMaps
- [ ] Every secret has a rotation period annotation
- [ ] Every pod has livenessProbe and readinessProbe
- [ ] No pod runs as root (runAsNonRoot: true)
- [ ] Resource limits set on every container
- [ ] NetworkPolicy restricts unnecessary pod-to-pod communication
- [ ] LoadBalancer only used where genuinely needed (not for internal services)
- [ ] StatefulSet used for any component with persistent storage

## Output format

Produce a single markdown file ready to commit to GitHub.
Include working YAML examples for every resource.
Use realistic resource values (not placeholder 1000m/1Gi on everything).
Label all resources consistently with app, component, and environment labels.

## Example invocation

User: "Create a K8s plan for a food delivery app with a React frontend,
       a Node.js API, a PostgreSQL database, a Redis cache, and a
       notification service that sends SMS."

Agent output: Complete plan with:
- 3 Deployments (frontend, API, notification)
- 2 StatefulSets (PostgreSQL, Redis)
- 5 Services (1 LoadBalancer, 3 ClusterIP, 1 ClusterIP headless)
- 2 ConfigMaps (app config, SMS templates)
- 4 Secrets (DB password, Redis password, SMS API key, JWT secret)
- 4 Namespaces (prod, staging, dev, monitoring)
- 5 ServiceAccounts + 4 Roles + 4 RoleBindings
- NetworkPolicy restricting DB access to API only
```

---

## Decision Framework — Built Into the Skill

The skill uses this decision tree for every application:

### Deployment vs StatefulSet

```
Does this component store data that must survive a pod restart?
    │
    ├── YES → Does it need a stable network identity (hostname)?
    │             │
    │             ├── YES → StatefulSet + headless Service
    │             └── NO  → StatefulSet (still needed for PVC guarantee)
    │
    └── NO  → Deployment (stateless — scale freely)

Examples:
  FastAPI backend     → Deployment  (stateless — DB is in Supabase)
  React frontend      → Deployment  (stateless — just serves HTML/JS)
  Notification svc    → Deployment  (stateless — just sends messages)
  PostgreSQL          → StatefulSet (stores data)
  Redis               → StatefulSet (stores session/cache data)
  OpenClaw agent      → StatefulSet (has SQLite memory database)
```

### Service type selection

```
Who needs to reach this service?
    │
    ├── Public internet users → LoadBalancer
    │       Example: app.komplio.com, api.myapp.com
    │
    ├── Only other pods inside the cluster → ClusterIP
    │       Example: FastAPI calling Whisper, API calling Redis
    │
    ├── Restricted access (admin, VPN only) → NodePort
    │       Example: Admin panel at port 30090 via VPN
    │
    └── StatefulSet pods need stable DNS → ClusterIP headless
            (clusterIP: None)
            Example: Redis StatefulSet, PostgreSQL StatefulSet
```

### Secret rotation periods

```
How sensitive is this secret? How damaging if compromised?

  Critical (DB password, JWT secret, master admin key) → 90 days
  High     (external API tokens, WhatsApp, email)      → 60 days
  Access   (VPN keys, gateway tokens)                  → 30 days
  Medium   (internal service passwords)                → 180 days
```

### Resource sizing guide

```
Component type          CPU Request  CPU Limit   RAM Request  RAM Limit
─────────────────────   ──────────── ─────────── ──────────── ──────────
Static frontend         50m          200m        64Mi         128Mi
Simple API (FastAPI)    100m         500m        128Mi        256Mi
Heavy API (ML/AI)       500m         2000m       512Mi        2Gi
Database (PostgreSQL)   250m         1000m       512Mi        1Gi
Cache (Redis)           100m         500m        128Mi        256Mi
AI Agent (LLM)          500m         2000m       512Mi        2Gi
Notification service    100m         300m        128Mi        256Mi
Sidecar (lightweight)   50m          200m        64Mi         128Mi
```

---

## Worked Examples

### Example 1 — E-commerce app

**Input:** "Plan K8s deployment for an e-commerce app: React storefront, Express.js API, PostgreSQL database, Redis cart cache, Stripe payment webhook receiver."

**Skill output summary:**

| Component | Kind | Service Type | CPU Req | RAM Req |
|---|---|---|---|---|
| React storefront | Deployment (2 replicas) | LoadBalancer | 50m | 64Mi |
| Express.js API | Deployment (2 replicas) | ClusterIP | 200m | 256Mi |
| PostgreSQL | StatefulSet (1 replica) | ClusterIP headless | 250m | 512Mi |
| Redis | StatefulSet (1 replica) | ClusterIP headless | 100m | 128Mi |
| Stripe webhook | Deployment (1 replica) | LoadBalancer | 100m | 128Mi |

Secrets: `db-secrets` (90d), `redis-secrets` (180d), `stripe-secrets` (60d), `jwt-secrets` (90d)

---

### Example 2 — AI Native Task Manager (Komplio)

**Input:** "Plan K8s deployment for an AI task manager with a Jinja2 frontend, FastAPI backend, OpenAI Whisper voice agent, WhatsApp notification service, Redis cache, multi-tenant."

**Skill output summary:**

| Component | Kind | Service Type | Namespace |
|---|---|---|---|
| Jinja2 frontend | Deployment (2) | LoadBalancer | komplio-prod |
| FastAPI backend | Deployment (2) | ClusterIP | komplio-prod |
| Whisper agent | Deployment (1–3) | ClusterIP | komplio-prod |
| Notification service | Deployment (1) | ClusterIP | komplio-prod |
| Admin panel | Deployment (1) | NodePort | komplio-prod |
| Redis | StatefulSet (1) | ClusterIP headless | komplio-prod |
| Per-tenant isolation | Namespace per tenant | ResourceQuota | komplio-tenant-{id} |

Secrets: `db-secrets`, `app-secrets`, `notification-secrets`, `ai-secrets`, `redis-secrets`

---

### Example 3 — Personal AI Employee (OpenClaw)

**Input:** "Plan K8s deployment for OpenClaw personal AI agent with security focus. Components: Gateway (Node.js), Agent (Claude LLM), Skills (plugins), Memory (SQLite). Must be secure."

**Skill output summary:**

| Component | Kind | Service Type | Security notes |
|---|---|---|---|
| Gateway + Agent + Skills | StatefulSet (1) | ClusterIP headless | Non-root, read-only FS, drop ALL caps |
| Tailscale access | Sidecar container | None | VPN-only access — no public port |
| Memory (SQLite) | PVC on StatefulSet | N/A | 5Gi block storage, encrypted at rest |

Secrets: `llm-secrets` (90d), `channel-secrets` (60d), `access-secrets` (30d), `skill-secrets` (90d)  
NetworkPolicy: Block all inbound except Tailscale. Allow outbound HTTPS only.

---

## Skill Evaluation Criteria

When evaluating the quality of a plan generated by this skill, check:

| Criterion | Pass | Fail |
|---|---|---|
| Correct Deployment vs StatefulSet | Stateful components use StatefulSet | Database uses Deployment |
| Correct Service types | Internal services use ClusterIP | All services use LoadBalancer |
| Secrets not in ConfigMap | Secrets in Secret objects only | API keys in ConfigMap data |
| Rotation periods defined | Every secret has expiry annotation | Secrets have no rotation plan |
| RBAC uses least privilege | Each SA can only access its own secrets | All SAs use cluster-admin |
| Resources set on all containers | Every container has requests and limits | Any container has no limits |
| Health probes defined | Every deployment has liveness + readiness | No probes defined |
| NetworkPolicy present | Communication matrix enforced | No network restrictions |
| Non-root containers | runAsNonRoot: true on all pods | Any pod runs as root |
| Namespaces for isolation | Separate namespace per environment | All resources in default namespace |

**Scoring:** 10/10 = production-ready plan. Below 7/10 = revise before submitting.

---

## How to Use This Skill in OpenClaw

```bash
# Install the skill in your OpenClaw instance
openclaw skills install ./k8-planning-skill.md

# Or register via ClawHub
openclaw skills install @komplio/k8-planning-skill

# Then trigger it via message:
# "Create a K8s deployment plan for my new chat app with
#  a React frontend, Node.js API, MongoDB database, and
#  a Socket.io real-time service"
```

## How to Use This Skill as a Prompt (without OpenClaw)

Paste the following into any AI assistant (Claude, GPT-4, Gemini):

```
You are a Kubernetes solutions architect. When I describe an application,
produce a complete K8s deployment plan in markdown format covering:

1. Pods and Deployments/StatefulSets (with YAML)
2. Services with correct types: LoadBalancer for public, ClusterIP for internal,
   NodePort for restricted, headless for StatefulSets
3. Resource requests and limits (realistic values, not placeholders)
4. ConfigMaps for all non-secret configuration
5. Secrets with rotation periods (annotate with expiry dates)
6. Namespaces for environment and tenant isolation
7. RBAC: ServiceAccounts, Roles, RoleBindings (least privilege)
8. Inter-service communication diagram + NetworkPolicy YAML

Rules:
- Stateful components (DB, cache, AI memory) always use StatefulSet
- Internal services always use ClusterIP
- Never put secret values in ConfigMaps
- Every pod must have livenessProbe and readinessProbe
- Every container must have resource limits
- Every secret must have a rotation period

My application: [DESCRIBE YOUR APP HERE]
```

---

## Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-04-03 | Initial release — covers all 8 K8s planning topics |

---

*This skill was developed as part of a Kubernetes deployment planning class project.  
It is designed to be reusable for any future project that requires K8s infrastructure planning.*
