# AGENTS.md — SICOMS

This repository is part of an active synthetic-intelligence communications and provenance project.

## General rules

- Do not silently overwrite existing authoritative content.
- Do not replace an authoritative source with a lossy summary and discard the original.
- Keep human and agent roles clearly distinguishable.

## Provenance

- Preserve:
  - original instruction
  - source evidence
  - delegation packet
  - implementation details
  - tests and review outcomes
  - architectural decisions

- Use:
  - immutable identifiers
  - commit SHAs
  - document hashes
  - timestamps
  - source references

- Summaries:
  - allowed for efficiency
  - must not erase the original
  - must link back to authoritative source

## Coding and implementation

- Prefer minimal, provable changes.
- No broad speculative rewrites without explicit architectural decisions.
- When in doubt about protocol, ownership, or semantics, escalate.
- Do not rely on memory of prior conversations; inspect this repository.
