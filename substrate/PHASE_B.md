# Substrate Phase B — Link-Sovereign Interface Management (R16)

**Status:** planned · **Roadmap:** HyperMesh Substrate Layer · **PDL phase:** `phase_1780516754686_skv43jxrm`
**Implements:** R16 · **Crate:** `core/base` · **Depends on:** Phase A (assigns the address Phase A derives)
**Canonical context:** `papers/SUBSTRATE.md` §7. This document is the executable design for Phase B.

---

## 1. Goal

Make the node own its link the way Phase A made it own its address. By the end of Phase B a node:

- Enumerates its own interfaces and reads carrier state — no hardcoded `[eth0, ens3, …]` guess.
- Assigns its Phase-A-derived `fd48:4d00::/32` address to a chosen interface **lease-free** (no DHCP).
- Watches link state and **self-heals**: on carrier loss / interface down it re-selects an interface, re-assigns the address, and signals STOQ to migrate connections.
- Degrades gracefully across capability tiers: **netlink → sysfs → unspecified-bind fallback**.

This is the interface-bounce problem (`networkctl down/up` on `eno1`) turned into a protocol behavior the node performs itself.

---

## 2. Design

### 2.1 Backends and tiers

Two real backends behind the `SubstrateAdapter` trait; the registry selects most-capable-first (`base/src/adapters/mod.rs`).

| Tier | Adapter | enumerate | carrier | assign | watch | Mechanism |
|---|---|:--:|:--:|:--:|:--:|---|
| 1 | `rtnetlink_linux` (feature `rtnetlink-backend`) | ✓ | ✓ | ✓ | ✓ | netlink sockets |
| 2 | `sysfs_fallback` | ✓ | ✓ | ✗ | ✗ | `/sys/class/net/` reads |
| 3 | (implicit) current STOQ behavior | — | — | — | — | `detect_outbound_interface()` + `UNSPECIFIED` bind |

If netlink is unavailable/unprivileged, the node still *observes* its link via sysfs (enumerate + carrier) and falls back to the existing bind behavior for the parts sysfs cannot do — it never hard-fails to "no network."

### 2.2 `rtnetlink_linux` operations

Using `rtnetlink` (async, tokio) + `netlink-packet-route`:

- **`enumerate_interfaces()`** — `handle.link().get().execute()` → stream of link messages. Map each to `InterfaceId { index, name }` (name from the `IFLA_IFNAME` attribute; index from the header). Skip loopback unless explicitly requested. Replaces `detect_outbound_interface()`’s probe list.
- **`carrier_state(iface)`** — read the link message flags for `iface.index`: `IFF_UP` (admin) and `IFF_RUNNING` / operstate (`IF_OPER_UP`) for carrier. Map to `LinkState::Up` / `Down` / `Carrier(bool)`. The `Carrier` distinction matters: an interface can be `Up` with no carrier (cable out / radio unassociated) — exactly the bounce case.
- **`assign_address(iface, InterfaceAddress { addr, prefix_len })`** — `handle.address().add(iface.index, IpAddr::V6(addr), prefix_len).execute()` (`RTM_NEWADDR`). Idempotent: if the address is already present, treat `EEXIST` as success. This is the lease-free assignment — the node puts its own derived address on the wire with no DHCP exchange.
- **link-event stream (for `watch_links`)** — open a second netlink socket subscribed to the `RTNLGRP_LINK` multicast group; decode `RTM_NEWLINK`/`RTM_DELLINK` into `LinkEvent { interface, state }`. Surface as a `BoxStream<'static, LinkEvent>` (matching the trait signature).

### 2.3 `sysfs_fallback` operations

Mirrors `hypermesh-ebpf`'s `NicCapabilities::detect()` sysfs approach:
- **`enumerate_interfaces()`** — read directory entries of `/sys/class/net/`; resolve index via `libc::if_nametoindex` (already used in `hypermesh-ebpf/src/af_xdp/helpers.rs`).
- **`carrier_state(iface)`** — read `/sys/class/net/{name}/carrier` (`1`/`0`) and `/sys/class/net/{name}/operstate`. Map to `LinkState`.
- **`assign_address` / `detect_reachability`** — already return `SubstrateError::Unsupported` in the scaffold; unchanged. Read-only tier.

### 2.4 Self-healing loop

A `Substrate`-level task (in `DefaultSubstrate`, not in an adapter) consumes `watch_links()`:

```text
on LinkEvent:
  if event indicates carrier loss or the active interface went Down:
      new_iface = select active interface (enumerate + prefer carrier-up, non-loopback)
      if new_iface differs from current OR address missing:
          assign_address(new_iface, derived_addr / 64)   # Phase A address, Phase B assignment
          emit MigrationSignal { new_bind: derived_addr, new_iface }   # to STOQ
```

**STOQ migration signal.** STOQ already has connection-migration machinery (`enable_migration`, `connection_migration_timeout_ms` in `config.rs`; "Connection migration scaffold" in `stoq/crate-status.toml`). Phase B does **not** add migration to STOQ — it provides the *trigger*. The signal is delivered the same injection-respecting way as Phase A: the `blockmatrix` orchestration layer subscribes to a Substrate-exposed channel and calls STOQ's existing migration entry point. No new STOQ→base dependency.

### 2.5 Interface selection policy

`active_interface()` returns the best current interface:
1. Prefer interfaces with `Carrier(true)`.
2. Among those, prefer non-loopback, then highest link speed if known (sysfs `speed`), else first stable by index.
3. If none have carrier, return the last-known or an `Unsupported`/`InterfaceNotFound` that lets STOQ fall back to `UNSPECIFIED` bind (tier 3) rather than failing the node.

---

## 3. musl build gate (hard prerequisite)

The deploy target builds **musl static-pie** (`core/CLAUDE.md` build notes). Before enabling `rtnetlink-backend` in production:

```
cargo build -p base --features rtnetlink-backend --target x86_64-unknown-linux-musl
```

`rtnetlink`/`netlink-packet-route`/`netlink-sys` are pure-Rust over raw sockets and are expected to build under musl, but this MUST be confirmed (B1 sprint deliverable). If a transitive dep breaks musl, the documented fallback is: ship the netlink backend on glibc nodes, ship `sysfs_fallback` (read-only) on musl-static nodes until resolved — the tier model already supports this split.

---

## 4. Data structures touched

| Type | Location | Change |
|---|---|---|
| `RtnetlinkLinuxAdapter` methods | `base/src/adapters/rtnetlink_linux.rs` | implement `enumerate`/`carrier`/`assign`/event-stream (replace `todo!()`) |
| `SysfsFallbackAdapter` enumerate/carrier | `base/src/adapters/sysfs_fallback.rs` | implement reads (replace `todo!()`) |
| `DefaultSubstrate` | `base/src/` | add `active_interface`, `watch_links`, self-healing task |
| `MigrationSignal` (new) | `base/src/link.rs` | struct carrying new bind addr + interface for STOQ migration |
| `detect_outbound_interface()` | `stoq/src/transport/manager/constructors.rs:36-47` | removed; replaced by injected `active_interface()` |
| migration subscriber | `blockmatrix/src/network/` | subscribe to Substrate channel, call STOQ migration |

---

## 5. Test strategy

**Adapter unit (require Linux; gate on `cfg(target_os="linux")` + feature):**
- `enumerate_interfaces()` returns `lo` plus at least one real interface on a normal host.
- `carrier_state()` for `lo` is up; for a real iface reflects `/sys/.../carrier`.
- `assign_address()` adds a `fd48:4d00::/64` address to a dummy interface (`ip link add dummy0 type dummy` in a test netns) and is idempotent on repeat.

**sysfs fallback:** point the reader at a temp fixture dir mimicking `/sys/class/net/` to test parsing without root.

**Self-healing (integration, network namespace):**
- Create `veth`/`dummy` pair in a test netns; assign derived address; bring carrier down (`ip link set … down`) and assert: a `LinkEvent` arrives, `active_interface()` re-selects, the address is re-assigned to the new interface, and a `MigrationSignal` is emitted.
- Capability degradation: with the netlink socket forced unavailable, assert the registry selects `sysfs_fallback` and the node still enumerates + reads carrier (no assign/watch), and STOQ falls back to tier-3 bind.

**Regression:** full STOQ suite green; the removal of `detect_outbound_interface()` must not change behavior on hosts where `active_interface()` returns the same interface the probe would have.

**musl:** the §3 build command succeeds in CI (or the documented split is recorded).

---

## 6. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Netlink requires `CAP_NET_ADMIN` to assign addresses | Detect at startup; if absent, advertise `assign_address: false` and degrade to sysfs/tier-3. Document the capability requirement for production nodes. |
| musl break in a netlink transitive dep | §3 gate + glibc/musl backend split fallback |
| Removing `detect_outbound_interface()` changes selection on some hosts | Selection policy (§2.5) is a superset; add a regression test pinning same-result on common configs; keep the probe list as the tier-3 *fallback* path, not the primary |
| Self-healing loop could thrash on a flapping link | Debounce link events (coalesce within a short window); only re-assign on a *stable* state change |
| Test suite needs root / netns for real netlink | Gate OS-level tests behind a CI job with `NET_ADMIN`; keep parsing/logic tests root-free via fixtures |

---

## 7. Acceptance criteria (R16)

Phase B is done when:

1. `rtnetlink_linux` enumerates real interfaces and reads carrier; `sysfs_fallback` does the same read-only; tier selection degrades netlink → sysfs verified.
2. The Phase-A-derived address is assigned lease-free to a selected interface (idempotent), confirmed in a netns test.
3. A simulated link flap produces: `LinkEvent` → interface re-selection → address re-assignment → `MigrationSignal` to STOQ (integration test green).
4. `detect_outbound_interface()` is removed; `active_interface()` (injected) drives interface selection with no STOQ→base dependency.
5. `cargo build -p base --features rtnetlink-backend --target x86_64-unknown-linux-musl` succeeds, or the glibc/musl split is documented.
6. No regression in STOQ tests; `crate-status.toml` Substrate.b items moved `planned → working`.

---

## 8. Out of scope (→ Phase C)

Any link that is not a kernel netdev: radio, device-to-device with no router, opportunistic peering. Phase B owns kernel-visible interfaces (ethernet, wifi as a netdev) via netlink/sysfs. Substrate.c is a separate research effort.
