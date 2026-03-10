# HyperMesh: A Sovereign Distributed Computing Protocol

**Version 0.4 -- March 2026**

**Abstract.** HyperMesh is a distributed computing protocol in which every node is sovereign over its own resources, identity, and state. The protocol replaces global consensus with bilateral Proof of State -- a four-proof authentication model (WHERE, WHO, WHAT, WHEN) that verifies through sovereign hash chains rather than network-wide agreement. All identity operations are secured with NIST-standardized post-quantum cryptography (FALCON-1024 for signatures, Kyber-1024 for encryption), and privacy is configurable along two independent axes at the protocol level. Six composable layers -- from kernel-integrated transport to optional economic interoperability -- communicate through trait-defined interfaces with strict upward dependency. A shared canonical type system ensures every layer speaks the same language. Layer 5 (Caesar) is an optional economic interop bridge (see the Caesar whitepaper), and Layer 6 (Engauge) is an optional execution and analytics layer. The result is a mesh that scales linearly with transaction volume rather than network size, resists quantum-capable adversaries, and permits each participant to choose its own balance between anonymity and accountability.


## 1. Introduction: The Problem

The infrastructure that operates the internet has consolidated into a small number of entities. Three cloud providers control the majority of public compute capacity. A handful of certificate authorities gate the issuance of TLS certificates. DNS resolution depends on thirteen root server clusters. This concentration creates structural fragility: a single provider outage can disable thousands of unrelated services, a compromised certificate authority can forge identities at scale, and a DNS failure can render entire regions unreachable.

Data sovereignty has been treated as an afterthought. Regulatory frameworks like GDPR and HIPAA impose residency and access controls, but these are bolted onto architectures that were designed for centralized storage and processing. Compliance becomes a constraint layer on top of systems that structurally resist it, producing complexity without genuine sovereignty.

Distributed systems that attempt to address these problems inherit a different failure mode: consensus cost. Byzantine fault-tolerant protocols require O(n^2) messages per state change in the general case. Raft improves this to O(n) but assumes a fixed, small membership. Neither scales to meshes of tens of thousands of nodes where membership is dynamic and geography is heterogeneous.

Meanwhile, the cryptographic assumptions underlying current transport security are under threat. Harvest-now-decrypt-later attacks -- where an adversary captures encrypted traffic today and decrypts it when a sufficiently capable quantum computer becomes available -- are a documented concern for any data with a secrecy requirement exceeding the timeline to fault-tolerant quantum computation. NIST completed standardization of post-quantum algorithms in 2024; adoption remains sparse.

Finally, privacy in existing protocols is binary. A connection is either public or tunneled through a VPN. There is no protocol-level mechanism for a node to be anonymous in one context and fully identified in another, or for two parties to interact with scoped identity disclosure without a third-party intermediary.

HyperMesh addresses each of these problems at the protocol level: sovereignty through local-first design, scalability through bilateral Proof of State, quantum resistance through NIST-standardized algorithms, and configurable privacy through a two-axis model embedded in the identity layer.


## 2. Design Principles

Five principles govern every architectural decision in HyperMesh.

**Sovereignty.** Every node controls its own resources, identity, and state. No protocol operation requires yielding control to a central coordinator. A node's local blockchain starts immediately on boot -- no network connectivity required. A node's participation in the broader mesh is voluntary, its data is self-custodied, and its identity is self-asserted through cryptographic proof.

**Locality.** Decisions are made from local information. Routing uses neighbor coordinates, not global tables. Verification is bilateral between transacting parties, not broadcast to the mesh. State is maintained per-node in a sovereign hash chain, not in a shared ledger.

**Composability.** The protocol is structured as six independent layers with trait-defined interfaces at each boundary. A shared canonical type system (hypermesh-lib) provides the common vocabulary -- NodeId, AssetId, PrivacyMode, MatrixPosition, ProofType -- that all layers reference without duplication. Lower layers know nothing about higher ones. Any layer can be replaced or extended without modifying adjacent layers.

**Post-quantum by default.** FALCON-1024 and Kyber-1024 -- both NIST-standardized as of 2024 -- are used throughout, not on a migration roadmap. The cipher suite is fixed per protocol version, eliminating downgrade attacks.

**Privacy as configuration.** Privacy is not a binary property. It is configured along two independent axes -- Access Scope (Bounded or Unbounded) and Tracked (true or false) -- producing three named presets (Anonymous, Private, Public) that a single node can use simultaneously on different connections. Privacy mode is enforced at the kernel level through eBPF policy maps, not just in application logic.


## 3. Protocol Requirements

The design principles (Section 2) produce concrete, testable requirements. Every compliant HyperMesh implementation MUST satisfy these. They are non-negotiable for protocol correctness.

**R1. Sovereign genesis with asset instantiation.** A node MUST produce a functioning local blockchain from a genesis block on boot, with zero network connectivity. The genesis block MUST instantiate the node's hardware as addressable assets -- CPU, GPU, Memory, Storage, Network interfaces -- each with an IPv6 address in the HyperMesh address space (`fd48:4d00::/32`) and an initial Proof of State. Hardware is assessed, not self-reported: the node measures its own capabilities (core count, clock speed, VRAM, RAM capacity, disk capacity and type, interface bandwidth) and records them as assets with proofs. These genesis assets form the root of the node's asset tree and determine what the node can contribute to the mesh.

**R2. Four-proof authentication.** Every state claim MUST carry a complete Proof of State: Proof of Space (WHERE), Proof of Stake (WHO), Proof of Work (WHAT), Proof of Time (WHEN). Partial proofs are invalid. Authentication is binary -- authentic or not. There are no trust scores, no reputation floats, and no partial validity.

**R3. Pipeline ordering.** Asset data MUST be processed in strict order: Compress (Brotli) -> Encrypt (Kyber-1024 KEM -> AES-256-GCM, whole blob) -> Shard (Reed-Solomon erasure coding) -> Distribute (tensor-weighted placement). Encryption operates on the entire compressed blob before sharding. Per-shard encryption is a protocol violation.

**R4. Content-addressed deduplication with privacy-scoped tracking.** Shards with identical content (identical BLAKE3 hash) MUST be stored once per storage domain with reference counting. Deduplication tracking is scoped to the privacy boundary: Device scope has full reference tracking, Private networks track within their trust boundary, Anonymous mode stores content-addressed shards but performs no cross-node reference tracking. Tamper detection is inherent in content addressing -- `BLAKE3(data) == expected_hash` -- and does not depend on reference metadata.

**R5. Erasure-coded redundancy.** Shards MUST be erasure-coded such that any k of n shards are sufficient for reconstruction (default: 10-of-14 Reed-Solomon). Loss of up to n-k shards MUST NOT cause data loss.

**R6. Instruction-based retrieval.** Asset transfer MUST transmit a shard map (content hash, shard hashes, matrix positions, fallback locations) -- not raw data. The receiver reconstructs from the map. Instruction payload MUST remain under 1 KB regardless of asset size.

**R7. Post-quantum storage encryption.** Data at rest MUST be encrypted with a post-quantum KEM (Kyber-1024). Classical-only encryption for stored assets is a protocol violation. Transport-layer key exchange (X25519) is acceptable because session keys are ephemeral.

**R8. Cipher suite policy.** The standard cipher suite (FALCON-1024, Kyber-1024, X25519, AES-256-GCM, BLAKE3) is the default for all networks and non-negotiable on the Public network. Private and Anonymous networks MAY agree on alternative cipher suites between consenting participants. Cross-network communication (via Gateway) always uses the standard suite.

**R9. Privacy-scope independence.** Network privacy (Anonymous/Private/Public) and blockchain scope (Device/Network) MUST be independently configurable. No combination is prohibited. Kernel-level isolation between privacy modes is required via eBPF policy maps.

**R10. Universal asset model with transmission.** Every resource MUST be represented as an asset with a universal AssetId, an IPv6 address, a Proof of State, and blockchain registration. System asset types: Cpu, Gpu, Memory, Storage, Network, Container, Transmission, Dns, Blockchain, Economic. Transmission is a first-class asset representing bandwidth available for mesh relay -- data communication across networks must exist without overhead beyond the Transmission asset's own proof. The genesis block instantiates initial assets from hardware assessment (R1).

**R11. Bilateral verification.** State verification MUST be bilateral between transacting parties. No operation requires global consensus, quorum, or leader election. Verification cost scales with transaction volume, not mesh size.

**R12. Shard commitment with swarm scaling.** Each block MUST commit to its shard distribution via `BLAKE3(sorted shard placements)`, anchoring the block's hash chain to its spatial evidence across the matrix. For popular content, consumers MUST become providers: shard distribution cascades through the swarm so that no single node bears more than O(log N) of the distribution load for N concurrent consumers. Replica discovery uses matrix-neighbor gossip, not centralized tracking. One million concurrent consumers of the same content MUST be served through peer-to-peer cascade with near-zero per-node overhead.

**R13. Minimum device specification.** The protocol MUST operate on devices with: 1 Mb/s network, 50 GB storage, 4 GB RAM, dual-core 1 GHz processor. Shard reconstruction MUST be streaming -- not buffer-all-then-reconstruct. This is the floor: the protocol must run on legacy hardware, low-tier consumer devices, and minimal virtual machines.

**R14. Adaptive shard sizing.** Erasure coding parameters MUST scale with asset size to keep individual shards within a transfer-efficient range. Post-creation shard splitting is prohibited -- it breaks content-addressing hashes (the shard's BLAKE3 hash is its identity for dedup, bucket assignment, retrieval maps, and block commitment). Wire-level chunking is handled by QUIC stream transfer in the transport layer, not by subdividing the content-addressed unit. Rebalancing MUST NOT require rehashing the network. Shard migration uses copy-then-redirect, not move-then-update. Adaptive sizing MUST NOT cause fault intolerance or throughput bottlenecks.


## 4. Architecture

HyperMesh is composed of six layers, ordered by dependency. Each layer depends only on the layer immediately below it. All layers share a canonical type system (hypermesh-lib) that defines the common identifiers, modes, and positions used across the protocol.

    +------------------------------------------------------+
    |  6. Engauge        Execution, analytics & metrics      |
    +------------------------------------------------------+
    |  5. Caesar        Economic interop bridge (optional)  |
    +------------------------------------------------------+
    |  4. Catalog       Asset types, discovery, delegation  |
    +------------------------------------------------------+
    |  3. BlockMatrix   Topology, routing, validation       |
    +------------------------------------------------------+
    |  2. TrustChain    Identity, certificates, privacy     |
    +------------------------------------------------------+
    |  1. STOQ          QUIC/IPv6 transport, eBPF, crypto   |
    +------------------------------------------------------+
    |  0. OS/Kernel     Linux, eBPF/XDP, AF_XDP             |
    +------------------------------------------------------+
    |  *  hypermesh-lib  Shared canonical types              |
    +------------------------------------------------------+

Layer 0 is the operating system kernel, where eBPF programs run at the XDP (eXpress Data Path) hook point and AF_XDP sockets provide zero-copy packet I/O. Layer 1 (STOQ) provides authenticated, multiplexed byte streams over QUIC and IPv6 with kernel-accelerated packet processing. Layer 2 (TrustChain) overlays identity, certificates, and privacy classification onto those streams. Layer 3 (BlockMatrix) assigns spatial coordinates, computes tensor-weighted routes, manages geospatial clusters, and validates state through sovereign hash chains using four-proof Proof of State. Layer 4 (Catalog) defines asset types, maintains a discovery registry, and delegates execution to mesh nodes. Layer 5 (Caesar) is an optional economic interop bridge that enables value transfer across the mesh and bridges to external payment systems (see the Caesar whitepaper for protocol details). Layer 6 (Engauge) is a planned execution and analytics layer for paid content hosting and network metrics.

Cross-layer communication occurs through Rust trait boundaries:

    Boundary                  Interface            Data Exchanged
    -----------------------   ------------------   ----------------------------
    OS -> STOQ                eBPF/XDP hooks       Raw packets via AF_XDP,
                                                    policy maps, flow metadata
    STOQ -> TrustChain        TransportStream      Authenticated byte streams
    TrustChain -> BlockMatrix TrustedChannel       Identity-tagged, privacy-
                                                    classified channels
    BlockMatrix -> Catalog    TopologyView         Coordinate positions,
                                                    tensor-weighted paths
    Catalog -> Caesar         AssetEvent           Asset allocation/release events
    Caesar <-> Engauge         RewardTrigger        Verified metrics and reward
                                                    distribution triggers

A minimal node runs layers 1 through 4 as a single statically-linked binary configured by a single TOML file. Caesar is an optional leaf; removing it changes nothing about the behavior of layers 1 through 4. The core mesh is a commons; Caesar and Engauge add an optional commercial layer.


## 5. Transport: STOQ Protocol

STOQ is built on QUIC (via the Quinn library in Rust) running exclusively over IPv6. The choice of QUIC provides multiplexed streams with independent flow control, 0-RTT reconnection for returning peers, and connection migration when a node's address changes. IPv6 gives every node a globally routable address, eliminating NAT traversal and enabling direct peer connectivity. The protocol identifier is `stoq/1.0` (ALPN negotiation per RFC 8439).

### 5.1 Kernel Integration

Packet processing begins in kernel space, before the operating system's network stack allocates socket buffers. A unified eBPF intelligence layer (hypermesh-ebpf) serves as the single source of truth for all packet decisions. Three consumers interact with it: STOQ as a thin transport wrapper, BlockMatrix as a policy configurator, and the kernel XDP program as the enforcement point.

Four classes of eBPF programs provide different capabilities. **XDP programs** classify and route packets at the earliest point in the network stack. The primary HyperMesh XDP program parses a custom extension header (identified by magic bytes `0x484D`) that carries Proof of State, asset hash, matrix routing, and privacy tier fields inline with every packet. Based on policy flags, the XDP program makes one of four decisions for each packet: Pass to the local network stack, Redirect to an AF_XDP socket for zero-copy delivery to STOQ, Forward via XDP_TX to another matrix node, or Drop with a recorded reason. **Kprobe programs** instrument kernel functions for zero-overhead observability -- connection lifecycle events, syscall patterns, and error conditions. **Tracepoint programs** capture structured kernel events for performance analysis. A **counter program** tracks packet rates and byte volumes per interface.

These four program types are compiled from C source by the build system and loaded into the kernel at runtime through the eBPF loader.

### 5.2 Zero-Copy Packet I/O

High-throughput data paths bypass the kernel's socket buffer allocation entirely through AF_XDP (Address Family XDP). When the XDP program redirects a packet, it lands directly in a shared memory region (UMEM) accessible to both kernel and userspace, eliminating copy operations.

The AF_XDP architecture uses four ring buffers mapped into userspace memory:

- **Fill Ring**: Userspace supplies empty UMEM frames for the kernel to fill with incoming packets.
- **RX Ring**: The kernel places received packet descriptors here for userspace consumption.
- **TX Ring**: Userspace places outgoing packet descriptors here for kernel transmission.
- **Completion Ring**: The kernel returns transmitted frame descriptors so userspace can reclaim them.

Each ring uses atomic producer and consumer pointers for lock-free operation. A frame allocator manages the UMEM region with batch allocation and a thread-safe free-list, allowing frames to be acquired and released without contention. Packet transmission uses the `sendto` syscall; reception uses `poll` and `recvfrom`. Batch operations amortize syscall overhead across multiple packets.

The system degrades gracefully across three tiers: full eBPF with AF_XDP zero-copy on capable kernels, eBPF without AF_XDP on older kernels, and pure userspace processing where eBPF is unavailable. The orchestrator detects kernel capabilities at startup and selects the highest-performance path available.

### 5.3 Policy Enforcement

Four BPF maps maintain kernel-side state that the XDP program consults on every packet:

- **policy_map**: Per-interface validation policy (whether to require Proof of State headers, validate asset hashes, or check matrix routing).
- **pos_header_map**: Cached Proof of State header data for fast lookup.
- **asset_hash_map**: Registered asset hashes for integrity verification at line rate.
- **xsk_map**: AF_XDP socket array for zero-copy redirect targeting.

Policy is serialized from userspace into a 32-byte little-endian format matching the C kernel struct, then synchronized to BPF maps through `sync_to_kernel`. This keeps kernel-space decisions consistent with userspace policy changes without requiring program reload.

### 5.4 Protocol Intelligence

STOQ is not a passive transport -- it provides protocol-level intelligence. Validation hooks allow two external systems to participate in packet-level decisions: STOQ registers a CertificateValidator (checking DER format, ASN.1 structure, and minimum certificate size) and a PacketValidator (checking QUIC header format, shard indices, and timestamp sanity with 5-minute clock skew tolerance). BlockMatrix registers an ExtensionValidator for HyperMesh-specific header fields (Proof of State, asset hash, matrix routing, privacy tier). All three validators are registered through `set_validation_hooks()` and invoked on the fast path.

Proof of State pre-validation occurs at the eBPF layer: format checks, algorithm indicator validation, and difficulty verification happen in kernel space. Deep cryptographic verification -- FALCON-1024 signature checks -- happens in userspace through TrustChain, and the results are fed back to the eBPF policy layer to update trust state.

### 5.5 Cryptographic Handshake and Congestion

The cryptographic handshake combines classical and post-quantum primitives. Key exchange uses X25519 elliptic-curve Diffie-Hellman, authenticated by FALCON-1024 lattice-based signatures. Each protocol frame carries a dynamic FALCON key_id, and real signature verification is performed by the FalconTrustChainClient using `pqcrypto_falcon::falcon1024`. Subsequent connections to the same peer use 0-RTT resumption.

Congestion control defaults to BBR v2, with CUBIC and NewReno available as alternatives. STOQ's BBR variant maintains per-path state, estimating bottleneck bandwidth and minimum RTT for each unique route. Six network tier profiles (Slow, Home, Standard, Performance, Enterprise, DataCenter) automatically configure buffer sizes and concurrency limits based on measured link capacity, from 256 KB send buffers with 10 concurrent streams on sub-100 Mbps links to 16 MB buffers with 1,000 concurrent streams on 10+ Gbps links.

### 5.6 Protocol Extensions

STOQ defines six custom QUIC frame types in the private-use range (`0xfe000001` through `0xfe000006`): token frames for Proof of State tokens, shard frames for distribution metadata, hop frames for routing, seed frames for initialization, and two FALCON frames for quantum-resistant authentication and key material. Transport parameters (`0xfe00` through `0xfe04`) negotiate extension support, FALCON enablement, public key exchange, maximum shard size (default: 1,400 bytes, MTU-safe), and token algorithm selection (SHA-256, SHA-384, SHA3-256, or BLAKE3).


### 5.7 Bootstrap Sequence

A circular dependency exists between the components required for a fully authenticated connection: TLS certificates require a TrustChain CA, CA discovery requires DNS, DNS registration requires a blockchain block with Proof of State, and block propagation requires a STOQ connection. HyperMesh breaks this cycle by separating encryption from authentication in the bootstrap phase.

**Phase 0 -- Bootstrap (implemented and verified).** A new node generates a self-signed certificate using its local TrustChain instance (no CA dependency). QUIC/TLS provides wire encryption; certificate validity is not enforced (all non-Public strategies use `AcceptAllVerifier`). The node then opens a bidirectional stream to a bootstrap peer and both sides exchange a flat JSON handshake containing their matrix coordinate, node ID, and a hex-encoded Proof of State token. The PoS token is validated bilaterally -- each side deserializes the peer's `StateProof`, verifies the four-proof structure, and rejects the connection if validation fails. Proof of State is the trust mechanism; TLS certificates are the encryption mechanism. This phase has been tested end-to-end with two independent nodes completing the full handshake sequence.

**Phase 1 -- Connected.** After PoS validation succeeds, nodes are trusted peers. DNS records are registered as blockchain assets, blocks propagate to connected peers, and shards distribute to matrix neighbors. All operations carry inline PoS headers verified at the eBPF layer (format checks) and in userspace (deep cryptographic verification).

**Phase 2 -- Full PKI (future work).** Once connected, nodes can request TrustChain-issued certificates from reflector or gateway nodes that serve as distributed CAs. These certificates replace the initial self-signed certificates, enabling strict TLS verification for long-lived connections. DNS resolution transitions from local HashMap to blockchain-backed queries propagated through the mesh. This phase is not yet implemented -- all current deployments operate with Phase 0 self-signed certificates and Phase 1 PoS-authenticated peer relationships.

This three-phase model ensures that a node can join the network with zero prior trust infrastructure. The minimum bootstrap requirement is a single reachable peer address and a valid Proof of State.


## 6. Identity: TrustChain

TrustChain provides identity without centralization through a federated certificate authority. Instead of a single root CA, signing authority is distributed using threshold cryptography: Shamir's Secret Sharing applied to FALCON-1024 key material, with a default threshold of 3-of-5. The full private key is never assembled in any single location. A certificate is valid when it carries signatures from the required threshold of independent CAs.

All identity operations use FALCON-1024, a lattice-based signature scheme standardized by NIST at Security Level V (256-bit security). Public keys are 1,793 bytes, secret keys are 2,305 bytes, and signatures are approximately 1,330 bytes. These sizes matter because signature verification is a hot path -- every authenticated stream, every certificate check, and every hash chain entry requires it. Message data is hashed with SHA-256 before signing to produce a fixed-size input regardless of payload.

Certificate Transparency is maintained through append-only Merkle tree logs using BLAKE3 as the hash function. Every certificate issuance, renewal, and revocation is recorded with a Signed Certificate Timestamp (SCT) and Merkle inclusion proof. Any node can verify log integrity by recomputing the tree and comparing root hashes. A CA that issues certificates without recording them in the transparency log is immediately detectable. A fingerprint tracker prevents duplicate certificate registration. Default certificate rotation is 24 hours.

DNS resolution operates over QUIC (DNS-over-QUIC) through the STOQ transport layer. The global gateway at `trust.hypermesh.online` provides the public entry point for mesh discovery. DNS names are themselves blockchain assets -- registration requires full Proof of State verification (WHERE the name resolves, WHO owns it, WHAT computational work supports it, WHEN it was registered).

### 6.1 Privacy Model

Privacy is configured along two axes, producing a grid:

                        Untracked           Tracked
                   +-------------------+-------------------+
    Unbounded      |   Anonymous       |   Public          |
                   |   Ephemeral keys, |   Registry-listed,|
                   |   unlinkable      |   fully           |
                   |                   |   discoverable    |
                   +-------------------+-------------------+
    Bounded        |   (valid but      |   Private         |
                   |    unnamed)       |   CA-scoped,      |
                   |                   |   compliance-     |
                   |                   |   ready           |
                   +-------------------+-------------------+

Anonymous mode uses ephemeral keys generated per connection and zeroized on disconnect. No identity persists across sessions. The connection timeout is 30 seconds. Private mode uses CA-issued certificates scoped to a specific federation, providing auditability within a trust boundary. The connection timeout extends to 90 seconds. Public mode registers identity in the global discovery registry, enabling full discoverability. Connections persist for up to 300 seconds. A single node operates with different presets simultaneously on different connections. The fourth quadrant (Bounded + Untracked) is valid but has no named preset.

Network isolation between privacy modes is cryptographic, not logical. Each mode maintains separate connection pools, separate routing state, and separate Proof of State chains. At the kernel level, privacy mode is encoded as a single byte in every packet's HyperMesh extension header (Anonymous = 0, Private = 2, Public = 3) and enforced by BPF policy maps -- isolation is not merely a software convention but a kernel-enforced boundary.

The privacy model operates on two independent dimensions that are often conflated in other systems. **Network privacy** (the two-axis model above) governs the transport layer: how packets are routed, whether connections are tracked, and what identity is disclosed. **Blockchain scope** (Section 7.2) governs the consensus layer: which nodes participate in state verification. These dimensions are fully orthogonal -- a Private blockchain can communicate over an Anonymous network for maximum security, and a Public blockchain can operate over a Federated network for controlled access.

### 6.2 Identity Lifecycle: Multi-Factor Mesh Authentication

Identity in HyperMesh must satisfy three invariants: it cannot be faked, it can be recovered, and it cannot be stolen. A single cryptographic key satisfies none of these -- keys can be copied (fakeable), lost (unrecoverable), and exfiltrated (stealable). HyperMesh addresses this through multi-factor mesh authentication (MFMA), mapping the three classical authentication factors to mesh-native primitives.

#### 6.2.1 Three Factors

**Something you HAVE: FALCON-1024 private key.** The node's signing key is a possession factor. It is generated at first boot, persisted on disk, and used to sign handshake challenges, state proofs, and key rotation entries. The key can be rotated -- possession of the current key is necessary for authentication but not sufficient.

**Something you KNOW: Recovery passphrase.** At genesis, the node operator provides (or the system generates) a recovery passphrase. The passphrase deterministically derives a recovery key commitment via `recovery_commitment = BLAKE3(HKDF-SHA512(passphrase, salt=node_id))`. This commitment is recorded in the genesis block as part of the `SystemAssetKind::Identity` asset entry. The passphrase is never stored -- only the commitment. The passphrase proves knowledge of a secret established at identity creation time without revealing the secret itself. This is the knowledge factor: something only the legitimate operator knows, independent of any key material on disk.

**Something you ARE: Chain history + hardware fingerprint + peer relationships.** The behavioral identity is the accumulated evidence that cannot be transferred or fabricated:

- **Chain history**: The sovereign hash chain is an append-only record of every block the node has produced, every asset it has registered, every DNS name it has claimed. An attacker cannot retroactively generate a consistent chain -- peers who synced blocks hold independent copies and can detect fabricated history.
- **Hardware fingerprint**: The genesis block records hardware assessment (R1) -- CPU cores, clock speed, RAM, storage, network interfaces. PoWork validations continuously verify that the node's computational output is consistent with its claimed hardware. A stolen key on different hardware produces inconsistent PoWork.
- **Peer relationships**: Each bilateral PoS handshake creates a mutual verification record. Over time, a node accumulates a web of peers who have independently verified its identity through the four proofs. This relationship graph cannot be forged -- each peer holds its own record of the verification.

The three factors are independent. Compromise of any single factor is insufficient for identity theft, and loss of any single factor is recoverable through the other two.

#### 6.2.2 Key Rotation

Key rotation is a first-class blockchain operation, not an out-of-band event. A rotation entry is a `BlockAssetEntry` with `SystemAssetKind::KeyRotation` containing:

- `old_pubkey_hash`: BLAKE3 hash of the outgoing FALCON public key
- `new_pubkey`: The full incoming FALCON-1024 public key (1,793 bytes)
- `new_kyber_pubkey`: The full incoming Kyber-1024 public key (1,568 bytes)
- `rotation_signature`: The OLD key signs `BLAKE3(old_pubkey_hash || new_pubkey || new_kyber_pubkey || block_index)`, proving the holder of the old key authorized the transition
- `reason`: Enum (Scheduled, Compromise, Upgrade, Recovery)

The `node_id` does NOT change on key rotation. After rotation, `node_id` remains `BLAKE3(genesis_falcon_pubkey)` -- the genesis key is the permanent anchor. Verification of a rotated identity requires walking the rotation chain: `genesis_key -> key_2 -> key_3 -> ... -> current_key`, where each transition is signed by the predecessor. This chain is bounded by the number of rotations (typically single digits over a node's lifetime), not by the chain length.

Peers who have bilateral PoS history with a node detect key rotation on the next handshake: the node presents its current key plus a proof that the key is authorized via the rotation chain. The peer verifies the chain back to the genesis key it has on record. If the rotation chain is valid and the other three proofs (PoSpace, PoWork, PoTime) are consistent with historical observations, the peer accepts the new key.

#### 6.2.3 Recovery

Recovery addresses two scenarios: key loss (the node operator can no longer sign with the current key) and data loss (the node's chain is lost from local storage).

**Key loss recovery.** Before keys are lost, the node must have established at least one recovery path:

1. **Passphrase recovery**: The operator provides the recovery passphrase established at genesis. The system derives the commitment and verifies it against the genesis block's `recovery_commitment`. If it matches, the operator can authorize a new keypair. The recovery entry is a special `KeyRotation` with `reason: Recovery` where the `rotation_signature` is replaced by `recovery_proof = BLAKE3(HKDF-SHA512(passphrase, salt=node_id) || new_pubkey)`. Peers verify this against the genesis commitment.

2. **Threshold peer recovery**: The node pre-registers recovery peers by recording a `recovery_peers` entry on its chain: `recovery_commitment_peers = BLAKE3(sorted(recovery_peer_node_ids))` with a threshold `k` (default: 3-of-5). When recovery is needed, `k` of the named peers co-sign an attestation: "We have bilateral PoS history with this node_id and attest that the entity presenting this new key is the same entity." Each attesting peer signs the attestation with their own FALCON key. The recovered node's chain includes the attestation as a `KeyRotation` entry with `reason: Recovery`.

3. **Shamir key backup**: At key creation or rotation, the node splits its FALCON secret key using Shamir's Secret Sharing (3-of-5 over GF(256)) and distributes shares to recovery peers via encrypted channels (Kyber-1024 KEM to each peer's public key). The shares are never stored together. Recovery requires contacting at least 3 of the 5 peers to reconstruct the original key. This is complementary to threshold peer recovery -- it recovers the original key rather than authorizing a new one.

**Data loss recovery.** If the node loses its local chain but retains its keys:

- Peers who participated in Network scope sync hold copies of the node's blocks (or at minimum, block hashes).
- The node requests its own chain from reflectors via the block fetching protocol (Section 7.5).
- After reconstruction, the node verifies chain integrity locally (BLAKE3 hash linkage) and resumes normal operation.
- If both keys and chain are lost, the operator uses passphrase recovery to authorize a new keypair, then fetches the chain from peers.

#### 6.2.4 Theft Prevention

A stolen FALCON private key alone is insufficient for identity theft because bilateral PoS requires all four proofs simultaneously:

- **PoSpace fails**: The attacker is at a different matrix position. Peers who have verified the legitimate node's position will see spatial inconsistency. The attacker cannot claim the same coordinates because RTT triangulation from reference nodes will produce different measurements.
- **PoWork fails**: The attacker's hardware differs from the genesis assessment. Computational output (benchmarks, hash rates, resource metrics) will not match the hardware fingerprint recorded in the genesis block.
- **PoTime fails**: Temporal continuity breaks. The legitimate node has an unbroken history of PoTime proofs. The attacker's chain fork starts with a temporal gap -- a discontinuity that peers detect.
- **Peer relationships fail**: Existing peers have accumulated bilateral verification history with the legitimate node. The attacker has no such history. Even if the attacker initiates new handshakes, the behavioral fingerprint (response patterns, resource availability, chain content) differs.

**Split-brain defense.** If an attacker with a stolen key attempts to rotate keys before the legitimate owner, two competing rotation entries may propagate. HyperMesh resolves this without global consensus: each peer independently decides which fork to trust based on its own bilateral verification history. The legitimate node has years of consistent PoS history; the attacker has none. A peer who has verified the legitimate node 1,000 times will not accept a competing rotation from an entity it has never verified. The legitimate owner needs to reach only one trusted peer to begin propagating the revocation -- the revocation then spreads through the mesh via normal block sync.

**Continuous verification.** Identity is not a one-time gate. Every bilateral PoS handshake re-verifies all four proofs. A compromised key is detected at the next handshake when the other three factors fail to match. This is zero-trust applied to mesh identity: never trust prior authentication, always verify the complete state proof. The MFA model ensures that identity is a continuous property of the node's behavior, not a static credential.

#### 6.2.5 Identity as Asset

The node's identity -- its FALCON and Kyber public keys -- is registered as a `SystemAssetKind::Identity` blockchain asset in the genesis block (R1, R10). This makes identity subject to the same Proof of State verification as any other asset. Key rotation entries are additional `SystemAssetKind::KeyRotation` asset entries on the same chain. The chain itself is the identity: a sovereign, append-only, tamper-evident record of every state transition the node has undergone. The key signs on behalf of the chain, but the chain IS the identity.

Identity is registered as a first-class blockchain asset (`SystemAssetKind::Identity`) in the genesis block. The asset's `definition` field contains the FALCON-1024 public key, `metadata` contains the Kyber-1024 public key, and `config` contains the recovery commitment (`BLAKE3(HKDF-SHA512(passphrase, salt=node_id, info="hypermesh-recovery-v1"))`).

The node's permanent identity (`node_id`) is `BLAKE3(genesis_falcon_pubkey)` -- derived from the genesis FALCON key and never changes, even after key rotation. This ensures that a node's identity in the mesh is stable across key lifecycle events.


## 7. Topology: BlockMatrix

Every node in the mesh occupies a position in a three-dimensional coordinate space. The x-axis encodes hop distance, the y-axis encodes latency profile, and the z-axis encodes bandwidth class. Three dimensions capture over 95% of routing variance in real mesh topologies; two dimensions cannot represent the asymmetry between latency and bandwidth that characterizes heterogeneous infrastructure. Four distance metrics are available: Euclidean (geometric distance), Manhattan (grid distance), Chebyshev (maximum-axis distance), and Haversine (great-circle distance for geospatial coordinates).

Coordinate assignment is decentralized. A joining node measures round-trip latency to a set of landmark nodes and computes a position that minimizes prediction error -- a variant of Vivaldi-style network coordinate embedding [7]. Coordinates converge in 5 to 10 cycles (50 to 100 seconds) and achieve 15% relative error for 95% of node pairs in meshes of 100 or more nodes.

Each link between two nodes is scored by a 4-element tensor: bandwidth (Gbps), latency (ms RTT), reliability (0 to 1), and load (0 to 1). Tensors are asymmetric -- the tensor from A to B differs from B to A, capturing differences in upload versus download capacity, routing path asymmetry, and load imbalance. Default weights for path optimization are bandwidth 0.2, latency 0.3, reliability 0.35, and load 0.15. Tensor values decay with a half-life of 30 seconds, ensuring stale measurements do not persist. A tensor operations library provides Vector3D and Matrix3x3 arithmetic, and A* pathfinding computes optimal routes through the coordinate space using the tensor-weighted cost function.

The topology serves a dual purpose: routing optimization and tamper evidence. Because shards are distributed to diverse matrix positions and each block commits to its shard distribution (Section 7.1), the matrix forms a 3D hash volume where every node's state is cross-linked to its spatial neighbors through shard evidence. Unlike a Merkle tree with a single root, the Block-MATRIX has no root — integrity emerges from the spatial consistency of overlapping evidence. Corrupting a node requires convincing all its shard holders (14+ independent positions per asset) to update their evidence, and those updates cascade through their own shard neighborhoods. The attack cost scales with the neighborhood size, not the chain depth.

Neighbor tables scale logarithmically: each node maintains detailed information about nearby nodes and progressively sparser information at greater coordinate distances, yielding O(neighbors) memory and O(log n) routing hops. Maximum neighbor count is 128 per node.

Geospatial clustering uses DBSCAN to form a three-level zone hierarchy: Local, Regional, and Global. Over 50 reference locations support GPS-to-matrix coordinate conversion. Zone membership determines replication boundaries, compliance regions, and aggregation domains.

### 7.1 Distribution Pipeline

The distribution pipeline processes every asset transfer through four stages in strict order. The ordering is deliberate: compression must precede encryption (encrypted data is incompressible), and encryption must precede sharding (each shard is individually meaningless without the encryption key, but the encryption operates on the whole blob so that key management is per-asset, not per-shard).

1. **Compression** -- Brotli streaming compression (levels 1 through 11, default level 4) on raw data. Chunk size is 64 KB for streaming operation.
2. **Encryption** -- Kyber-1024 key encapsulation generates a shared secret, which derives an AES-256-GCM symmetric key (256-bit key, 96-bit nonce). The entire compressed blob is encrypted as a single unit, producing authenticated ciphertext. This protects data at rest against harvest-now-decrypt-later attacks from quantum-capable adversaries.
3. **Erasure coding** -- Reed-Solomon coding with 10 data shards and 4 parity shards splits the encrypted blob. Any 10 of the 14 shards are sufficient for reconstruction, tolerating up to 4-shard loss at 40% storage overhead. Each shard carries a BLAKE3 hash, index, size, original size, and parity flag.
4. **Topology-aware placement** -- Shards are distributed across diverse matrix coordinates using the tensor-weighted cost function. Placement optimizes for fault tolerance (shards on nodes in different failure domains) and retrieval proximity (shards near likely consumers).

Retrieval is instruction-based rather than data-based. Instead of transferring raw data, the sender transmits a shard map: the content hash (BLAKE3, 32 bytes), individual shard hashes, matrix positions for each shard, and fallback locations. The instruction payload is under 1 KB regardless of asset size. The receiver queries the nearest matrix positions, fetches shards in parallel from multiple nodes, and reconstructs locally. This inverts the traditional model -- bandwidth cost is borne by the receiver pulling from nearby nodes, not by the sender pushing to a remote destination.

**Torrent-like shard distribution.** Shard distribution follows a seeding model, not a push model. The lifecycle has four phases:

1. **Store.** The creator compresses, encrypts, and erasure-codes the asset. Shard hashes and their matrix placements are recorded on-chain in a `BlockAssetEntry`. The shards themselves are offered to the reflector pool -- the on-chain entry records WHAT exists (asset hash, proof hash, state proof, storage pointer), while the reflector pool stores WHERE the data lives. This is the separation of concerns: integrity on-chain, availability off-chain.

2. **Seed.** Reflectors that accept shards become seeders for those shards. Each shard has its own AssetAddress (Section 7.3.1), making it independently routable and fetchable. As an asset grows in popularity, more reflectors hold its shards. The reflector pool for a popular asset grows organically with demand -- this is R12 (swarm scaling) in action: O(log N) per-node load for N concurrent consumers.

3. **Fetch.** A consumer receives a shard map (R6: under 1 KB) listing shard hashes and AssetAddresses. The shard map is the "torrent file" -- a compact instruction set that tells the consumer which shards to pull and where to find them. The consumer fetches shards from any available reflector, not necessarily the origin node. Because only 10 of 14 shards are needed for reconstruction (R5), the consumer can tolerate 4 unavailable seeders without data loss.

4. **Replicate.** After reconstruction, the consumer holds all shards it fetched. It can then serve those shards to other consumers -- the consumer becomes a provider. The reflector pool for the asset expands with every successful retrieval. A file accessed by 1,000 nodes is seeded by up to 1,000 nodes. A file accessed by one node is seeded by one node plus the original reflectors. Availability scales with demand, not with pre-provisioned infrastructure.

The shard map functions as a content manifest: it lists shard hashes (for integrity verification), AssetAddresses (for routing), and fallback locations (for resilience). A consumer that receives a shard map can verify each shard independently -- `BLAKE3(shard_data) == shard_hash` -- without trusting the provider. The combination of content-addressed shards, erasure coding, and consumer-becomes-provider replication produces a distribution system where popular content becomes more available, not less, and where no single node is a bottleneck.

**Storage pointers.** Each block entry carries a storage pointer that records where the asset data lives. For locally stored assets the pointer is a path. For distributed assets the pointer contains shard hashes and their matrix placements -- a per-entry commitment anchoring that entry's hash chain to its spatial shard evidence across the matrix. Private or device-scoped assets may have no shards at all (local storage only); sharding is a distribution concern, not a ledger requirement. The commitment creates cross-links between the originating node and every node holding its shards -- if the originating block is rewritten, the entry's content hash changes, but the shard evidence at neighboring positions still references the original hash. Integrity emerges from spatial consistency across overlapping shard evidence at independent matrix positions, not from a single Merkle root.

### 7.2 Blockchain Scopes

Every node maintains its own sovereign hash chain. The block structure is:

```
C = Brotli(A)
hA = BLAKE3(C)
π = PoS(A)
hπ = BLAKE3(π)

Block_i = { prev_hash, entries: [{ hA, hπ, state_proof, ptr }, ...] }
block_hash_i = BLAKE3(Block_i)
```

Each block is a batch of asset entries. Each entry carries its own content hash (hA), proof integrity hash (hπ), full four-proof state proof (WHO/WHEN/WHERE/WHAT), and a storage pointer (local path or shard placements). A block has no timestamp, no node coordinate, and no nonce -- temporal ordering lives in the state proof's PoTime, spatial location lives in PoSpace, and same content produces the same hash. The ledger secures integrity; the storage layer holds data.

The chain starts on boot with a genesis block containing the node's initial hardware-assessed assets, requiring no network connectivity. Identity attribution comes from the node's TrustChain certificate association, not per-entry signing.

Blockchain scope is a binary operating mode:

- **Device Scope**: A single node's local chain. Starts on boot, requires nothing external. Every node always runs a Device chain.
- **Network Scope**: Nodes synchronize their chains to shared state via reflector pooling. Devices in a network either reflect (all devices share the same pool of data) or they don't (purely peer-to-peer). For Public networks, spatial hash-bucket filtering (Section 7.4) limits each node's sync responsibility to blocks referencing its matrix neighborhood.

PrivacyMode (Anonymous/Private/Public) controls who can participate in a network. A family sharing devices is a Private (Bounded, tracked) network. A company is a larger Private network. An open community is a Public (Unbounded, tracked) network. Sub-federation -- groups within organizations within federations -- is handled by nesting Private networks, not by separate scope types. The same protocol operates at every level.

Cross-network asset transfers require Proof of State verification in both the source and destination networks. Gateway nodes bridge between networks, maintaining partial state from multiple networks and routing cross-network requests based on matrix topology.

The independence of blockchain scope from network privacy (Section 6.1) enables powerful combinations. A Network Scope blockchain communicating over an Anonymous transport provides private group computing with untraceable packets. A Network Scope blockchain over a Public transport provides shared state with full transparency. The two dimensions compose freely.

### 7.3 Asset System

Everything in BlockMatrix is an asset: compute, storage, memory, network bandwidth, GPU cycles, containers, and DNS names. The asset model is defined by three pillars in the canonical type system (hypermesh-lib) and implemented by each layer that manages resources.

**Kind** classifies assets at two levels. System assets (Cpu, Gpu, Memory, Storage, Network, Container, Economic, Blockchain, Dns) have stable numeric identifiers used in serialization and eBPF maps. User-defined assets are registered through Catalog with a content-addressed type hash derived from their package definition. The two-level `AssetKind` enum (`System(SystemAssetKind)` or `UserDefined(UserAssetKind)`) provides exhaustive classification without constraining what assets can represent.

**Status** is a programmable state machine. Every asset has an infrastructure-level lifecycle state (`BaseState`: Available, Allocated, InUse, Suspended, Maintenance, Failed) that the runtime uses for scheduling, health checks, and resource accounting. Domain-specific assets define richer state graphs -- a manufacturing asset might progress through Designed, Manufacturing, QA, Shipped, InService, and Retired -- where each domain state maps to one of the six BaseState values. The `AssetStatusTrait` defines the interface: current state name, base state mapping, transition validation, and available transitions. Implementations are provided by each asset type, not by a generic state machine in the type library.

**Adapter** is the fully programmable runtime interface. The `AssetAdapter` trait is the asset's executable behavior: lifecycle hooks (`on_create`, `on_transition`, `on_destroy`), a command/query dispatch interface (`execute` for mutating operations, `query` for reads), data validation, and self-describing capabilities. A GPU adapter exposes compute dispatch. A storage adapter handles encrypted sharding. A middleware adapter interfaces with other assets. A data pipeline adapter implements manufacturing-style staged processing. The adapter declares what it can do through `AdapterCapabilities` (supported commands, queries, streaming support, composition support, resource dependencies); the BlockMatrix runtime decides what to allow based on consensus proofs, resource limits, and sandboxing policy. Security is enforced by the runtime, not by the trait.

Six system adapters are implemented:

- **CPU Adapter**: Time-based scheduling with Proof of Work validation.
- **GPU Adapter**: FALCON-1024 quantum security for GPU memory, NAT-like addressing for remote GPU access.
- **Memory Adapter**: NAT-like proxy addressing that translates global IPv6-style addresses to local memory regions, enabling remote memory access across the mesh.
- **Storage Adapter**: Kyber-1024 encrypted sharding with content-aware deduplication through hash buckets.
- **Network Adapter**: Bandwidth allocation and QoS enforcement.
- **Container Adapter**: Resource isolation and orchestration for containerized workloads.

All assets are registered in the node's blockchain with a universal AssetId. Resource allocation uses tensor operations on the Block-MATRIX coordinate space -- the matrix position of available nodes, their measured capabilities, and the tensor-weighted distances all feed into placement decisions.

Privacy-aware allocation respects the node's configured privacy mode. Anonymous assets have no identity tracking. Private assets are bounded to specific networks. Public assets are fully discoverable. Users control resource allocation percentages, concurrent usage limits, and state proof requirements per asset.

#### 7.3.1 Asset Addressing

Every instantiated asset receives an IPv6 address in the HyperMesh ULA prefix (`fd48:4d00::/32`). The address encodes three pieces of information:

- **Matrix coordinates** (WHERE the asset lives in the topology)
- **Content fingerprint** (BLAKE3 hash of the compressed asset, truncated to fit the address space)
- **Shard index** (0 through n-1 for individual shard addressing within an erasure-coded asset)

This addressing scheme (R10) makes assets first-class network citizens. A shard is not just a data blob stored on a node -- it has a routable address. Any node in the mesh can reach a specific shard of a specific asset by addressing it directly. The shard index component means that individual shards can be requested without knowing which node currently holds them; the address encodes the content identity, and the reflector pool (Section 7.3.2) resolves the address to a provider.

#### 7.3.2 Asset-Level Trust

TrustChain can issue certificates not only for nodes but also for assets. An asset certificate binds an AssetAddress to three claims:

- **Provenance**: Which node created the asset (the creator's TrustChain identity and the block index where the asset was registered).
- **Access policy**: Which nodes or privacy scopes are authorized to fetch the asset's shards. A Public asset has an open policy; a Private asset lists authorized network IDs; an Anonymous asset has no certificate at all.
- **Integrity**: The asset's content hash (BLAKE3 of the compressed blob before encryption). The certificate commits to this hash, so any recipient can verify that reconstructed content matches the certified original.

Asset certificates are optional -- they add provenance guarantees on top of the content-addressing integrity that every asset already has. For Public assets that earn Caesar rewards, asset certificates provide the verifiable attribution chain that reward distribution requires: who created the content, when, and that the content has not been altered since certification.

The certificate is small (AssetAddress + content hash + policy flags + FALCON-1024 signature, approximately 1,500 bytes) and travels with the shard map (R6). A consumer receiving a shard map can verify the asset certificate before fetching any shards, confirming provenance and access rights without downloading the data first.

### 7.4 Block Synchronization: HashMatrix

Block synchronization strategy depends on blockchain scope and network privacy mode. Three modes cover all cases.

**Device Scope: no sync.** A Device chain is sovereign. It exists on one node, starts on boot, and never leaves. There is nothing to synchronize.

**Network Scope, Private: full replication.** A Private network is a bounded group -- a family, a team, a company. Every node in the group holds the complete chain. When a node appends a block, it broadcasts the block to all peers via reflector pooling. Every peer validates and appends. This is straightforward because Private networks are small by definition (bounded membership). Full replication provides the strongest consistency guarantee: every participant sees exactly the same chain at all times.

**Network Scope, Public: HashMatrix spatial filtering.** Public networks are unbounded. Full replication is not viable -- a mesh of 100,000 nodes cannot require every node to validate every block. HashMatrix solves this by aligning block validation responsibility with shard placement responsibility.

Each block entry's storage pointer (Section 7.1) contains shard hashes and their matrix placements -- actual 3D coordinates in the Block-MATRIX. A node's hash bucket is therefore spatial: it accepts and validates blocks whose entries have shard placements within its matrix neighborhood. If a node holds the shards, it validates the blocks that reference them. This unifies two concerns that other protocols treat separately.

The neighborhood radius adapts to peer density. In dense regions of the matrix (many nodes close together), each node covers a small spatial volume. In sparse regions, a single node covers a larger volume. The adaptation follows octree-like spatial partitioning: the mesh is recursively subdivided until each partition contains a manageable number of nodes. Boundary refinement uses the aggregate of neighboring node positions as the partitioning pivot. The result is self-balancing -- as nodes join a dense area, each node's radius shrinks automatically; as nodes leave, radius grows.

The validation cost per block is O(neighbors), not O(n). A node with 50 matrix neighbors validates blocks from those 50 neighbors, regardless of whether the mesh contains 1,000 or 1,000,000 total nodes. Neighborhood size is bounded by the adaptive radius, not by mesh size.

HashMatrix is consistent with PoSPing (Section 8.3). PoSPing verifies spatial consistency by ray-casting into the 3D hash volume and cross-checking shard evidence at claimed positions. HashMatrix block sync uses the same spatial model -- the same coordinates that determine where shards are placed determine who validates the blocks referencing them. The two mechanisms reinforce each other: HashMatrix ensures local consistency within a neighborhood, PoSPing samples global consistency across neighborhoods.

The matrix operations involved are coordinate distance comparisons, sorted neighbor lists, and radius calculations -- not tensor decompositions or gradient descents. Computational cost is O(n log k) where n is peers considered and k is the square root of nearby peers. No GPU is required.

    Approach                          Validation Cost     Topology-Aware    Shard-Aligned
    --------------------------------  ------------------  ----------------  ---------------
    Full replication (Bitcoin, etc.)   O(n) per block      No                N/A
    Committee sharding (Eth 2.0)      O(committee) random  No                No
    Hash-range partitioning (DHT)     O(1) per key        No                No
    HyperMesh HashMatrix              O(neighbors)        Yes               Yes

The key distinction is the final column. In committee-based sharding, the nodes validating a block have no relationship to the nodes storing the data that block references. In hash-range partitioning, keys map to nodes by hash prefix, ignoring network topology entirely. HashMatrix aligns validation with storage and both with spatial locality. A node validates what it stores, stores what is near it, and is near what it can reach quickly.

### 7.5 Block Fetching Protocol

Nodes that fall behind (offline, new join, partition recovery) fetch missing blocks from reflectors or peers via a two-phase pull protocol over STOQ streams:

**Phase 1 -- Sync Discovery**: The requesting node opens a STOQ stream and sends a `SyncRequest` containing its current chain height and head hash. The responder replies with a `SyncResponse` listing block hashes the requester is missing.

**Phase 2 -- Block Fetch**: The requester opens a second STOQ stream with a `BlockFetchRequest` containing the hashes of blocks it needs. The responder serializes each requested block and sends them in a `BlockFetchResponse`. The requester verifies `BLAKE3(block_bytes) == requested_hash` before insertion.

Each fetched block is validated through the standard chain integrity checks (index ordering, previous_hash linkage, per-entry proof verification) before insertion into the local chain. Blocks that fail validation are silently dropped.

The protocol is intentionally simple -- no Merkle proofs or range queries. A node that needs to catch up asks "what do you have that I don't?" and then "send me those blocks." STOQ's QUIC transport handles reliability, ordering, and flow control.

### 7.6 Network Scope Sync Architecture

Network scope blockchains synchronize across participating nodes via three components:

**SyncManager**: Maintains the set of known peers and their reported chain heights. Runs periodic sync rounds (default: 30 seconds) that compare local height against peer heights and initiate block fetching from peers that are ahead.

**ReflectorPool**: A subset of well-connected nodes that serve as block relay points. Reflectors accept blocks from any peer, validate them, and re-propagate to their connected peers. A node joins the reflector pool by announcing availability and maintaining uptime above a configurable threshold.

**Block Propagation**: When a node creates a new block, it announces the block hash to connected peers via `TAG_BLOCK_ANNOUNCE` (push). Peers that receive an announcement for a block they don't have initiate a block fetch (pull). This push-announce/pull-fetch model ensures blocks propagate without requiring every node to maintain connections to every other node.

The sync protocol is scope-aware:
- **Device scope**: No sync. Chain is local-only.
- **Network scope**: Full replication within the network. Every participating node holds every block.
- **Public scope (future)**: HashMatrix spatial filtering -- nodes only replicate blocks whose spatial hash falls within their neighborhood radius.

### 7.7 Metrics Pipeline

BlockMatrix nodes emit capacity metrics to Engauge via UDP datagrams every 30 seconds. Each datagram contains a `MetricsFrame` with:
- Chain height and block count
- Connected peer count
- CPU and memory utilization
- Shard storage metrics
- Sequence number for ordering

The UDP push is best-effort with backoff -- if Engauge is unreachable, the node backs off for 10 cycles (~5 minutes) before retrying. This avoids coupling node operation to metrics infrastructure availability.

Engauge aggregates frames from multiple nodes, applies differential privacy filtering (Laplace noise injection calibrated by epsilon), and produces routing intelligence feeds that inform BlockMatrix's tensor-weighted routing decisions.


## 8. Validation: Proof of State

Global consensus does not scale to large dynamic meshes. Raft requires O(n) messages per state change and assumes stable membership. PBFT requires O(n^2) messages and tolerates at most (n-1)/3 Byzantine faults. In a mesh of 10,000 nodes, a single state change under PBFT would generate on the order of 100 million messages.

HyperMesh replaces global consensus with **Proof of State** -- a bilateral authentication model. The distinction is fundamental: Proof of State does not seek agreement among many parties. It verifies that a specific claim about state is authentic. Something is either authentic or it isn't. There is no voting, no quorum, and no leader election.

Each node maintains its own sovereign hash chain: an append-only sequence of entries where each entry contains the operation, a timestamp, and a BLAKE3 hash linking to the previous entry. Identity attribution comes from the node's TrustChain certificate, not per-entry signing -- FALCON-1024 is used at the certificate level (TrustChain CA issuance, STOQ handshake authentication), not per block or per asset. The chain is tamper-evident -- modifying any entry invalidates all subsequent hashes.

Verification is bilateral. When node A needs to verify node B's state (for an asset transfer, workload delegation, or resource access), A requests a segment of B's hash chain covering the relevant period. A checks internal BLAKE3 hash consistency, confirms that the claimed state transitions are consistent with A's own observations, and verifies B's TrustChain certificate. No other nodes need to participate. There are no reputation scores, no time-decay violation tracking, and no float-based trust levels. Authentication is binary: authentic or not.

Verification depth scales with trust context. A routine transfer between established neighbors requires verification of the most recent 10 chain entries. A node joining a new federation requires full-chain verification. This graduated approach avoids unnecessary work for common operations while maintaining strong guarantees for high-stakes ones.

### 8.1 Four Proofs

Each operation requires a Proof of State composed of four independent proofs that together answer the fundamental questions of any state claim:

- **Proof of Space (WHERE)**: Verifies the physical resources -- storage, compute, bandwidth, and matrix position -- the node claims to contribute. Encoded as 16 bytes (IPv6-style matrix position) in every packet's extension header.
- **Proof of Stake (WHO)**: Verifies ownership, access rights, and economic stake. Encoded as 32 bytes identifying the claiming entity.
- **Proof of Work (WHAT/HOW)**: Verifies the computational capacity the node has demonstrated and the processing performed. Encoded as 32 bytes with hash-meets-difficulty validation at the eBPF layer.
- **Proof of Time (WHEN)**: Verifies temporal validity -- not expired, not future-dated. Encoded as a Unix-epoch microsecond timestamp (8 bytes) with 5-minute clock skew tolerance and 24-hour maximum age.

The complete Proof of State header is 88 bytes and travels inline with every packet in the HyperMesh extension header. Pre-validation (format checks, difficulty verification, timestamp bounds) happens at kernel speed in the XDP program. Deep cryptographic verification (TrustChain certificate validation) happens in userspace, and the results are fed back to the eBPF policy layer to update the node's authentication state.

### 8.2 Capability-Based Verification

Verification in HyperMesh is capability-based: usage IS verification. There are no proactive challenges, no resource type verification polls, and no periodic proof-of-capability requests. A node's capabilities are validated by observing its actual work output. If a node claims 8 GPUs but only delivers throughput consistent with 2, subsequent Proof of Work validations will reflect the actual capacity -- not the claimed capacity. There is no separate mechanism for "catching" overstatement; the bilateral authentication model inherently detects it through observed performance.

Matrix positions are derived from measured network latencies (RTT triangulation), not self-reported. A node cannot claim to be at position (10, 20, 5) if its round-trip times to known reference nodes are inconsistent with that position. Position verification happens naturally through the transport layer -- every STOQ connection provides latency measurements that the matrix can cross-reference.

For high-value transactions, optional witness corroboration adds assurance: witnesses from different BlockMatrix regions independently verify the transaction, providing defense against bilateral collusion. Witness selection uses coordinate diversity to ensure geographic distribution.

The result is that verification cost is proportional to transaction volume, not mesh size. A mesh of 10,000 nodes where 100 transactions occur per second requires verification of 100 transactions per second -- not 100 transactions multiplied by 10,000 nodes. Storage cost for the sovereign hash chain is approximately 200 MB per 1 million entries.

### 8.3 Spatial Verification: PoSPing

While bilateral hash chain verification (Section 8) detects tampering within a single chain, it cannot detect a node that rewrites its entire chain from genesis. Shard commitments (Section 7.1) create spatial cross-links that make such rewrites detectable, but cross-links are only useful if someone checks them. PoSPing provides the checking mechanism.

PoSPing is a lightweight, epoch-seeded probe protocol that verifies spatial consistency in the Block-MATRIX. Each probe is a "ray cast" into the 3D hash volume: a node queries a target's chain head, shard commitment, and shard map, then cross-checks with shard holders at the claimed positions. The result is binary: consistent or inconsistent.

Probe targets are selected deterministically from epoch entropy: `BLAKE3(epoch_seed || prober_position || epoch_counter)`. This prevents targeted harassment while ensuring uniform coverage of the matrix. Both prober and target can independently verify that a probe was legitimately assigned.

PoSPing respects the two-axis privacy model. The **Scope** axis gates probe access: Unbounded nodes accept probes from any node; Bounded nodes accept probes only from federation members. The **Traceability** axis filters the response: untracked nodes return shard positions without identity attribution; tracked nodes include NodeIds. The commitment hash itself is always verifiable — only the preimage disclosure varies.

The result is that the Block-MATRIX functions as a self-verifying 3D hash field. Each block's shard commitment creates spatial extent (like a Gaussian splat's covariance), PoSPing probes sample the field for consistency (like ray-casting), and the aggregated results (via Engauge) produce a rendered view of verified network capacity. Inconsistencies appear as spatial artifacts — a node whose chain head doesn't match its shard evidence at neighboring positions.

Verification cost remains proportional to probe rate, not mesh size. A mesh of 10,000 nodes with 3 probes per epoch per node generates 30,000 probes per minute — each requiring only a handful of hash comparisons.

### 8.4 WireSignedProof: Cryptographic State Proof Envelope

State proofs travel between nodes as `WireSignedProof` envelopes that bundle the proof data with a FALCON-1024 signature:

```
WireSignedProof = {
    proof_bytes: serialize(StateProof),
    nonce: random([u8; 32]),
    signature: FALCON-1024-sign(secret_key, BLAKE3(proof_bytes || nonce)),
    signer_pubkey: FALCON-1024-public-key
}
```

The verification path:
1. Deserialize the envelope
2. Recompute `BLAKE3(proof_bytes || nonce)`
3. Verify the FALCON-1024 signature against `signer_pubkey`
4. If valid, deserialize inner `proof_bytes` as `StateProof` and validate thresholds

This two-layer design separates identity binding (FALCON signature proves WHO generated the proof) from proof content (four sub-proofs answer WHERE/WHO/WHAT/WHEN). A valid envelope with failing sub-proofs means the node is honest but under-resourced. An invalid envelope means the proof should be discarded entirely.

For backward compatibility during rolling upgrades, validators attempt `WireSignedProof` deserialization first, falling back to raw `StateProof` bytes with a warning.


## 9. Cryptographic Foundation

HyperMesh uses a fixed cipher suite with deliberate separation of concerns. Each cryptographic function is assigned to the layer where its threat model applies.

    Function              Algorithm        Layer        Notes
    --------------------  -------------    ----------   --------------------------
    Signatures            FALCON-1024      TrustChain   NIST Level V, certificate-level only
    Key exchange          X25519           STOQ         Classical ECDH, FALCON-auth
    Storage encryption    Kyber-1024 KEM   BlockMatrix  Post-quantum, data at rest
    Key derivation        HKDF-SHA512      STOQ         RFC 5869
    Symmetric encryption  AES-256-GCM      All          256-bit key, 96-bit nonce
    Hashing               BLAKE3           All          Chunk integrity, hash chains

The separation between transport-layer and storage-layer cryptography is deliberate and reflects different threat models.

**FALCON-1024** secures identity at the certificate level: TrustChain CA issuance, STOQ handshake authentication, and threshold key distribution. It is NOT used per-block, per-entry, or per-asset -- sovereign hash chains use BLAKE3 hash linking for integrity, with identity attribution from TrustChain certificate association. FALCON provides compact signatures (~1,330 bytes) at NIST's highest security level (Level V, 256-bit security). Public keys are 1,793 bytes; secret keys are 2,305 bytes. NIST standardized FALCON (as FN-DSA) alongside ML-DSA (DILITHIUM) in 2024; FALCON was selected for HyperMesh due to its signature compactness. Both Falcon-512 (Level I, 128-bit) and Falcon-1024 (Level V, 256-bit) are supported, with Falcon-1024 as the default.

**Kyber-1024** (ML-KEM) protects stored data through lattice-based key encapsulation. The KEM produces a shared secret from which an AES-256-GCM symmetric key is derived. Encrypted shards may persist on disk for weeks or months -- well within the window where a quantum-capable adversary could mount a harvest-now-decrypt-later attack. Post-quantum encryption for data at rest is therefore a strict requirement. Encryption operates on whole assets before sharding, so key management is per-asset rather than per-shard.

**X25519** secures transport-layer key exchange. Session keys are ephemeral and exist only in memory for the duration of a connection. A quantum adversary would need to perform real-time cryptanalysis during the connection lifetime -- a substantially weaker threat model than breaking stored ciphertext offline. Using classical key exchange at the transport layer keeps the handshake small (X25519 adds 32 bytes versus approximately 1,568 bytes for Kyber-1024) and fast (0.12 ms for X25519 key agreement).

**BLAKE3** provides integrity hashing throughout the protocol. It produces 32-byte digests, runs approximately 3 times faster than SHA-256, and is inherently parallelizable. Every chunk, every hash chain entry, every Merkle tree node, and every asset hash uses BLAKE3.

The cipher suite is not negotiable per-connection. All nodes run the same algorithms. This eliminates downgrade attacks and simplifies verification: a node never needs to check which algorithm was used, only whether the proof is valid.


## 10. Applications

**Distributed compute.** Catalog's execution delegation model places workloads on mesh nodes selected by BlockMatrix coordinates. The scheduler evaluates available nodes by distance to required data, current load, available resources, and trust context. Every execution requires Proof of State verification, and results are recorded in the executor's sovereign hash chain, producing an auditable, tamper-evident computation trail. Failover is automatic: when a node fails during computation, the workload migrates to the nearest equivalent node and resumes from the last checkpoint stored through the distribution pipeline.

**Enterprise private meshes.** An enterprise deploys HyperMesh within its own infrastructure, controlling the root CA, node membership, and trust boundaries. BlockMatrix's geospatial clustering enforces data residency -- shards are constrained to nodes within designated geographic zones. TrustChain's Certificate Transparency provides a complete audit trail. The Private privacy preset (Bounded + Tracked) delivers compliance readiness for frameworks including GDPR, HIPAA, and SOC 2. Gateway nodes enable selective peering with the public mesh or other private deployments under controlled policy.

**Sovereign infrastructure.** Any participant can contribute compute, storage, and bandwidth as discoverable mesh assets. Resources are registered through Catalog with capability declarations and availability parameters. The node's sovereign hash chain records all resource contributions and verifications, building a cryptographically attested operational history. Mesh positioning through BlockMatrix coordinates reflects actual network performance, not static configuration.

**Economic interop (Caesar).** Caesar is an optional economic interop bridge for value transfer across the mesh. The CAES token is a purely intermediary exchange medium -- not a store of value. Caesar bridges fiat and cryptocurrency systems through a normalized payment interface. For full protocol details, see the Caesar whitepaper.

**Execution & analytics (Engauge).** Engauge provides privacy-preserving network metrics and verification analytics. Four streaming payload types (Capacity, Congestion, Routing, Economic) flow through a differential privacy filter calibrated to each node's privacy mode: Anonymous nodes share nothing, Private nodes share capacity and congestion within their federation, Public nodes share all payloads mesh-wide. A fifth payload type (Verification) carries PoSPing consistency results, enabling RegionalAggregates to report verified capacity — the actual measured and cross-checked capability of a network region, not self-reported claims. Caesar reward eligibility requires Public Engauge participation with passing PoSPing verification.


## 11. Implementation

HyperMesh is implemented in Rust across nine crates with a shared canonical type system. The node ships as a single statically-linked binary with a single TOML configuration file. The project is open source.

The implementation is structured to reflect the layered architecture: STOQ provides QUIC transport with eBPF kernel integration and FALCON-1024 cryptography. TrustChain implements the federated certificate authority, Certificate Transparency, and DNS-over-QUIC resolution. BlockMatrix implements the 3D coordinate system, tensor operations, every-node blockchain, geospatial clustering, matrix persistence (write-ahead log with incremental snapshots), and the complete asset system with six resource adapters. Catalog provides the asset package registry with execution delegation. Caesar implements token economics and multi-chain bridging. The eBPF intelligence layer compiles C kernel programs (XDP, kprobe, tracepoint) and manages AF_XDP zero-copy sockets. Gateway provides the HTTP/3 entry point at `trust.hypermesh.online`.

The protocol specification and implementation continue to evolve. This document describes the architecture as designed and implemented in March 2026.


## References

[1] National Institute of Standards and Technology, "Module-Lattice-Based Key-Encapsulation Mechanism Standard" (FIPS 203), August 2024.

[2] National Institute of Standards and Technology, "Module-Lattice-Based Digital Signature Standard" (FIPS 204), August 2024.

[3] National Institute of Standards and Technology, "Stateless Hash-Based Digital Signature Standard" (FIPS 205), August 2024.

[4] P. Fouque et al., "FALCON: Fast-Fourier Lattice-based Compact Signatures over NTRU," NIST Post-Quantum Cryptography Standardization, Round 3 Submission, 2020.

[5] R. Avanzi et al., "CRYSTALS-Kyber (ML-KEM): Algorithm Specifications and Supporting Documentation," NIST Post-Quantum Cryptography Standardization, 2024.

[6] J. O'Connor et al., "BLAKE3: One Function, Fast Everywhere," 2020.

[7] F. Dabek et al., "Vivaldi: A Decentralized Network Coordinate System," ACM SIGCOMM, 2004.

[8] N. Cardwell et al., "BBR: Congestion-Based Congestion Control," ACM Queue, vol. 14, no. 5, 2016.

[9] M. Castro and B. Liskov, "Practical Byzantine Fault Tolerance," OSDI, 1999.

[10] D. Ongaro and J. Ousterhout, "In Search of an Understandable Consensus Algorithm," USENIX ATC, 2014.
