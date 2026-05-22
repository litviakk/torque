# nool local engine — client-side connectivity checks

Client-side connectivity checks. Runs on your local machine (developer machine or CI runner) to verify you can reach a service over the network.

`src/clj/torque/nool/local.clj` (Phase 1)

## Check Pipeline

Checks run on the developer's machine. Layer L3 → L7, cascade on failure.

| Layer | Check | What it verifies | Platform |
|-------|-------|------------------|----------|
| **L3** | WireGuard tunnel | Is the tunnel active? | macOS only (Phase 1) |
| **L3** | DNS resolver | Is the VPN DNS resolver active? | macOS only (Phase 1) |
| **L3** | DNS resolution | Does service hostname resolve to expected IP? | macOS only (Phase 1) |
| **L4** | Transport: TCP | Can we open a TCP connection to the service port? | all |
| **L5** | Session: TLS | Does the TLS handshake complete? | all |
| **L6** | Certificate | Is the cert trusted and issued for this hostname? | all |
| **L7** | Application: HTTP | Does the service return expected status code on public endpoint? | all |

Each check returns a result map with `:status`, `:detail`, `:fix`. Renders with cascade: if TCP fails, TLS/cert/HTTP skipped.

## Platform Support (contract)

Phase 1 supports **macOS only** (arm64 + amd64) — those are the only platforms the team runs today.

The contract for unsupported platforms is: **fail fast with a clear error message naming the platform**. Not a silent skip, not "best effort". Implementation:

- Check functions declare `:platform #{:macos}` (a set; expands when more platforms are added).
- When invoked on a platform not in the set, the engine throws `ex-info` with `:fix "Phase 1 supports macOS only. Linux support is tracked for Phase 2."` and the result map carries `:status :skip` with the same detail.
- A Phase 1 acceptance test runs the engine on Linux (or with the platform spoofed) and confirms the error message names the platform explicitly.

Linux support (`resolvectl`, `ip link show`, `/etc/resolv.conf`) is deferred to Phase 2.

## L3 Checks (macOS only)

WireGuard tunnel, DNS resolver, and DNS resolution checks are macOS-only. Underlying syscalls (`scutil --dns`, `ifconfig utun*`) are Darwin-specific; see Platform Support above.

### WireGuard handshake threshold

The WireGuard tunnel check inspects the last successful handshake timestamp and warns if it is stale. **Use 5 minutes** as the staleness threshold, not 3.

WireGuard rekeys on a ~3-minute rotation, so a handshake at exactly 3:00 is still a live tunnel. Using `< 3 min` as the pass condition warns on healthy tunnels after every rekey cycle. One full rekey-interval of slack (5 minutes) gives the right signal-to-noise.

This is **check policy** — it lives in the engine, not in `dial` (dial is pure UX) and not in `core` (core wires modules, it doesn't own per-check thresholds).

## Certificate Trust (L6)

Services accessed via HTTPS have certificates issued from one of several CAs. The trust check must know **which CA to expect for which service** and **which trust store on the client machine confirms that CA**. The mapping:

| Cert source | Issuer | Services | Client trust store |
|---|---|---|---|
| Let's Encrypt (nginx certbot) | Let's Encrypt R3/E1 (publicly trusted) | External subdomains (`*.dev.balkan.coffee` served by nginx on the VPS) | macOS system keychain (default trust) |
| OpenBao internal CA | self-hosted root CA | Incus internal endpoints (`*.incus`) and any service signed via OpenBao PKI | macOS system keychain *only if the OpenBao CA was installed locally* (manual step) |
| Incus server CA | self-hosted CA at `/var/lib/incus/server.ca` | Incus API endpoints used during CI bootstrap | macOS system keychain *only if installed locally* (see `project_incus_ca_trust.md`) |
| SSH host keys | host keypair per VPS | Not relevant to HTTPS trust — SSH engine concern, listed for completeness | `~/.ssh/known_hosts` |

The L6 check resolves the **expected issuer** per service (from registry metadata; see `:cert-source` field below) and verifies the leaf cert chains to that issuer using the local trust store. If the expected issuer is the OpenBao or Incus CA and the chain cannot be built, the check returns `:warn` with `:fix "install the OpenBao/Incus CA locally; see project_incus_ca_trust.md"` — not `:fail`. The service is healthy from the server's perspective; the client just lacks trust.

Registry contract: `:cert-source` is defined and owned by `registry.md`. The local engine reads it from the service record and refuses to start an L6 check if the value is unknown.

## Open Questions

**OQ-B: WireGuard tunnel check (macOS, Phase 1)** — How to detect if WireGuard tunnel is active on macOS? Options:
- `wg show` (requires `brew install wireguard-tools`)
- Probe gateway with `nc`
- Other platform-specific methods

**Decision needed before Phase 1 L3 check implementation.**

**OQ-C: DNS resolver check (macOS, Phase 1)** — How to verify the VPN DNS resolver (e.g., `10.211.139.1`) is the active resolver on macOS? Option:
- `scutil --dns` (output is complex to parse)

Linux support (resolvectl, /etc/resolv.conf) deferred beyond Phase 1.

**Decision needed before Phase 1 L3 check implementation.**

## Contract

Check functions implement the nool engine contract:
- Accept service name + any config
- Return result map: `:status`, `:detail`, `:fix`
- Throw `ex-info` with error details on exception
- Engine owns execution policy: timeouts, retries, confirmations. See [core.md §Execution policy](../core.md#execution-policy-owned-by-the-engine).
