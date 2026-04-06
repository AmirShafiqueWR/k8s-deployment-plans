# Kubernetes Deployment Plans

**Student:** Amir Shafique
**Project:** Kubernetes Infrastructure Planning
**Submission:** Class Project — AI Systems Deployment

---

## Repository Contents

| File | Description |
|---|---|
| `plan-1-komplio-k8s.md` | K8s deployment plan for Komplio — AI Native Task Manager (UI, Backend APIs, Whisper Task Agent, WhatsApp Notification Service) |
| `plan-2-openclaw-k8s.md` | K8s deployment plan for OpenClaw — Personal AI Employee with full security considerations |
| `k8-planning-skill.md` | Reusable Agent Skill that generates K8s deployment plans for any future project |

---

## Scenario 1 — AI Native Task Manager (Komplio)
Komplio is a multi-tenant SaaS task management platform. The K8s plan covers
5 Deployments, 1 StatefulSet, 6 Services, 2 ConfigMaps, 5 Secrets with rotation
handling, multi-tenant Namespaces, full RBAC, and inter-service communication.

## Scenario 2 — AI Employee (OpenClaw)
OpenClaw is a personal AI agent. The K8s plan focuses on security — StatefulSet
with Tailscale sidecar, ClusterIP-only exposure, secret expiry handling, threat
model covering prompt injection, RBAC with least privilege, and NetworkPolicy
blocking internal cluster access.

## K8 Planning Skill
A reusable agent skill with decision frameworks, worked examples, and an
evaluation rubric that can generate K8s plans for any application.
