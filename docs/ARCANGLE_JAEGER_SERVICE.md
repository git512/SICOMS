# ArcAngle Jaeger Tracing Service

## Purpose

ArcAngle now runs a local Jaeger instance as the distributed-tracing backend for OpenTelemetry work, initially to diagnose OpenClaw heartbeat / Llama Proxy inference-lifecycle behavior and later to support SICOMS GPU-lease and orchestration tracing.

## Deployment

- Host: `ArcAngle`
- Service definition: `/servers/jaeger/compose.yaml`
- Runtime: Docker Compose
- Container name: `jaeger`
- Restart policy: `unless-stopped`
- Image: `cr.jaegertracing.io/jaegertracing/jaeger:2.22.0`

## Ports

The host uses deliberately non-default ports to avoid collisions with existing ArcAngle services:

- Jaeger UI: `27186` -> container `16686`
- OTLP/HTTP: `24318` -> container `4318`
- OTLP/gRPC: `24317` -> container `4317`

Useful endpoints:

- UI from LAN/Tailscale: `http://ArcAngle:27186`
- Local OTLP/HTTP exporter target: `http://127.0.0.1:24318`
- Local OTLP/gRPC exporter target: `127.0.0.1:24317`

## Current storage policy

The initial deployment uses Jaeger's built-in in-memory storage. Trace data is therefore intentionally ephemeral and is lost when the Jaeger container restarts.

This is acceptable for the current debugging phase. Persistent trace storage should be added only when there is a demonstrated operational need rather than making observability infrastructure larger before the tracing path itself is proven useful.

## Initial tracing target

The first use is to correlate one OpenClaw heartbeat run across:

1. OpenClaw heartbeat / agent-run lifecycle
2. OpenClaw model-call and tool-call spans
3. Llama Proxy ingress on port `19092`
4. JIT lock acquisition and release
5. model load / slot restore
6. llama.cpp backend request and SSE lifetime
7. slot persistence
8. model unload / hibernate
9. eventual GPU-lease / SICOMS state transitions

The desired end state is one causally coherent distributed trace showing exactly where time is spent and which component still considers a request active.
