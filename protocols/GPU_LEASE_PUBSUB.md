# GPU Lease Pub/Sub Coordination Protocol

**Status:** Proposed cross-project orchestration protocol  
**Established:** 2026-09-27  
**Scope:** ArcAngle GPU ownership, inference handoff, media workloads, agent frameworks, inter-project and intra-project coordination

## Purpose

This protocol defines a transport-agnostic GPU lease and event-publication abstraction for ArcAngle.

The intent is broader than a simple "GPU available / GPU busy" lock. A requester should be able to ask an orchestrator for a bounded GPU lease, receive a deterministic handoff schedule, and publish/subscribe to lifecycle events so independent components and agent frameworks can coordinate without requiring an inference model to remain active during the transition.

The same abstraction is intended to be useful both:

- **inter-project** — for example ArcMusic, ArcControl, openclaw, ComfyUI, video generation, benchmark tooling, and future projects sharing the same GPU;
- **intra-project** — for example multiple workers, stages, or services inside one larger project coordinating residency and execution.

This document specifies semantics, not a specific message broker. NATS, MQTT, Redis Streams, an HTTP/SSE/WebSocket implementation, or another transport may implement the protocol as long as the behavioral contract is preserved.

## Core design principle

A GPU lease is a **time-bounded, observable ownership transaction**, not merely permission to use the GPU.

A successful lease response may communicate:

- whether control is available immediately;
- when the requester may take control;
- the latest time by which current owners are expected to yield;
- the maximum continuous lease duration;
- whether the lease is preemptible;
- the earliest time the requester may request an extension or renewed control;
- which event topic/stream carries lifecycle updates;
- which correlation/lease identifier must accompany later actions.

This allows a conversational agent or inference framework to receive a handoff notice, finish or stop an inference run, publish that it has yielded, and then go offline while mechanical orchestration continues.

## Non-goals

This protocol does not require:

- a language model to remain resident while a handoff occurs;
- every project to know how another project frees VRAM;
- a specific broker implementation;
- callers to micromanage llama.cpp, ComfyUI, ArcControl, or other backend-specific start/stop mechanics.

Those mechanics belong behind the orchestrator boundary.

## Lease lifecycle

A recommended lifecycle is:

`REQUESTED -> PENDING -> NOTICE -> READY -> ACTIVE -> RELEASING -> RELEASED`

Additional terminal/error states may include:

- `DENIED`
- `EXPIRED`
- `PREEMPT_REQUESTED`
- `PREEMPTED`
- `FAILED`
- `ORPHANED`

### REQUESTED

A client submits an idempotent lease request.

### PENDING

The orchestrator has accepted the request but cannot yet grant ownership.

### NOTICE

Current subscribers/owners are informed that a handoff is expected. This phase provides a grace window for inference frameworks or other GPU clients to stop cleanly, persist state, or acknowledge readiness.

### READY

The orchestrator has determined that the requester may take control at or after the published `acquire_after` time.

### ACTIVE

The requester owns the lease. The lease remains bounded by its expiry/max-duration policy.

### RELEASING

The owner has begun release/cleanup and the orchestrator is waiting for the GPU to return to a known state.

### RELEASED

Ownership has ended and subscribers may be informed that normal workloads can resume or compete for subsequent leases.

## Request schema

A lease request should be able to express at least:

- `request_id` — caller-generated idempotency key;
- `requester` — stable logical identity, not merely a PID;
- `project` — owning project/workflow;
- `job_id` — durable work item identifier when applicable;
- `resource` — normally a logical GPU resource such as `gpu:primary`;
- `priority` — orchestrator-interpreted priority class;
- `requested_at`;
- `not_before` — optional earliest acceptable acquisition time;
- `requested_duration` — expected duration;
- `max_duration` — caller-declared hard upper bound if known;
- `grace_period` — desired notice interval before takeover;
- `preemptible` — whether the new lease itself may be interrupted;
- `metadata` — bounded project-specific data;
- `reply_topic` or equivalent response channel when supported.

A lease request is declarative. It describes the resource requirement and timing intent, not backend-specific shutdown commands.

## Grant / schedule schema

A grant or pending schedule should expose at least:

- `lease_id`;
- `request_id`;
- `state`;
- `resource`;
- `requester`;
- `issued_at`;
- `notice_at`;
- `acquire_after`;
- `must_yield_by` for current owners when applicable;
- `lease_expires_at`;
- `max_continuous_duration`;
- `renew_after` or earliest extension-request time;
- `preemptible`;
- `event_topic`;
- `heartbeat_interval` if heartbeats are required;
- `orchestrator_epoch` or equivalent generation identifier so stale grants can be rejected after controller restart/failover.

The important distinction is that `acquire_after` and `lease_expires_at` are protocol facts, while the specific mechanism used to stop or restore inference is implementation detail.

## Pub/Sub event model

Subscribers should be able to observe GPU coordination without polling one specific project.

Recommended logical topic hierarchy:

- `gpu.lease.requested`
- `gpu.lease.pending`
- `gpu.lease.notice`
- `gpu.lease.ready`
- `gpu.lease.active`
- `gpu.lease.heartbeat`
- `gpu.lease.release_requested`
- `gpu.lease.released`
- `gpu.lease.preempt_requested`
- `gpu.lease.preempted`
- `gpu.lease.expired`
- `gpu.lease.failed`
- `gpu.state.changed`

A transport may map these to one stream with typed events instead of literal topics.

Every event should carry:

- event ID;
- timestamp;
- lease ID when one exists;
- request ID when one exists;
- resource;
- state/event type;
- requester/owner;
- orchestrator epoch/generation;
- causal parent or correlation ID where useful;
- bounded metadata;
- monotonic sequence number per lease or stream when supported.

Events are evidence. Consumers should not infer ownership solely from missing events; authoritative lease state must remain queryable from the orchestrator.

## Subscriber behavior

An agent framework, inference service, or project may subscribe to lease/state events and act mechanically.

Example inference subscriber behavior:

1. Receive `gpu.lease.notice` with `must_yield_by`.
2. Stop accepting new GPU work.
3. Finish, cancel, checkpoint, or hibernate the active inference transaction according to local policy.
4. Publish/acknowledge `yield_ready` or equivalent.
5. Remain offline if necessary.
6. Later observe `gpu.lease.released` or an explicit restore instruction.
7. Restore normal GPU availability according to policy.

No inference model is required to reason during steps 2-7.

## Ownership and safety invariants

1. **Exactly one authoritative lease owner** exists for an exclusive GPU resource at a time.
2. A lease ID is never reused.
3. A stale lease from an earlier orchestrator epoch must not authorize ownership.
4. Lease expiry must fail closed: expiration removes permission to continue using the resource.
5. A requester must not assume ownership merely because the expected acquisition time passed; it must have a valid READY/ACTIVE grant or equivalent authoritative state.
6. Current owners receive explicit notice when possible before preemption.
7. Unknown/ambiguous ownership is not silently treated as free.
8. Release must be explicit and observable.
9. Backend-specific recovery/restoration failures must be published rather than hidden behind a successful release event.
10. Durable jobs survive caller disconnect; lease lifecycle must not depend on the initiating HTTP connection remaining open.

## Time-bounded control and renewal

The protocol intentionally supports bounded ownership.

A grant may say, in effect:

- "You may acquire immediately."
- "You may acquire after 14:32:10."
- "The current owner has until 14:32:10 to yield."
- "You may hold the GPU for up to 8 minutes."
- "You may request extension after 6 minutes."
- "You are preemptible by priority class X."

This makes long-running media generation, benchmarking, inference, and agent workloads schedulable without treating any one client as permanently entitled to the GPU.

Renewal should be an explicit transaction tied to the same lease lineage. It must not silently turn a bounded lease into indefinite ownership.

## Heartbeats and orphan detection

For workloads long enough to justify them, the orchestrator may require heartbeats.

A heartbeat should not itself grant ownership. It only confirms the current lease holder is alive.

If heartbeats stop:

1. mark the lease suspect;
2. apply a bounded grace interval if policy allows;
3. mark it orphaned/expired;
4. initiate backend-specific cleanup;
5. publish the resulting state transition.

## ArcMusic integration requirement

ArcMusic should expose a setting conceptually named **orchestrator request**.

When disabled:

- ArcMusic may use its existing/direct local GPU resource path according to local configuration.

When enabled:

- ArcMusic must request a GPU lease from the configured orchestrator before beginning a GPU-exclusive production stage;
- it must honor `acquire_after`, expiry, and release semantics;
- it must publish/attach its durable ArcMusic job ID to the lease transaction;
- it must not require the initiating model/agent turn to remain active;
- the returned studio/job URL remains usable while orchestration proceeds asynchronously.

The UI/API should treat this as an orchestration policy, not expose backend-specific inference stop/start plumbing to ordinary callers.

## LLaMA Proxy / orchestrator relationship

Current implementation direction: **LLaMA Proxy is being evolved into the dedicated orchestration service.**

This is intentional because LLaMA Proxy already sits on the ingress path for ordinary inference calls, including calls from "dumb" clients that are unaware of GPU leasing or orchestration. That allows the orchestrator to enforce lease/admission policy centrally: orchestration-aware clients can request explicit leases, while unaware clients can simply be held, queued, delayed, or routed until inference is legally available again.

ArcControl remains a natural telemetry/control-plane subscriber and operator surface, but GPU lease authority should not be duplicated there. ArcMusic and other clients should depend on the stable lease protocol rather than LLaMA Proxy internals.

The orchestrator implementation must still preserve transport independence at the client contract boundary so the pub/sub/event transport can evolve without forcing every client to change.

## Cross-project usefulness

The same lease/event abstraction should be reusable for:

- ArcMusic song/image/video generation;
- interactive llama.cpp inference;
- openclaw agent workloads;
- GPU ASR/TTS where exclusive memory is required;
- ComfyUI workflows;
- ArcBench benchmarking;
- future image/video/research systems;
- intra-project stage coordination.

A subscriber may also use state events to determine when a component may come online, warm a model, queue work, or remain offline.

## Observability

The orchestrator should expose:

- current authoritative owner;
- pending lease queue;
- active lease and expiry;
- recent lease/event history;
- subscriber acknowledgements when used;
- preemption/expiry reasons;
- restoration result;
- correlation IDs linking project job -> lease -> backend actions.

The event stream should be suitable for both machine consumption and later forensic reconstruction.

## Persistence and restart behavior

Authoritative lease state and enough recent event history must survive orchestrator restart or fail safely.

On restart, the orchestrator must not assume that "no in-memory lease" means "GPU is free." It should reconcile observed backend/process/GPU state, advance its epoch/generation, invalidate stale grants, and publish the reconciled state.

## Open design decisions

The following are intentionally not fixed yet:

- broker/transport selection;
- exact priority classes;
- exact default grace periods;
- whether acknowledgements are broker-native or protocol messages;
- whether lease scheduling is centralized entirely in ArcControl or delegated to a dedicated residency/orchestration service;
- exact preemption hierarchy;
- exact interaction with llama.cpp slot save/sleep/wake once those mechanisms are verified.

These should be decided from empirical system behavior rather than prematurely reified.

## Initial implementation target

A minimal conforming first implementation should provide:

1. lease request;
2. idempotent request handling;
3. pending/granted/released states;
4. `acquire_after` and expiry timestamps;
5. a notice/grace interval;
6. a subscribable event stream;
7. authoritative status query;
8. release;
9. restart-safe epoch/generation handling;
10. one ArcMusic integration using the `orchestrator request` setting.

More complex priority/preemption/renewal policy can follow without changing the core abstraction.
