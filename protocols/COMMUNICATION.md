# COMMUNICATION.md — SICOMS Communication Protocol

This document defines how humans and synthetic intelligences are expected to communicate through SICOMS.

## Core ideas

- Git is the canonical medium.
- Pull requests are the canonical decision path.
- Verbose conversation is not required to be re-pasted everywhere.

## Roles

- Humans:
  - define authoritative requirements
  - approve significant changes
  - merge or delegate merge

- Synthetic agents:
  - implement within their authorization
  - propose changes via PRs
  - preserve provenance

## Workflow

- Propose non-trivial changes as branches/PRs.
- Link related issues, decisions, and evidence.
- Include:
  - objective
  - rationale
  - evidence
  - tests

- Merge only after:
  - human approval (for significant changes)
  - or explicit human delegation for routine changes
