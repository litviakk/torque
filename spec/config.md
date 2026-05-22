# config

torque's configuration module. Owns `torque.edn` — reads, writes, and initialises it.

`src/clj/torque/config/core.clj`

## What config is

Config is the module that manages `torque.edn`. It owns the file lifecycle: creating it from scratch, reading values, and writing values. No other module writes `torque.edn`.

## What config is not

- **Not a validator of other modules' concerns.** Config reads and writes `torque.edn` as-is. Modules validate their own slice of config when they need it — not config's job.
- **Not required for torque to run.** `torque.edn` is optional. Torque operates with defaults when it is absent.

## Usage

```bash
bb torque config                        # interactive editor — show current values, edit in place
bb torque config init                   # interactive wizard — create torque.edn with prompts
bb torque config list                   # print all current config values (non-interactive)
bb torque config get :labels            # read the global label context
bb torque config set :labels :env :dev  # write a single label key/value
bb torque init                          # alias: runs config init then registry init in sequence
```

**Interactive modes.** `config init` and `config` (no subcommand) are separate implementations with separate flows — neither delegates to the other:

- **`config init`** (wizard) — first-run. Prompts for every key including `:passe :services` (default `[]`, hint: "space-separated service names managed by passe, e.g. mlflow"). Creates `torque.edn` from scratch. Equivalent to `npm init`.
- **`config`** (interactive editor) — edit existing values. For each key currently in `torque.edn`, prints the current value and prompts to accept (press Enter) or replace (type a new value). Keys absent from the file are skipped — use `config set` to add them.

Both modes share the same visual style in Phase 0/1 (line-by-line prompts). In Phase 2 both use `dial`'s TUI component library — same components, separate code paths. Both respect `--yes` (accept all values without prompting) and `--non-interactive` (skip prompts, use current values, no output beyond errors).

**Key paths.** Nested keys are space-separated keyword segments: `:labels :env` resolves to `(get-in config [:labels :env])`. `config set` uses `assoc-in` with the same path. Any depth is supported.

**`config set` value syntax.** All args form the key path except the last, which is the scalar value. One setting per invocation. To set multiple values, run `config set` once per key or use `bb torque config` (interactive editor).

```bash
bb torque config set :registry :backend :local
# path = [:registry :backend], value = :local

bb torque config set :labels :env :dev
# path = [:labels :env], value = :dev

bb torque config set :labels :project :shadowing
# path = [:labels :project], value = :shadowing
```

Any number of path segments is supported; the last arg is always the value. Fewer than two args (no path + value) throws `ex-info` with a usage hint.

## torque.edn

`torque.edn` is user-owned config. Discovery order at startup:

1. Path in `TORQUE_CONFIG` env var (if set)
2. `./torque.edn` in the current working directory

If neither is found, torque runs with defaults — no error.

### Shape

```edn
{:hash     "sha256-abc123..."        ; written by torque; detects out-of-band edits on next load
 :registry {:backend :local          ; :local | :s3 | :git (future)
            :path    "registry.edn"} ; path relative to config file location
 :labels   {:env :dev}               ; global label context — applied to all registry lookups
 :passe    {:services [:mlflow]}}    ; services managed by passe (used by passe list)
```

`:hash` is a SHA-256 of the EDN content excluding the `:hash` field itself, written by `config/write` (used by `config init` and `config set`). On load, `config/load` recomputes and compares — a mismatch means the file was edited outside of torque and emits a warning. Manual edits are valid; the warning prompts you to run `config set` or `config init` to refresh the hash.

All keys are optional. `config init` generates this skeleton with comments.

### Defaults (when torque.edn is absent or key is missing)

| Key path | Default |
|----------|---------|
| `:registry :backend` | `:local` |
| `:registry :path` | `"registry.edn"` (CWD) |
| `:labels` | none — falls back to `:default` partition or alphabetical |
| `:passe :services` | `[]` — no managed services; `passe list` outputs empty |

## `bb torque init`

Core-level bootstrap alias — not a registered module, does not appear in `bb torque` listing. Hard-wired in `core.clj` to run:

1. `bb torque config init` — interactive wizard to create `torque.edn`
2. `bb torque registry init` — creates skeleton `registry.edn`

Prompts before overwriting either file if it already exists. Respects `--yes` and `--non-interactive`.

Phase 1 addition: after both files are created, prompts "Run connectivity checks now? [y/N]" and optionally dispatches to `nool`.

## Registration

`config` self-registers with core. See `torque.config.core/register` and the contract in [core.md](core.md#module-self-registration).

## Related Documentation

- [core.md](core.md) — dispatch, torque.edn discovery
- [registry.md](registry.md) — registry module (owns registry.edn; called by `torque init`)
- [torque.md](torque.md) — module table and delivery phases
