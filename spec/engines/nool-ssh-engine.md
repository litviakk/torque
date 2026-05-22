# nool SSH engine — server-side health checks

Server-side health checks. Runs checks **on the server** via SSH to verify service is healthy from the infrastructure's perspective.

`src/clj/torque/nool/ssh.clj` (Phase 1, planned)

## Check Pipeline

Checks run on the server via SSH. Downstream skipped if upstream failed (cascade logic).

| Layer | Check | What it verifies |
|-------|-------|------------------|
| **Process** | systemd unit active? | `systemctl is-active <service>` |
| **Socket** | Listening on expected port? | `ss -tlnp` or equivalent, check port in output |
| **L7** | Localhost HTTP response | `curl http://localhost:<port>/health` from inside container |

All checks run **on the server**. No TLS checks here — internal services use plain HTTP on localhost; TLS terminates at nginx.

## Context: Incus Services

Internal services (mlflow, keycloak, zot, openbao) run in Incus containers on a VPS. Dnsmasq resolves `.incus` hostnames and `*.dev.balkan.coffee` subdomains. 

The SSH engine logs into the VPS and runs health checks **inside each service container** or on the host Incus system.

## Platform Support

The SSH engine runs checks **on the server** — it is Linux-only. The client that initiates SSH may be any platform; only the target host matters, and all managed services run on Linux.

The contract for being called on an unsupported target: the engine checks the target platform via SSH before running health checks and throws `ex-info` with `:fix "SSH engine targets Linux services only"` if the target reports a non-Linux OS.

## Phase 1 Scope

SSH engine ships in Phase 1 after:
- Local engine (Phase 1) proves nool architecture works end-to-end
- Decisions on SSH execution mechanism (OQ-A) are made
- All services in registry (not just `:mlflow`)

## Open Questions

**OQ-A: SSH execution mechanism (Phase 1)** — How should SSH engine execute checks on the server? Options:
1. Direct SSH: `ssh mlflow.dev.balkan.coffee 'systemctl is-active mlflow'`
2. Babashka's `babashka.process/shell` with SSH flag (if available)
3. Ansible integration: leverage existing ansible-inventory to get SSH host config
4. Custom Clojure SSH client (heavy)

**Decision needed before Phase 1 SSH engine implementation.**

## Contract

Check functions implement the nool engine contract:
- Accept service name + SSH config
- Return result map: `:status`, `:detail`, `:fix`
- Throw `ex-info` with error details on exception
- Engine owns execution policy: timeouts, retries, confirmations. See [core.md §Execution policy](../core.md#execution-policy-owned-by-the-engine).
