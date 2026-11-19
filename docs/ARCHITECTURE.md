# Marketing Manager — Architecture Spec

Status: **draft, decisions locked through the MVP design discussion**. This
document consolidates the architecture review conversation into a single
reference. Open items are called out explicitly at the bottom — this is not
a complete spec yet.

## 1. System overview

A self-hosted AI marketing agency stack. One human operator runs a fleet of
AI agents that onboard clients, execute marketing/SEO/creative work, and
route everything through governance gates (budget caps, human approval)
before anything goes live.

### Tiers

1. **Agency Command Center** — Agency Portal (dashboard): Add Client intake,
   Manage Clients directory, approval queue, budget readouts.
2. **Nervous System & Provisioning** — n8n: webhook router, onboarding
   provisioner, GPU job dispatcher.
3. **Control Plane & Observability** — Paperclip (org chart, task queue,
   locks, approval gates, budget enforcement) + Langfuse (LLM tracing &
   spend tracking).
4. **Agent Workforce** — Claude/Claude Code (strategy, creative, dev),
   Hermes (account management, SEO), Maus (social, production ops).
5. **Workstations & Tools** — Postiz (social scheduling), OpenSEO (crawls/
   SERP audits), Gemini MCP (vision critique, search), Cloud GPU farm
   (ComfyUI, Duix-Avatar).

### Flows

- **Full Architecture** — everything, all tiers.
- **Add Client (Onboarding)** — Portal → n8n → provisions Postiz/OpenSEO
  workspace + Paperclip kickoff ticket.
- **Manage Clients (Ops & Approval)** — Portal → Paperclip task/approval
  queue → agent workforce → Langfuse trace/spend → back to Portal.

## 2. Multi-tenancy model

**One shared install per service, isolated by `client_id` — not separate
installs per client.** The only resource that scales per-client is budget/
priority, not infrastructure:

| Service | Isolation mechanism |
|---|---|
| Paperclip | Rows scoped by `client_id`; tasks, locks, budget state all per-client |
| Langfuse | Per-client project/tag |
| Postiz / OpenSEO | Per-client workspace within one shared instance |
| GPU farm | Per-client job priority/quota in the n8n dispatcher, not dedicated hardware |

Paperclip is the **source of truth** for client config: budget thresholds,
service tier, enabled services. Other services read from or report back to
it rather than enforcing tenancy independently.

## 3. Infrastructure (MVP: 1–10 clients)

| Component | Choice | Notes |
|---|---|---|
| Core VPS | 4 vCPU / 16 GB RAM / 160 GB NVMe, ~20 TB bandwidth | Hetzner CPX/CCX preferred on price/perf — **verify current availability/pricing before provisioning**, pricing changed mid-2026 and the CX line may be unavailable for new orders in some regions. Contabo/OVHcloud as fallback. |
| Services on core VPS | Agency Portal, n8n, Paperclip + Postgres, Postiz, OpenSEO | Single Docker Compose stack |
| LLM tracing / spend | **Langfuse Cloud** (hosted) | Avoids self-hosting Postgres+ClickHouse+Redis+MinIO; revisit self-hosting only if scale or data-residency needs force it |
| Reverse proxy / edge | Cloudflare | TLS, CDN, DDoS protection in front of the VPS |
| Media storage | Cloudflare R2 | GPU farm writes rendered media (images/video) directly here, never to VPS disk — keeps disk flat regardless of client volume |
| GPU compute | Rented on-demand (RunPod / JarvisLabs), per-second billing | No owned GPU hardware — jobs are bursty, dispatched async via n8n. Confirm current per-hour rates before budgeting; third-party pricing aggregators disagree significantly as of late 2026. |

Growth tier (10–50 clients, if/when needed): 8–16 vCPU / 32 GB RAM, split
into app-tier + data-tier boxes, self-hosted Langfuse reconsidered at that
point.

## 4. Connectivity model

Two distinct concerns with different tooling — do not conflate them.

### 4a. Development & deployment (GitOps, no MCP needed)

1. Code changes pushed to GitHub.
2. GitHub Actions (or equivalent) SSHes into the VPS and runs
   `docker compose pull && up -d` (or `--build`).
3. For hands-on VPS work: Claude Code CLI installed directly on the VPS,
   accessed via SSH like any other dev box.

No custom "MCP control plane for the VPS" — that duplicates what SSH +
CI/CD already solves and adds unnecessary attack surface.

### 4b. Runtime agent tool access (MCP, per-service)

| Component | MCP? | How |
|---|---|---|
| n8n | Yes (free) | Native **MCP Server Trigger** node (n8n ≥1.88) exposes any workflow as a callable tool |
| Postiz | Yes | Ships its own MCP server |
| Paperclip | Yes (custom-built) | Build an MCP server alongside the REST API from day one — see §5 adapter contract |
| OpenSEO | No | Called as plain REST by Hermes; not worth wrapping for MVP |
| Langfuse | No | Transparent proxy/sink in the LLM call path, not a tool agents call |
| GPU farm | No | Dispatched via n8n → provider REST API |

## 5. Paperclip design decisions

### Task queue & locking

Locks and concurrency limits are scoped **per client** (at most per-task),
never a single global mutex/queue. A stuck approval gate for one client must
never block another client's tasks. Add a per-client concurrency cap so one
client can't starve others either.

### Agent adapter contract

All three agent families (Claude, Hermes, Maus) implement the same minimal
contract against Paperclip, regardless of what each is built on internally:

```
POST /tasks/claim
POST /tasks/:id/complete
POST /tasks/:id/fail
GET  /tasks/:id/context
```

Each agent's own adapter layer translates this into whatever it needs
internally. Paperclip only ever speaks this one contract — avoids three
divergent integrations to maintain.

### Onboarding provisioning (idempotency)

Modeled as a checkpointed state machine on the client record:

```
pending → postiz_ready → openseo_ready → workspace_ready
```

The n8n onboarding workflow writes a checkpoint to Paperclip after each
successful step. Every step is check-then-create (safe to retry). A
partial/failed onboarding is visible in the Portal at its last checkpoint
instead of silently lost; retries resume rather than restart.

### Budget enforcement

Three thresholds per client, stored on the Paperclip client record:

| Threshold | Example | Effect |
|---|---|---|
| `alert_threshold` | ₹8,000 | Notification fires. No change to task behavior. |
| `stop_threshold` | ₹9,800 | Paperclip stops letting agents **claim new tasks** for this client. Nothing new starts. |
| `hard_cap` | ₹10,000 | Safety margin only — absorbs overshoot from tasks already in flight when `stop_threshold` was crossed. **Size the gap to `stop_threshold`→`hard_cap` off the single most expensive possible task**, not a flat percentage. |

Enforcement happens **at task-claim time** in Paperclip, checking spend
pulled from Langfuse (near-real-time sync — poll interval or webhook,
tight enough that the gap above covers worst-case lag).

When `stop_threshold` is hit, affected tasks move to a `held_for_budget`
state — the same mechanism as the existing human-approval gate, surfaced
in the Portal. **Policy is uniform across all clients regardless of tier:**
reschedule / drop / raise-cap is always a human decision, never automatic.

## 6. Open items (not yet decided)

- Security / secrets handling (credential storage for client platform
  connections, VPS access control, Langfuse/Portal auth model)
- Portal approval-gate UI — exact fields/actions an operator needs to see
  to make a reschedule/drop/raise-cap decision quickly
- Client offboarding & data deletion (what happens to a departed client's
  Postiz/OpenSEO workspace, stored media in R2, Langfuse project)
- Self-hosting Langfuse migration path, if/when MVP outgrows Langfuse Cloud
- Growth-tier infra specifics (when to split app/data tiers, at what client
  count)

## 7. Source diagram

The original interactive architecture diagram (nodes/edges/flows) that this
spec was derived from is kept at
`docs/reference/interactive-architecture-diagram.html` for reference. It is
a standalone HTML/SVG/JS widget (generated via an AI design tool) — treat it
as a visual reference only, not something to build on top of; this document
is the source of truth going forward.
