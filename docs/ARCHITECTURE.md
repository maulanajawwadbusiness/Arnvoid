# Architecture

## 1. System Overview -- Request Flow

```
 +-----------------------------+
 |  PROMPT SCREEN (EnterPrompt)|
 |  User uploads PDF + prompt  |
 +-------------+---------------+
               |
               v
 +-----------------------------+
 |  DOCUMENT PARSE             |
 |  src/store/documentStore.tsx|
 |  Worker-based PDF/TXT/MD    |
 |  -> extracted text          |
 +-------------+---------------+
               |
               v
 +-----------------------------+
 |  ANALYSIS REQUEST ASSEMBLY  |
 |  src/ai/analysisRouter.ts   |
 |  resolveAnalyzeModePure.ts  |
 |  (shared pure fn, client +  |
 |   server) proposes mode     |
 +-------------+---------------+
               |
               |  POST /api/llm/paper-analyze
               |  {document_text, instruction_text, requested_mode}
               v
 +-----------------------------+
 |  CLOUDFLARE WORKER          |
 |  (arnvoid-api-dev)          |
 |                             |
 |  1. Session cookie validate |
 |  2. Rupiah balance check    |
 |  3. analyzePreflight.ts     |
 |     - slot acquisition      |
 |     - caps rejection        |
 |  4. resolveServerExecuted   |
 |     Mode.ts                 |
 |     SERVER RE-DECIDES mode  |
 |     (choke point)           |
 +-------------+---------------+
               |
               v
 +-----------------------------+
 |  LLM GATEWAY                |
 |  llmGateway.ts              |
 |  isolates provider details  |
 |  from analysis logic        |
 |                             |
 |  -> OpenRouter -> Cerebras  |
 |     model: gpt-oss-120b     |
 +-------------+---------------+
               |
               |  structured JSON colony spec
               |  (typed neurons + directed edges)
               v
 +-----------------------------+
 |  RESPONSE PROCESSING        |
 |  skeletonV2WorkerParse.ts   |
 |  skeletonV2WorkerValidation |
 |  strict schema check        |
 |                             |
 |  if fail -> semantic repair |
 |  skeletonV2SemanticRepair   |
 +-------------+---------------+
               |
               |  validated colony
               v
 +-----------------------------+
 |  TOPOLOGY APPLY             |
 |  nodeBinding.ts             |
 |  -> topologyControl.ts      |
 |     setTopology() /         |
 |     patchTopology()         |
 |     (only mutation seam)    |
 |  -> physics engine settles  |
 |     neurons under xPBD      |
 +-------------+---------------+
               |
               v
 +-----------------------------+
 |  PERSISTENCE                |
 |  Saved Interface committed  |
 |  locally + outbox sync to   |
 |  Cloudflare D1              |
 +-------------+---------------+
               |
               v
 +-----------------------------+
 |  NEURAL MAP RENDERED        |
 |  Canvas 2D draws graph      |
 |  xPBD controls positions    |
 |  user interacts directly    |
 +-----------------------------+
```

### Flow Invariants

| Invariant | Owner | Rule |
|---|---|---|
| Server executes the final mode decision | `resolveServerExecutedMode.ts` | The client may request a mode, but the Worker independently resolves the mode that actually runs. |
| Invalid LLM JSON can be repaired before failure | `skeletonV2SemanticRepair.ts` | Structurally invalid responses enter semantic repair when recoverable. |
| Graph topology mutates through one API | `topologyControl.ts` | Graph structure changes must use `setTopology()` or `patchTopology()`. |

The client and server both use shared mode-resolution policy, but the server-side decision is authoritative. This prevents a client request from directly selecting an unsupported or unauthorized analysis strategy.

LLM output is validated before it reaches the graph runtime. If the response is close enough to the expected structure, semantic repair attempts to recover the payload without discarding the paid analysis result.

Topology control is the only supported graph mutation path. This keeps rest-length policy, directed-link handling, and spring derivation consistent across analysis, restore, and UI-driven updates.

## 2. Frontend Runtime Stack -- Layer Ownership

```
 +=====================================================+
 |  UI SURFACES                                         |
 |  Sidebar . PromptCard . ModalLayer . OnboardingChrome|
 |  (src/components/, src/screens/)                     |
 +=====================================================+
                         |
 +=====================================================+
 |  APPSHELL                                            |
 |  src/screens/AppShell.tsx (447 lines, wiring-only)   |
 |  Screen state machine . analysis session authority   |
 |  Saved interface writes . overlay sequencing         |
 |  Domain seams in src/screens/appshell/*              |
 +=====================================================+
                         |
 +=====================================================+
 |  OVERLAY LAYER                                       |
 |  SV2 D1/D2 windows (neuron speaking)                 |
 |  Projected SVG edge connectors (OverlayEdgesLayer)   |
 |  Tooltip provider . hover/focus state                |
 |  src/playground/graphPhysicsPlayground/sv2Windows/   |
 +=====================================================+
                         |
 +=====================================================+
 |  GRAPH RUNTIME SHELL                                 |
 |  GraphPhysicsPlaygroundShell.tsx (orchestration only)|
 |  Hooks/services per concern:                         |
 |    - useGraphAnalysisSessionBridge                    |
 |    - useSv2OverlaySessionController                   |
 |    - useGraphPlaygroundPerfRuntime                    |
 |    - useGraphPlaygroundSurfaceRuntime                 |
 |  src/playground/graphPhysicsPlayground/               |
 +=====================================================+
                         |
 +=====================================================+
 |  CANVAS 2D RENDERER                                  |
 |  src/playground/rendering/                           |
 |  graphRenderingLoop.ts . camera.ts                   |
 |  Camera containment . DPR management                 |
 |  Pan-led follow . spike rails . closure-aware stop   |
 +=====================================================+
                         |
 +=====================================================+
 |  xPBD PHYSICS ENGINE                                 |
 |  src/physics/engine/ (33 files)                      |
 |  Fixed-timestep scheduler (debt-drop, no time-stretch)|
 |  Force pass -> Euler integration ->                  |
 |  XPBD constraint solve -> post-solve ->              |
 |  energy-gated spacing -> velocity -> sleep detection |
 |  Constraint budgets . correction residuals           |
 |  Adaptive degradation (Stressed -> Fatal)            |
 +=====================================================+
```

### Frontend Ownership Rules

| Area | Owner | Responsibility |
|---|---|---|
| Screen orchestration | `src/screens/AppShell.tsx` | Wires screen state, analysis session authority, saved-interface writes, and overlay sequencing. |
| AppShell domains | `src/screens/appshell/` | Owns screen flow, overlays, transitions, saved interfaces, and render mapping. |
| Graph runtime shell | `GraphPhysicsPlaygroundShell.tsx` | Composes graph hooks and services without owning domain logic directly. |
| Graph rendering | `src/playground/rendering/` | Owns Canvas 2D draw loop, camera containment, DPR handling, and render guards. |
| Physics runtime | `src/physics/engine/` | Owns xPBD simulation, fixed timestep scheduling, constraint budgets, and sleep detection. |

### Input Ownership

Panels and overlays must own pointer and wheel events while active. Overlay wrappers use `pointerEvents: 'auto'`, interactive children stop `onPointerDown` propagation, and full-screen backdrops shield click-outside flows. The canvas should not receive leaked input while a panel, modal, or D1/D2 window is active.

### Frame Scheduling

The physics runtime uses a fixed timestep. When rendering falls behind, the scheduler drops accumulated simulation debt instead of stretching simulation time. Expensive passes such as spacing are energy-gated and phase-staggered across frames. Resting neurons can sleep until user interaction or graph motion wakes them.

## 3. Backend & Edge Architecture

```
 +---------------------------------------------------------------+
 |  VERCEL (frontend CDN)                                         |
 |  www.arnvoid.com                                               |
 |  React SPA build, deploys from git push to master              |
 +----------------------------+----------------------------------+
                              |
                              | HTTPS (credentials: include)
                              | cookie: arnvoid_session
                              v
 +---------------------------------------------------------------+
 |  CLOUDFLARE WORKER (arnvoid-api-dev)                          |
 |  THE LIVE BACKEND -- handles ALL active routes                |
 |                                                                |
 |  Auth:          /auth/google, /auth/logout, /me                |
 |  Saved IFs:     /api/saved-interfaces (CRUD + outbox sync)     |
 |  Doc Artifacts: /api/document-artifacts (R2 upload/retrieve)   |
 |  LLM Analyze:   /api/llm/paper-analyze                         |
 |  Payments:      /api/payments/* (Midtrans, balance, estimate)  |
 |  Beta Usage:    /api/beta/usage/today                          |
 |  Health:        /health, /config-health, /db-health            |
 |                                                                |
 |  Shell: cloudflare-worker/src/index.ts (49 lines)              |
 |  Router: cloudflare-worker/src/app/router.ts                   |
 |  LLM pipeline: cloudflare-worker/src/llm/                      |
 |    - analyzePreflight . analyzeModeAuthority                   |
 |    - analyzeBillingSession . skeletonV2Executor                |
 |    - skeletonV2WorkerAnalyzePrompt . skeletonV2WorkerParse     |
 |    - skeletonV2WorkerValidation . service.ts                   |
 |  Payments: cloudflare-worker/src/payments/service.ts           |
 |  Auth: cloudflare-worker/src/auth/ (google.ts, session.ts)     |
 +----------+-------------------------+---------------------------+
            |                         |
            v                         v
 +----------------------+   +----------------------+
 |  CLOUDFLARE D1       |   |  CLOUDFLARE R2       |
 |  (arnvoid-db-dev)    |   |  (arnvoid-docs-dev)  |
 |                      |   |                      |
 |  - users             |   |  Document artifact   |
 |  - sessions          |   |  bytes (PDF/TXT/MD)  |
 |  - saved_interfaces  |   |                      |
 |  - rupiah_ledger     |   |                      |
 |  - llm_request_audit |   |                      |
 |  - payments          |   |                      |
 +----------------------+   +----------------------+

 +---------------------------------------------------------------+
 |  GCP (legacy, decommissioned)                                  |
 |  Google Cloud Run + Cloud SQL (PostgreSQL)                     |
 |  Serves no live traffic. Migration to Cloudflare complete.     |
 +---------------------------------------------------------------+
```

### Backend Runtime

The Cloudflare Worker is the active application backend. It handles auth, LLM orchestration, billing, payments, saved interfaces, document artifacts, and persistence. It is not only an edge proxy.

### Live Resource Names

| Resource | Name | Status |
|---|---|---|
| Worker | `arnvoid-api-dev` | Live backend for production traffic |
| D1 | `arnvoid-db-dev` | Live database |
| R2 | `arnvoid-docs-dev` | Live document artifact bucket |
| GCP Cloud Run + Cloud SQL | legacy resources | Decommissioned; serves no live traffic |

The `-dev` suffix on Cloudflare resources is historical and misleading. Treat those resources as live production infrastructure.

### Backend Responsibilities

| Concern | Implementation |
|---|---|
| Auth | Google OAuth, server-side `arnvoid_session` cookie, D1-backed sessions |
| Billing | Tiered flat charge: `ceil(charCount / 50000) * 750 Rp` |
| Ledger safety | D1 `rupiah_ledger` idempotency on `(reason, ref_type, ref_id)` |
| Payments | Midtrans GoPay QRIS top-ups |
| LLM execution | Preflight balance check, server-side mode decision, provider gateway, strict schema validation, semantic repair |
| Storage | D1 for relational state, R2 for document artifacts |

Every user-state route is authenticated through the Worker. Retries and Worker replays reuse the ledger idempotency key, so an analysis cannot be charged twice for the same billing reference.
