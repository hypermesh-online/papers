# HyperMesh Architecture Map — Ground Truth

> **Purpose:** Anchor HyperMesh's architecture to *what actually exists* in the codebase,
> so design decisions stop drifting from reality. Every primitive below is marked
> **REAL / STUB / ABSENT** with a `file:line` citation. Read this before redesigning anything.

## Thesis: Kubernetes, inverted

In **Kubernetes**, a control plane hoists *many* machines into *one* cluster that an
operator drives top-down. **HyperMesh inverts this.** The user's own device — and its OS
function — is hoisted into its *own sovereign cluster*, and **the user is its admin**.

Clusters **nest**:
- A device is a cluster of *its assets*.
- A user's multiple devices form a cluster.
- Networks nest recursively (domains depend on networks, recursive/nested).

The stack is **complete** — the architecture is not missing a layer. What was missing is the
**orchestration control plane wired to the dataplane**. The components below already exist;
the gap is the wiring between them.

> **Naming note:** the crate on disk is `engauge/`; the project refers to it as **NGauge**.

---

## The Mapping (k8s concept → HyperMesh component → status → citation)

| Kubernetes | HyperMesh | Status | Citation |
|---|---|---|---|
| Control plane (API server + scheduler + reconcilers) | **NGauge** — orchestration + routing intelligence | **STUB** (traits exist, **UNWIRED**) | `engauge/src/routing_intel.rs` (PathAdvisor / RoutingAdvisor / EbpfPolicyFeedback traits); "not yet emitted from real metrics" — `blockmatrix/tests/ebpf_feedback_integration.rs` |
| etcd (cluster state / source of truth) | **TrustChain** — network state: CA \| CT \| DNS \| identity/PoS | **REAL** | see TrustChain breakdown below |
| └ CA | FALCON-1024 issuance, four-proof validation | **REAL** | `trustchain/src/ca/certificate_authority.rs` |
| └ CT log | Ed25519 signed log; **merkle tree is a placeholder** `Arc<RwLock<()>>` | **REAL (merkle STUB)** | `trustchain/src/ct/certificate_transparency/operations.rs` |
| └ DNS | resolver real; **server accept-loop NOT wired** (returns error) | **REAL resolver / STUB server** | `trustchain/src/dns/mod.rs:298` (`start()` warns STOQ DNS listener not implemented) |
| └ Identity | `node_id = BLAKE3(falcon_pubkey)` — the *who*, not an address | **REAL** | `trustchain/src/identity.rs` |
| └ Proof of State | StateProof four-proof, WireSignedProof FALCON-1024, binary pass/fail | **REAL** | `trustchain/src/proof_of_state/` |
| Networking dataplane / CNI | **STOQ** — realizes addressing + forwarding; eBPF beneath it | **REAL transport / forwarding GAP** | `stoq/src/transport/` (production QUIC), multipath scheduler `stoq/src/transport/multipath/scheduler.rs`, `stoq/src/transport/ebpf.rs` |
| └ matrix-position routing | matrix coord param dropped, not used on the wire | **ABSENT (unwired)** | `stoq/src/transport/multipath/policy.rs:206` (`validate_path` drops `_privacy_mode`/`_network_id`/etc.) |
| RBAC | **PoS four-proof + AccessLevel + privacy grid** | **REAL** | `trustchain/src/proof_of_state/validation.rs` (binary pass/fail); `blockmatrix/src/proof_of_state/mod.rs` (AccessLevel: None/Public/Private/Federated/Restricted/Verified); `lib/src/types.rs` (PrivacyMode scope×tracked grid); `blockmatrix/src/assets/core/privacy.rs` |
| Services ↔ Endpoints/EndpointSlices | **AssetAddress** (content-addressed) ↔ **swarm shard-holders** | **REAL** | `lib/src/address.rs` (prefix + matrix x,y,z + BLAKE3 fingerprint + shard index); `blockmatrix/src/transfer/mod.rs` (`readdress_asset`: same content hash, new matrix coord on transfer); `blockmatrix/src/distribution/swarm.rs:156` (cascade: requester auto-announced as provider — "consumers become providers", R12); `blockmatrix/src/retrieval/shard_map.rs`; `blockmatrix/src/assets/storage/{content_address,deduplication}.rs` |
| Cluster DNS (CoreDNS) | **TrustChain DNS-over-STOQ + domain-as-network** | **REAL resolver / STUB server** | `trustchain/src/dns/mod.rs`; `blockmatrix/src/dns/domain.rs` (domain → Network-scope chain via BLAKE3(domain_name); hierarchical/nested via `parent_network_id`) |
| Pod-to-pod flat networking | **the distribution pipeline** | **REAL** | `blockmatrix/src/assets/pipeline/` (Compress→Encrypt→Shard→Distribute, Reed-Solomon 10+4) |

---

## The Addressing Model — read this to avoid repeating the error

Addresses are **content-derived and causally placed**, *never identity-flat*.

- An asset's address = **BLAKE3(content) fingerprint + its matrix coordinate** (where it was
  causally placed) **+ shard index**. — `lib/src/address.rs`
  (ULA prefix `fd48:4d00` → net → matrix coords → asset fingerprint → shard).
- A **node is NOT directly addressed.** It is traceable *through the assets it holds/mirrors*.
- `node_id = BLAKE3(falcon_pubkey)` is only the **WHO** of the four-proof StateProof —
  **NOT an address.** — `trustchain/src/identity.rs`
- Identity = the **signed StateProof** (`WireSignedProof`).
- "`local_address`" (Substrate sense) = a **pointer to *this* node** — its PoS + its own
  Device-scope chain head — **never a derived IPv6**.

> **Do not repeat this error.** A prior attempt derived a *flat node IPv6 from the pubkey*.
> That was **WRONG and has been reverted**. **R15 was withdrawn** for exactly this reason
> (`core/CLAUDE.md:24` — "R15 (withdrawn — wrong addressing model)"). Nodes are addressed by
> their assets; identity is the signed proof, not a hash-of-pubkey IPv6.

---

## The Genuine Gaps — the only things actually missing

1. **eBPF zero-copy *pass-through* forwarding is ABSENT.** `XDP_REDIRECT` to AF_XDP for
   *this node's* traffic is real (`hypermesh-ebpf/programs/hypermesh_xdp.c:577`,
   `bpf_redirect_map(&xsk_map, …)`), but there is **no `XDP_TX` / forward-when-not-mine**.
   `PacketDecision::Forward { next_hop }` is declared in types but the kernel program never uses it.
2. **Matrix-position-aware routing is unwired.** `stoq/src/transport/multipath/policy.rs:206`
   drops its policy params; tensor routing exists in blockmatrix but doesn't drive STOQ/eBPF.
3. **NGauge routing-intelligence → eBPF feedback is traits-only.**
   `engauge/src/routing_intel.rs` — "not yet emitted from real metrics."
4. **Link/carrier self-management is genuinely unbuilt** (the `eno1` problem). The surviving
   base-crate scaffold (`core/base/`, `SubstrateAdapter` traits + adapters) is its home.
5. **STOQ content reflection (mirror-and-forward) is a stub** (`SeedInfo` unused in
   `stoq/src/extensions.rs`). Real reflection lives in the **blockmatrix swarm cascade**, not
   the STOQ reflector (which is blockchain-sync only).

---

## The Substrate's Corrected Role

The Substrate is **NOT an addressing layer.** It is the orchestration spine:

> **NGauge** (control plane / reconcilers)
> → commands **STOQ** (dataplane / CNI: realizes addresses on the wire + eBPF forwarding)
> → reconciling against **TrustChain** (state)
> → over the **blockmatrix** asset/swarm layer.

The **base crate** is the link/carrier **floor under STOQ's dataplane** (interface / carrier /
self-heal, acting as a reconciler). The Substrate **realizes** content/PoS addresses on the
wire — it does **not invent** them.
