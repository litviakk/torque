# passe

Passe — passe-partout, passport; eases token-based travel across the network. Create, rotate, and revoke API tokens across different storage backends and external service APIs.

`src/clj/torque/passe/core.clj`

## Usage

```bash
bb torque passe list                   # list all managed tokens + expiry (alias: cles)
bb torque passe cles                   # alias for list
bb torque passe create <service>       # provision token; idempotent (skip if unexpired)
bb torque passe rotate <service>       # new token → overwrite KV → notify → revoke old
bb torque passe revoke <service>       # revoke immediately → create replacement
```

## Scope

Phase 0 covers MLflow tokens only. Adding a new service means adding a management backend — the storage backend and keycloak engine are shared across all services.

**Service discovery.** `passe list` reads the managed service list from `torque.edn :passe :services` (e.g. `[:mlflow]`). Each service name in the list is checked against the token KV for expiry. Passe does not scan the registry; it uses the configured list.

Passe passes `ctx :system.labels` to `registry/service` when resolving service hostnames, so label-scoped registries (Phase 1+) resolve the correct service record. In Phase 0 with a single `:default` partition this is a no-op.

## Backend Types

passe has two pluggable backend types. They are independent axes: any storage backend can pair with any management backend.

### Storage backends

Own all vault I/O — reading credentials, writing and reading token records.

| Backend | Description | Status |
|---------|-------------|--------|
| `bao` | OpenBao KV — reads OIDC credentials, reads/writes composite token entries | Phase 0 |

### Management backends

Own the external service API — token creation, revocation, and any service-specific auth flow.

| Backend | Description | Status |
|---------|-------------|--------|
| `keycloak` | Exchanges OIDC credential for a JWT; required by mlflow-api | Phase 0 |
| `mlflow-api` | Calls MLflow token API: create (90-day expiry), revoke | Phase 0 |

Adding a new managed service = add a management backend. Storage backend and keycloak are reused as-is.

## Engines

Engines implement backend types. Core orchestrates across them; each engine owns one system boundary.

**bao engine** (storage) — reads and writes OpenBao KV. Owns: OIDC credential fetch, token record read/write. Auth: reads `BAO_TOKEN` env var (fallback: `VAULT_TOKEN`); throws `ex-info` with `:category :auth` if absent. Error categories: 401/403 = `:auth`; connection refused/timeout = `:network`; 5xx = `:transient`. Provides: `:passe/kv-read`, `:passe/kv-write`.

**keycloak engine** (management) — exchanges an OIDC credential for a Keycloak JWT. Owns: token endpoint call, JWT validation. Provides: `:passe/jwt-exchange`.

**mlflow-api engine** (management) — calls the MLflow token API. Owns: create (with expiry), revoke. Provides: `:passe/token-create`, `:passe/token-revoke`.

## Orchestration

`passe/core.clj` directly requires and calls `bao.clj`, `keycloak.clj`, and `mlflow_api.clj`. Sub-steps are **not** dispatched through `core/dispatch` and do not appear in the `--plan` effect trace. `--plan` for a passe command returns a description of the high-level workflow steps, not the individual engine calls.

## KV Schema

All token state lives in a single composite entry per service:

```edn
;; path: kv/bot-app/<service>   e.g. kv/bot-app/mlflow
{:token-id             "mlflow-token-abc123"
 :expires-at           "2026-08-18T00:00:00Z"   ; ISO-8601 UTC
 :previous-token-id    "mlflow-token-old456"     ; set during rotate; cleared after revoke
 :previous-expires-at  "2026-05-20T00:00:00Z"   ; kept so rotate can confirm old is safe to drop
}
```

`:previous-token-id` is `nil` at rest. Rotate sets it; revoke clears it. If rotate fails before clearing, the entry is left with both IDs and a re-run of rotate detects the dirty state.

## Workflows

### create

1. **bao** — read `kv/bot-app/<service>`; if `:expires-at` is > 7 days out, return `:skip` (idempotent).
2. **bao** — read `kv/terraform-ci/oidc` → OIDC credential.
3. **keycloak** — exchange credential → JWT.
4. **mlflow-api** — create token, 90-day expiry → `{:token-id, :expires-at}`.
5. **bao** — write composite entry (`:token-id`, `:expires-at`; `:previous-*` fields absent).
6. Return `:ok` with `:detail` showing service name and expiry date.

### rotate

Zero-downtime: old token stays valid until explicit revoke.

1. Steps 2–4 of `create` (skip check bypassed — rotate always provisions a new token).
2. **bao** — read current entry → stash `{:token-id, :expires-at}` as `previous-*`.
3. **bao** — write composite entry: new `:token-id` + `:expires-at`, plus `previous-*` fields.
4. Return `:ok` with `:next-step` "restart bot-app to apply, then run: `bb torque passe revoke <service>`".

### revoke

1. **bao** — read entry; resolve which token to revoke:
   - If `:previous-token-id` exists → revoke that (post-rotate cleanup).
   - If only `:token-id` exists → emergency revoke; provision replacement first (steps 2–4 of create), then revoke old.
2. **mlflow-api** — revoke the target token.
3. **bao** — clear `previous-*` fields (or write fresh entry after emergency replacement).
4. Return `:ok` with `:detail` "token revoked; previous-token-id cleared".

## Idempotency

`create` is idempotent: skips if `expires-at > now + 7d`. Rotate is intentionally non-idempotent (always provisions). A dirty composite entry (both current + previous set) is recoverable — re-running `revoke` clears it.

**Renewal flow.** `create` is the renewal mechanism. Run `bb torque passe create <service>` periodically (cron, pre-deploy hook) to auto-renew tokens approaching expiry. Create provisions a new token when `expires-at ≤ now + 7d` and is safe to run at any frequency.

**Orphaned tokens.** A crash between KV-read and KV-write in rotate step 2 can leave a token created in MLflow but not recorded in KV. That token is orphaned — torque cannot revoke it. Accepted risk: the token expires at the service-configured TTL (MLflow default: 90 days). No manual recovery path is needed.

## Phase 0 Acceptance Criterion

- `bb torque passe list` returns the configured services (from `torque.edn :passe :services`) with their token expiry dates.
- `bb torque passe rotate mlflow` completes the full bao→keycloak→mlflow-api→bao sequence and returns `:ok` with `:next-step` carrying the restart instruction.

## Notification (no restart ownership)

passe does not restart services. After rotate, the result carries `:next-step`:

```clojure
{:status    :ok
 :detail    "token rotated for mlflow; expires 2026-08-18"
 :next-step "restart bot-app to pick up the new token, then run: bb torque passe revoke mlflow"}
```

Any `:fix :effect` that carries `passe rotate` or `passe revoke` must have `:agent-safe false` — these are destructive operations that require human confirmation before dispatch.

Dial renders `:next-step` as a highlighted prompt after the result.

## Open Questions

**OQ-S4: Dirty-state detection.** If rotate crashes after writing the composite entry, the KV has both current + previous IDs set but bot-app hasn't restarted. Should `rotate` on re-run fail fast ("dirty entry, run revoke first") or auto-recover? **Tentative: fail fast and instruct.**

## Registration

`passe` self-registers with core (see [core.md §Module Self-Registration](core.md#module-self-registration)). Engines self-register separately: `bao`, `keycloak`, `mlflow-api` each declare `:provides` keys under the `passe` namespace.

## File Layout

```
src/clj/torque/passe/
  core.clj        — module spec, subcommand dispatch, cross-engine orchestration
  bao.clj         — storage backend: KV read/write, OIDC credential fetch
  keycloak.clj    — management backend: JWT exchange
  mlflow_api.clj  — management backend: token create + revoke
```

## Related Documentation

- [core.md](core.md) — dispatch, engine selection, cross-engine orchestration
- [dial.md](dial.md) — UX layer (`:next-step` rendering)
- [registry.md](registry.md) — service registry (used to resolve service hostnames)
- [torque.md](torque.md) — module table and delivery phases
