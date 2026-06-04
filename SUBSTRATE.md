# Substrate: A Sovereign Network Floor for HyperMesh

**Version 0.1 — June 2026**
**HyperMesh Project — hypermesh.online**

---

## Abstract

HyperMesh is defined from the operating-system kernel upward. The kernel — where eBPF/XDP programs run and AF_XDP sockets move packets — is the protocol's "Layer 0" (HYPERMESH.md §4). But the kernel does not conjure a network out of nothing: it assumes a live carrier, a routable IPv6 address, and a configured interface already exist, handed to it by the incumbent regime — a DHCP server, an ISP-assigned prefix, a routing table the node does not own. Every sovereignty claim above transport is therefore real only down to the point where that borrowed link begins. Below that line, a "sovereign" node is a tenant.

The **Substrate** (layer S) is the floor beneath the floor. It owns the network reality the kernel assumes: a self-assigned IPv6 address derived from the node's cryptographic identity and verifiable by any peer; an enumerated, carrier-monitored network interface managed by the node itself rather than an external daemon; and a known reachability path. With the Substrate in place, the thing underneath the kernel stops being "the network the incumbent gave us" and becomes the mesh's own substrate. This is the difference between a better protocol layered over the existing internet and a network that does not require the existing internet to exist.

The Substrate does not introduce new ambitions — it produces requirements the protocol already mandates. R1 already requires every node to instantiate its network interfaces as assets each carrying an IPv6 address in the `fd48:4d00::/32` space; nothing in the codebase derives those addresses. This document specifies the derivation (R15), the link and carrier management that makes it real (R16), and the pluggable backend model that lets the mesh reclaim the substrate one tier at a time — from sovereign addressing over existing links, down to link/carrier self-management, and ultimately to physical device-to-device links with no provider at all.

---

## 1. Motivation: three disconnected seams

The need is not theoretical. Three pieces already exist in the codebase, each correct in isolation, with no layer connecting them — and the gap is exactly the place a manual interface bounce (`networkctl down/up` + DHCP renewal) reveals: when the link isn't there, none of the six layers have anything to say.

1. **Identity exists but never becomes an address.** `trustchain/src/identity.rs` derives `node_id = BLAKE3(falcon_public_key)` (a 32-byte digest, canonicalized as `hypermesh_lib::NodeId`). It is used for signing and verification — never to derive a network address, though R1 says every node carries one in `fd48:4d00::/32`.

2. **Transport takes its address on faith.** `stoq/src/transport/config.rs` exposes `bind_address: Ipv6Addr` (defaulting to `UNSPECIFIED`), `public_ipv6: Option<Ipv6Addr>` (which **must be set manually — there is no discovery mechanism**), and `ebpf_interface: Option<String>`. STOQ binds whatever address it is handed and assumes it is live and routable.

3. **The interface is a hardcoded guess.** `stoq/src/transport/manager/constructors.rs` (`detect_outbound_interface()`) probes a fixed list — `[eth0, ens3, ens4, enp0s3, wlan0, wlp2s0]` — and falls back to `lo`. There is no enumeration, no carrier check, no awareness that the chosen interface might have no link.

Across all of `core`, there is no netlink/rtnetlink integration, no DHCP client, no STUN/ICE/hole-punching, no carrier or link-state monitoring, and no identity-to-address derivation. The Substrate is the layer that bridges seams (1)→(2)→(3) and supplies what is genuinely absent.

---

## 2. Scope

The Substrate is three sub-strata, reclaimed bottom-up. Each tier moves the sovereignty line one notch lower.

- **Substrate.a — Sovereign addressing & reachability.** Derive a verifiable `fd48:4d00::/32` address from `node_id`; discover reachability (external address / path), extending the existing STOQ reflector pool rather than introducing a new discovery service. Feeds STOQ's `bind_address` and `public_ipv6` instead of those being configured by hand. **Buildable now — roadmap Phase 1.**
- **Substrate.b — Link/carrier/interface management.** Enumerate interfaces and monitor carrier/link state over netlink (with a sysfs read-only fallback); assign the derived address lease-free; self-heal on link flap and signal the transport layer for QUIC connection migration. Replaces the hardcoded interface guess. This is the interface-bounce problem made a protocol concern. **Buildable, OS-integration heavy — roadmap Phase 2.**
- **Substrate.c — Physical/radio.** Device-to-device links with zero incumbent infrastructure: wireless mesh radio, opportunistic peering. The network exists before any provider does. **Roadmap-only R&D; scaffolded as a stub adapter, not built — roadmap Phase 3.**

---

## 3. Position in the stack

The kernel already owns the name "Layer 0." The Substrate sits *beneath* it and is **never** called Layer 0; it is layer S. It is the lowest software layer the mesh controls, below the kernel's packet path and below `hypermesh-lib`'s canonical types in the dependency sense only — it depends on `hypermesh-lib` for `NodeId` and nothing else in the workspace.

```
+------------------------------------------------------+
|  1. STOQ          QUIC/IPv6 transport, eBPF, crypto   |
+------------------------------------------------------+
|  0. OS/Kernel     Linux, eBPF/XDP, AF_XDP             |
+------------------------------------------------------+
|  S. Substrate     Self-assigned addr, link/carrier    |   <-- this document
+------------------------------------------------------+
|  *  hypermesh-lib  Shared canonical types              |
+------------------------------------------------------+
```

**Relationship to `hypermesh-ebpf`.** The kernel layer attaches XDP and binds AF_XDP queues *to an interface that already exists and is up* (`hypermesh-ebpf` takes an interface name as a parameter; `af_xdp/helpers.rs` resolves it via `libc::if_nametoindex`). The Substrate is what guarantees that interface exists, has carrier, and carries the node's address — it produces the precondition the kernel layer consumes. The two are complementary: Substrate manages the link; the kernel layer accelerates packets over it.

**Cross-layer boundary.** `Substrate -> OS` via `SubstrateView`: self-assigned IPv6, active interface + carrier state, reachability hints (added to HYPERMESH.md §4 boundary table).

---

## 4. The Substrate contract

The Substrate exposes two traits, modeled on the async-trait `AssetAdapter` pattern in `blockmatrix/src/assets/core/adapter.rs`. (Rust signatures live in `core/base/src/substrate.rs`; this section is the normative description.)

**`Substrate`** — the service the node binary consumes:
- `local_address(node_id) -> Ipv6Addr` — the derived `fd48:4d00::/32` address (Substrate.a, R15). Feeds STOQ `bind_address`.
- `reachability() -> Reachability` — externally visible address + path kind (Substrate.a). Feeds STOQ `public_ipv6`.
- `active_interface() -> InterfaceId` — the selected outbound interface (Substrate.b, R16). Replaces `detect_outbound_interface()`; feeds STOQ `ebpf_interface`.
- `watch_links() -> Stream<LinkEvent>` — link-state changes for self-healing (Substrate.b, R16).

**Layering rule (normative).** `base` depends only on `hypermesh-lib`. STOQ does **not** depend on `base`. The node binary constructs the Substrate and *injects* the resolved `bind_address` / `public_ipv6` / `ebpf_interface` into STOQ's `TransportConfig`. This keeps STOQ transport-only and keeps the Substrate strictly beneath transport — no upward dependency, no cycle (in particular, `base` consumes the canonical `NodeId` from `hypermesh-lib`, not `FalconIdentity` from `trustchain`).

---

## 5. The SubstrateAdapter capability pattern

A single host can have several possible backends (Linux netlink, a read-only sysfs path, a future radio stack, a future Windows API). The **`SubstrateAdapter`** trait abstracts the backend; the **`SubstrateAdapterRegistry`** selects the highest-capability one available at runtime and degrades across tiers — exactly the graceful-degradation model `hypermesh-ebpf` uses to pick kernel capability tiers (HYPERMESH.md §5.2).

Each adapter advertises a `SubstrateCapabilities` set (`enumerate`, `carrier`, `assign_address`, `watch`, `reachability`) and implements `enumerate_interfaces`, `carrier_state`, `assign_address`, and `detect_reachability`.

**Adapters scaffolded up front** (so pilot-funded R&D fills them without restructuring):

| Adapter | Tier | Status |
|---|---|---|
| `rtnetlink_linux` | netlink — full link mgmt + address assign | real near-term backend (feature-gated), bodies land Phase 1/2 |
| `sysfs_fallback` | read-only `/sys/class/net/` | degraded tier; reuses the `NicCapabilities::detect()` sysfs pattern |
| `radio_mesh` | Substrate.c device-to-device | **roadmap stub** — advertises no capabilities, never selected |
| `windows` | future cross-platform (IP Helper API) | **roadmap stub** — advertises no capabilities |

The two stubs advertise an empty capability set, so the registry never selects them today; they exist to give R&D a concrete home and to justify the abstraction (HyperMesh already has a cross-platform OS abstraction in `blockmatrix/src/os_integration/`).

---

## 6. Substrate.a — sovereign addressing

**Derivation (normative).** Given `node_id = BLAKE3(falcon_public_key)` (32 bytes) and a 32-bit `subnet`:

```
  bits   0..32   fd48:4d00          fixed HyperMesh ULA prefix (R1)
  bits  32..64   <subnet>           network membership (0 = Device scope)
  bits  64..128  node_id[24..32]    interface identifier (low 8 bytes of the digest)
```

The address is `fd48:4d00:<subnet-hi>:<subnet-lo>:<8 bytes from node_id>`. Because `node_id` is itself `BLAKE3(falcon_public_key)`, the full address is a pure, deterministic function of the public key. **Verifiability (R15):** any peer holding the FALCON-1024 public key recomputes `NodeId::from_public_key(pubkey)`, re-runs the derivation, and confirms a claimed address belongs to the identity that signed for it. There is no DHCP, no lease, no external authority.

The `subnet` slot (bits 32..64) is reserved for network membership: a node not joined to a Network-scope blockchain uses `0` (Device scope); joining a network sets it. This composes with the existing Device/Network scope model without changing it.

(Note: `hypermesh-lib` already has an address type at `lib/src/address.rs`; the Substrate derivation produces a `std::net::Ipv6Addr` and does not collide with it. Bridging to the lib address type, if desired, is a Phase 1 detail.)

**Reachability.** `reachability()` returns the externally visible address (if discovered) and a `PathKind` (`Direct`, `Traversed`, `Unknown`). Discovery is a Phase 1+ concern and is expected to extend the existing STOQ reflector pool (`stoq/src/transport/reflector/`) — a node can learn its observed address from a reflector it already talks to — rather than introducing a standalone STUN service.

**STOQ integration points (normative — targets, changed in Phase 1, not in the scaffold pass):**

| STOQ location | Today | After |
|---|---|---|
| `config.rs:109` `bind_address` | caller-provided, default `UNSPECIFIED` | `Substrate::local_address(node_id)` |
| `config.rs:169` `public_ipv6` | manual, no discovery | `Substrate::reachability()` |
| `config.rs:159` `ebpf_interface` | auto-resolved from bind_address | `Substrate::active_interface()` |
| `constructors.rs:36-47` `detect_outbound_interface()` | hardcoded probe list | replaced by `Substrate::active_interface()` |

Values are injected at the node binary; `constructors.rs:56` (`resolve_ebpf_interface`) and `:227` (`UdpSocket::bind`) become the consuming call sites.

---

## 7. Substrate.b — link, carrier, interface management

What it owns, and what it replaces:

- **Interface enumeration** over netlink (`RTM_GETLINK`), replacing the hardcoded candidate list. Sysfs fallback enumerates `/sys/class/net/` directory entries.
- **Carrier/link-state monitoring** — the distinction the bounce reveals: an interface can be administratively up while having no carrier. `LinkState::Carrier(bool)` captures this; netlink reads operstate, sysfs reads `/sys/class/net/{iface}/carrier`.
- **Lease-free address assignment** — the Substrate.a-derived address is assigned directly to the chosen interface (`RTM_NEWADDR`), no DHCP. This is the concrete realization of "sovereign self-assigned addressing."
- **Self-healing on link flap** — `watch_links()` subscribes to `RTMGRP_LINK`. On carrier loss / interface down, the node re-selects an active interface, re-assigns its derived address, and emits a signal the transport layer uses for QUIC connection migration (HYPERMESH.md §5 already cites QUIC migration when a node's address changes; the Substrate is the producer of that change event).

Capability degradation (R16): netlink (full) → sysfs (read-only: enumerate + carrier, no assign/watch) → unspecified-bind fallback (today's behavior). The node manages its own link without an external network daemon.

---

## 8. Substrate.c — physical/radio (deferred R&D)

The endgame of substrate sovereignty: links that exist with no incumbent at all. This is **roadmap-only** and intentionally **not** given a protocol R-number — R-numbers must be concrete and testable, and radio is not yet that. It is scaffolded as the `radio_mesh` adapter (no capabilities) so it has a home in the registry.

Open R&D questions (to be answered when a pilot funds the work, not designed here):
- Radio hardware abstraction and driver access across devices.
- Neighbor discovery and association without a router or DHCP.
- Opportunistic peering and store-carry-forward when links are intermittent.
- Spectrum, regulatory, and power constraints.
- How Substrate.a addressing and Substrate.b carrier semantics map onto a radio link with no kernel netdev.

---

## 9. New protocol requirements

Added to HYPERMESH.md §3 (verbatim there; summarized here):

- **R15. Sovereign self-assigned addressing.** Derive `fd48:4d00::/32` deterministically and verifiably from `node_id = BLAKE3(falcon_public_key)`; no DHCP/lease/authority; peer-verifiable. Producer for R1 and for transport bind/public address config.
- **R16. Link-sovereign interface management.** Enumerate own interfaces and monitor carrier without a hardcoded list; on link change re-select, re-assign the derived address, and signal connection migration; degrade gracefully netlink → sysfs → fallback; manage the link without an external daemon.

Substrate.c (radio) is roadmap prose, not an R-number, until it is concrete and testable.

---

## 10. Integration points (normative summary)

| File | Change | Phase |
|---|---|---|
| `core/base/` | new crate — traits, types, adapter registry, stubs | scaffold (done) |
| `core/Cargo.toml` | add `base` member; add `rtnetlink`/`netlink-packet-route` (feature-gated) | scaffold (done) |
| `stoq/src/transport/config.rs` | `bind_address`/`public_ipv6`/`ebpf_interface` ← Substrate (injected) | Phase 1 |
| `stoq/src/transport/manager/constructors.rs` | replace `detect_outbound_interface()` with `Substrate::active_interface()` | Phase 1 |
| node binary (`blockmatrix/src/bin/node/`) | construct Substrate, inject resolved values | Phase 1 |

---

## 11. Crate skeleton

```
core/base/
  Cargo.toml          # name = "base", edition 2021; rtnetlink-backend feature off by default
  crate-status.toml   # phase = "planning"
  CLAUDE.md           # crate-local engineering context
  SPEC.md             # implementation contract (cites this whitepaper)
  src/
    lib.rs            # module tree + re-exports
    substrate.rs      # Substrate + SubstrateAdapter traits, SubstrateCapabilities
    error.rs          # SubstrateError (thiserror), SubstrateResult
    address.rs        # Substrate.a derivation + verification (R15)
    reachability.rs   # Reachability, PathKind
    link.rs           # InterfaceId, LinkState, LinkEvent, InterfaceAddress
    adapters/
      mod.rs          # SubstrateAdapterRegistry (capability-tier selection)
      rtnetlink_linux/ # real backend (feature rtnetlink-backend)
      sysfs_fallback/  # read-only degraded tier
      radio_mesh/      # Substrate.c stub
      windows/         # future cross-platform stub
```

Dependencies: `hypermesh-lib`, `tokio`, `async-trait`, `futures`, `thiserror`, `tracing`, `blake3`; `rtnetlink` + `netlink-packet-route` behind the `rtnetlink-backend` feature. **Musl note:** the deploy target builds musl static-pie (core/CLAUDE.md); netlink crates' musl-buildability MUST be verified before enabling the feature in Phase 1.

---

## 12. Roadmap mapping

| Phase | Deliverable | Requirement | Status |
|---|---|---|---|
| Scaffold | Spec + `base` crate skeleton + R15/R16 + workspace wiring | R15/R16 stated | done |
| Phase 1 | Substrate.a derivation + `rtnetlink_linux` enumeration; inject into STOQ | R15 | planned |
| Phase 2 | Substrate.b carrier monitoring, lease-free assign, link-flap self-heal + migration; `sysfs_fallback` | R16 | planned |
| Phase 3 | `radio_mesh` (Substrate.c), `windows` adapter — pilot-funded R&D | — | roadmap stub |

Status tracking follows the `crate-status.toml` convention (core/CLAUDE.md): items move `planned → in_development → working` as phases land.

---

## 13. Non-goals and confusions to avoid

- **This is not "Layer 0."** The kernel owns that name; the Substrate is layer S, beneath it.
- **This is not the BlockMatrix proxy/NAT system.** `blockmatrix/src/assets/proxy/nat_translation.rs` is *internal memory addressing* (mapping global asset addresses to local process memory). It is unrelated to network reachability or NAT traversal. The Substrate's reachability is real network-layer addressing; the naming overlap is coincidental.
- **Substrate.c radio is not built** in this work, and addressing/link semantics for radio are deliberately left open.
- **The Substrate does not replace STOQ, the kernel layer, or `hypermesh-ebpf`.** It produces the link and address they consume.
