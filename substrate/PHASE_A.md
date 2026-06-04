# Substrate Phase A — Sovereign Addressing (R15)

**Status:** planned · **Roadmap:** HyperMesh Substrate Layer · **PDL phase:** `phase_1780516747187_b552ct4j9`
**Implements:** R15 · **Crate:** `core/base` · **Depends on:** the scaffold (landed 2026-06-03)
**Canonical context:** `papers/SUBSTRATE.md` §6. This document is the executable design for Phase A.

---

## 1. Goal

Make a node's network address a property of *who it is*, not of *what the network handed it*. By the end of Phase A:

- `core/base/src/address.rs` derives a verifiable `fd48:4d00::/32` address from `NodeId` (bodies, not `todo!()`).
- `core/base/src/reachability.rs` reports how the node is reachable, learning its external address from a reflector it already talks to.
- The node's STOQ `TransportConfig` is populated with Substrate-derived `bind_address` / `public_ipv6` / `ebpf_interface` at construction, replacing the current hand-set `LOCALHOST`/`UNSPECIFIED` defaults — **without STOQ gaining a dependency on `base`**.

Phase A deliberately stops short of touching interfaces at the OS level (that is Phase B). The derived address is computed and *advertised*; assigning it to a live interface lease-free is Phase B. This split keeps Phase A pure-and-testable (math + wiring) and isolates the OS-integration risk into Phase B.

---

## 2. Design

### 2.1 Address derivation (`address.rs`)

The mapping is fixed by R15 and `SUBSTRATE.md` §6:

```
  bits   0..32   fd48:4d00          fixed HyperMesh ULA prefix (HYPERMESH_PREFIX)
  bits  32..64   subnet (u32)       network membership; SUBNET_DEVICE_SCOPE = 0 when unjoined
  bits  64..128  node_id[24..32]    interface identifier = last 8 bytes of the 32-byte digest
```

```rust
pub fn derive_address(node_id: &NodeId, subnet: u32) -> SubstrateResult<Ipv6Addr> {
    let id = node_id.as_bytes();            // &[u8; 32]
    let mut octets = [0u8; 16];
    octets[0..4].copy_from_slice(&HYPERMESH_PREFIX);   // fd48:4d00
    octets[4..8].copy_from_slice(&subnet.to_be_bytes()); // subnet slot
    octets[8..16].copy_from_slice(&id[24..32]);        // IID from digest tail
    Ok(Ipv6Addr::from(octets))
}

pub fn verify_address(addr: &Ipv6Addr, node_id: &NodeId, subnet: u32) -> SubstrateResult<bool> {
    Ok(*addr == derive_address(node_id, subnet)?)
}
```

**Why the tail bytes (`[24..32]`).** BLAKE3 output is uniformly distributed, so any 8-byte window is collision-equivalent; the tail is chosen by convention and fixed forever (changing it would re-address every node). The function never fails today, but the `SubstrateResult` return is retained so a future stricter variant (e.g. rejecting a reserved subnet) needs no signature change.

**Verifiability is the point (R15).** Because `node_id = NodeId::from_public_key(falcon_pubkey)` (`lib/src/types.rs:19`), any peer holding the FALCON public key recomputes the `NodeId` and re-derives the address. A node cannot claim an address that does not hash from its key — address spoofing reduces to key forgery.

### 2.2 Reachability (`reachability.rs`)

`Reachability { public_v6: Option<Ipv6Addr>, path: PathKind }` already exists. Phase A implements *discovery*:

- **Direct:** the derived/bound address is globally routable and observed unchanged by a peer. `public_v6 = None` (advertise the bind address as-is).
- **Traversed:** a peer (reflector) observes a *different* source address than the node bound — the node is behind NAT. `public_v6 = Some(observed)`, `path = Traversed`.
- **Unknown:** no reflector has reported yet.

**Mechanism.** Extend the existing reflector path (`stoq/src/transport/reflector/{block_transport,bridge}.rs`) with a lightweight "observed-address" echo: when a node completes a reflector handshake, the reflector reports the source address it saw. This reuses an existing relationship — no new STUN service, no new port. The reflector pool today only exposes `check_reflector_health` at the module root; Phase A adds an observed-address field to the reflector message/bridge and a `Substrate`-side consumer. *This is the only networked work in Phase A and the main source of its risk; see §5.*

### 2.3 Injection into STOQ (the layering-preserving wiring)

STOQ is constructed in several places, all setting `bind_address` by hand:
- `blockmatrix/src/network/mod.rs:769` (`LOCALHOST`)
- `blockmatrix/src/bin/node/commands/{ping,connect}.rs` (`UNSPECIFIED`)
- `blockmatrix/src/network/stoq_integration.rs:822`

There is already an address-resolution seam: `stoq_integration.rs:698-704` resolves the advertised address as `public_ipv6().or(bind_address)`. Phase A inserts the Substrate **above** these call sites:

```rust
// at the node binary / network construction layer (NOT inside stoq):
let substrate = /* Substrate impl */;
let addr = substrate.local_address(&node_id).await?;
let reach = substrate.reachability().await?;
let config = stoq::TransportConfig {
    bind_address: addr,
    public_ipv6: reach.public_v6,
    ebpf_interface: None, // Phase B fills this via active_interface(); None keeps current behavior
    ..stoq::TransportConfig::default()
};
```

**Invariant:** `base` is imported by `blockmatrix` (the binary/orchestration layer), never by `stoq`. STOQ keeps taking a plain `Ipv6Addr`. Verified by `cargo tree -p stoq | grep base` staying empty.

A `DefaultSubstrate` struct in `base` implements the `Substrate` trait, holding the chosen `SubstrateAdapter` from the registry. In Phase A it answers `local_address` (pure), `reachability` (reflector), and returns `Unsupported` for `active_interface`/`watch_links` (those land in Phase B) — so callers pass `ebpf_interface: None` and the existing `resolve_ebpf_interface` fallback continues to operate unchanged.

---

## 3. Data structures touched

| Type | Location | Change |
|---|---|---|
| `derive_address`, `verify_address` | `base/src/address.rs` | implement bodies |
| `Reachability`, `PathKind` | `base/src/reachability.rs` | already defined; add `detect` producer |
| `DefaultSubstrate` (new) | `base/src/substrate.rs` or new `base/src/default.rs` | implements `Substrate` for Phase A scope |
| reflector observed-address echo | `stoq/src/transport/reflector/{message,bridge}.rs` | add observed source addr to the exchange |
| STOQ construction sites | `blockmatrix/src/network/mod.rs`, `bin/node/commands/*` | inject Substrate-derived values |

---

## 4. Test strategy

**Unit / property (the testable core of R15) — `base/src/address.rs` tests:**
- *Determinism:* `derive_address(id, s)` equal across repeated calls and processes.
- *Round-trip:* `verify_address(derive_address(id, s), id, s) == true`.
- *Prefix:* first two octets always `0xfd, 0x48`; next two `0x4d, 0x00`.
- *Subnet placement:* octets `[4..8] == subnet.to_be_bytes()`; `SUBNET_DEVICE_SCOPE` → zeros.
- *IID source:* octets `[8..16] == node_id.as_bytes()[24..32]`.
- *Distinctness:* distinct `NodeId`s → distinct addresses (probabilistic; assert on a sampled set).
- *Cross-identity rejection:* `verify_address(addr_of_A, id_B, s) == false`.
- Use `proptest` if available in-workspace; otherwise table-driven with fixed `NodeId`s from `hypermesh_lib::test_utils::test_node_id`.

**Integration:**
- Node boots, constructs `DefaultSubstrate`, derives an address, builds `TransportConfig`, and `StoqTransport::new` binds it successfully (loopback-scoped test using a derived address mapped onto `lo`).
- Reflector observed-address echo: two-node local test where node B reports node A's source address; assert A's `reachability()` transitions `Unknown → Direct` (same address) and would report `Traversed` if the observed differs (simulated).

**Layering guard (CI):** `cargo tree -p stoq | grep -q base && exit 1` — STOQ must not depend on `base`.

---

## 5. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Reflector echo is the only networked piece and could balloon into a STUN/ICE effort | Scope it to "report the source address you saw" over the existing handshake; full NAT traversal (hole-punching) is explicitly out of Phase A — `Traversed` may just mean "advertise the observed address," with traversal deferred |
| Derived address may collide with an OS-assigned address on the same host during the loopback test | Test binds within `fd48:4d00::/32` on `lo`; production assignment is Phase B's concern |
| Multiple STOQ construction sites mean injection is touch-many | Centralize via a single helper in `blockmatrix` (`substrate_transport_config(node_id) -> TransportConfig`) that all sites call |
| `subnet` semantics interact with Device/Network scope model | Phase A uses only `SUBNET_DEVICE_SCOPE = 0`; network-membership subnet assignment is a documented follow-on, not Phase A |

---

## 6. Acceptance criteria (R15)

Phase A is done when:

1. `derive_address`/`verify_address` implemented; all §4 property tests pass.
2. A node boots with a derived `fd48:4d00::/32` address injected into STOQ's `TransportConfig`, and STOQ binds it (integration test green).
3. `reachability()` returns `Direct` with no external address when source is observed unchanged, and carries `public_v6` when a reflector observes a different source.
4. `cargo tree -p stoq` shows no `base` dependency; `cargo tree -p base` shows no `stoq`/`trustchain`/`blockmatrix`.
5. No regression in existing STOQ test suite.
6. `crate-status.toml` items for Substrate.a moved `planned → working`; `core/CLAUDE.md` R15 line updated from "produced by the Substrate layer" to reflect implemented status.

---

## 7. Out of scope (→ Phase B)

Assigning the derived address to a real interface lease-free; interface enumeration; carrier monitoring; link-flap self-healing; replacing `detect_outbound_interface()`. Phase A computes and advertises the address; Phase B makes the kernel actually carry it.
