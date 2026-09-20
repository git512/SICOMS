# ArcAngle Continuous Upstream Integration and Agent Coordination

**Status:** Active coordination plan  
**Established:** 2026-09-20  
**Scope:** llama.cpp, openclaw, OpenWebUI, ArcAngle integration/testing, autonomous maintenance

## Bootstrap context: read this first

This document is written so that a fresh model, agent, or human can reconstruct the work without assuming access to prior conversations.

Do **not** infer missing history from phrases such as "the current work," "our patch," or "the other thread." If a fact is not documented here or in the project-local state, inspect the actual repositories/runtime/evidence before acting.

The central coordination problem is that Patrick is actively working on ArcAngle from more than one ChatGPT thread at the same time. One thread is primarily a laptop/support implementation thread; another is primarily a phone/architecture thread. These conversations may contain useful reasoning, but they are not durable authority and must not be treated as if every future agent has seen them.

Repository state, captured evidence, explicit provenance records, tests, and committed coordination documents are intended to become the authoritative shared context.

## Current system context

ArcAngle is Patrick's local inference/workstation system. The relevant stack currently includes, at minimum:

- llama.cpp for local model inference
- LLaMA Proxy as an inference routing/proxy layer
- openclaw as a persistent agent/runtime layer
- OpenWebUI as a user-facing multi-assistant/chat interface
- ArcControl and related observer/telemetry components
- GPU-based voice/ASR workloads that compete for VRAM with interactive LLM inference

The exact deployed versions, active service targets, ports, branches, and runtime state must be inspected from the actual machine/project state rather than assumed from this document.

## Why this downstream-maintenance system is being created

Several separate problems converged into the same architectural need.

### 1. Active support work exposed a need for a small custom llama.cpp patch surface

The laptop/support thread is working from an ArcAngle incident package and has already identified multiple integration/telemetry issues, including requester attribution, request correlation, inference telemetry quality, CPU inference routing, OpenWebUI-to-openclaw session continuity, and missing cache-reuse telemetry.

During that work, a dedicated lightweight llama.cpp trace/correlation endpoint such as `/arc/trace` became attractive because it could create a deterministic marker inside llama.cpp's own logging/timing domain. This would make request provenance stronger than trying to correlate independent timelines only by nearby timestamps.

**Status:** proposed downstream llama.cpp feature; not assumed to be implemented yet.

### 2. Voice/ASR reliability and VRAM contention are materially harming interactive use

Patrick uses voice heavily and currently considers voice reliability a major usability problem. GPU ASR can consume enough VRAM to collide with interactive LLM inference.

The desired orchestration model is therefore able to temporarily free GPU residency for voice/ASR and then restore the interrupted inference workload with as little lost state as possible.

### 3. Current llama.cpp cache/persistence behavior is not reliable enough for the intended workflow

In Patrick's actual use, useful cache state frequently has to be rebuilt rather than reused. Qwen3.6 checkpoint/cache persistence has been observed as unreliable in the current workflow.

The immediate practical objective is not a complete llama.cpp runtime redesign. It is to expose and improve mechanisms that already exist:

- callable sleep
- callable wake
- slot save/restore
- persistence to RAM or disk by explicit choice
- instrumentation that proves whether restore actually reused state or merely reported success before a full re-prefill

Desired operational sequence:

`slot save -> sleep -> temporary GPU workload such as ASR -> wake -> slot restore -> resume`

If cache restoration still requires a re-prefill initially, explicit reliable VRAM eviction is still valuable because it makes voice usable on demand.

**Status:** intended custom work; do not assume it exists until verified in the relevant branch/runtime.

### 4. A custom llama.cpp branch creates an upstream-maintenance problem unless it is continuously reconciled

Patrick does not want a stale long-lived fork. The intended model is to remain close to current upstream and carry a small, explicit ArcAngle patch stack protected by regression tests.

The same maintenance pattern is desirable for openclaw and OpenWebUI because both evolve quickly and local modifications/integrations can otherwise make upgrades painful.

### 5. Autonomous maintenance changes the economics of carrying downstream patches

Eliska Iskra is Patrick's persistent local agent and has a normal heartbeat cadence of approximately 30 minutes.

ArcAngle is also being set up for continuously available CPU inference. That creates an opportunity for routine upstream tracking, reconciliation, testing, documentation, and low-risk coding work to run in the background without unnecessarily displacing interactive GPU inference.

Routine work should therefore preferentially use CPU inference where practical. Difficult work may escalate to stronger GPU/local coding models, Codex, Grok Build, or another explicitly selected worker.

The current intended local coding preference is to use Ornith-family coding profiles for ordinary maintenance, with Ornith Precise/Q6 as a stronger local option for more difficult work. This is a routing preference, not a rigid invariant.

### 6. Regression tests should accumulate

Every material bug fixed in the maintained downstream trees should, where practical, gain a regression test protecting the recovered invariant.

The long-term objective is that maintenance becomes more mechanical over time, not more dependent on remembering why an old patch existed.

### 7. GitHub is intended as durable provenance and a review boundary

Patrick wants the maintained repositories pushed to his GitHub.

Pull requests can serve as:

- durable diffs
- review objects
- external provenance
- backup
- triggers for independent review by another model/agent

GitHub is not intended to replace local runtime authority.

## Existing facts versus work being established

The following distinctions are important.

### Known existing conventions / state

- llama.cpp dated build outputs use the convention `/servers/llcpp/main/YYYY-MM-DD/`.
- Historical known-good llama.cpp builds are intentionally preserved.
- Eliska Iskra has a normal heartbeat cadence of approximately 30 minutes.
- There are currently multiple active ChatGPT workstreams touching overlapping ArcAngle components.
- The laptop/support thread is working from captured support evidence rather than only conversational description.
- Current interactive use is being harmed by cache/prefill disruption and voice/ASR resource contention.

### Being established now

- canonical `/repos/llama/`, `/repos/openclaw/`, and `/repos/openwebui/` maintenance namespaces
- pristine upstream and tested bleeding trees
- temporary work-branch workflow
- shared human-readable and machine-readable maintenance state
- autonomous upstream polling/reconciliation
- CPU-preferred background maintenance
- difficulty-based model/worker escalation
- cumulative regression infrastructure
- GitHub push/PR review workflow
- cross-project compatibility tracking

### Proposed or intended, not assumed implemented

- llama.cpp `/arc/trace`-style correlation endpoint
- explicit externally callable llama.cpp sleep/wake control for orchestration
- slot persistence selectable between RAM and disk
- unified/richer serialization behavior for RAM/disk persistence
- repaired Qwen3.6 hybrid/recurrent checkpoint/cache restore behavior
- automatic PR-triggered independent review
- fully automated promotion/reconciliation pipeline

Agents must verify actual implementation status before acting.

## Purpose

ArcAngle is moving toward a continuously maintained downstream-development model for three fast-moving upstream projects:

- llama.cpp
- openclaw
- OpenWebUI

The goal is to remain very close to current upstream while carrying a small, explicit, test-protected ArcAngle patch set. Autonomous maintenance by Eliska Iskra should reduce merge debt by reconciling upstream changes frequently instead of allowing long-lived divergence.

Conversation threads are inputs to this process, not authoritative project state.

## Repository topology

Canonical local namespaces:

- `/repos/llama/`
- `/repos/openclaw/`
- `/repos/openwebui/`

Each project should contain:

- `<project>-upstream/` — pristine upstream-tracking checkout; no ArcAngle-specific commits.
- `<project>-bleeding/` — latest ArcAngle-maintained state that has passed the required validation gates.
- `state/` — machine-readable and human-readable reconciliation/provenance records.
- `tools/` — project-specific maintenance, testing, build, and reconciliation helpers.

Shared generic maintenance machinery may live under:

- `/repos/tools/upstream-sync/`

Source authority and build/runtime artifacts must remain separate.

Existing llama.cpp build convention remains:

- `/servers/llcpp/main/YYYY-MM-DD/`

Historical known-good builds are not to be deleted or silently replaced outside the existing same-day rebuild convention. Production switching remains a separate explicit action.

## Branch semantics

`upstream` and `bleeding` are not scratch branches.

Autonomous or human development should occur on short-lived work branches created from the current reconciled state, for example:

- `work/llama-slot-hibernate`
- `work/arc-trace`
- `work/openclaw-session-shim`

Only validated work is merged into `bleeding`.

Therefore:

- **upstream** = pristine external source
- **work branch** = active modification/reconciliation
- **bleeding** = latest accepted/tested ArcAngle downstream state
- **production** = separately promoted known-good runtime state

A clean textual merge/rebase is not proof of semantic compatibility.

## Continuous upstream maintenance loop

Eliska Iskra acts as orchestrator and custodian of authoritative maintenance state.

On the normal heartbeat cadence, currently every 30 minutes, she may:

1. Check whether upstream moved.
2. Record the exact new upstream SHA(s).
3. Determine which ArcAngle patches or tests may be affected.
4. Create/update a bounded work branch.
5. Classify implementation/review difficulty.
6. Delegate routine background work to CPU inference when practical.
7. Escalate difficult or high-risk work to stronger GPU/local coding models, Codex, Grok Build, or another explicitly selected worker.
8. Build and run required per-change regression tests.
9. Record conflicts, resolutions, test evidence, and semantic changes.
10. Merge into `bleeding` only after required validation succeeds.
11. Push review-ready or accepted state to GitHub.
12. Leave production unchanged unless separately instructed.

Repeated execution against the same upstream SHA should be idempotent.

Dirty or unexplained source trees are a hard stop for automated reconciliation.

No ArcAngle patch may be silently dropped. If upstream supersedes a local patch, retirement must be explicit and must record the upstream change that made the patch unnecessary.

## Context-confidence rule for agents

At the start of any maintenance transaction, distinguish:

- **documented fact**
- **observed current machine/repository state**
- **proposal/design intent**
- **inference/hypothesis**
- **unknown**

Do not silently upgrade a proposal into an existing feature or an inference into a fact.

When required context is absent, inspect authoritative state/evidence before modifying code. If the missing information cannot be recovered from authoritative sources, record the uncertainty rather than manufacturing a plausible history.

This is especially important at bootstrap because an incorrect initial assumption can propagate through every later decision and test.

## CPU versus GPU work

Background maintenance should preferentially avoid perturbing interactive GPU inference.

Candidate CPU tasks include:

- heartbeat reasoning
- upstream polling
- routine diff inspection
- straightforward reconciliation
- repository bookkeeping
- documentation maintenance
- regression-result inspection
- low-risk review

Interactive/user inference has priority over background GPU work.

Before escalating background work to the GPU, the orchestrator should consider whether active user inference is occurring. CPU inference can continue independently at lower throughput when latency is not important.

A stronger model should be selected according to actual task difficulty, not merely because it is available.

Current intended coding hierarchy is approximately:

- routine/low-risk work: suitable Ornith profile
- moderate or patch-overlap work: Ornith Precise, preferably Q6 where practical
- difficult/high-risk semantic conflicts or subsystem changes: stronger independent review or implementation using Codex, Grok Build, or another appropriate high-end coding worker

The difficulty classification and its later success/failure should itself be recorded so routing can improve over time.

## Regression policy

A material bug fix should normally gain a regression test protecting the recovered invariant.

Testing is cumulative.

Three useful test classes are:

### Per-change tests

Run whenever upstream movement or a local patch touches relevant behavior.

### Daily integration tests

Exercise expensive or cross-component behavior, including real runtime interactions.

### Periodic soak/regression tests

Catch state-dependent or timing-dependent failures that short tests may miss.

A passing file load, API response, or merge is not sufficient when the desired invariant is behavioral.

For example, llama.cpp slot restore must distinguish:

- state loaded successfully
- token count restored
- actual KV/cache state reused
- prefix silently reprocessed

## Initial llama.cpp downstream work

The first intended custom llama.cpp capabilities include:

1. A lightweight `/arc/trace`-style correlation mechanism allowing LLaMA Proxy to create a deterministic marker inside llama.cpp's own log/timing domain.
2. Explicit callable sleep.
3. Explicit callable wake.
4. Slot persistence selectable between RAM and disk.
5. Reuse of the richest correct state representation for both RAM and disk persistence rather than maintaining two unrelated persistence mechanisms.
6. Instrumentation proving whether restored state is actually reused.
7. Investigation and repair of Qwen3.6 hybrid/recurrent checkpoint/cache persistence as required.

Desired operational sequence:

`slot save -> sleep -> temporary GPU workload (for example ASR) -> wake -> slot restore -> resume`

Even when cache restoration still requires re-prefill, reliable explicit VRAM eviction has immediate operational value because it allows voice/ASR workloads to acquire GPU memory on demand.

## LLaMA Proxy / ArcControl integration context

Related active work includes:

- durable requester attribution between LLaMA Proxy and ArcControl
- request/correlation IDs carried across inference phases
- higher-resolution/event-driven inference telemetry
- model-change visibility
- cache reuse telemetry
- CPU inference routing analogous to existing GPU routing
- OpenWebUI-to-openclaw session mapping moved into LLaMA Proxy so openclaw can remain closer to stock

These changes should use the same provenance/test discipline as the upstream-maintenance workflow.

## GitHub role

GitHub provides durable remote provenance, reviewable diffs, backup, and an independent review boundary.

Eliska may push validated or review-ready work branches and use pull requests as coordination/review objects.

GitHub does not replace local runtime authority.

A useful pattern is:

`local autonomous work -> tests -> PR/push -> independent review -> accepted bleeding state -> separate production promotion`

Pull-request events may be used to trigger independent review by another model or agent.

## Cross-thread coordination

At present there are multiple active ChatGPT workstreams.

### Laptop/support thread

This thread began from an ArcAngle support transaction using a captured incident package. Its primary responsibility is empirical implementation/debugging against that captured/current system evidence.

Known work in that thread includes:

- ArcControl requester attribution
- LLaMA Proxy request provenance/correlation
- inference telemetry improvements
- model-change telemetry
- miscellaneous/system VRAM accounting
- CPU inference routing
- OpenWebUI-to-openclaw session mapping
- Arc Shepherd / observer cache-reuse telemetry

### Phone/architecture thread

This thread is primarily responsible for broader architectural design and coordination, including:

- llama.cpp residency/sleep/wake/slot-persistence design
- voice/ASR VRAM scheduling
- CPU-background inference strategy
- autonomous upstream-maintenance design
- cross-project repository/testing architecture
- GitHub/PR coordination strategy

Neither conversation is authoritative.

Material decisions, implementation state, test evidence, current SHAs, and unresolved work should be written into repository/project state so either thread can reconstruct the present situation without relying on conversational memory.

## Shared state requirements

At minimum, the maintenance state for each project should record:

- current upstream SHA
- current bleeding SHA
- upstream base SHA
- active work branch
- ArcAngle-specific commits/patches
- affected upstream files/functions where useful
- current work-item state
- originating context/reference
- assigned worker/model
- difficulty classification
- conflicts encountered
- conflict-resolution reasoning
- build result
- regression-test result
- runtime-validation result
- retired/superseded patch relationships
- last reconciliation timestamp
- next concrete task

The shared cross-project state should additionally record which tested versions of llama.cpp, openclaw, OpenWebUI, LLaMA Proxy, and ArcControl are known to work together.

## Core invariant

The intended system is not "an always-mutating fork."

It is a continuously reconciled, test-gated downstream distribution whose divergence from upstream remains small, explicit, attributable, reversible, and mechanically defended by regression tests.

A fresh agent should be able to enter from this document plus project-local state, distinguish fact from proposal, inspect current reality, and continue safely without pretending to remember conversations it never saw.
