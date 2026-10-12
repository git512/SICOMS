# ArcAngle Jaeger Documentation Handoff

## Recipient

Eliska Iskra / ArcAngle documentation-maintenance workflow.

## Objective

Add the newly deployed Jaeger service to ArcAngle's canonical local documentation store so its location, ports, purpose, and current limitations are discoverable from ArcAngle itself rather than only from SICOMS/GitHub.

Do not invent a new documentation root. Locate and use the existing canonical ArcAngle documentation folder / document store already used for system-service documentation.

## Source of truth

SICOMS now contains the deployment record at:

`docs/ARCANGLE_JAEGER_SERVICE.md`

Use that document as the authoritative source for this handoff.

## Facts to preserve

- Service definition: `/servers/jaeger/compose.yaml`
- Runtime: Docker Compose
- Container name: `jaeger`
- Restart policy: `unless-stopped`
- Image: `cr.jaegertracing.io/jaegertracing/jaeger:2.22.0`
- Jaeger UI host port: `27186`
- OTLP/HTTP host port: `24318`
- OTLP/gRPC host port: `24317`
- UI: `http://ArcAngle:27186`
- Local OTLP/HTTP target: `http://127.0.0.1:24318`
- Local OTLP/gRPC target: `127.0.0.1:24317`
- Current storage is in-memory / ephemeral; traces are lost when the Jaeger container restarts.
- Initial purpose is distributed tracing of OpenClaw heartbeat, Llama Proxy ingress `19092`, llama.cpp inference lifecycle, and later SICOMS GPU-lease/orchestration state.

## Work

1. Locate ArcAngle's existing canonical documentation store/folder.
2. Add or update the appropriate service/infrastructure documentation with the facts above.
3. Link back to the SICOMS source document and record the relevant SICOMS commit/PR provenance.
4. Preserve any existing documentation conventions and indexes.
5. Verify the documentation is readable from the normal ArcAngle documentation workflow.
6. Report the exact local documentation path changed and any index/reference updated.

## Acceptance criteria

- Jaeger can be found from ArcAngle's own documentation without consulting chat history.
- The non-default ports are explicitly recorded.
- The ephemeral-storage limitation is explicit.
- The SICOMS source record is referenced.
- No duplicate competing documentation root is created.
