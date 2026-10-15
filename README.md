# SIYA — Spread & Interleg Yield Algorithm

**Strategy ID:** `bitunix_mlrv`  
**Target Venue:** Bitunix Futures (USDT-M Perpetual)  
**Governing Strategy Baseline:** V6.3.4b Consolidated Specification  
**Production Runtime Language:** Rust (sole production language)  
**Specification Validation Tooling:** Python (validation and CI only)

---

## 1. Project Overview

**SIYA** (Spread & Interleg Yield Algorithm) is an institutional-grade, multi-leg relative-value execution engine targeted at Bitunix futures markets. The engine identifies, qualifies, executes, and hedges relative-value opportunities across futures structures under strict economic and risk-containment invariants.

### Key Tenets

* **Fail-Closed Safety:** If venue behavior, market data, account state, or order outcomes are ambiguous or unresolved, the system treats them as `UNKNOWN` and immediately fails closed for all dependent entry behavior.
* **Separation of Safety and Profitability:** A trade passing all safety and risk gates does not imply it is economically viable. SIYA requires positive net trading edge after all modelled costs (fees, funding, basis, slippage, and latency decay). Rebate and fee-share revenues are strictly tracked in separate ledgers and must never rescue negative trading economics.
* **No Invented Venue Semantics:** Venue attributes—such as contract multipliers, quantity lot steps, rounding rules, maintenance margin models, and liquidation mechanics—must be grounded in authoritative Bitunix evidence. Missing facts are treated as `UNKNOWN` rather than assigned guessed numeric defaults.
* **Exchange-Grounded Truth:** Local engine intent is subordinate to verified exchange state. A position is never considered `FLAT` without exchange-grounded multi-tier confirmation.
* **Single Authoritative Progress Tracking:** All implementation tasks, milestone dependencies, and validation records are centrally tracked in [`TASKS.md`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/TASKS.md).

---

## 2. Authoritative Specification & Governance

SIYA’s design and runtime contracts are governed by versioned documentation in the repository. Architectural changes require explicit project-owner authorization and formal versioned amendments.

| Document | Purpose / Scope |
|---|---|
| [`docs/bitunix_multi_leg_futures_strategy_v6_3_4_consolidated.md`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/docs/bitunix_multi_leg_futures_strategy_v6_3_4_consolidated.md) | Governing strategy and runtime contract (V6.3.4b). Section 61 defines the sole composite entry gate. |
| [`docs/bitunix_multi_leg_futures_strategy_v6_3_4_gate_registry.yaml`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/docs/bitunix_multi_leg_futures_strategy_v6_3_4_gate_registry.yaml) | Single machine-readable source of truth for all 36 entry gates, failure reasons, parameter references, and required test IDs. |
| [`docs/bitunix_multi_leg_futures_strategy_v6_3_4_gate_registry.schema.json`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/docs/bitunix_multi_leg_futures_strategy_v6_3_4_gate_registry.schema.json) | JSON Schema defining the structural specification of the gate registry. |
| [`docs/ACCOUNT_CONTROL_WORKER_IMPLEMENTATION_CONTRACT.md`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/docs/ACCOUNT_CONTROL_WORKER_IMPLEMENTATION_CONTRACT.md) | Contract for account-control worker activation, snapshot immutability, freshness, and failure semantics. |
| [`docs/SIYA_RUNTIME_ENV.example`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/docs/SIYA_RUNTIME_ENV.example) | Complete reference specification for runtime environment variables and secrets. |
| [`AGENTS.md`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/AGENTS.md) | Protected project policy defining safety rules, implementation boundaries, and development invariants. |
| [`TASKS.md`](file:///c:/Users/Pravesh/Desktop/arbspot/codebase/siya/TASKS.md) | The single authoritative progress tracker for all phases (0 through 29). |

---

## 3. Permanent Safety Invariants

1. **`UNKNOWN` is a First-Class State:**  
   `UNKNOWN != FAILED` and `TIMEOUT != NOT_EXECUTED`. A timeout during order submission or cancellation does not imply non-execution. Ambiguous states require active reconciliation before any retry or recovery decision.
2. **Never Blindly Retry:**  
   Orders are never retried blindly. Reconciliation must first prove that a retry cannot create duplicate or unintended market exposure.
3. **Execution Non-Atomicity:**  
   Multi-leg orders, Fill-or-Kill (FOK), and Immediate-or-Cancel (IOC) orders are never assumed to execute atomically. Partial fills activate exposure management and independent risk monitoring immediately.
4. **Market Data Integrity:**  
   Trading on stale, invalid, or sequence-gapped order books is strictly prohibited. Unverified depth or crossed books fail closed.
5. **Decoupled Risk Clock:**  
   The Independent Risk Monitor evaluates portfolio exposure on an autonomous timer (the Risk Clock) and must never be blocked or starved by slow exchange REST responses or network reconciliation.
6. **Protection Tracking:**  
   Protective orders (stop-loss and take-profit) are tracked explicitly. A protection gap is treated as unhedged exposure, not assumed coverage.
7. **Zero Secret Leakage:**  
   Credentials, API keys, and private signing secrets are wrapped in redaction primitives (`Redacted<T>`) preventing exposure across `Debug`, `Display`, serialization, or structured logs.

---

## 4. Architecture & Subsystems

```
┌────────────────────────────────────────────────────────────────────────┐
│                              SIYA Runtime                              │
├──────────────────────────┬─────────────────────────────────────────────┤
│ Strategy & Sizing        │ Order Gateway & Adapter                     │
│  - Multi-leg evaluation  │  - Normalized order lifecycle               │
│  - Composite entry gates │  - Bitunix REST & WebSocket clients         │
│  - Economic edge checks  │  - Rate-limiting & quota reservations       │
├──────────────────────────┼─────────────────────────────────────────────┤
│ Risk & Protection        │ State & Persistence                         │
│  - Independent Risk Clock│  - Append-only durable journal (SHA-256)    │
│  - 8-D MAX_RISK vector   │  - Exchange-grounded reconciliation         │
│  - Event-driven TP/SL    │  - Deterministic replay & fault injection   │
└──────────────────────────┴─────────────────────────────────────────────┘
```

### Staged Deployment Models

* **Owner-Only V1 (Initial Phase):**  
  Dedicated single-account operation. Bitunix API credentials are read from local environment variables into runtime memory. No PostgreSQL, Redis, tenant resolution, or user database exists on the trading execution path. `SIYA_ENABLED=false` halts new entries while preserving risk management and reconciliation for existing exposure.
* **Future Multi-User Deployment:**  
  PostgreSQL becomes authoritative for account controls. A separate, asynchronous Account Control Worker periodically publishes complete, immutable, versioned in-memory snapshots. Redis acts strictly as an optional distribution cache. Worker failure or stale snapshots block new entries without impairing existing position protection.

### Canonical Promotion Lifecycle (§40, §63)

Code compilation and test passes never authorize live trading. Promotion requires advancing through strictly gated stages:
1. **Day-0 / Phase 0:** Venue capability verification, pre-registered statistical plans, and historical replay.
2. **Backend Engineering (Phases 1–26):** Safe domain foundation, adapters, risk clock, protection, and journal.
3. **Paper Trading (Phase 26A):** Stage 2 simulation with user-configurable virtual capital, realistic slippage/latency models, and segregated P&L/rebate ledgers.
4. **Micro-Live (Stage 3):** Human-authorized live execution under restricted risk budgets and strict observation periods.

---

## 5. Repository Structure

```text
siya/
├── Cargo.toml                       # Root Cargo workspace manifest
├── AGENTS.md                        # Protected project policy and rules
├── TASKS.md                         # Single authoritative roadmap and progress log
├── requirements-dev.txt             # Python dev dependencies (PyYAML)
├── .venv/                           # Project-local Python validation environment
├── crates/
│   └── siya-domain/                 # Safe domain foundations (Phase 1.1)
│       ├── Cargo.toml               # Domain crate manifest
│       └── src/
│           ├── lib.rs               # Crate root, strict lints (#![forbid(unsafe_code)])
│           ├── units.rs             # Typed newtypes, VenueValue<T>, MAX_RISK vector
│           ├── ids.rs               # Opaque type-safe entity IDs
│           ├── states.rs            # Canonical order & transaction state machines
│           ├── errors.rs            # DomainError, canonical 36 Section 61 reason codes
│           ├── clock.rs             # Clock trait, SystemClock, ManualClock, SequentialClock
│           ├── logging.rs           # Redacted<T> wrapper and LogContext
│           ├── version.rs           # 7-field VersionProvenance bundle (V6.3.4 §41)
│           └── ports.rs             # Subsystem trait interfaces (Gateway, Risk, Recon, etc.)
└── docs/
    ├── bitunix_multi_leg_futures_strategy_v6_3_4_consolidated.md
    ├── bitunix_multi_leg_futures_strategy_v6_3_4_gate_registry.yaml
    ├── bitunix_multi_leg_futures_strategy_v6_3_4_gate_registry.schema.json
    ├── bitunix_multi_leg_futures_strategy_v6_3_4_ci.py
    ├── ACCOUNT_CONTROL_WORKER_IMPLEMENTATION_CONTRACT.md
    └── SIYA_RUNTIME_ENV.example
```

---

## 6. Development & Verification

### Prerequisites

* **Python 3.14+** (with `.venv` containing `PyYAML` from `requirements-dev.txt`) for running specification integrity CI.
* **Rust 1.75+** (`cargo`, `rustc`) for building and testing the production workspace.

### Specification Integrity Check

Before changing or evaluating any domain code or configuration, execute the authoritative specification CI check:

```bash
# Windows PowerShell
.venv\Scripts\python.exe docs\bitunix_multi_leg_futures_strategy_v6_3_4_ci.py

# Linux / macOS
.venv/bin/python docs/bitunix_multi_leg_futures_strategy_v6_3_4_ci.py
```

This verifies:
- Bidirectional consistency between Section 61 composite entry gates and the YAML registry.
- Exact match of all 36 gate IDs, failure reason codes, test IDs, and parameter references.
- Schema conformity and deterministic AMR (Automatic Margin Replenishment) test matrices.

### Rust Workspace Commands

Once the Rust toolchain is available on the host environment:

```bash
# Format check
cargo fmt --all -- --check

# Strict compilation check across workspace
cargo check --workspace --all-targets

# Run unit and integration tests
cargo test --workspace
```

---

## 7. Safety Notice

> [!CAUTION]
> SIYA interacts directly with cryptocurrency derivatives markets. Under no circumstances should test or mock credentials be pointed at live production endpoints, nor should any order submission mechanism be executed without passing all prerequisite Phase 0 through Phase 29 promotion gates and obtaining explicit human approval.

