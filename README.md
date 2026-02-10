# Cage

Secure sandbox infrastructure for running AI agents and bots in isolated environments, powered by Azure Container Instances (ACI).

## Problem

Running autonomous agents and bots (e.g., code execution agents, web-scraping bots, LLM tool-use agents) on bare metal or shared hosts introduces significant security risks — arbitrary code execution, network exfiltration, resource exhaustion, and privilege escalation. A purpose-built sandbox layer is needed to enforce strict isolation, resource limits, and ephemeral lifecycles.

## Goals

- **Isolation** — Each agent/bot runs in its own container with no access to host resources or other workloads.
- **Ephemeral by default** — Containers are created on demand and destroyed after execution; no persistent state leaks between runs.
- **Resource-bounded** — CPU, memory, and network quotas enforced per container.
- **Network control** — Outbound traffic restricted by policy; no inbound exposure unless explicitly configured.
- **Observable** — Structured logging, execution traces, and resource metrics collected for every run.
- **Simple API surface** — A thin orchestration layer to create, monitor, and tear down sandboxes programmatically.

## Why Azure Container Instances

- No cluster to manage (serverless containers).
- Sub-second billing — pay only for execution time.
- Native VNet integration for network isolation.
- Built-in resource limits (CPU/memory caps per container group).
- Azure Identity integration for secret-free auth to other Azure services.
- Supports both Linux and Windows containers.

## Architecture (Planned)

```
┌─────────────────────────────────────────────────────┐
│                   Caller / API                      │
│            (CLI, HTTP API, or SDK)                   │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              Orchestrator Service                    │
│  - Accepts run requests                             │
│  - Provisions ACI container groups                  │
│  - Enforces policies (resource limits, networking)  │
│  - Collects logs & results                          │
│  - Tears down containers on completion / timeout    │
└──────┬──────────────────────────────────┬───────────┘
       │                                  │
       ▼                                  ▼
┌──────────────┐                 ┌──────────────────┐
│  ACI Sandbox │                 │  ACI Sandbox     │
│  (agent run) │                 │  (bot run)       │
│              │                 │                  │
│  - Isolated  │                 │  - Isolated      │
│  - Ephemeral │                 │  - Ephemeral     │
│  - Capped    │                 │  - Capped        │
└──────────────┘                 └──────────────────┘
       │                                  │
       ▼                                  ▼
┌─────────────────────────────────────────────────────┐
│              Azure VNet (optional)                   │
│  - NSG rules for outbound filtering                 │
│  - Private DNS for internal resolution              │
└─────────────────────────────────────────────────────┘
```

## Plan

### Phase 1 — Foundation

1. Define container image requirements (base images, pre-installed tooling, entrypoint conventions).
2. Create Terraform / Bicep templates for provisioning ACI container groups with:
   - Resource limits (CPU, memory).
   - VNet integration and NSG rules.
   - Managed Identity assignment.
3. Build a minimal CLI that can launch a sandbox, stream logs, and destroy it.

### Phase 2 — Orchestration

4. Implement the orchestrator service (lightweight API server) that:
   - Accepts sandbox run requests (image, env vars, timeout, resource policy).
   - Manages ACI lifecycle (create → poll → collect → delete).
   - Enforces concurrency limits and quotas.
5. Add execution timeout enforcement and automatic cleanup.
6. Integrate structured logging (Azure Monitor / Log Analytics).

### Phase 3 — Security Hardening

7. Network policies — outbound allowlists, DNS filtering, block metadata endpoint.
8. Read-only root filesystem; tmpfs for scratch space.
9. Drop all Linux capabilities; run as non-root.
10. Secret injection via Azure Key Vault references (not env vars).
11. Image signing and admission policy (only trusted images).

### Phase 4 — Observability & Operations

12. Per-run metrics: CPU/memory usage, wall-clock time, exit code.
13. Cost tracking and reporting per agent/bot.
14. Alerting on anomalous runs (OOM kills, timeouts, high egress).

### Phase 5 — Evaluate & Extend

15. Benchmark ACI cold-start latency and evaluate alternatives (ACA, AKS with virtual nodes) if needed.
16. Support multi-step agent workflows (chained sandbox runs with artifact passing).
17. Optional warm-pool for latency-sensitive workloads.

## Tech Stack (Planned)

| Component       | Choice                        |
|-----------------|-------------------------------|
| IaC             | Terraform or Bicep            |
| Orchestrator    | Go or Python                  |
| Container runtime | Azure Container Instances   |
| Networking      | Azure VNet + NSG              |
| Secrets         | Azure Key Vault               |
| Logging         | Azure Monitor / Log Analytics |
| CI/CD           | GitHub Actions                |

## Status

🚧 **Planning** — architecture and IaC templates in progress.
