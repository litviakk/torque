# Torque Testing Strategy

Testing is **contract-driven**. Each module and engine declares a contract; tests verify that contract holds.

## Philosophy

- **Contracts are the unit of test.** A function's promise (inputs, outputs, errors) is what tests should pin down. Implementation can change; the contract is stable.
- **Core tests = glue.** The core business logic module (`src/clj/torque/core.clj`) wires modules and dispatches to engines. Core tests verify the wiring — that the right engine is selected, that results flow back, that workflow control (sequential/parallel, cascade on failure) behaves as declared.
- **Engine tests = implementation.** Each engine tests its own implementation of the principle it owns (e.g. the local nool engine tests that an L4 TCP check actually opens a socket and returns a result map in the agreed shape).
- **Don't duplicate library tests.** Don't re-test `babashka.process` or `clojure.spec`. Trust deps.

**North star:** "If this test passes, the contract holds. If it fails, the contract is broken."

---

## What each layer tests

### Core (`torque.core`)

The glue. Tests cover:
- **Engine selection** — given a command + flags + config, the correct engine is dispatched.
- **Workflow control** — sequential vs. parallel check execution, cascade-on-failure, flag propagation (`--plan`, `--non-interactive`, `--yes`) from CLI down to the chosen engine.
- **Result aggregation** — engine results reach dial in the expected shape.

Tests live in `test/clj/torque/core_test.clj`. No real I/O; engines are stubbed.

### Modules (`config`, `registry`, `nool`, `passe`, `dial`)

Each module declares an interface contract (see the module's `.md`). Tests verify that contract independent of engine.

Examples:
- `registry/service :mlflow` returns a single service record; throws `ex-info` if not found.
- `registry/services` returns a sequence of service records for the resolved partition; `registry/services filter` honors the filter shape (Phase 1+).
- `registry/validate!` throws `ex-info` with a `:fix` hint on schema violation.
- `dial/render` produces a string containing the status icon and detail.

These tests use in-memory data; no engine is exercised.

### Engines (`nool-local`, `nool-ssh`, `tf-registry`)

Each engine tests **the principle it implements** first (mandatory), then **nuances** (optional, expanded over time).

Mandatory:
- `nool-local` (Phase 1): an L4 TCP check opens a socket and returns the agreed result map.
- `nool-ssh` (Phase 1): can dispatch a command to a server and parse the output into the agreed result map.
- `tf-registry` (Phase 1): can read a Terraform-derived inventory and write a valid `registry.edn`.

Optional (expand over phases):
- Edge cases (malformed inventory, transient SSH failures, platform-detection fallback, etc.).

### Meta-test: contract coverage

For each module that declares "engines must implement X" (e.g. `dial.md` Engine Contract), a meta-test verifies every registered engine actually implements X. Catches drift between contract and engines.

---

## What's out of scope (for now)

- **Full integration tests across engines.** Each engine is tested in isolation. Cross-engine integration is deferred.
- **CI pipeline definition.** Not in scope for this document. Tests should be cheap enough to run locally; CI hookup is a later concern.
- **Manual test plans.** Users will report issues; that loop is the manual signal.
- **Comprehensive happy-path coverage.** Aim is critical paths, not exhaustiveness.

---

## Running tests

```bash
bb test:clj
```

Test files live under `test/clj/torque/...` mirroring the source tree.
