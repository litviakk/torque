# core

torque's glue layer. Wires modules to engines, owns CLI dispatch, owns workflow control.

`src/clj/torque/core.clj`

## What core is

Core is the **business-logic glue**: the code that turns a CLI invocation into the right sequence of module + engine calls, then hands results to `dial` for rendering. Core does **not** implement checks or any other dev operation (lint, push, build, deploy, etc) — those live in modules and their engines. Core also does not render output — that's `dial`.

Everything that doesn't belong to a single module or engine belongs here — including the "how do these pieces fit together" decisions that other docs delegate to "the core module".

## Responsibilities

1. **Dispatch.** Parse `bb torque <module> <subcommand> [args] [label-args] [flags]` and route to the registered module entry. Unknown commands fail fast with a usage hint. **Label args** (`:key value` pairs) always trail positional args: `bb torque passe rotate mlflow :env prod`. **`--plan`** is consumed by core before ctx is built — it is not in the Ctx schema. All other flags (`--json`, `--yes`, `--non-interactive`) are in the Ctx schema and propagated to engines.

**`torque.edn` read timing.** Core reads `torque.edn` eagerly at `dispatch` entry — once per invocation, before `build-ctx`. Core calls `(config/load)` directly; the function returns `{}` if the file is absent. No circular dependency: `config/load` is a pure file→map function that does not call `core/register`.
2. **Engine wiring.** Resolve the engine providing the requested function (engines self-declare what they implement — see [Engine Contract](#engine-contract)), propagate parsed flags (`--plan`, `--non-interactive`, `--yes`, `--json`), and orchestrate across engines when a workflow spans more than one. Intra-engine workflow (sequential/parallel, cascade-on-fail) is the engine's choice; cross-engine ordering is core's.
3. **Result plumbing.** Catch `ex-info` from engines, normalise, group by `:layer`, hand the sequence to `dial` for rendering.

## What core is not

- **Not the UX layer.** All output, spinners, formatting go through `dial`.
- **Not a business-operation implementer.** Checks, registry I/O, builds, deploys — and the execution policy around them (timeouts, retries, confirmations) — belong to modules and their engines. Core wires them together; it does not run them.

## Public API (Phase 0)

```clojure
(core/register module-spec)       ; module self-registration (see Module Self-Registration)
(core/register-engine engine-spec); engine self-registration (see Engine Contract)
(core/dispatch args)              ; entry point from bb.edn; args = *command-line-args*
                                  ; returns an execution result map (see below)
(core/resolve-engine fn-key ctx)  ; pick the unique engine providing fn-key under ctx
                                  ; Internal; pinned by tests.
```

`dispatch` is the only function called from `bb.edn`. The others are internal but documented because tests pin them.

### Execution Result

`core/dispatch` produces an execution result map. All observers derive from it — dial renders it for humans, `--json` serializes it, exit code is a pure function of `:status`.

```clojure
{:run-id  "uuid-..."
 :status  :ok | :warn | :fail    ; rollup — worst :status across :results
 :results [{:status   :ok
             :layer    "L4"
             :detail   "TCP connection established"}
            {:status   :fail
             :layer    "L5"
             :category :network
             :detail   "TLS handshake failed"
             :fix      {:hint   "OpenBao CA not trusted locally"
                        :effect {:module :nool :subcommand :install-ca}}}]}
```

**Exit codes** (derived from `:status`):

| Code | Condition |
|------|-----------|
| `0` | `:status` is `:ok` or `:warn` |
| `1` | `:status` is `:fail` |
| `2` | Usage error — unknown command, missing required arg; thrown before execution |

## Module Self-Registration

**There is exactly one `bb.edn` task:** `torque`. It calls `core/dispatch`. Adding a module never requires editing `bb.edn`.

Every module exposes a `register` function (by convention: `torque.<module>.core/register`) that returns a **module spec** — a pure-data declaration of its CLI surface. The spec is the single source of truth: help text, dispatch table, docs lookup, and skill metadata are all derived from it. No duplication.

```clojure
;; in src/clj/torque/nool/core.clj
(defn register []
  {:name        :nool
   :summary     "Diagnostic checks: service health or client connectivity."
   :doc         "Runs a series of checks to verify a service is healthy
                 (default mode, SSH engine, Phase 1) or that this machine
                 can reach it (--local, local engine, Phase 1)."
   :subcommands
   [{:name     :check                                ; default subcommand
     :args     [{:name :service :required true :doc "service name as in registry"}]
     :flags    [{:name :local :type :boolean :doc "client-side checks instead of server-side"}]
     :examples ["bb torque nool mlflow --local"]
     :entry    nool/check}]})
;; Modules declare *what they do*. Engines (see Engine Contract below) declare
;; *what they implement and when they apply*. The module spec carries no
;; engine references; core resolves engines at dispatch time by intent.
```

`core/register` validates the spec and stores it in an internal registry. At dispatch time, core looks up the module by name, parses args/flags against the spec (uniform parser, no hand-rolled per-module parsing), and calls the entry function.

**What registration gives you, for free:**

| Surface | Source | Example |
|---|---|---|
| Dispatch | `:subcommands[].entry` | `bb torque nool mlflow --local` → `nool/check` |
| Help (module) | `:doc` + `:subcommands` | `bb torque nool --help` |
| Help (subcommand) | `:subcommands[].doc/args/flags/examples` | `bb torque nool check --help` |
| Listing | `:name` + `:summary` | `bb torque` (no args) lists all modules |
| Docs lookup | full spec | `bb torque docs nool` returns the spec as EDN/JSON (consumed by skills, agents) |
| Skill metadata | `:summary` + `:examples` | scraped at build time into AI-agent skill descriptors |

**Registration trigger:** `torque.core` requires each module namespace at startup; the act of requiring triggers a `(core/register (register))` call at the bottom of each module's `core.clj`. Modules can also be required directly as a library — in that case the consumer calls `register` explicitly.

**Module `:requires`** — a module spec may declare cross-module effect dependencies:

```clojure
;; in src/clj/torque/passe/core.clj
{:name     :passe
 :requires [{:type :effect :ref :passe/kv-read}
            {:type :effect :ref :passe/kv-write}
            {:type :effect :ref :passe/jwt-exchange}
            {:type :effect :ref :passe/token-create}
            {:type :effect :ref :passe/token-revoke}]
 ...}
```

After all modules and engines are registered (after all `require` calls in `torque.core`), core validates that every declared `:effect` ref has a registered engine implementing it. Validation failure throws `ex-info` with the missing ref and a hint. This is checked once at startup, before first dispatch.

**Contract:**
- A module spec is pure data. No I/O, no side effects until `dispatch` actually runs an entry.
- The same spec drives help, dispatch, and docs. Help text never duplicates information from arg/flag declarations.
- If a module's spec is malformed, `core/register` throws `ex-info` with `:fix` describing the missing or invalid key.
- Module `:requires` checks run after all modules/engines are registered — startup fails if a declared effect has no engine.
- Removing a module = remove the `(require)` line in `torque.core`. No other place to touch.

## Dispatch Context

Core assembles one `ctx` map per invocation and passes it unchanged to every engine function (`:applies?`, `:execute`, `:plan`). Engines read from it; they never write to it.

### Shape

```clojure
{:run-id   "uuid-..."                          ; trace identifier, generated by core
 :cli      {:flags {:local           false     ; parsed CLI flags
                    :json            false
                    :yes             false
                    :non-interactive false}
            :args  {:service :mlflow}}         ; parsed positional args
 :registry {:name     :mlflow                    ; resolved service record (post label filter)
            :hostname "mlflow.dev.balkan.coffee"
            :ports    [443]}
 :system   {:platform :macos                   ; :macos | :linux
            :labels   {:env :dev               ; from torque.edn :labels; always present (min {})
                       :project :shadowing}}}  ; any key/value; used to filter registry entries
```

### Enforcement

Validated with malli at construction time in `core/build-ctx`, before any engine is called. Invalid ctx throws `ex-info` with `:fix`. Engines receive a guaranteed-valid shape and can destructure freely — no defensive checks inside `:applies?` or `:execute`.

```clojure
(def Ctx
  [:map
   [:run-id :uuid]
   [:cli      [:map
               [:flags [:map
                        [:local           {:optional true} :boolean]
                        [:json            {:optional true} :boolean]
                        [:yes             {:optional true} :boolean]
                        [:non-interactive {:optional true} :boolean]]]
               [:args :map]]]
   [:registry {:optional true} [:map
               [:name     :keyword]
               [:hostname :string]
               [:ports    [:vector :int]]]]
   [:system   [:map
               [:platform [:enum :macos :linux]]
               [:labels   [:map-of :keyword :keyword]]]]])
```

The schema is the contract — it lives in code, not docs, so it can't drift.

## Engine Contract

Engines implement module-declared operations. They are the only layer that talks to the world (subprocesses, network, filesystem, terminal prompts). The contract below applies to **every** engine — nool's local and SSH engines, the TF registry engine, and future build/deploy engines.

### Engine self-registration

Engines register with core. Modules **do not** carry engine references — the wiring is engine-side, not module-side. This keeps modules portable as a library and lets new engines drop in without touching module code.

`core/register-engine` registers two things simultaneously: the engine for dispatch (`:applies?`, `:execute`, `:plan`, `:requires`, `:ready?`) and the doc for capability discovery (`:doc`). `bb torque docs <module>` aggregates the module spec with all registered engine docs into a single response for agents and humans.

```clojure
;; in src/clj/torque/nool/local.clj
(defn register []
  {:provides :nool/check                     ; the function this engine implements
   :applies? (fn [ctx] (-> ctx :cli :flags :local boolean))  ; pure predicate — no I/O
   :ready?   local-engine/ready?             ; optional — effectful probe (I/O allowed);
                                             ;   checks runtime conditions (tunnel up, CA trusted)
   :requires [{:type :tool :name :wg}]       ; static deps checked at registration:
                                             ;   :tool = binary must be on PATH
                                             ;   :effect = EffectRef must have a registered engine
   :platform #{:macos}                       ; supported platforms; fail-fast elsewhere
   :execute  local-engine/run                ; (execute ctx) → seq of result maps
   :plan     local-engine/plan               ; (plan ctx) → pure data description; REQUIRED
   :doc    {:summary  "Check connectivity to a service from your local machine"
              :when     "Use when verifying you can reach a service over WireGuard"
              :examples ["bb torque nool mlflow --local"]}}) ; loaded at registration;
                                             ; drives bb torque docs + agent capability discovery
```

At dispatch time, core enumerates engines providing the requested function, filters by `:applies?` against the live context, and picks the unique match. Resolution outcomes:

- **Exactly one match** — invoke `:execute` (or `:plan` if `--plan` is set; `:plan` is required — an engine without it fails registration).
- **No match** — fail fast with `ex-info` whose `:fix` names the function and the active context (e.g. `"no engine implements :nool/check for flags {:local false}; --local selects the local engine"`).
- **More than one match** — `:applies?` mutual exclusion is a discipline requirement, not a mechanical guarantee. Core cannot evaluate `:applies?` at registration without a live ctx. At dispatch, if multiple engines match the same `:provides` key, core throws `ex-info` with `:category :config` naming both engines — no priority ordering; ambiguity is always an error.

`:provides` is namespaced: `:<module>/<function>`. This is what the module's entry function calls into; the module doesn't know which engine answered.

**Static vs runtime dependencies:**

| Field | Purpose | When checked | I/O allowed |
|---|---|---|---|
| `:requires` | Static wiring deps — tools on PATH, registered effects | At registration time | No |
| `:ready?` | Runtime conditions — tunnel active, CA trusted, file exists | At dispatch time, before `:execute` | Yes |

`:requires` accepts two shapes:
```clojure
{:type :tool   :name :wg}              ; binary must be on PATH
{:type :effect :ref  :registry/sync}   ; EffectRef must have a registered engine
```

If a `:requires` check fails at registration, core throws immediately — the engine is not registered. If `:ready?` returns falsy at dispatch, core fails fast with the engine's `:fix` before any execution begins.

### Result contract

Engines return result maps. Each map:

```clojure
{:status   :ok | :warn | :fail | :skip
 :category :transient | :permanent | :config | :auth | :network | :usage | :infra
            ; required when :status is not :ok; lets agents dispatch on failure type
 :detail   "human-readable result"
 :fix      {:hint     "WireGuard tunnel is down"               ; always present
            :effect   {:module     :nool                      ; optional — structured
                       :subcommand :check                     ; torque call; core
                       :args       {:service :gateway}        ; dispatches it
                       :flags      {:local true}
                       :agent-safe true                       ; default false; true = medicin may
                       :requires   #{:wireguard}}}            ;   auto-dispatch without human gate
            ; :fix required when :status is not :ok; :effect and :requires optional within :fix
            ; :agent-safe false = destructive ops (passe rotate/revoke); must not be auto-dispatched
 :layer    "L3" | "L4" | ...           ; optional; groups results in output
 :service  :mlflow                     ; optional; included when relevant
 :next-step "human-readable follow-up action"  ; optional; rendered by dial as a highlighted prompt
}
```

**Return vs throw.** Engines **return** result maps for *expected* outcomes — including diagnostic failures (`:fail`). Engines **throw** `ex-info` only for *engine-internal errors* the engine itself cannot turn into a meaningful result: subprocess crash, missing binary, malformed inventory, network unreachable from the runner.

`ex-info` carries the same `:category` and `:fix` shape:
```clojure
(throw (ex-info "SSH binary not found"
                {:category :config
                 :fix      {:hint   "SSH binary not found on PATH"
                            :effect {:module :nool :subcommand :install-deps}}}))
```

Dial renders both shapes uniformly. The distinction matters because returned `:fail` participates in cascade and aggregation; a thrown `ex-info` aborts the current engine invocation.

**Scrubbing.** Core strips the `:data` key from thrown `ex-info` maps before emitting to audit records and `--json` output — only `:message`, `:category`, `:fix` are propagated. Engine `:detail` strings must not include credential values, tokens, or raw HTTP response headers.

### Execution policy (owned by the engine)

The engine, not dial and not core, implements:

- **Timeouts** — kill operations that exceed the per-check/per-step limit.
- **Retries + backoff** — for operations the engine knows are safely retryable.
- **Confirmations** — `[y/N]` prompts for destructive ops; respect `--yes` and `--non-interactive`.
- **Non-interactive mode** — when `--non-interactive` or `--json` is set, no prompts; structured output only.

Core propagates the flags; engines honour them. `--plan` is routed by core: it calls `:plan` instead of `:execute` on the resolved engine. Every engine must implement `:plan`.

### Intra-engine workflow

The engine decides whether its checks run sequentially or in parallel, and whether to cascade-on-failure inside the pipeline. Core enforces only the public contract: results arrive in a defined order, cascade emits `:skip` for downstream entries when an upstream is `:fail`. Cross-engine orchestration (when a workflow spans more than one engine) is core's concern, not the engine's.

### Phase 0 effective resolution

| Function | Engine | Applies when |
|---|---|---|
| `:registry/sync` | `ansible` | always (only engine in Phase 0) |
| `:passe/kv-read`, `:passe/kv-write` | `bao` | always |
| `:passe/jwt-exchange` | `keycloak` | always |
| `:passe/token-create`, `:passe/token-revoke` | `mlflow-api` | always |

nool engines ship in Phase 1. See [torque.md §Delivery Phases](torque.md#delivery-phases).

## Cross-engine orchestration (Phase 0)

Intra-engine workflow is each engine's concern (see the contract in `nool.md`). Core's orchestration job kicks in when a workflow spans engines or modules.

Phase 0 cases:

- **`registry sync` (single shot)** — load inventory → validate → write `registry.edn`. One engine, one step.
- **`passe` (multi-engine sequence)** — bao → keycloak → mlflow-api → bao. Core sequences the steps, propagates flags, halts on failure. See [passe.md](passe.md).
- **`torque init` (bootstrap alias)** — hard-wired core alias, not a registered module. Dispatches `config init` then `registry init` in sequence. Does not appear in `bb torque` module listing. Respects `--yes` and `--non-interactive`. This hard-coding is intentional and permanent — `torque init` is a one-off bootstrap alias, not a precedent. `build init`, `deploy init`, etc. are subcommands registered by their modules and do not follow the alias pattern. There is no alias registration API.
- **`medicin` (runtime)** — dispatches each `:agent-safe true` `:fix :effect` once via `core/dispatch`. Single round per invocation: does not re-run `:ready?` probes after applying fixes, does not recurse. `:agent-safe false` effects are surfaced to the operator but not auto-dispatched.

Phase 2+ will introduce real multi-engine workflows (e.g. `deploy` chains `build → registry update → service restart`, each potentially on a different engine). Core orchestrates those — orders steps, propagates flags, aggregates results, halts on failure — without dictating each engine's internal execution shape.

## Audit Record

Core emits one audit record per invocation to **stderr** as newline-terminated JSON. No other module writes audit records.

```clojure
{:run-id      "uuid-..."         ; same as ctx :run-id
 :timestamp   "2026-05-21T..."   ; ISO-8601 UTC, invocation start
 :user        "k"                ; $USER env var
 :module      :nool              ; module name
 :subcommand  :check             ; subcommand name
 :status      :ok                ; rollup :status from execution result
 :exit-code   0                  ; 0, 1, or 2 (see exit codes above)
 :duration-ms 142}               ; wall-clock ms from parse to result
```

Audit records are ephemeral in Phase 0 — they go to stderr and are not persisted. Redirect stderr to capture them. Phase 2 ships the `logs` module with persistent, queryable backends.

**`--json` and audit are independent streams.** When `--json` is set: stdout = JSON execution result, stderr = JSON audit record. Both are emitted; they do not interfere.

## Agent Contract

The machine-readable surface agents couple to. These shapes are stable across patch releases within a phase; breaking changes move to the next phase.

### `--json` stdout

When `--json` is set, dial suppresses all human output and writes one JSON object to stdout:

```json
{
  "run-id": "uuid-...",
  "status": "ok",
  "results": [
    {"status": "ok",  "layer": "L4", "detail": "TCP connection established"},
    {"status": "fail","layer": "L5", "category": "network",
     "detail": "TLS handshake failed",
     "fix": {"hint": "OpenBao CA not trusted locally",
             "effect": {"module": "nool", "subcommand": "install-ca"}}}
  ]
}
```

`:category`, `:layer`, `:fix`, `:next-step` are present only when populated by the engine.

### `bb torque docs <module>`

Returns the registered module spec merged with its engines' `:doc` entries, serialised as EDN (default) or JSON (`--json`):

```edn
{:name        :nool
 :summary     "Diagnostic checks: service health or client connectivity."
 :doc         "..."
 :subcommands [{:name :check :args [...] :flags [...] :examples [...]}]
 :engines     [{:provides :nool/check
                :platform #{:macos}
                :doc {:summary "..." :when "..." :examples [...]}}]}
```

### Exit codes

| Code | Condition |
|------|-----------|
| `0` | `:status` is `:ok` or `:warn` |
| `1` | `:status` is `:fail` |
| `2` | Usage error — unknown command, missing required arg |

## Testing

See [testing.md](testing.md). Core tests cover **engine selection**, **cross-engine orchestration** (step ordering, halt-on-failure across engines), and **flag propagation** — using stubbed engines so no real I/O happens. Intra-engine workflow (sequential/parallel inside one engine) is tested by the engine, not by core.

## Related Documentation

- [torque.md](torque.md) — top-level architecture and module table
- [dial.md](dial.md) — UX contract that core's results flow into
- [nool.md](nool.md) — module that core dispatches checks to
- [registry.md](registry.md) — interface core reads from
- [testing.md](testing.md) — contract-driven test strategy
