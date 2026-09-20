# ArcAngle Continuous Upstream Integration and Agent Coordination

**Status:** Active coordination plan  
**Established:** 2026-09-20  
**Scope:** llama.cpp, openclaw, OpenWebUI, ArcAngle integration/testing, autonomous maintenance

## Purpose

ArcAngle is moving toward a continuously maintained downstream-development model for three fast-moving upstream projects:

- llama.cpp
- openclaw
- OpenWebUI

The goal is to remain very close to current upstream while carrying a small, explicit, test-protected ArcAngle patch set. Autonomous maintenance by Eliska Iskra should reduce merge debt by reconciling upstream changes frequently instead of allowing long-lived divergence.

Conversation threads are not authoritative project state. Repository state, provenance records, tests, and this coordination record are the durable synchronization layer.

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

Primary role:

- current ArcAngle support transaction
- ArcControl issues
- LLaMA Proxy issues
- telemetry
- attribution
- CPU-routing and OpenWebUI/openclaw integration work
- implementation against captured support evidence

### Phone/architecture thread

Primary role:

- broader orchestration architecture
- llama.cpp residency/sleep/wake/slot-persistence design
- voice/ASR VRAM scheduling
- CPU-background inference strategy
- autonomous upstream-maintenance design
- cross-project repository/testing architecture

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
