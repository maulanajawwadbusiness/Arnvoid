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
 |  WebGL canvas draws frame   |
 |  neurons exist under        |
 |  physical law               |
 |  user interacts directly    |
 +-----------------------------+
```

The single most important decision in this flow is the server-side executed-mode choke point (`resolveServerExecutedMode.ts`): the client proposes a mode through the shared pure resolver, but the server independently re-decides which mode actually runs, preventing client-side manipulation of the analysis strategy. The second is the semantic repair pass (`skeletonV2SemanticRepair.ts`) -- when the LLM returns structurally invalid JSON, the system attempts to salvage the response by repairing structural issues without losing content, rather than failing the entire paid LLM call. The third is the topology mutation seam: all changes to the graph structure must flow through `setTopology()` or `patchTopology()` in `topologyControl.ts`, ensuring rest-length policy and spring derivation stay consistent -- no code path can mutate the graph outside this API.

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
 |  WEBGL CANVAS RENDERER                               |
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

Each layer owns only its concern and exposes a narrow seam to the layer above. The AppShell is wiring-only -- it delegates to domain-specific hooks under `src/screens/appshell/` for screen flow, overlays, transitions, saved interfaces, and render mapping. The graph runtime shell is orchestration-only -- each concern (analysis session bridge, overlay session, perf runtime, surface runtime) lives in its own hook. Pointer and wheel events are explicitly shielded at every boundary: when any overlay or panel is open, it must fully own input events through `onPointerDown` stop-propagation, `pointerEvents: 'auto'` wrappers, and full-screen backdrops -- the canvas never reacts to leaked input underneath. The physics engine runs at a fixed timestep with debt-drop: if the renderer falls behind, the scheduler drops accumulated time rather than stretching simulation time, preferring visual stutter over temporal lag. Expensive passes (spacing) are energy-gated and phase-staggered across frames, and neurons at rest enter sleep state to save CPU until interaction wakes them.

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

The entire backend runs on Cloudflare -- the Worker is not an edge proxy but a full application backend handling auth, LLM orchestration, payments, and persistence. The `-dev` naming on Worker, D1, and R2 resources is misleading: they are live production. Auth uses Google OAuth with server-side session cookies stored in D1, validated on every request. The billing system computes a tiered flat charge (`ceil(charCount / 50000) * 750 Rp`) on the Worker and persists to D1 `rupiah_ledger` with idempotency on `(reason, ref_type, ref_id)`, meaning network retries or worker replay cannot double-charge a user. Users top up their balance via Midtrans GoPay QRIS. The LLM pipeline runs entirely on the Worker: preflight balance check, server-authoritative mode decision, single gateway seam to OpenRouter-backed Cerebras (`gpt-oss-120b`), strict schema validation with semantic repair, and colony persistence. GCP infrastructure (Cloud Run, Cloud SQL) is decommissioned legacy -- the migration completed and those resources serve zero live traffic.
