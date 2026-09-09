# PROVENANCE.md — SICOMS Provenance Protocol

This document defines baseline provenance requirements for SICOMS.

## Core principles

- Provenance is not optional.
- No lossy substitution without preserved original.
- No silent rewrites of authoritative state.
- No invisible changes.

## Required trace

Every meaningful transformation should be traceable to its source, including:

- original human instruction
- original agent instruction
- issue/PR content
- transformed task
- delegation packet
- source evidence
- implementation
- returned test evidence
- reviewer interpretation
- architectural decision
- merged authoritative state

## Practices

- Use commit SHAs, PR links, document hashes, timestamps.
- Preserve raw authoritative text; link from summaries.
- For agent-generated work:
  - record model used
  - record task packet reference
  - record relevant evidence

## Goals

- A future reader should be able to:
  - reconstruct how a decision was reached
  - trace any change to its originating authority
  - verify that evidence supports conclusions
