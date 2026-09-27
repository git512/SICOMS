# ArcMusic Studio — Phased Build Plan

**Status:** Active build plan  
**Established:** 2026-09-27

## Core invariants

- Model weights stay under `/models`; runtimes may reference or symlink them but must not silently relocate them.
- Durable ArcMusic jobs, provenance, candidates, canonical selections, critiques, ratings, and generated assets live under `/data/arcmusic`.
- The browser UI and documented API are peers.
- Long-running jobs are asynchronous and survive caller disconnect.
- Aggregate scores may be computed, but raw measurements, critiques, ratings, pairwise comparisons, prompts, settings, and lineage are preserved.
- Candidate media stays preserved; canonical audio/art/video selection is explicit.
- GPU-exclusive work may be delegated to the LLaMA Proxy orchestration service through the SICOMS lease protocol.
- Observability, provenance, and agent-readable documentation are implementation requirements.

## Current empirical generator status

- **YuE2-3B:** known working baseline in ArcMusic.
- **HeartMuLa 3B:** previous ArcMusic attempt failed; requires debugging.
- **MiniMax Music 3:** not yet empirically exercised through ArcMusic.
- **ACE-Step 1.5:** ArcControl integration exists; ArcMusic end-to-end validation pending.

Do not upgrade these statuses without actual run evidence.

## Phase 0 — Studio kernel and agent API

Durable queue/state, persisted requests, retry/recovery, idempotency, HTTP 202 submissions, studio deep links, `/agents.md`, OpenAPI, stable `/api/v1`, capabilities, model-path audit, canonical-asset foundation, orchestrator-request setting, LLaMA Proxy lease boundary, versioned docs.

**Exit:** a fresh agent can discover ArcMusic, submit a song/album job, receive status/studio URLs, end its turn, and later reconstruct what happened.

## Phase 1 — Generator bring-up lab

Revalidate YuE2; diagnose HeartMuLa; first MiniMax Music 3 run; first ACE-Step 1.5 ArcMusic run; unified generator health/capability contract; adapter-specific provenance.

## Phase 2 — Candidate sets and canonical audio

N candidates per concept; candidate families; audition UI; keep/reject; explicit canonical audio; seed/settings capture; comparison metadata.

## Phase 3 — Music Flamingo critic

Structured evaluation axes; raw critique; pairwise candidate comparison; revision improvement/regression comparison; evaluator provenance.

## Phase 4 — Human preference layer

Per-axis ratings; overall rating; A/B preference; textual rationale; explicit machine-vs-human disagreement retention.

## Phase 5 — Learned scoring and conceptualization-space feedback

Derived aggregate score or small score vector; learned metric weighting; latent preference representation; recomputable historical scoring; producer feedback toward preferred conceptual regions.

## Phase 6 — Iterative studio-director loop

`concept -> candidates -> critique -> revision brief -> regenerate -> compare`

Add iteration budgets, stagnation/regression detection, approval gates, and canonical promotion policy.

## Phase 7 — Artist identity and album coherence

Persistent sonic/lyrical identity, album concept, track roles, recurring motifs, cross-track consistency evaluation.

## Phase 8 — Artwork studio

Qwen-Image-2.1-class profiles, album-cover candidates, per-track art candidates, visual persona/reference assets, canonical art, MP3 embedding, sidecars, full prompt/model/seed provenance.

## Phase 9 — Lyric-video studio

Lyric timing, visualizer/compositor, canonical art/keyframes, 16:9 and 9:16 exports, candidate and canonical lyric videos.

## Phase 10 — Generative music-video studio

Profile-driven text-to-video, image-to-video, image+text-to-video, hybrid shots, multishot backends, per-shot regeneration. Candidate families include Wan, HunyuanVideo, LTX, CogVideoX, and future frontier models.

## Phase 11 — Video-understanding critic loop

Use video-capable inference to inspect semantic fit, temporal coherence, identity drift, malformed objects/anatomy, corrupted text, camera logic, continuity, and song-section fit. Regenerate bad shots individually.

## Phase 12 — Canonical release package

Canonical audio, album art, track art, lyric video, music video, lyrics/metadata, Navidrome publication, and release manifest.

## Phase 13 — Studio-director autonomy

Agent-orchestrated producer/generator/critic/visual/publisher workflow with configurable approval gates and complete evidence retention.

## GPU orchestration

LLaMA Proxy is being evolved into the dedicated orchestration service because it already receives ordinary inference calls from clients that may not understand orchestration.

ArcMusic supports an **orchestrator request** mode:

- disabled: configured direct resource behavior;
- enabled: request a GPU lease, honor acquisition timing/expiry, associate the lease with the durable job, then release mechanically.

See `protocols/GPU_LEASE_PUBSUB.md`.

## Immediate sequence

1. Finish Phase 0.
2. Audit model paths.
3. Revalidate YuE2.
4. Debug HeartMuLa.
5. Exercise MiniMax Music 3.
6. Exercise ACE-Step 1.5.
7. Build candidate sets.
8. Integrate Music Flamingo.
