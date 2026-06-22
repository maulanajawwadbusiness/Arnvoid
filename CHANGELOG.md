# Changelog

All major milestones for Arnvoid (2KI -- 2D Knowledge Interface).

## 2026-02-21 -- AppShell and Sidebar Modularization Plan

Established the modularization strategy for the four largest monolithic components: AppShell, Sidebar, ModalLayer, and CanvasOverlays. Each was documented with a bedrock report identifying concern seams, authority boundaries, and phased extraction order before any code was moved.

## 2026-02-25 -- Skeleton Physics Hardening

Fixed rest-length immutability under interaction, click-branch selection authority, focus gate sequencing, and drag-only focus hardening. The xPBD solver now uses policy-level rest length instead of spawn distance, and interaction does not rewrite constraint state.

## 2026-02-28 -- Skeleton V2 Canonicalization and Schema Wiring

Defined the SV2 canonical serialization contract with deterministic fingerprinting, built the strict schema with role-enum validation, and wired the end-to-end pipeline from LLM response parse through canonicalize to apply unit. Five hardening runs closed topology strict-apply, server branch totality, protocol parity, session ownership, and saved-interface CAS conflict.

## 2026-03-01 -- SV2 Debuggability Bedrock and Strict Schema

Added bounded in-memory SV2 debug dump store with TTL and redaction, protected debug retrieval endpoint, dev raw console logger, and runtime mode authority provenance instrumentation. Enforced recursive strict schema parity rule (every properties key mirrored in required, additionalProperties false) and centralized prompt/gate semantic parity into a single policy file.

## 2026-03-02 -- SV2 Single Authority and D1/D2 Window System

Established single reducer/state owner for SV2 window authority with one-way legacy bridge. Built SV2-native D1 and D2 window systems independent from NodePopup, with window anchor measurement, coordinate domain parity, and overlay content as single source of truth for analysis text.

## 2026-03-03 -- SV2 Cold-Boot Restore Gates and CI Acceptance Gate

Restore-flight coordinator with boundary invalidation queue, known-good session fixture, and CI-wired acceptance test suite. Track B D1/D2 window infra hardening with coordinate domain parity lock, anchor measurement churn lock, and D2 per-D1 window render selectors.

## 2026-03-06 -- AppShell Modularization (Stage 0-6)

Extracted six domain seams from monolithic AppShell.tsx: analysis session controller, saved interfaces controller, graph loading gate controller, balance modal controller, render model, and surface extraction. AppShell.tsx became a wiring-only orchestration shell.

## 2026-03-07 -- Document Artifacts Pipeline (Stage 1-10)

Built the full document artifact lifecycle: ingest-time artifact coordination, R2 upload/auth, saved interface artifact reference contract, viewer source resolution, restore loading barrier, stale session guards, and pristine persistence forensic fix. Closed the gap where saved sessions could lose their attached document.

## 2026-03-08 -- Prompt/Document Path Unification

Unified the split prompt-only and prompt+document request paths into one canonical submit contract with composite pending payload. Removed prompt/document contamination from analysis state. Save/restore truth unified for both paths.

## 2026-03-11 -- OpenRouter Credit Universal Blocker

Persistent hysteresis-based credit blocker with D1-backed state (enter at $0.50, clear at $5.00). Universal blocking overlay with graph interaction cancellation through a shared blocker-entry epoch. Offline blocking overlay added for connectivity failures.

## 2026-03-12 -- Direct Cerebras Strict Cutover and SV2 D1 Window System

Strict analysis path routed directly to Cerebras (gpt-oss-120b) through the existing strict structured client seam, with provider-boundary schema authority unchanged. SV2 D1 window system matured with typography resize, scrollbar, unified scale math, title badge, text reflow mask, and click-toggle cycle.

## 2026-03-16 -- LLM Analyze Route Modularization

Nine-phase modularization of the analyze request lifecycle into dedicated seams: terminal state, billing snapshot, executor failure telemetry, success terminal truth unification, preflight gate, mode executor contract, request orchestrator, route trim under 450 lines, and presenter extraction. Each phase produced a side report with acceptance checks.

## 2026-04-02 -- Sidebar Modularization (Phase 1-7)

Seven-phase extraction: pure helpers (placement, presentation, rename, icons), render surface decomposition into section components, row menu/rename controller with capture-phase outside-click, avatar menu controller, more menu controller, motion controller with reduced-motion detection, and dev/debug probe seam. Sidebar.tsx became a composition shell.

## 2026-04-07 -- XPBD Tick and Saved Interfaces Sync Modularization

Four-phase extraction of engineTickXPBD into constraint inventory, solver, post-solve, and diagnostics seams. Five-phase extraction of saved interfaces sync into policy, remote outbox, session bridge, hydration, and conflict resolver seams. Both became composition shells over extracted authority.

## 2026-04-11 -- Camera Containment and Premium Motion

Pan-led follow containment with drag-freeze, spike-only velocity rails, closure-aware stop, filtered feel-dt, and session-aware reseed reasons (session_replace, restore_apply, viewport_replace, continuity_break). Drag keeps visible camera frozen while containment targets update, so release does not wake from stale state.

## 2026-04-12 -- Mobile Sidebar and Touch Menu Hardening

Phone full-screen sidebar overlay with viewport-aware placement, safe-area bottom inset computation, row menu left-of-trigger placement on phone, visual viewport resize and orientation-change repositioning. Shared responsive breakpoint contract for phone/tablet/desktop classification.

## 2026-04-13 -- Phone Sidebar Auto-Collapse (Wave 1-3)

Acceptance-aware phone-only auto-collapse for document viewer, balance, create new, search interfaces, profile, and logout actions. Each handler returns a boolean acceptance result so collapse fires only when the action is accepted. Tablet and desktop remain unchanged.

## 2026-04-14 -- Touch Interaction System (Pan/Zoom Stages 1-7)

Complete touch interaction stack: one-finger pan with momentum and settle behavior, two-finger pinch-zoom with camera authority, wheel zoom with controlled smooth response, user authority over containment, release settle, and active affordance hints. Seven stages with adversarial review at each stage.

## 2026-04-22 -- GCP to Cloudflare Migration Complete

Full backend migration from Google Cloud Run + Cloud SQL to Cloudflare Worker + D1 + R2: auth, saved interfaces, document artifacts, payments, and LLM analyze all now run on the Worker edge. GCP infrastructure serves zero live traffic after this point; COOP popup fix and dev login bootstrap added for Google Identity Services interoperability and Playwright testing.

## 2026-05-01 -- Touch Hardening Wave

Comprehensive touch hardening across the entire UI: sidebar session row (long press, highlight, text immunity), search overlay (fade presence, close control, scrollbar), profile button (hold press feedback), payment modal (state choreography), child modals (fade unification, scrim order, pressed feedback), enterPrompt (login overlay, tagline, device close), and appshell overlays (scroll owner, three-line input, phone portrait reachability).

## 2026-05-03 -- Peta Neural Naming and D1 Typography

User-facing graph map renamed from Antarmuka to Peta Neural. D1 body paragraph line spacing knob added (SV2_D1_BODY_LINE_HEIGHT_SCALE) with consumer seam in sv2WindowTypographyTokens.ts. Fade presence protocol standardized at 200ms for overlays. Touch block suppression and text immunity scoped to specific elements.

## 2026-05-07 -- Flat Billing and Reveal Motion

Analysis billing switched to flat 750 Rp per successful analysis with tiered charge formula (ceil(charCount / 50000) * 750 Rp) on the Worker, persisted to D1 rupiah_ledger with idempotency on (reason, ref_type, ref_id). Graph reveal motion split camera release from physics gating so topology is applied before Show Analysis but physics does not settle behind the loading gate, and prompt-only guard blocks text-only submissions without a document.
