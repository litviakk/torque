# dial

torque's UX contract. dial is an **observer of the effect trace** — it consumes structured results emitted by engines and renders them. It does not drive execution.

`src/clj/torque/dial.clj`

## Responsibilities

- **Spinners** — animated progress indicators during long operations (interactive only).
- **Result rendering** — formats structured result maps (`:status`, `:detail`, `:fix`, `:layer`) for display; adapts to interactive vs `--non-interactive`/`--json` context.
- **Next-step rendering** — renders `:next-step` from result maps as a highlighted follow-up prompt after the result.
- **Error formatting** — catches `ex-info`, extracts message + `:fix` hint, renders uniformly with returned results.

The result-map shape and the return-vs-throw rule are defined in [core.md §Engine Contract](core.md#engine-contract). Dial consumes the contract; it does not define it.

## What dial is not

- **Not the policy owner.** Timeouts, retries, confirmations, non-interactive mode — all engine concerns. Dial does not enforce them; engines do.
- **Not the cascade decider.** Cascade-on-failure is intra-engine workflow. The engine emits `:skip` for downstream entries when an upstream is `:fail`; dial renders what arrives, in the order it arrives.
- **Not a logic layer.** Dial never decides what's wrong, only how to present it.

## Principles

- **Pure UX layer.** Dial owns *how things look*. Nothing about correctness lives here.
- **No UX outside dial.** Modules and engines never call `println`, never manage spinners, never render. All output goes through dial.

## Public API

```clojure
(dial/render result)                 ; render one result (formatting only)
(dial/render-grouped results)        ; render sequence, grouped by :layer
(dial/spinner label f)               ; show spinner while f runs (interactive only); return f's value
```

**Note on `dial/spinner`.** Write a fresh spinner. `apps/bot-app/src/build_artifact/artifact.clj` is a reference implementation — do not port it directly as it lives outside the torque namespace.

## JSON output (`--json`)

When `--json` is set in ctx, dial serialises the full execution result map (as returned by `core/dispatch`) to stdout as JSON and suppresses all other output. The shape mirrors the Clojure map exactly — `:status`, `:results`, `:run-id`. Engines and modules never branch on `--json`; dial detects it at render time.

