# registry

torque's shared service registry. Single source of truth for service metadata — pure EDN, no code.

`src/clj/torque/registry.edn`

## Concept

You own the registry. It is your curated source of truth for service metadata — you edit it directly and all torque modules read from it. No hardcoded hostnames or IPs anywhere in module code.

**Do not commit `registry.edn` to git.** It may contain private information: internal hostnames, IP addresses, environment variables, and service credentials (e.g. `:env {:MLFLOW_BACKEND_STORE_URI "postgresql://..."}`). Keep it out of version control — add it to `.gitignore`. For team sharing, use a remote backend (S3 or git-backed private store — Phase 2).

`registry sync` (TF-backed) is a convenience that automates populating the registry from Terraform state — not a requirement. The baseline is a hand-edited EDN file.

Remote backends (S3, git) for team collaboration and CI are a future concern, inspired by how Terraform handles remote state. For now: local file, you own it.

**Scope (Phase 0–1)**: Services only. Artifacts and images stay in the build module, not in the registry.

## Interface Contract

The **registry module** provides a contract for reading service metadata:

```clojure
(registry/load)                      ; load registry.edn from disk and validate — never returns an invalid registry
(registry/service name)              ; return single service by name (default partition); throws ex-info if not found
(registry/service name labels)       ; return single service by name in the partition matching labels
(registry/services)                  ; return all services in the resolved default label partition
(registry/services filter)           ; return services matching filter (Phase 1+)
(registry/validate!)                 ; validate schema, throw ex-info on error
```

**Default label resolution.** `(registry/services)` and `(registry/service name)` with no explicit label resolve the active partition in this order:
1. Default label from `torque.edn` `{:labels ...}` if set
2. `:default` partition in registry.edn if present
3. Alphabetical — first partition by label key/value wins

**Multi-key selector matching.** When the active selector has multiple keys (e.g. `{:env :dev :project :shadowing}`), a partition matches if all its keys are present in the selector with equal values (subset match). Extra keys in the selector are ignored — `{:env :dev}` matches `{:env :dev :project :shadowing}`. If multiple partitions subset-match, the first alphabetically by key-name wins.

`torque.edn` is optional. `:default` partition is the recommended Phase 0 convention (explicit beats implicit). Alphabetical is the last-resort fallback for labelled multi-partition registries with no configured default.

**Filtering.** The filter grammar (Phase 1+) is intentionally deferred until multiple partitions exist and callers need more than the no-arg form. See [open question](#open-questions).

Pluggable engines implement how to populate the registry:

| Engine | Source | Status |
|--------|--------|--------|
| `ansible` | `inventory.yml` — already exists; Phase 0 | Phase 0 |
| `tf` | Terraform state + outputs | Phase 1 |

See [engines/tf-registry.md](engines/tf-registry.md).

## Open Questions

**OQ-R1: Filter grammar.** What concrete shapes does the filter argument accept (keyword for exact-name match, predicate map, predicate function, query DSL)? What is the lookup-vs-search semantics (does an unmatched name throw, or return empty)? **Decision needed before Phase 1 when multiple partitions exist.**

## Data Shape

Registry contains services only. Artifacts and images are in the build module.

**Partition key type:**

```clojure
(def LabelSelector
  [:or :keyword [:map-of :keyword :keyword]])
```

Partition keys are either a bare keyword (`:default`) or a keyword→keyword map (`{:env :dev}`).

**Phase 0 — single default partition:**

```edn
{:default
  [{:name     :mlflow
    :label    "MLflow Tracking"
    :hostname "mlflow.dev.balkan.coffee"
    :ip       "10.211.139.10"
    :ports    [443]
    :backend  :docker
    :env      {:MLFLOW_BACKEND_STORE_URI "postgresql://..."}}]}
```

**Phase 1+ — env-labelled partitions:**

```edn
{{:env :dev}
  [{:name     :mlflow
    :label    "MLflow Tracking"
    :hostname "mlflow.dev.balkan.coffee"
    :ip       "10.211.139.10"
    :ports    [443]
    :backend  :docker
    :env      {:MLFLOW_BACKEND_STORE_URI "postgresql://..."}}
   {:name     :keycloak
    :label    "Keycloak"
    :hostname "keycloak.dev.balkan.coffee"
    :ports    [443]
    :backend  :docker}]

 {:env :prod}
  [{:name     :mlflow
    :label    "MLflow Tracking"
    :hostname "mlflow.balkan.coffee"
    :ip       "10.0.0.10"
    :ports    [443]
    :backend  :docker
    :env      {:MLFLOW_BACKEND_STORE_URI "postgresql://..."}}]}
```

**Structure:** the top-level map is keyed by `LabelSelector`. Each key is a label partition; each value is the services in that partition. The tf-registry engine owns exactly one key per sync run — it writes its partition and nothing else.

**EDN-only constraint.** Phase 1 partition keys (`{:env :dev}`) are Clojure maps — valid EDN but not valid JSON map keys. The registry is EDN-native. Agent consumers working with partition keys must parse them from EDN strings, not JSON. `--json` serializes partition keys as stringified EDN (e.g. `"{:env :dev}"`).

**Lookup:** `registry/services` with a selector finds the partition whose key matches the selector and returns its services.

**Service fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `:name` | yes | Logical service name — used in CLI args (`bb torque nool mlflow`) |
| `:label` | yes | Human-friendly display name |
| `:hostname` | yes | DNS FQDN for the service |
| `:ip` | yes (Phase 1) | Internal IP for DNS resolution check (nool local engine). **Phase 0 entries may omit this field** — it is required by the nool L3 check which ships Phase 1 alongside this field. |
| `:ports` | yes | TCP ports to probe |
| `:backend` | no | Backend type (`:docker`, `:systemd`, etc.) for deploy module |
| `:env` | no | Environment variables for the service |
| `:cert-source` | no | CA that issued the service cert: `:lets-encrypt` (default) \| `:openbao` \| `:incus`. Required by nool L6 cert check for non-public CAs. |

## CLI

```bash
bb torque registry init              # create skeleton registry.edn with :default partition
bb torque registry list              # list services in default partition
bb torque registry list :env dev     # list services in {:env :dev} partition
bb torque registry get mlflow        # get single service by name (default partition)
bb torque registry get mlflow :env prod  # get service in specific partition
bb torque registry sync              # populate registry from inventory (Ansible Phase 0; TF Phase 1)
```

**Label args** follow the `:key value` convention, consistent with `bb tf :env dev`. Multiple labels: `:env dev :project shadowing`. Labels are optional — omit to use default partition resolution.

## Registration

`registry` self-registers with core. See `torque.registry.core/register` and the contract in [core.md](core.md#module-self-registration). The spec declares the `:registry` module, the `init`/`sync`/`list`/`get` subcommands, the `:ansible` engine (Phase 0) and `:tf` engine (Phase 1), and help text. No `bb.edn` task is added per module.

## Schema Validation

The `registry` module validates on load — throws `ex-info` with a fix hint if schema is violated.

**Services require:** `:name`, `:label`, `:hostname`, `:ports`

**Non-empty:** an empty registry (no partitions) fails validation with: `"registry.edn has no partitions. Run: bb torque registry init"`

## Related Documentation

- [engines/tf-registry.md](engines/tf-registry.md) — Terraform-backed sync engine (how registry is populated)
- [config.md](config.md) — torque config module (owns torque.edn)
- [nool.md](nool.md) — diagnostic checks (reads from registry)
