# OpenClaw External-Supervisor Update Integration TODO

## Context

ArcAngle runs the OpenClaw Gateway under a systemd user service with a custom launcher:

`systemd --user -> /servers/arc-bench/scripts/openclaw-gate.sh -> node -> OpenClaw Gateway`

The OpenClaw Control UI exposes an update action when an update is available and the current user is authorized, but the currently installed updater can fail during activation because the Gateway does not own the supervisor lifecycle that must restart it. OpenClaw's safety logic correctly refuses to stop a parent process it cannot safely replace.

A temporary local helper, `openclaw-update-shim`, performs `openclaw update --no-restart` and then delegates activation to the user systemd manager with a detached delayed restart of `openclaw-gateway.service`.

## Immediate invariant

Do not weaken or bypass OpenClaw's parent-process safety checks. The integration must adapt the restart handoff to the external supervisor rather than convincing OpenClaw that it owns processes it does not own.

## Desired end state

The Control UI update button should work normally on ArcAngle while preserving systemd as the sole physical owner of Gateway lifecycle.

Semantic ownership should be separated from process ownership:

- OpenClaw owns the semantic operation: install/update and request activation.
- systemd owns the physical operation: stop/start/restart the Gateway process tree.

## Work items

1. Trace the exact current OpenClaw Control UI `update.run` activation path and identify the point where managed restart handoff is selected or rejected.
2. Check current upstream OpenClaw behavior and configuration for externally managed gateways before carrying any local patch forward. In particular, inspect current support for external service-repair/restart policy and newer Control UI updater behavior.
3. Determine whether ArcAngle can use an existing supported external-supervisor hook or policy without patching OpenClaw core.
4. If a local adapter is still required, implement the narrowest possible integration at the restart-handoff boundary rather than patching Doctor safety logic.
5. Preferred activation mechanism: enqueue a detached user-systemd restart of `openclaw-gateway.service`, allowing the updater/Gateway process to exit without synchronously killing its own supervisor chain.
6. Preserve graceful drain behavior and any updater durable run/sentinel semantics needed for post-restart verification.
7. Verify that the newly started Gateway reports the expected updated version/build and passes OpenClaw health/readiness checks.
8. Make the integration update-resilient: avoid edits to OpenClaw core if a wrapper, supported hook, service policy, or stable adapter boundary exists.
9. Add structured logging around update start, package mutation, restart handoff acceptance, systemd activation, new Gateway identity, verification success/failure, and rollback/recovery state.
10. Document recovery behavior when package update succeeds but restart fails, including the exact manual systemd recovery command.
11. Once the durable integration is proven, remove or demote the temporary `openclaw-update-shim` helper so there is one authoritative update path.

## Acceptance criteria

- Control UI update works from the ArcAngle deployment without manual terminal intervention.
- OpenClaw never attempts to kill or replace the external parent supervisor.
- systemd remains authoritative for Gateway lifecycle.
- No second competing Gateway service/unit is installed.
- Existing ArcBench launcher behavior remains intact.
- Update success is not reported until the restarted Gateway is actually healthy and running the expected version/build.
- Failure leaves a clear diagnostic trail and a deterministic recovery path.

## Temporary operational path

Until the durable integration is complete, use the local `openclaw-update-shim` helper. It updates with restart suppressed, then hands the restart to the systemd user manager asynchronously.

This helper is intentionally a tactical bridge, not the final architecture.
