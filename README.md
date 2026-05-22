# torque

Swiss army knife ops CLI for the infrastructure. Single `bb torque` entry point covering the full operational lifecycle — diagnose, secrets, registry, build, deploy.

```bash
bb torque nool mlflow --local     # can I reach mlflow from here?
bb torque nool mlflow             # is mlflow healthy on the server?
bb torque passe cles              # list managed tokens + expiry
bb torque passe rotate mlflow     # rotate mlflow API token
bb torque registry list           # list services in active partition
bb torque config set :labels :env :dev  # set active label context
```

## Quick start

```bash
bb torque init          # create torque.edn + registry.edn interactively
bb torque               # list all modules
bb torque <module> --help
```

## Docs

| Doc | What it covers |
|-----|---------------|
| [torque.md](spec/torque.md) | Vision, scope, delivery phases, architecture |
| [core.md](spec/core.md) | Dispatch, engine contract, ctx schema, agent contract |
| [config.md](spec/config.md) | `torque.edn` — init, get, set, list |
| [registry.md](spec/registry.md) | `registry.edn` — service metadata, label partitions |
| [passe.md](spec/passe.md) | Token lifecycle — create, rotate, revoke |
| [nool.md](spec/nool.md) | Diagnostic checks — local + SSH engines |
| [dial.md](spec/dial.md) | UX layer — spinners, rendering, JSON output |
| [testing.md](spec/testing.md) | Contract-driven test strategy |
| [engines/](spec/engines/) | Engine specs (nool local, nool SSH, TF registry) |

## Key concepts

- **Modules** declare intents; **engines** implement them; **core** dispatches
- `registry.edn` — do not commit to git (contains private hostnames, IPs, env vars)
- `torque.edn` — user config; optional; torque runs with defaults if absent
- `--plan` — returns the effect description as data without executing anything
- `--json` — machine-readable output for agents and CI
- Label args always trail positional args: `bb torque passe rotate mlflow :env prod`
