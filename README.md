# Arnvoid -- 2KI (2D Knowledge Interface)

A spatial thinking environment where self-analyzing neurons exist under physical laws, each one an independent agent holding its own reasoned viewpoint about a document -- and you interact with them directly with your hands.

**Live product:** [arnvoid.com](https://arnvoid.com) . Closed source . Built solo by Maulana Jawwad Wirasyahputra

---

## What It Does

Arnvoid puts self-analyzing neurons in your hands for the purpose of deep reasoning over documents. You upload a research paper or writing -- PDF, TXT, or Markdown -- and the LLM acts as hive swarm source, spawning a colony of neurons. Each neuron is not a label on a graph. It is a self-agentic unit, independently responsible for a slice of reasoning: one neuron holds a claim, another holds the evidence supporting it, another holds a hidden assumption, another holds a limitation. Each neuron carries its own analysis, its own confidence about what it knows, its own framing of its portion of the document. Knowledge links are directed connections between neurons -- they encode which neuron's reasoning depends on, contradicts, or extends another's. You do not look at the document through a graph. You are looking at the AI's thinking process made physical and touchable. You press a neuron and it speaks -- opening a window that reveals its analysis, its confidence score, its specific reasoning about its slice of the document.

Neurons exist under physical laws, not in a decorative layout. The xPBD constraint solver governs their spatial relationships: knowledge link tension pulls connected neurons together, repulsion forces maintain readable separation, and the whole colony drifts toward equilibrium like a system finding its rest state. When two neurons are strongly linked, the physics pulls them closer -- that tension IS the relationship, made physical. When you drag a neuron, you are moving an agent through mind-space, and its neighbors follow because they are connected by law, not by a layout algorithm. The physics is the spatial reality these agents inhabit. The system is built for deep reading, research synthesis, and understanding complex documents through direct interaction with individual reasoning units.

---

## Screens

<!-- GIF_PLACEHOLDER: prompt-screen -- the onboarding/prompt entry screen with the live physics graph preview floating above the prompt input -->

<!-- GIF_PLACEHOLDER: neural-map -- the main graph screen showing neurons and knowledge links under physics simulation, with the sidebar expanded showing saved Peta Neural sessions -->

<!-- GIF_PLACEHOLDER: d1-window -- a neuron being tapped, the D1 analysis window opening with edge connectors back to the graph, showing the analysis text panel -->

---

## Architecture

The system is split into three runtime layers that serve one surface -- the user's browser, running a React SPA that renders a WebGL canvas where neurons exist under physical law. When a user submits a document on the prompt screen, the request hits the Cloudflare Worker edge layer first. The Worker validates the session cookie, checks the user's rupiah balance, executes the mode authority decision (classic vs skeleton modes), and dispatches the LLM call through a gateway seam that isolates provider details from analysis logic. The LLM acts as hive swarm source -- its structured JSON response is not a summary but a colony specification: typed neurons (claim, evidence, method, assumption, limitation, bridge, definition) each initialized with their own analytical role, connected by directed semantic edges (supports, challenges, depends_on, produces, limits). This specification is parsed, validated against a strict schema with optional semantic repair, and persisted to D1. The Worker returns the colony to the frontend, which applies it through a centralized topology control seam. The physics engine then governs how these new neurons settle into their spatial relationships under xPBD constraints, and the WebGL renderer draws the frame.

The frontend layer is organized around the AppShell -- a thin orchestration component that wires together screen rendering, overlay layering, sidebar state, and transition sequencing. Screen content, onboarding flow, modals, saved interface sync, and the graph playground are all extracted into domain-specific seams under `src/screens/appshell/`. The graph playground layer owns the physics engine lifecycle, the custom WebGL rendering loop with camera containment, and the SV2-native D1/D2 window system. When you press a neuron, the D1 window is that neuron speaking -- it reveals the agent's own analysis, confidence, and reasoning. Input events follow a strict doctrine: when any overlay or panel is open, it must fully own pointer and wheel events so the canvas never reacts underneath.

The worker layer on Cloudflare handles everything that touches user data or infrastructure. Auth is Google OAuth with server-side session cookies. Saved interfaces sync through an outbox pattern with timestamp-based ordering. Document artifacts go to R2. The billing system computes a tiered flat charge -- `ceil(charCount / 50000) * 750 Rp` -- and persists each charge to D1's `rupiah_ledger` with idempotency on `(reason, ref_type, ref_id)`. Users top up their balance via Midtrans GoPay QRIS. Every flow -- analyze, save, restore, pay -- is session-authenticated and gated through the Worker, making the edge layer the single authority for user state.

### Layer Map

| Layer | Location | Role |
|---|---|---|
| WebGL Canvas | `src/playground/rendering/` | Custom render loop, camera containment, DPR management |
| Physics Engine | `src/physics/engine/` (33 files) | xPBD solver governing spatial relationships between neurons |
| AppShell | `src/screens/appshell/` | Screen orchestration, overlay/transition sequencing, sidebar state |
| Frontend SPA | `src/` (React + Vite) | App structure, panel composition, input routing, canvas binding |
| Cloudflare Worker | `cloudflare-worker/src/` | Auth, session, LLM analyze dispatch, payment routes, D1/R2 persistence |
| LLM Pipeline | `cloudflare-worker/src/llm/` | Mode authority, gateway seam, prompt building, strict schema parsing, semantic repair |
| Billing & Ledger | `cloudflare-worker/src/payments/`, `cloudflare-worker/src/shared/common.ts` | Tiered flat charge, rupiah ledger persistence, Midtrans integration |
| Auth | `cloudflare-worker/src/auth/` | Google OAuth, server-side session cookies, CORS hardening |
| Storage | Cloudflare D1 + R2 | Relational data (sessions, saved interfaces, rupiah ledger) + document blobs |

---

## Engineering Highlights

**Custom xPBD Physics Solver as Spatial Law.** Not a D3 or force-graph wrapper. The engine in `src/physics/engine/` (33 files) implements a multi-stage tick pipeline: preflight, force calculation, semi-implicit Euler integration, Gauss-Seidel constraint iteration, post-solve history reconciliation, energy-gated spacing pass, velocity damping, and sleep detection. The solver is not a layout algorithm -- it is the spatial law that governs how neurons relate to each other in mind-space. Knowledge link tension pulls connected neurons together; that tension IS the relationship made physical. When the system falls behind real time, it drops frames rather than accumulating lag -- visual stutter over temporal syrup. The solver has explicit constraint budgets, correction residual carry-over for clipped iterations, and adaptive degradation levels (Stressed -> Fatal).

**Server-Authoritative Mode Resolution.** Three analysis modes (classic, skeleton_v1, skeleton_v2) with a shared pure resolver (`resolveAnalyzeModePure.ts`) running on both client and server. The server independently re-decides the executed mode through `resolveServerExecutedMode.ts`, preventing client-side mode manipulation. Each mode has a full registry entry defining its physics profile, UI semantics, spacing behavior, and restore policy in `modeStrategyRegistry.ts`. The mode determines how the hive swarm source spawns the neuron colony -- which roles exist, how they connect, how they settle.

**Tiered Flat-Charge Billing with Idempotency.** Every analysis is charged `ceil(charCount / 50000) * 750 Rp`, computed on the Worker and persisted to D1 `rupiah_ledger` with a composite idempotency key on `(reason, ref_type, ref_id)`. This means network retries or worker replay cannot double-charge a user. The billing session lifecycle spans three phases: preflight balance check, mid-execution checkpoint commit, and post-execution final reconciliation. Free-pool is legacy code and does not run.

**Strict Structured JSON with Semantic Repair.** The LLM response must conform to a recursive strict schema where every `properties` key is mirrored in `required` and `additionalProperties` is false at every object level. When the raw model output fails validation, a semantic repair pass (`skeletonV2SemanticRepair.ts`) attempts to salvage the response by repairing structural issues without losing content -- a pragmatic middle ground between total failure and blind acceptance. This matters because the colony specification must be structurally sound before neurons can be born from it.

**Shared Contract Mirrors with CJS/ESM Interop.** The analyze mode contracts, SV2 schema, canonical response types, and resolution logic are maintained in three mirror files (`.ts`, `.cjs`, `.js`) that must always agree. A dedicated contract test suite (`test:mode-strategy-registry-contracts`, `test:analyze-module-interop-contracts`) catches drift between mirrors before it reaches production. CJS files are strictly gated from browser imports -- this interop discipline was hard-won from production crashes.

**Overlay-First Input Doctrine.** The canvas is the substrate where neurons exist; everything else floats above it. When any panel or overlay is open, it must fully own pointer and wheel events within its bounds so the canvas never reacts to leaked input. This is enforced through explicit `onPointerDown` stop-propagation on every interactive child, backdrop shielding, and a `pointerEvents: 'auto'` overlay wrapper. The doctrine is documented alongside the invariant that panels are "input black holes."

**Saved Interfaces Outbox Sync.** The save/restore system uses an outbox pattern: local commits happen immediately, then an async sync pushes changes to the Worker with timestamp-based ordering truth. The sync layer includes retry with backoff, identity isolation, and conflict resolution -- all in dedicated seams under `src/screens/appshell/savedInterfaces/`. Restore operations use a flight coordinator (`restoreFlightCoordinator.ts`) that queues boundary invalidations until restore settles, preventing mid-restore state corruption when a saved colony of neurons is brought back to life.

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite | SPA with WebGL canvas |
| Physics Engine | Custom xPBD (33 files) | Hand-authored constraint solver governing neuron spatial law |
| Canvas Rendering | Custom WebGL | Camera containment, DPR handling |
| Backend API | Cloudflare Workers | Auth, LLM dispatch, payments, persistence |
| Database | Cloudflare D1 | Sessions, saved interfaces, billing ledger |
| Blob Storage | Cloudflare R2 | Uploaded document artifacts |
| LLM Provider | OpenRouter -> Cerebras | Single model: `gpt-oss-120b` |
| Payments | Midtrans | GoPay QRIS top-ups |
| Auth | Google OAuth (GIS) | Server-side cookie sessions |
| Testing | Vitest, Playwright, contract suites | Mode strategy, schema parity, eval harness |
| CI/CD | GitHub Actions | Acceptance gate, eval guard |

---

## Who Built This

I am Maulana Jawwad Wirasyahputra, a software engineer with 7 years of experience spanning full-stack development, AI systems engineering, and infrastructure. I have been building production AI applications for 3 years and have operated my own software venture for 3 years. This project is the result of that experience applied end-to-end.

When I say built solo, I mean the entire system: the xPBD physics engine and its 33-file solver pipeline that governs how neurons relate in mind-space, the WebGL canvas renderer with camera containment, the AppShell architecture with its overlay sequencing and input doctrine, the LLM analysis pipeline where the hive swarm source spawns colonies of self-agentic neurons through server-authoritative mode resolution and strict schema enforcement, the Cloudflare Worker backend with auth, session management, saved interface sync, and rupiah ledger billing, the Midtrans payment integration, and the frontend UI from prompt screen through neural map. No off-the-shelf physics, no boilerplate backend framework, no third-party graph library. Every seam was an engineering decision.

---

## License

All rights reserved. This is a closed-source commercial product. The repository exists to demonstrate engineering capability -- the code itself is private.
