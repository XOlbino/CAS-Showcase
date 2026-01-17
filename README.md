# CAS: A Desktop Orchestrator for High-Confidence Delivery

CAS is a **Windows desktop** development orchestrator built to turn intent into **audited, verifiable output**—with plan-first governance, contract checks, and a 50‑point inspection before you ship.

This repository is a **public showcase** of the product surface area, architecture, and key workflows.

---

## Tech Stack

- **Rust** (native orchestration + filesystem/process execution)
- **Tauri** (desktop runtime + secure bridge between UI and native commands)
- **React + TypeScript** (orchestration UX and visualization)
- **Tailwind CSS** (fast UI iteration and consistent styling)

---

## Core Features

Summarized from `CAS-Showcase/CAS_MANUAL.md` (the repo’s current source-of-truth manual).

- **SpecScout (Intent Intake / Plan-First Governance)**
  - Generates a structured plan (Steps, File Impacts, Risk) and gates execution on approval.
  - Supports XAI-style reporting to different audiences (e.g., Junior / Tech Lead / Client).

- **Intent Capsules (Intent Blocks)**
  - Treats tasks as first-class UI objects with type labeling (🛡️ BUGFIX / ✨ FEATURE / 🧹 REFACTOR).
  - Persists outputs and reports so work survives project tab switching.

- **50-Point Inspection (Health Check Dashboard)**
  - Runs a governance-grade checklist (e.g., `cargo check`, `tsc`, `eslint`, contract validation).
  - Streams per-check logs into the UI and renders a “Certificate” view when passing.
  - Enforces strict posture: missing toolchains/scripts (e.g., ESLint) are treated as **FAIL**.

- **MRI / Bridge Contract Validation**
  - Detects Rust ↔ TypeScript bridge mismatches and surfaces breaches as part of governance.

- **LLM Streaming Viewer**
  - Separates prose vs. code blocks for legibility and shows a streaming indicator while live.

- **Knowledge Vault (RAG Namespaces)**
  - Upload and search documents in named namespaces from within the app.
  - Includes a **Global Library** pattern (`global_specs/`) for universal docs shared across projects.

- **Agent Status Overlay + War Room**
  - A collapsible overlay for agent state/controls, with a dedicated “War Room” entry point.

- **Git-lite Checkpoints / Rollback**
  - Creates filesystem snapshots per intent and supports safe rollback via restore workflows.

---

## Freelance Power-Pack

CAS includes “client-facing” workflow accelerators designed for high-trust delivery:

- **Client Update Tab**
  - Generates a **human-friendly status report** from technical diffs/changes.
  - Presents the result in an editable draft so you can finalize tone and claims before sending.

- **Contra-ready Export**
  - Exports a single Markdown write-up (Problem → Process → Result) via a “Save As” dialog,
    suitable for portfolio/project descriptions on platforms like Contra.

- **Record Demo Shortcut**
  - Launches your preferred recorder (configurable path), with a fallback to [Loom Desktop](https://www.loom.com/desktop).

---

## Screenshots & Demo

- **Screenshots**:
  - Contract-Driven Implementation.png
  - Deterministic Project Auditing.png
  - Intent Intake & Live Reasoning.png

---

## Quickstart

### Development

```bash
npm install
npm run tauri:dev
```

### Useful checks

```bash
npm run check   # TypeScript typecheck (tsc --noEmit)
npm test        # Vitest
```

### Clean build cache (PowerShell)

```powershell
Remove-Item -Recurse -Force src-tauri\target\release -ErrorAction SilentlyContinue
```

---

## Repo Pointers (Start Here)

- **App shell / layout / orchestration UX**: `src/App.tsx`
- **Tauri orchestration backend**: `src-tauri/src/orchestrator.rs`
- **Bridge types + command wrappers**: `src/bridge/index.ts`
- **50-point inspection UI**: `src/components/panels/HealthCheckDashboard.tsx`
- **Agent Status overlay**: `src/components/overlay/AgentStatusDashboard.tsx`
- **LLM stream viewer**: `src/components/panels/LlmStreamViewer.tsx`
- **Knowledge Vault UI**: `src/components/panels/KnowledgeVault.tsx`

---

## Security Note (Showcase Repo)

This repository is intentionally a **technical showcase + documentation hub**.

- **No API keys should be committed**. Secrets are maintained in a **secure, local environment** (e.g., local configuration via the Tauri backend and/or environment variables).
- The **proprietary orchestration core** and any private operational data are maintained outside this public showcase footprint.

