# CAS Manual (Development Orchestrator)

This document is the **source of truth** for the current CAS system as implemented in this repository (Windows Tauri Desktop app).
It is written to get a new collaborator productive in **~5 minutes**.

---

## TL;DR (How to run)

```bash
npm install
npm run tauri:dev
```

Useful checks:

```bash
npm run check        # TypeScript typecheck (tsc --noEmit)
npm test             # Vitest
```

Build cleanup (PowerShell):

```powershell
Remove-Item -Recurse -Force src-tauri\target\release -ErrorAction SilentlyContinue
```

---

## System Architecture Overview

CAS is a **Tauri** desktop application:

- **Frontend**: React + TypeScript + Tailwind (in `src/`)
- **Backend**: Rust (Tauri commands + orchestration) in `src-tauri/src/`
- **Bridge**: TypeScript wrappers for Tauri commands and DTOs in `src/bridge/index.ts`

### High-level data flow

1. The user interacts with React UI (`src/App.tsx` and `src/components/**`).
2. The UI calls into the Rust backend via `invoke()` wrappers in `src/bridge/index.ts`.
3. Long-running / streaming operations publish events from Rust → UI (e.g. health check logs).
4. UI subscribes to events via `@tauri-apps/api/event` (`listen(...)`) and updates React state.

### Why this split matters

- **Rust** owns filesystem + process execution (native performance and Windows path safety).
- **React** owns visualization, orchestration UX, and governance gating.
- **Bridge** keeps the contract between the layers typed and explicit.

---

## UI Architecture: The 3-Column Layout (Current)

CAS uses a 3-column layout constrained to **1672px** wide (see `src/App.tsx`), implemented as a responsive grid:

- At large breakpoints: `lg:grid-cols-[minmax(0,1fr)_minmax(0,1fr)_930px]`
- The right column is **exactly 930px** (`w-[930px] flex-none`)

Scrolling model (current):

- The **entire app** scrolls: outer container is `h-screen overflow-y-auto` in `src/App.tsx`.
- Individual panels may still have local scroll regions (e.g., log lists) for usability.

### Left Column (primary control surface)

Owned in `src/App.tsx` (first `<section>`). Commonly includes:

- **Intent Intake (SpecScout)**: prompt input, plan generation, approvals, success criteria.
- **Automation Console**: moved into the left column (renders `ConsolePanel` titled “Automation Console”).
- **War Room trigger**: no longer in the left header; it is grouped under the Agent Status overlay (see below).

### Middle Column (status + diagnostics)

Owned in `src/App.tsx` (second `<section>`). Currently focuses on system introspection such as:

- **Live Reasoning Trace** (and related stream diagnostics)

### Right Column (930px fixed) (execution artifacts + governance)

Owned in `src/App.tsx` (third `<section>`, fixed width). Current content includes:

- **Artifacts / Code Result**
- **Senior Audit** controls and grading UI
- **Impact Dashboard**
- **Repo Map**
- **Knowledge Vault**
- **LLM Stream** viewer
- **50-Point Inspection / Governance** UI (`HealthCheckDashboard`)

> Note: Some older design notes may describe Governance/MRI as “middle column”. The **source of truth** is the current `src/App.tsx` layout.

---

## Core Features & Logic

### Health Check Dashboard (50-point inspection)

**Purpose**: enforce “execution blockers” and verify governance requirements after automation work.

#### Backend: generating the report

Rust implementation: `src-tauri/src/orchestrator.rs` (function `run_health_check`).

Key properties:

- Produces a `HealthCheckReport` containing:
  - `ok` (boolean)
  - `intent_type` (string)
  - `files_verified`, `contract_breaches`, `type_safety_pct`
  - `items[]`: each `HealthCheckItem` has `id`, `label`, `summary`, `status`, `column`, `logs`
- Emits streamed logs per check via a frontend event (consumed as `logsByCheckId` in React).
- **Persists** the report into the active `IntentBlock` so the UI survives tab switching.

Current check IDs (as emitted by Rust):

- **Static Analysis**:
  - `cargo_check`
  - `cargo_clippy`
  - `tsc`
  - `eslint` (**strict governance**: missing toolchain/script is treated as FAIL)
- **System Integrity**:
  - `mri_contracts` (contract validation)
  - `path_safety` (repo root canonicalization + expected directories exist)
- **Documentation**:
  - `rationales` (junior/lead/client rationale presence)

#### Frontend: rendering + mapping

React implementation: `src/components/panels/HealthCheckDashboard.tsx`.

Rules:

- **Grouping** uses `item.column` when it matches one of:
  - `Static Analysis`
  - `System Integrity`
  - `Documentation`
- **Fallback mapping** ensures checks are not “lost” if a column is missing/unknown:
  - `cargo_check`, `cargo_clippy`, `tsc`, `eslint` → Static Analysis
  - `mri_contracts`, `path_safety` → System Integrity
  - `rationales` → Documentation
- **Certificate Mode**: if `report.ok === true`, the dashboard shows a “Certificate of Completion”.
- **Live Logs**: the “View Logs” panel uses `logsByCheckId[item.id]` as the primary source.

Intent emoji mapping:

- Uses the shared helper `intentTypeLabel()` (from `src/agents/specScout.ts`), which maps:
  - `BUGFIX` → 🛡️
  - `FEATURE` → ✨
  - `REFACTOR` → 🧹

### Agent Status UI (collapsible overlay + War Room)

Component: `src/components/overlay/AgentStatusDashboard.tsx`

Behavior (wired in `src/App.tsx`):

- Overlay is shown when `agentStatusOpen === true`.
- Overlay provides:
  - **War Room** button (calls `onOpenWarRoom`)
  - **Close (X)** button (calls `onClose`)
- When closed, `App.tsx` renders a small fixed “Agent Status” button (top-right) that re-opens the overlay.

Layering:

- Uses the shared z-index map in `src/constants/zindex.ts`
- Agent Status overlay is rendered at `Z_INDEX.debug` (overlay-safe).

### LLM Streaming (prose vs code + streaming indicator)

Streaming lifecycle (simplified):

- UI triggers a stream via `startLlmStream(...)` (bridge wrapper → Rust).
- React state `isStreaming` controls the UI’s “live” indicators.
- Stream text accumulates into `llmOutput` in `src/App.tsx`.

Viewer implementation:

- Component: `src/components/panels/LlmStreamViewer.tsx`
- Splits output into segments using Markdown code fences:
  - Prose: rendered in **sans** with higher legibility
  - Code blocks: rendered in **mono** with distinct background + border
- Streaming affordance:
  - Shows a pulsing cursor block (`▍`) when `streaming === true`

---

## Other Major Subsystems (at a glance)

### Plan-first governance (SpecScout → approval gate)

- Plans are generated before execution actions are enabled.
- Execution controls are gated on plan state and approvals (in `src/App.tsx` + `src/agents/specScout.ts`).

### MRI / Contract validation

- The backend produces a `ContractReport` and exposes it through the bridge.
- `mri_contracts` is also represented as a health check item in the 50-point inspection.

### Git-lite Checkpoint / Rollback

Backend (Rust):

- Checkpoints stored under `checkpoints/{intent_id}/{timestamp}/` with a `manifest.json`.
- Rollback restores via atomic temp-to-rename strategy to avoid corruption.

Frontend (React):

- Exposes checkpoint/rollback through bridge wrappers (see `src/bridge/index.ts`) and orchestration UI in `src/App.tsx`.

### Knowledge Vault (RAG namespaces)

Component: `src/components/panels/KnowledgeVault.tsx`

Key UX:

- **Namespace selection** is now a prominent **“Namespace” button** in the header toolbar (next to “Upload Doc”).
- Clicking it opens a dropdown menu of saved namespaces.
- Upload and search actions operate against the selected namespace.

---

## Workflow Instructions

### Development (recommended)

1. Install dependencies:

```bash
npm install
```

2. Run the app:

```bash
npm run tauri:dev
```

Notes:

- Tauri is configured to run Vite on port **5199** (see `src-tauri/tauri.conf.json`).

### Production build

```bash
npm run tauri:build
```

### Clean build cache (Windows / PowerShell)

Use this before a “final” production build or when debugging odd Tauri build artifacts:

```powershell
Remove-Item -Recurse -Force src-tauri\target\release -ErrorAction SilentlyContinue
```

### “PowerShell shortcut script” (recommended pattern)

This repo does **not** currently ship a `.ps1` helper script.
Many teams create a small local script (not committed) to speed up repetitive cleanup tasks, e.g.:

```powershell
# clean-release.ps1 (example)
Remove-Item -Recurse -Force src-tauri\target\release -ErrorAction SilentlyContinue
npm run check
```

Purpose:

- Reduce “human variance” in build prep.
- Make production build steps repeatable and quick.

---

## Tauri Configuration (Window constraints)

File: `src-tauri/tauri.conf.json`

Current window sizing is locked to support the intended 3-column layout width:

- **width**: 1672
- **height**: 1200
- **minWidth**: 1672
- **minHeight**: 700

Why lock width:

- The UI is designed around a fixed 3-column information architecture.
- The right column is a fixed **930px** and contains dense operational tooling; shrinking width risks overlap and unreadable panels.

---

## Repo Pointers (Where to look first)

- **App shell / layout / orchestration**: `src/App.tsx`
- **Tauri orchestration backend**: `src-tauri/src/orchestrator.rs`
- **Bridge types and command wrappers**: `src/bridge/index.ts`
- **50-point inspection UI**: `src/components/panels/HealthCheckDashboard.tsx`
- **Agent Status overlay**: `src/components/overlay/AgentStatusDashboard.tsx`
- **LLM stream viewer**: `src/components/panels/LlmStreamViewer.tsx`
- **Knowledge Vault UI**: `src/components/panels/KnowledgeVault.tsx`
- **Z-index map**: `src/constants/zindex.ts`

