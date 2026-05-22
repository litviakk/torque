# nool

torque's diagnostic module. Runs a series of checks to verify service health or connectivity.

`src/clj/torque/nool/core.clj`

## Usage

```bash
bb torque nool <service>          # service health checks (SSH engine, Phase 1)
bb torque nool <service> --local  # client-side connectivity checks (local engine, Phase 1)
```

nool ships in Phase 1 — there is no Phase 0 stub. In Phase 0, `bb torque nool` returns core's standard "unknown module" error with a usage hint. No special wiring required.

## Engines

nool delegates check execution to pluggable engines. Each engine implements the same check interface; the difference is **where** and **how** checks run.

**Local nool engine (`--local`)** — checks if *this client* can reach the service. Runs on your local machine. See [engines/nool-local-engine.md](engines/nool-local-engine.md).

**SSH nool engine (default)** — checks if the service is healthy **from the server's perspective**. SSH to the server, run checks inside the service container or on the host. Phase 1. See [engines/nool-ssh-engine.md](engines/nool-ssh-engine.md).

## Module Contract

nool's public surface is a single function — `:nool/check` — that takes a service name and returns a sequence of result maps. The shape of those maps, the return-vs-throw rule, execution policy, and platform support are defined in [core.md §Engine Contract](core.md#engine-contract); nool does not redefine them.

What is nool-specific:
- **Function key**: `:nool/check`. Engines providing nool implement this.
- **Selection**: the `--local` flag picks the local engine; absence picks the SSH engine. Concrete selection lives in the engines' `:applies?` predicates (see [core.md §Engine Contract](core.md#engine-contract)).
- **Check layers**: engines return results tagged with `:layer` so dial can group them (e.g. L3/L4/L5/L6/L7 for the local engine). The set of layers is engine-specific.

## Registration

`nool` self-registers with core. See `torque.nool.core/register` and the contract in [core.md](core.md#module-self-registration). The spec declares the `:nool` module, its single subcommand (`check`), the `--local` flag, and help text. No `bb.edn` task is added per module; engines self-register separately (see [core.md §Engine Contract](core.md#engine-contract)).

## Related Documentation

- [engines/nool-local-engine.md](engines/nool-local-engine.md) — local nool engine: L3–L7 client-side checks (macOS, Phase 1)
- [engines/nool-ssh-engine.md](engines/nool-ssh-engine.md) — SSH nool engine: server-side health checks via SSH (Phase 1)
- [core.md](core.md) — dispatch, engine selection, workflow control
- [dial.md](dial.md) — UX layer (spinner, result rendering)
- [registry.md](registry.md) — service registry
