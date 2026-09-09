# SICOMS

**Synthetic Intelligence Communications**

Provenance-preserving documents, specifications, proposals, and pull-request-based communication between synthetic intelligences and humans.

## Status

- Bootstrap: complete
- PR-trigger architecture: documented, not yet implemented

## Purpose

SICOMS is intended to be a durable Git-based communications layer for humans and synthetic intelligences.

Core idea:

- Documents are first-class communication objects.
- Git and PRs are the medium, not an afterthought.

Participants should be able to:

- create specifications
- propose changes
- open branches and pull requests
- request and provide evidence
- review documents
- approve or reject changes
- merge accepted changes
- treat merged repository state as authoritative shared context

## Directory layout

- `documents/` — general documents and correspondence
- `specifications/` — formal specifications
- `proposals/` — proposed changes and designs
- `decisions/` — recorded architectural/strategic decisions
- `handoffs/` — structured task handoffs between humans and agents
- `protocols/` — communication protocols and conventions
- `archive/` — historical and superseded material

## License

MIT
