# Self-hosted backend plan (without CopilotKit Intelligence)

**Status:** design only — no runtime swap in this change.  
**Audience:** Happy Horizon fork operators planning a Google Cloud deployment that keeps OpenBot MIT-compatible and does **not** use, bypass, or emulate CopilotKit’s proprietary Intelligence service or its licence checks.

Today the API server refuses to start unless `INTELLIGENCE_API_URL`, `INTELLIGENCE_GATEWAY_WS_URL`, and `INTELLIGENCE_API_KEY` are set (`server/src/config.ts` → `runtimeCapabilities`). `COPILOTKIT_LICENSE_TOKEN` is optional and only forwarded when present. Architecture and configuration docs treat Intelligence as required for durable threads, memory/learning, and the realtime gateway.

This plan inventories those dependencies, proposes a GCP-backed replacement using CopilotKit’s public TypeScript `AgentRunner` seam, and phases the work so upstream merges stay manageable.

**Non-goals**

- Running or reverse-engineering CopilotKit Intelligence (managed or self-hosted Helm/ECS).
- Emulating Intelligence licence / entitlement APIs (`getRuntimeEntitlements`, licence tokens).
- Changing governance (computer gateway, policy, audit) except where thread identity must point at a new store.

**Verification note (this PR)**  
No server was started and no Intelligence credentials were used. Conclusions come from reading source, Helm/chart values, `docs/*`, CopilotKit’s public AgentRunner docs (`https://docs.copilotkit.ai/backend/agent-runner`), and the Google provider wiring in `agent-langgraph` / `shared/model-providers.json`.

---

## 1. Inventory: where OpenBot depends on Intelligence

Rough count: **~45 distinct call sites / modules** across server, app, desktop, Helm, scripts, and docs (plus a large test surface that mocks Intelligence). Grouped by concern.

### 1.1 Startup validation and config surface

| Path | What it uses Intelligence for |
| --- | --- |
| `server/src/config.ts` | `RuntimeCapabilities` is a single mode: `mode: "intelligence"`. Requires the three `INTELLIGENCE_*` vars as a set; reads optional `COPILOTKIT_LICENSE_TOKEN`. Comment block states Intelligence is required for durable threads and memory. |
| `docs/configuration.md` | Documents the three required vars + optional licence token. |
| `.env.example` | Ships the Intelligence env contract for local/desktop setup. |
| `charts/openbot/values.yaml` | `config.intelligence.{apiUrl,gatewayWsUrl}` + secrets `intelligenceApiKey` / `licenseToken`. |
| `charts/openbot/ci/gke-values.yaml` (and other CI values) | Example GKE values still point at managed Intelligence URLs and Secret Manager keys for the API/licence tokens. |
| `docs/deployment.md` / `docs/architecture.md` / `README.md` | Product narrative: threads/memory/realtime live in Intelligence; Postgres holds product data only. |

### 1.2 Runtime client, runner, and CopilotKit mount

| Path | What it uses Intelligence for |
| --- | --- |
| `server/src/intelligence-client.ts` | Builds `CopilotKitIntelligence`; proxies auth failures (401) into the desktop “Intelligence connection failed” path. |
| `server/src/copilot.ts` | **Primary integration.** Always mounts `CopilotRuntime` with `intelligence:` (no non-Intelligence branch). Subclasses `CopilotKitIntelligence` as `IntelligenceKnowingANewThread` so empty/new threads do not 500 on history. Passes `licenseToken`. Exposes `runnerConnection()` from `ɵgetRunnerWsUrl` / `ɵgetRunnerAuthToken`, thread lock acquire/renew/release via `ɵacquireThreadLock` / `ɵrenewThreadLock` / `ɵcleanupThreadLock`, and `history` via `getThreadMessages`. Wires Automatic Learning container assignment into `getOrCreateThread`. |
| `server/src/index.ts` | Process-wide `routineIntelligence` + `IntelligenceAgentRunner` for routines, handoffs, responsibilities, group turns, proactive follow-ups. Uses `getThreadMessages` for history replay. Builds a second client for the self-host banner entitlements. Comment explicitly says `RuntimeCapabilities` has one mode and any second mode must grow a guard here. |
| `server/src/routines/run-turn.ts` | Headless turn engine: mirrors Intelligence channel-manager behaviour. Depends on `getOrCreateThread`, `getThreadMessages`, and the three `ɵ*ThreadLock` methods, then drives `IntelligenceAgentRunner.run/stop`. Documents that the realtime gateway is on the critical path for routines. |
| `server/src/agents/handoff-runner.ts` (via `index.ts`) | Handoff delivery uses a fresh `IntelligenceAgentRunner` from `copilotRuntime.runnerConnection()`. |
| `app/src/lib/copilot/provider.tsx` | Browser talks only to `/api/copilotkit` with session cookies — **no Intelligence secrets in the browser**. Still depends on the server runtime being in Intelligence mode for thread/stream behaviour. |
| `server/src/app.ts` | `/api/capabilities` projects `mode` / `durableHistory` but **must not** expose Intelligence keys (already guarded). Self-host banner projection ties to Intelligence entitlements. |

### 1.3 Thread creation, listing, and per-bot / channel mapping

| Path | What it uses Intelligence for |
| --- | --- |
| `server/src/db/schema/core.ts` → `intelligence_channel_mappings` | Postgres maps `(userId, channelId) → threadId`. The **thread body** (messages/runs) lives in Intelligence; the mapping is local. |
| `server/src/db/schema/group.ts` → `group_bot_threads` | One Intelligence thread per Bot in a group (Intelligence admits one agent per thread). Shared human transcript is local (`group_messages`). |
| `server/src/channels/routes.ts` | On channel create/list, inserts/joins `intelligence_channel_mappings` with minted thread ids. |
| `server/src/channels/group.ts` | Creates/deletes mappings; `groupSourceForRun` resolves group threads via `group_bot_threads` (not the channel mapping table). |
| `server/src/channels/thread-identity.ts` | Embeds a deployment fingerprint in UUID v8 thread ids so one shared Intelligence **project** can separate deployments. Less critical if threads live only in our Postgres, but still useful for multi-env hygiene. |
| `server/src/channels/thread-routes.ts` | `POST /mint` (local) + `GET /:threadId` that asks Intelligence whether the thread still exists (avoids silent empty-thread resurrection). |
| `server/src/channels/thread-status.ts` | `createThreadReader` → `intelligence.getThread`. |
| `server/src/channels/summary.ts` | Channel summarisation reads history with `getThreadMessages` (no lock). |
| `server/src/channels/attachments.ts`, `server/src/activity/activity.ts`, `server/src/passwords/resume.ts`, and several `index.ts` lookups | Resolve channel from `threadId` via `intelligence_channel_mappings`. |

### 1.4 History replay

| Path | What it uses Intelligence for |
| --- | --- |
| `server/src/copilot.ts` (`history`, `IntelligenceKnowingANewThread`) | Replay for hops / UI-adjacent reads; treats 404-as-empty for brand-new threads. |
| `server/src/index.ts` | Routine / delivery paths re-fetch messages after a short delay because Intelligence persists slightly behind the stream. |
| `server/src/routines/run-turn.ts` | Seeds headless turns with `persistedInputMessages` from Intelligence history. |
| `app/src/lib/copilot/thread-messages.ts`, channel chat components | Consume history through CopilotKit / channel APIs that ultimately hit the Intelligence-backed runtime. |

### 1.5 Memory vs Automatic Learning (important distinction)

OpenBot already has **product personal memory in Postgres**, independent of Intelligence User Memories:

| Path | Role |
| --- | --- |
| `server/src/memory/*` (`store.ts`, `tools.ts`, `ingestion.ts`, `routes.ts`) | `personal_memories` / `memory_sources` in Cloud SQL–compatible Postgres. Injected into runs via `PersonalMemoryMiddleware`. |
| `server/src/db/schema/memory.ts` | Schema for the above. |

Intelligence **Automatic Learning** (evidence → skill candidates → published skills) is a separate product surface and **does** depend on Intelligence:

| Path | What it uses Intelligence for |
| --- | --- |
| `server/src/learning/runtime.ts` | `CopilotKitIntelligence` for thread binding, `getInspectorLearning`, skill snapshot delivery. |
| `server/src/learning/routes.ts` | Admin Learning UI; errors name “Intelligence Learning”. |
| `server/src/learning/settings.ts` / `server/src/db/schema/learning.ts` | Local settings + `learning_thread_bindings`; container IDs are Intelligence resources. |
| `server/src/learning/remote.ts` | Forwards verified skill snapshots to remote Bots (credential stays on server). |
| `scripts/setup-learning.ts` (+ tests) | Provisions Intelligence Learning containers / keys. |
| `docs/automatic-learning.md` | Operator runbook for managed and self-hosted Intelligence Learning. |
| Env: `CPK_INTELLIGENCE_LEARNING_CONTAINER_ID`, `CPK_INTELLIGENCE_SKILLS_REVISION` | Optional defaults, not startup-required. |

Architecture diagrams that say “memory lives in Intelligence” conflate platform memory/learning with OpenBot’s own Postgres memory. For a Happy Horizon swap, **keep OpenBot personal memory**; treat Intelligence Learning as out of scope until a later phase (or permanently omit).

### 1.6 Realtime gateway / WebSocket

| Path | What it uses Intelligence for |
| --- | --- |
| `IntelligenceAgentRunner` + `INTELLIGENCE_GATEWAY_WS_URL` | AG-UI stream durability, reconnect/`connect`, cross-replica run join, event acknowledgement deadlines (documented in `run-turn.ts`). |
| `server/src/copilot.ts` `runnerConnection()` | Anything that drives a run outside the browser (routines, handoffs) must use the same gateway credentials the runtime uses. |
| `server/src/channels/socket.ts` | **Not** the Intelligence gateway — OpenBot’s own channel event WebSocket. Survives a swap, but multi-replica fan-out of channel events is a separate concern (Postgres `LISTEN`/`NOTIFY` or Redis). |

### 1.7 Licence token and entitlements (do not emulate)

| Path | What it uses Intelligence for |
| --- | --- |
| `COPILOTKIT_LICENSE_TOKEN` / `config.runtime.intelligence.licenseToken` | Forwarded into `CopilotRuntime` for self-hosted Intelligence that issues a licence. Managed Intelligence often has none. |
| `server/src/self-host-banner.ts` | Calls `getRuntimeEntitlements()` to hide the promotional self-host bar on paid plans. |
| Desktop `desktop/src/App.tsx` / tests | Full managed + self-hosted Intelligence sign-in UX (project pick, provisioned key, API/WS URLs). |

**Replacement stance:** under `THREAD_BACKEND=postgres`, skip licence token entirely, force self-host banner off (or always-on for the fork), and stop calling entitlements. Do **not** fake paid-plan responses.

### 1.8 Desktop, Helm, docs, tests (secondary but large)

| Area | Notes |
| --- | --- |
| `desktop/src/App.tsx`, `ProviderPicker.tsx`, extensive `App.test.tsx` | Setup wizard is Intelligence-centric. Fork setup should eventually offer “Postgres / self-hosted backend” instead of “Point at your own Intelligence server”. |
| Helm / compose / CI | Charts and docs assume Intelligence secrets. GKE CI values still reference managed Intelligence. |
| Tests under `server/tests/*` | Many integration tests stub Intelligence clients, locks, or runner behaviour (`thread-*.test.ts`, `routine-*.test.ts`, `learning-*.test.ts`, etc.). |
| Docs | `architecture.md`, `configuration.md`, `deployment.md`, `automatic-learning.md`, `development.md`, `releasing.md`, `coworkers.md`. |

---

## 2. Proposed replacement architecture (Google Cloud)

```text
                    ┌─────────────────────────────┐
                    │  App (React)                │
                    │  CopilotKitProvider →       │
                    │  /api/copilotkit (cookies)  │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────▼───────────────┐
                    │  OpenBot server (GKE)       │
                    │  CopilotRuntime +           │
                    │  PostgresAgentRunner        │
                    │  (THREAD_BACKEND=postgres)  │
                    └──────┬───────────┬──────────┘
                           │           │
              ┌────────────▼──┐   ┌────▼────────────────┐
              │ Cloud SQL     │   │ Memorystore Redis    │
              │ Postgres+     │   │ (phase 3+: pub/sub   │
              │ pgvector      │   │  for multi-replica)  │
              │ threads,      │   └─────────────────────┘
              │ history,      │
              │ personal      │   ┌─────────────────────┐
              │ memory,       │   │ Vertex AI Gemini    │
              │ product data  │   │ (ADC / SA)          │
              └───────────────┘   └─────────────────────┘

              Computers: prefer computers.mode=sandbox
              (agent-sandbox CRD), not Docker-socket supervisor
```

### 2.1 Threads and history — custom `AgentRunner` on Cloud SQL

CopilotKit’s TypeScript runtime already supports replacing Intelligence with a runner:

- Abstract `AgentRunner`: `run`, `connect`, `isRunning`, `stop`.
- Built-ins: `InMemoryAgentRunner`, `SqliteAgentRunner` (`@copilotkit/sqlite-runner`), `IntelligenceAgentRunner` (Intelligence only).
- Docs recommend subclassing `InMemoryAgentRunner` (or implementing `AgentRunner`) for Postgres/Redis; AWS AgentCore is the reference pattern.
- **Intelligence runtimes reject a custom `runner`** — so the OpenBot mount must use the **non-Intelligence** `CopilotRuntime` constructor path when `THREAD_BACKEND=postgres`.

**Proposal:** implement `PostgresAgentRunner` in this fork (e.g. `server/src/runner/postgres-agent-runner.ts`):

- Persist thread metadata, runs, and AG-UI event/message snapshots in Cloud SQL (reuse existing `DATABASE_URL` / OpenBot Postgres; enable `vector` only if we later embed).
- Own thread creation/listing/history APIs that today go through `CopilotKitIntelligence` (`getOrCreateThread`, `getThread`, `getThreadMessages`).
- Replace Intelligence thread locks with Postgres row locks / `SELECT … FOR UPDATE` (or lease columns on a `thread_locks` table), shared across replicas — same pattern OpenBot already uses for `work_items`.
- Keep `intelligence_channel_mappings` / `group_bot_threads` as **local** maps (optionally rename later); only the backing store for message bodies changes.
- Phase 1 may start from `SqliteAgentRunner` or an `InMemoryAgentRunner` subclass that flushes to Postgres after each run, then harden to a first-class durable runner.

**Do not** call Intelligence HTTP/WS endpoints from this path.

### 2.2 Memory

| Option | Fit | Trade-offs |
| --- | --- | --- |
| **A. Keep OpenBot `personal_memories` (recommended default)** | Already MIT, in Postgres, wired into runs and tools. | No automatic fact extraction from transcripts; no Intelligence Learning skills. |
| **B. pgvector embeddings on Cloud SQL** | Same DB; was used historically for document chunks (dropped in migration `0010`). Good for similarity recall later. | You build extraction, chunking, and retention; ops stay in-house. |
| **C. Vertex AI Memory Bank** | Managed extract/consolidate/retrieve across sessions; ADC-friendly on GCP. | New Google dependency and billing; API oriented around Agent Engine sessions; does not replace thread history; couples memory UX to Google’s product cadence. |

**Recommendation:** Phase 1–2 keep option A only. Revisit B vs C when Automatic Learning / long-term recall becomes a product requirement. Do not block the Intelligence removal on Memory Bank.

### 2.3 Realtime and multi-replica behaviour

| Phase | Approach |
| --- | --- |
| **Start** | **Single server replica.** `InMemoryAgentRunner`-style connect works in-process; Postgres holds durable history across restarts. Routines CronJob + work queue already serialize background work via Postgres. |
| **Later** | **Memorystore for Redis** pub/sub (or Postgres `LISTEN`/`NOTIFY` for lighter fan-out) so `connect()` / live AG-UI events reach the replica holding the socket, and channel WebSocket events fan out. Thread locks must already be in Postgres before scaling out. |

Risk if you scale replicas before shared locks/pubsub: duplicate runs, lost reconnects, or “thread already running” races — the PR template already calls this out.

### 2.4 Hosting on GKE (computers / supervisor)

| Mode | GKE fit |
| --- | --- |
| Docker **supervisor** (`supervisor/`, Docker socket) | Laptop / VM. Holding a Docker socket in GKE is operationally painful (privileged DaemonSet / DinD) and a large blast radius. **Not recommended** as the primary Happy Horizon path. |
| Helm `computers.mode: shared` | One shared browser StatefulSet — simplest GKE start; shared logins/files across Bots. |
| Helm `computers.mode: sandbox` | Per-Bot `Sandbox` via `kubernetes-sigs/agent-sandbox` (chart already supports this; EKS sandbox CI values exist). **Preferred** for per-Bot isolation on GKE without Docker socket. Requires controller install + `networkPolicy.kubernetesApiCidr`. |
| `computers.mode: external` | Point at a computer service you run elsewhere. |

`charts/openbot/ci/gke-values.yaml` today uses `computers.mode: shared` and still configures Intelligence — swap those secrets for Postgres-backend config when implementing.

Server, worker CronJobs, Cloud SQL (existing secret pattern), Workload Identity, and Gateway API already match Happy Horizon’s target shape.

### 2.5 Gemini via Vertex AI (service account)

**Current behaviour**

- `BOT_PROVIDER=google` is supported (`shared/model-providers.json`: `GOOGLE_API_KEY` + optional `GOOGLE_GENERATIVE_AI_BASE_URL`).
- `agent-langgraph` uses `ChatGoogleGenerativeAI` from `@langchain/google-genai` with an **API key** (Google AI Studio / Generative Language API style), not Vertex ADC.

**Change needed for Vertex + Workload Identity**

1. Add a Vertex path (new provider id such as `google-vertex`, or a flag like `GOOGLE_GENAI_USE_VERTEX=true`) using `@langchain/google-vertexai` / `ChatVertexAI` (or the current LangChain Vertex chat class for the pinned version).
2. Auth via Application Default Credentials from the GKE service account (`iam.gke.io/gcp-service-account` already in gke-values) — **no** `GOOGLE_API_KEY` required when ADC works.
3. Set project/location (`GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` / Vertex location).
4. Mirror the same choice in Python bots that read `BOT_PROVIDER` (`shared/model_providers.py`) if those harnesses are deployed.
5. Desktop/setup docs: “Google API key” vs “Vertex on GKE”.

Until that lands, Happy Horizon can still call Gemini with an API key on GCP, but that is not the service-account posture the org wants long term.

---

## 3. Making Intelligence optional (`THREAD_BACKEND`)

Goal: **small, reviewable diffs against upstream** while the fork can run without Intelligence.

### 3.1 Config switch

```text
THREAD_BACKEND=intelligence|postgres   # default: intelligence (upstream-compatible)
```

| Value | Behaviour |
| --- | --- |
| `intelligence` (default) | Today’s behaviour: require `INTELLIGENCE_*`; mount Intelligence runtime; unchanged for upstream merges. |
| `postgres` | **Do not require** Intelligence env vars or licence token. Mount `CopilotRuntime` with `runner: PostgresAgentRunner` (no `intelligence:`). Implement local thread/history/lock ports used by routines and handoffs. Disable or no-op Automatic Learning and entitlements. |

Optional companions:

- `OPENBOT_SELF_HOST_BANNER=false` (already exists) — set in fork defaults.
- Later: `REALTIME_BACKEND=none|redis` once multi-replica lands.

### 3.2 Code structure to keep the upstream diff small

1. **Widen `RuntimeCapabilities`** in `config.ts` to a discriminated union (`mode: "intelligence" | "postgres"`) instead of scattering `if` checks — `index.ts` already names this as the line that must grow.
2. **Extract an internal `ThreadStore` / `Transcript` / `ThreadLock` interface** from the methods `IntelligenceLike` already documents in `run-turn.ts`, so routines/handoffs depend on the interface, not `CopilotKitIntelligence`.
3. **Keep Intelligence path verbatim** behind `mode === "intelligence"` so cherry-picks/merges mostly conflict in `config.ts` + a thin factory in `copilot.ts`.
4. **Put Postgres runner + SQL migrations in new files** (`server/src/runner/*`, new drizzle migration) — low merge conflict.
5. **Do not delete** desktop Intelligence UX in the first implementation PR; gate it or leave it for operators still on `intelligence`. Fork product polish can follow.
6. **Learning:** under `postgres`, refuse Learning admin actions with a clear “not configured” rather than calling Intelligence.

### 3.3 What stays shared

- Channel membership, audit, policy, computers, plugins, personal memory tables.
- Thread id minting (`thread-identity.ts`) — still useful.
- Mapping tables (names can stay `intelligence_channel_mappings` until a rename migration is worth the churn).

---

## 4. Phased implementation plan

Relative size: **S** ≈ small config/docs, **M** ≈ one subsystem, **L** ≈ cross-cutting, **XL** ≈ multi-service product.

### Phase 1 — Minimal working Postgres threads (single replica) — **L**

**Outcome:** Server starts with `THREAD_BACKEND=postgres` and no Intelligence env. A person can chat in a channel; history survives process restart via Cloud SQL. Single API replica.

- Config union + startup validation fork.
- `PostgresAgentRunner` (or Sqlite→Postgres bridge) wired into `mountCopilotRuntime`.
- Local implementations of get-or-create thread, get thread, get messages, and locks for interactive runs.
- Headless paths either deferred (routines/handoffs disabled with clear errors) **or** included if the `ThreadStore` interface is ready — prefer including interactive + one headless path if time allows.
- Helm/env examples for Happy Horizon; docs pointer from `docs/configuration.md`.
- Tests with Testcontainers/pg against the new runner.

**Main risks:** CopilotKit runtime upgrades (OpenBot is alpha; already pinning `@copilotkit/runtime` 1.77) break private assumptions; `run-turn.ts` already warns that `ɵ*` APIs are unstable. Prefer public `AgentRunner` + our own SQL over copying more Intelligence internals.

### Phase 2 — Headless parity (routines, handoffs, group turns) — **L**

**Outcome:** Routines CronJob, handoffs, responsibilities, and group Bot turns persist into the same Postgres threads without Intelligence locks/gateway.

- Finish `ThreadStore` for all `IntelligenceLike` / runner call sites in `index.ts`.
- Ensure lock TTL/renewal matches today’s “person cannot chat while lock held” UX.
- Replay timing: remove Intelligence-specific “persist lags the stream” retries where our writer is synchronous.

**Main risks:** Subtle lock/lease bugs lock users out of channels; multi-agent group threads must keep one-agent-per-thread semantics locally.

### Phase 3 — Memory / learning product decisions — **M–L**

**Outcome:** Explicit Happy Horizon choice among: Postgres personal memory only; pgvector recall; Vertex Memory Bank; custom learning pipeline.

- Default: document that Automatic Learning is unavailable without Intelligence and is not emulated.
- Optional: pgvector-backed recall or Vertex Memory Bank integration behind `MEMORY_BACKEND=…`.

**Main risks:** Scope creep; confusing OpenBot personal memory with Intelligence Learning in UX copy.

### Phase 4 — Realtime + multi-replica — **L**

**Outcome:** Horizontal server replicas on GKE with correct reconnect/`connect` and channel event fan-out.

- Memorystore Redis pub/sub (or equivalent) for live streams + channel sockets.
- Load tests for concurrent runs and CronJob + API replica interaction (work queue should already serialize).

**Main risks:** Shipping multiple replicas before shared locks/pubsub (silent production bugs). Stick to single replica until this phase completes.

### Phase 5 — GKE hardening + Vertex Gemini — **M**

**Outcome:** Production-shaped chart values: Cloud SQL, Workload Identity, `computers.mode: sandbox` (or shared), Vertex Gemini via ADC, Intelligence secrets removed from Happy Horizon values.

- Vertex provider wiring (see §2.5).
- Fork defaults: `THREAD_BACKEND=postgres`, banner off, no Intelligence secret refs.
- NetworkPolicy / sandbox controller runbook for the cluster.

**Main risks:** agent-sandbox operational learning curve; Vertex quota/region; image pull and computer cold-start latency.

---

## 5. Open questions for Happy Horizon

1. **Default backend in the fork:** Keep upstream-compatible `THREAD_BACKEND=intelligence` default, or default the fork to `postgres` and document Intelligence as unsupported?
2. **Automatic Learning:** Permanently out of scope, or schedule a Happy Horizon–owned learning pipeline (and if so, pgvector vs Vertex Memory Bank)?
3. **Computers on GKE:** Start with `shared` (faster) or invest immediately in `sandbox` + agent-sandbox controller?
4. **Multi-replica timing:** Is a single API replica acceptable for the first internal production, or must HA land before first users?
5. **Gemini auth:** Accept `GOOGLE_API_KEY` temporarily, or is Vertex ADC a launch blocker?
6. **Desktop / one-container story:** Does the fork still need the Tauri desktop Intelligence wizard, or is GKE/Helm the only supported path?
7. **Upstream merge policy:** How often to merge CopilotKit/openbot `main`, and who owns conflict resolution on `config.ts` / `copilot.ts`?
8. **Rename trap:** Rename `intelligence_channel_mappings` early (cleaner) or keep the name indefinitely (smaller diffs)?
9. **Compliance:** Any requirement that conversation bytes never leave the VPC (rules out managed Memory Bank and any residual CopilotKit SaaS)?
10. **Licence posture:** Confirm legal/engineering agreement that we will not run CopilotKit’s proprietary Intelligence charts even if a trial key is available — this plan assumes yes.

---

## 6. Recommended approach (summary)

1. **Inventory:** Intelligence is deeply wired (~45 modules/surfaces): startup, `CopilotRuntime` mount, `IntelligenceAgentRunner`, thread mappings, history, headless turns, Automatic Learning, entitlements/licence, desktop setup, Helm, and docs. OpenBot **personal memory is already Postgres** and should not be conflated with Intelligence.
2. **Replace** durable threads/history/locks with a **Postgres `AgentRunner`** on existing Cloud SQL; make Intelligence optional via **`THREAD_BACKEND=intelligence|postgres`** with a discriminated `RuntimeCapabilities` and a small `ThreadStore` interface so upstream merges stay localized.
3. **Ship phases:** (1) single-replica Postgres chat history → (2) headless parity → (3) memory/learning decision → (4) Redis multi-replica → (5) GKE sandbox + Vertex ADC.
4. **Do not** emulate Intelligence licence or entitlements; turn the self-host banner off in the fork and omit Learning until a deliberate replacement exists.
