# Arnvoid (beta)

<img src="screens/Arnvoid 1.gif" alt="Front screen of webapp" width="100%" /></td>

Arnvoid create an AI operating system named 2KI (2D Knowledge Interface). This system teach an LLM to process information as a series of neurons in a dark spatial environment. The neuron represent a self-thinking unit - a unit that self-analyze and self-intrepret information for user in a deep reasoning analysis. The neuron grow itself, connect itself, creating a network of neurons serve to run under user's interaction. 

**Live product:** [arnvoid.com](https://arnvoid.com)
Closed source. Built solo over 9 months.

---

## What It Does

Upload a research paper. The AI breaks it into neurons — self-analyzing thought units — connected by knowledge links on a physics-driven spatial map. Tap a neuron to open its analysis window. Interact with the neurons further to synthesize insights.

<table>
      <tr>
        <td><img src="screens/Arnvoid 2.gif" alt="Neural Map" width="100%" /></td>
        <td><img src="screens/Arnvoid 3.gif" alt="Analysis from Neuron Unit" width="100%" /></td>
      </tr>
      <tr>
        <td align="center">A Neural Map</td>
        <td align="center">Analysis from Neuron Unit</td>
      </tr>
</table>


---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite | SPA with WebGL canvas |
| Physics engine | Custom xPBD | Hand-authored constraint solver |
| Canvas rendering | HTML Canvas 2D | Camera containment and DPR handling |
| Backend API | Cloudflare Workers | Auth, LLM dispatch, payments, persistence |
| Database | Cloudflare D1 | Sessions, saved interfaces, billing ledger |
| Blob storage | Cloudflare R2 | Uploaded document artifacts |
| LLM provider | Cerebras | Current model: `gpt-oss-120b` |
| Payments | Midtrans | GoPay QRIS top-ups |
| Auth | Google OAuth (GIS) | Server-side cookie sessions |

---

## Architecture

Arnvoid is split into three runtime layers: the browser app, the Cloudflare Worker API, and Cloudflare storage. The browser owns the interactive neural map. The Worker owns authenticated application state, LLM execution, billing, and payments. D1 and R2 persist sessions, saved maps, ledger entries, and uploaded document artifacts.

### Runtime Layers

| Layer | Location | Owns |
|---|---|---|
| Browser app | `src/` | React UI, WebGL graph rendering, xPBD physics, D1/D2 windows, saved-map restore UI |
| Worker API | `cloudflare-worker/src/` | Google OAuth sessions, analyze requests, mode resolution, LLM provider routing, billing, payment routes |
| Storage | Cloudflare D1 + R2 | Sessions, saved interfaces, rupiah ledger rows, document artifact blobs |

### Analysis Flow

1. The user submits a research paper, an instruction, or both from the prompt screen.
2. The Worker validates the `arnvoid_session` cookie and checks the user's rupiah balance.
3. The server resolves the executed analysis mode instead of trusting the client request directly.
4. The LLM pipeline builds the prompt, calls the configured provider, and expects strict graph-shaped JSON.
5. The response is validated against the schema, repaired when possible, charged, and persisted.
6. The browser receives the graph payload and applies it through the topology control layer.
7. The xPBD solver positions the neurons while the WebGL renderer draws the map.
8. Pressing a neuron opens a D1 window with that neuron's analysis.

### Browser Runtime

The React app is organized around `AppShell`, which coordinates screen rendering, overlay order, sidebar state, transitions, saved-interface sync, and graph entry. Domain logic is kept under `src/screens/appshell/` so the shell remains orchestration-focused.

The graph runtime owns the physics lifecycle, camera containment, WebGL render loop, and SV2 D1/D2 windows. Topology updates are centralized before they reach the solver, so analysis results, restore flows, and UI actions all mutate the graph through the same control path.

Input routing is explicit. Panels, overlays, and D1 windows must own pointer and wheel events while open so the canvas cannot react underneath them.

### Worker Runtime

The Worker is the authority for user-facing state. It handles Google OAuth sessions, CORS, balance checks, analysis execution, saved-interface sync, document artifact storage, Midtrans payment routes, and D1/R2 access.

Billing uses a tiered flat charge: `ceil(charCount / 50000) * 750 Rp`. Each analysis charge is written to D1's `rupiah_ledger` with idempotency on `(reason, ref_type, ref_id)`, so retries cannot double-charge a user.

Saved interfaces sync through an outbox pattern with timestamp-based ordering. Uploaded document artifacts are stored in R2, while D1 stores session records, saved-map metadata, and ledger entries.

---

## Core Systems

| System | Main files | Responsibility |
|---|---|---|
| Physics solver | `src/physics/engine/` | Runs the xPBD simulation that positions neurons and knowledge links |
| WebGL renderer | `src/playground/rendering/` | Draws the graph, camera, labels, DPR-aware canvas output, and overlays |
| Mode authority | `src/shared/policies/analyze/`, `cloudflare-worker/src/llm/` | Resolves requested and executed analysis modes across client and server |
| LLM schema pipeline | `cloudflare-worker/src/llm/` | Builds prompts, validates structured graph JSON, and repairs invalid responses when possible |
| Billing ledger | `cloudflare-worker/src/payments/`, `cloudflare-worker/src/shared/common.ts` | Computes analysis charges and records idempotent ledger entries |
| Saved interfaces | `src/screens/appshell/savedInterfaces/`, Cloudflare D1 | Saves, restores, and syncs neural maps across sessions |
| Input ownership | `src/screens/appshell/`, `src/playground/` | Prevents graph pointer events from leaking through active panels and overlays |

### Physics Solver

The graph is not laid out with D3 or a force-graph package. Arnvoid uses a custom xPBD solver in `src/physics/engine/`. The tick pipeline covers force calculation, integration, constraint solving, post-solve reconciliation, velocity damping, and sleep detection.

The solver is bounded per frame. If runtime load is too high, the renderer drops simulation debt instead of letting graph time drift behind real time.

### Mode Authority

Analysis mode selection is resolved on both sides of the system. The client uses the shared resolver in `src/shared/policies/analyze/resolveAnalyzeModePure.ts`; the Worker resolves the executed mode again before running analysis. The server result is the authority.

Mode behavior is centralized in `modeStrategyRegistry.ts`, including physics profile, UI semantics, spacing policy, and restore behavior.

### Structured LLM Output

The LLM returns graph-shaped JSON, not free-form prose. The payload describes typed neurons and directed semantic links. Before the graph reaches the browser, the Worker validates it against the SV2 schema and attempts semantic repair when the response is structurally close but invalid.

This keeps the browser graph runtime separate from provider-specific response quirks.

### Billing

Analysis billing uses a tiered flat charge:

```text
ceil(charCount / 50000) * 750 Rp
```

The Worker writes charges to D1's `rupiah_ledger` with idempotency on `(reason, ref_type, ref_id)`. Retries and Worker replays reuse the same ledger identity instead of charging twice.

### Saved Interfaces

Saved neural maps are committed locally first, then synced to the Worker through an outbox. Timestamp-based ordering decides which write wins. Restore paths use session and restore-flight guards so stale async work cannot overwrite the active graph.

### Input Ownership

The graph canvas is the base interaction layer. Any active panel, modal, or D1 window must own pointer and wheel events inside its bounds. Overlay code uses explicit event shielding so graph drag, pan, and zoom handlers do not receive leaked input.


## License

All rights reserved. This is a closed-source product. The repository exists to demonstrate engineering capability -- the code itself is private.
