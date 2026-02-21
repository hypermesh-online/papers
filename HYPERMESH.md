# HyperMesh: A Sovereign Distributed Computing Protocol

**Version 0.2 -- February 2026**

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


## 3. Architecture

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


## 4. Transport: STOQ Protocol

STOQ is built on QUIC (via the Quinn library in Rust) running exclusively over IPv6. The choice of QUIC provides multiplexed streams with independent flow control, 0-RTT reconnection for returning peers, and connection migration when a node's address changes. IPv6 gives every node a globally routable address, eliminating NAT traversal and enabling direct peer connectivity. The protocol identifier is `stoq/1.0` (ALPN negotiation per RFC 8439).

### 4.1 Kernel Integration

Packet processing begins in kernel space, before the operating system's network stack allocates socket buffers. A unified eBPF intelligence layer (hypermesh-ebpf) serves as the single source of truth for all packet decisions. Three consumers interact with it: STOQ as a thin transport wrapper, BlockMatrix as a policy configurator, and the kernel XDP program as the enforcement point.

Four classes of eBPF programs provide different capabilities. **XDP programs** classify and route packets at the earliest point in the network stack. The primary HyperMesh XDP program parses a custom extension header (identified by magic bytes `0x484D`) that carries Proof of State, asset hash, matrix routing, and privacy tier fields inline with every packet. Based on policy flags, the XDP program makes one of four decisions for each packet: Pass to the local network stack, Redirect to an AF_XDP socket for zero-copy delivery to STOQ, Forward via XDP_TX to another matrix node, or Drop with a recorded reason. **Kprobe programs** instrument kernel functions for zero-overhead observability -- connection lifecycle events, syscall patterns, and error conditions. **Tracepoint programs** capture structured kernel events for performance analysis. A **counter program** tracks packet rates and byte volumes per interface.

These four program types are compiled from C source by the build system and loaded into the kernel at runtime through the eBPF loader.

### 4.2 Zero-Copy Packet I/O

High-throughput data paths bypass the kernel's socket buffer allocation entirely through AF_XDP (Address Family XDP). When the XDP program redirects a packet, it lands directly in a shared memory region (UMEM) accessible to both kernel and userspace, eliminating copy operations.

The AF_XDP architecture uses four ring buffers mapped into userspace memory:

- **Fill Ring**: Userspace supplies empty UMEM frames for the kernel to fill with incoming packets.
- **RX Ring**: The kernel places received packet descriptors here for userspace consumption.
- **TX Ring**: Userspace places outgoing packet descriptors here for kernel transmission.
- **Completion Ring**: The kernel returns transmitted frame descriptors so userspace can reclaim them.

Each ring uses atomic producer and consumer pointers for lock-free operation. A frame allocator manages the UMEM region with batch allocation and a thread-safe free-list, allowing frames to be acquired and released without contention. Packet transmission uses the `sendto` syscall; reception uses `poll` and `recvfrom`. Batch operations amortize syscall overhead across multiple packets.

The system degrades gracefully across three tiers: full eBPF with AF_XDP zero-copy on capable kernels, eBPF without AF_XDP on older kernels, and pure userspace processing where eBPF is unavailable. The orchestrator detects kernel capabilities at startup and selects the highest-performance path available.

### 4.3 Policy Enforcement

Four BPF maps maintain kernel-side state that the XDP program consults on every packet:

- **policy_map**: Per-interface validation policy (whether to require Proof of State headers, validate asset hashes, or check matrix routing).
- **pos_header_map**: Cached Proof of State header data for fast lookup.
- **asset_hash_map**: Registered asset hashes for integrity verification at line rate.
- **xsk_map**: AF_XDP socket array for zero-copy redirect targeting.

Policy is serialized from userspace into a 32-byte little-endian format matching the C kernel struct, then synchronized to BPF maps through `sync_to_kernel`. This keeps kernel-space decisions consistent with userspace policy changes without requiring program reload.

### 4.4 Protocol Intelligence

STOQ is not a passive transport -- it provides protocol-level intelligence. Validation hooks allow two external systems to participate in packet-level decisions: STOQ registers a CertificateValidator (checking DER format, ASN.1 structure, and minimum certificate size) and a PacketValidator (checking QUIC header format, shard indices, and timestamp sanity with 5-minute clock skew tolerance). BlockMatrix registers an ExtensionValidator for HyperMesh-specific header fields (Proof of State, asset hash, matrix routing, privacy tier). All three validators are registered through `set_validation_hooks()` and invoked on the fast path.

Proof of State pre-validation occurs at the eBPF layer: format checks, algorithm indicator validation, and difficulty verification happen in kernel space. Deep cryptographic verification -- FALCON-1024 signature checks -- happens in userspace through TrustChain, and the results are fed back to the eBPF policy layer to update trust state.

### 4.5 Cryptographic Handshake and Congestion

The cryptographic handshake combines classical and post-quantum primitives. Key exchange uses X25519 elliptic-curve Diffie-Hellman, authenticated by FALCON-1024 lattice-based signatures. Each protocol frame carries a dynamic FALCON key_id, and real signature verification is performed by the FalconTrustChainClient using `pqcrypto_falcon::falcon1024`. Subsequent connections to the same peer use 0-RTT resumption.

Congestion control defaults to BBR v2, with CUBIC and NewReno available as alternatives. STOQ's BBR variant maintains per-path state, estimating bottleneck bandwidth and minimum RTT for each unique route. Six network tier profiles (Slow, Home, Standard, Performance, Enterprise, DataCenter) automatically configure buffer sizes and concurrency limits based on measured link capacity, from 256 KB send buffers with 10 concurrent streams on sub-100 Mbps links to 16 MB buffers with 1,000 concurrent streams on 10+ Gbps links.

### 4.6 Protocol Extensions

STOQ defines six custom QUIC frame types in the private-use range (`0xfe000001` through `0xfe000006`): token frames for Proof of State tokens, shard frames for distribution metadata, hop frames for routing, seed frames for initialization, and two FALCON frames for quantum-resistant authentication and key material. Transport parameters (`0xfe00` through `0xfe04`) negotiate extension support, FALCON enablement, public key exchange, maximum shard size (default: 1,400 bytes, MTU-safe), and token algorithm selection (SHA-256, SHA-384, SHA3-256, or BLAKE3).


## 5. Identity: TrustChain

TrustChain provides identity without centralization through a federated certificate authority. Instead of a single root CA, signing authority is distributed using threshold cryptography: Shamir's Secret Sharing applied to FALCON-1024 key material, with a default threshold of 3-of-5. The full private key is never assembled in any single location. A certificate is valid when it carries signatures from the required threshold of independent CAs.

All identity operations use FALCON-1024, a lattice-based signature scheme standardized by NIST at Security Level V (256-bit security). Public keys are 1,793 bytes, secret keys are 2,305 bytes, and signatures are approximately 1,330 bytes. These sizes matter because signature verification is a hot path -- every authenticated stream, every certificate check, and every hash chain entry requires it. Message data is hashed with SHA-256 before signing to produce a fixed-size input regardless of payload.

Certificate Transparency is maintained through append-only Merkle tree logs using BLAKE3 as the hash function. Every certificate issuance, renewal, and revocation is recorded with a Signed Certificate Timestamp (SCT) and Merkle inclusion proof. Any node can verify log integrity by recomputing the tree and comparing root hashes. A CA that issues certificates without recording them in the transparency log is immediately detectable. A fingerprint tracker prevents duplicate certificate registration. Default certificate rotation is 24 hours.

DNS resolution operates over QUIC (DNS-over-QUIC) through the STOQ transport layer. The global gateway at `trust.hypermesh.online` provides the public entry point for mesh discovery. DNS names are themselves blockchain assets -- registration requires full Proof of State verification (WHERE the name resolves, WHO owns it, WHAT computational work supports it, WHEN it was registered).

### 5.1 Privacy Model

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

Anonymous mode uses ephemeral keys generated per connection and zeroized on disconnect. No identity persists across sessions. The connection timeout is 30 seconds. Private mode uses CA-issued certificates scoped to a specific federation, providing auditability within a trust boundary. The connection timeout extends to 90 seconds. Public mode registers identity in the global discovery registry, enabling reputation and full discoverability. Connections persist for up to 300 seconds. A single node operates with different presets simultaneously on different connections. The fourth quadrant (Bounded + Untracked) is valid but has no named preset.

Network isolation between privacy modes is cryptographic, not logical. Each mode maintains separate connection pools, separate routing state, and separate Proof of State chains. At the kernel level, privacy mode is encoded as a single byte in every packet's HyperMesh extension header (Anonymous = 0, Private = 2, Public = 3) and enforced by BPF policy maps -- isolation is not merely a software convention but a kernel-enforced boundary.

The privacy model operates on two independent dimensions that are often conflated in other systems. **Network privacy** (the two-axis model above) governs the transport layer: how packets are routed, whether connections are tracked, and what identity is disclosed. **Blockchain scope** (Section 6.2) governs the consensus layer: which nodes participate in state verification. These dimensions are fully orthogonal -- a Private blockchain can communicate over an Anonymous network for maximum security, and a Public blockchain can operate over a Federated network for controlled access.


## 6. Topology: BlockMatrix

Every node in the mesh occupies a position in a three-dimensional coordinate space. The x-axis encodes hop distance, the y-axis encodes latency profile, and the z-axis encodes bandwidth class. Three dimensions capture over 95% of routing variance in real mesh topologies; two dimensions cannot represent the asymmetry between latency and bandwidth that characterizes heterogeneous infrastructure. Four distance metrics are available: Euclidean (geometric distance), Manhattan (grid distance), Chebyshev (maximum-axis distance), and Haversine (great-circle distance for geospatial coordinates).

Coordinate assignment is decentralized. A joining node measures round-trip latency to a set of landmark nodes and computes a position that minimizes prediction error -- a variant of Vivaldi-style network coordinate embedding [7]. Coordinates converge in 5 to 10 cycles (50 to 100 seconds) and achieve 15% relative error for 95% of node pairs in meshes of 100 or more nodes.

Each link between two nodes is scored by a 4-element tensor: bandwidth (Gbps), latency (ms RTT), reliability (0 to 1), and load (0 to 1). Tensors are asymmetric -- the tensor from A to B differs from B to A, capturing differences in upload versus download capacity, routing path asymmetry, and load imbalance. Default weights for path optimization are bandwidth 0.2, latency 0.3, reliability 0.35, and load 0.15. Tensor values decay with a half-life of 30 seconds, ensuring stale measurements do not persist. A tensor operations library provides Vector3D and Matrix3x3 arithmetic, and A* pathfinding computes optimal routes through the coordinate space using the tensor-weighted cost function.

Neighbor tables scale logarithmically: each node maintains detailed information about nearby nodes and progressively sparser information at greater coordinate distances, yielding O(neighbors) memory and O(log n) routing hops. Maximum neighbor count is 128 per node.

Geospatial clustering uses DBSCAN to form a three-level zone hierarchy: Local, Regional, and Global. Over 50 reference locations support GPS-to-matrix coordinate conversion. Zone membership determines replication boundaries, compliance regions, and aggregation domains.

### 6.1 Distribution Pipeline

The distribution pipeline processes every asset transfer through four stages in strict order. The ordering is deliberate: compression must precede encryption (encrypted data is incompressible), and encryption must precede sharding (each shard is individually meaningless without the encryption key, but the encryption operates on the whole blob so that key management is per-asset, not per-shard).

1. **Compression** -- Brotli streaming compression (levels 1 through 11, default level 4) on raw data. Chunk size is 64 KB for streaming operation.
2. **Encryption** -- Kyber-1024 key encapsulation generates a shared secret, which derives an AES-256-GCM symmetric key (256-bit key, 96-bit nonce). The entire compressed blob is encrypted as a single unit, producing authenticated ciphertext. This protects data at rest against harvest-now-decrypt-later attacks from quantum-capable adversaries.
3. **Erasure coding** -- Reed-Solomon coding with 10 data shards and 4 parity shards splits the encrypted blob. Any 10 of the 14 shards are sufficient for reconstruction, tolerating up to 4-shard loss at 40% storage overhead. Each shard carries a BLAKE3 hash, index, size, original size, and parity flag.
4. **Topology-aware placement** -- Shards are distributed across diverse matrix coordinates using the tensor-weighted cost function. Placement optimizes for fault tolerance (shards on nodes in different failure domains) and retrieval proximity (shards near likely consumers).

Retrieval is instruction-based rather than data-based. Instead of transferring raw data, the sender transmits a shard map: the content hash (BLAKE3, 32 bytes), individual shard hashes, matrix positions for each shard, and fallback locations. The instruction payload is under 1 KB regardless of asset size. The receiver queries the nearest matrix positions, fetches shards in parallel from multiple nodes, and reconstructs locally. This inverts the traditional model -- bandwidth cost is borne by the receiver pulling from nearby nodes, not by the sender pushing to a remote destination.

### 6.2 Blockchain Scopes

Every node maintains its own sovereign hash chain -- an append-only sequence of entries where each entry contains the operation, a BLAKE3 hash linking to the previous entry, and a FALCON-1024 signature from the node's TrustChain identity. The chain starts on boot with a unique genesis block, requiring no network connectivity. This local blockchain is the foundation of all state verification.

Blockchain scope is a binary operating mode:

- **Device Scope**: A single node's local chain. Starts on boot, requires nothing external. Every node always runs a Device chain.
- **Network Scope**: Nodes synchronize their chains to shared state via reflector pooling. Devices in a network either reflect (all devices share the same pool of data) or they don't (purely peer-to-peer).

PrivacyMode (Anonymous/Private/Public) controls who can participate in a network. A family sharing devices is a Private (Bounded, tracked) network. A company is a larger Private network. An open community is a Public (Unbounded, tracked) network. Sub-federation -- groups within organizations within federations -- is handled by nesting Private networks, not by separate scope types. The same protocol operates at every level.

Cross-network asset transfers require Proof of State verification in both the source and destination networks. Gateway nodes bridge between networks, maintaining partial state from multiple networks and routing cross-network requests based on matrix topology.

The independence of blockchain scope from network privacy (Section 5.1) enables powerful combinations. A Network Scope blockchain communicating over an Anonymous transport provides private group computing with untraceable packets. A Network Scope blockchain over a Public transport provides shared state with full transparency. The two dimensions compose freely.

### 6.3 Asset System

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

Privacy-aware allocation respects the node's configured privacy mode. Anonymous assets have no identity tracking. Private assets are bounded to specific networks. Public assets are fully discoverable. Users control resource allocation percentages, concurrent usage limits, and consensus requirements per asset.


## 7. Validation: Proof of State

Global consensus does not scale to large dynamic meshes. Raft requires O(n) messages per state change and assumes stable membership. PBFT requires O(n^2) messages and tolerates at most (n-1)/3 Byzantine faults. In a mesh of 10,000 nodes, a single state change under PBFT would generate on the order of 100 million messages.

HyperMesh replaces global consensus with **Proof of State** -- a bilateral authentication model. The distinction is fundamental: Proof of State does not seek agreement among many parties. It verifies that a specific claim about state is authentic. Something is either authentic or it isn't. There is no voting, no quorum, and no leader election.

Each node maintains its own sovereign hash chain: an append-only sequence of entries where each entry contains the operation, a timestamp, a BLAKE3 hash linking to the previous entry, and a FALCON-1024 signature from the node's TrustChain identity. The chain is tamper-evident -- modifying any entry invalidates all subsequent hashes.

Verification is bilateral. When node A needs to verify node B's state (for an asset transfer, workload delegation, or trust evaluation), A requests a segment of B's hash chain covering the relevant period. A checks internal hash consistency, validates the FALCON-1024 signatures, and confirms that the claimed state transitions are consistent with A's own observations. No other nodes need to participate.

Verification depth scales with trust context. A routine transfer between established neighbors requires verification of the most recent 10 chain entries. A node joining a new federation requires full-chain verification. This graduated approach avoids unnecessary work for common operations while maintaining strong guarantees for high-stakes ones.

### 7.1 Four Proofs

Each operation requires a Proof of State composed of four independent proofs that together answer the fundamental questions of any state claim:

- **Proof of Space (WHERE)**: Verifies the physical resources -- storage, compute, bandwidth, and matrix position -- the node claims to contribute. Encoded as 16 bytes (IPv6-style matrix position) in every packet's extension header.
- **Proof of Stake (WHO)**: Verifies ownership, access rights, and economic stake. Encoded as 32 bytes identifying the claiming entity.
- **Proof of Work (WHAT/HOW)**: Verifies the computational capacity the node has demonstrated and the processing performed. Encoded as 32 bytes with hash-meets-difficulty validation at the eBPF layer.
- **Proof of Time (WHEN)**: Verifies temporal validity -- not expired, not future-dated. Encoded as a Unix-epoch microsecond timestamp (8 bytes) with 5-minute clock skew tolerance and 24-hour maximum age.

The complete Proof of State header is 88 bytes and travels inline with every packet in the HyperMesh extension header. Pre-validation (format checks, difficulty verification, timestamp bounds) happens at kernel speed in the XDP program. Deep cryptographic verification (FALCON-1024 signature validation) happens in userspace through TrustChain, and the results are fed back to the eBPF policy layer to update the node's trust state.

For high-value transactions, optional witness corroboration adds assurance: witnesses from different BlockMatrix regions independently verify the transaction, providing defense against bilateral collusion. Witness selection uses coordinate diversity to ensure geographic distribution.

The result is that verification cost is proportional to transaction volume, not mesh size. A mesh of 10,000 nodes where 100 transactions occur per second requires verification of 100 transactions per second -- not 100 transactions multiplied by 10,000 nodes. Storage cost for the sovereign hash chain is approximately 200 MB per 1 million entries.


## 8. Cryptographic Foundation

HyperMesh uses a fixed cipher suite with deliberate separation of concerns. Each cryptographic function is assigned to the layer where its threat model applies.

    Function              Algorithm        Layer        Notes
    --------------------  -------------    ----------   --------------------------
    Signatures            FALCON-1024      TrustChain   NIST Level V, lattice-based
    Key exchange          X25519           STOQ         Classical ECDH, FALCON-auth
    Storage encryption    Kyber-1024 KEM   BlockMatrix  Post-quantum, data at rest
    Key derivation        HKDF-SHA512      STOQ         RFC 5869
    Symmetric encryption  AES-256-GCM      All          256-bit key, 96-bit nonce
    Hashing               BLAKE3           All          Chunk integrity, hash chains

The separation between transport-layer and storage-layer cryptography is deliberate and reflects different threat models.

**FALCON-1024** secures all identity operations. It provides compact signatures (~1,330 bytes) at NIST's highest security level (Level V, 256-bit security), making it suitable for the mesh's signature-heavy workload. Public keys are 1,793 bytes; secret keys are 2,305 bytes. NIST standardized FALCON (as FN-DSA) alongside ML-DSA (DILITHIUM) in 2024; FALCON was selected for HyperMesh due to its signature compactness. Both Falcon-512 (Level I, 128-bit) and Falcon-1024 (Level V, 256-bit) are supported, with Falcon-1024 as the default.

**Kyber-1024** (ML-KEM) protects stored data through lattice-based key encapsulation. The KEM produces a shared secret from which an AES-256-GCM symmetric key is derived. Encrypted shards may persist on disk for weeks or months -- well within the window where a quantum-capable adversary could mount a harvest-now-decrypt-later attack. Post-quantum encryption for data at rest is therefore a strict requirement. Encryption operates on whole assets before sharding, so key management is per-asset rather than per-shard.

**X25519** secures transport-layer key exchange. Session keys are ephemeral and exist only in memory for the duration of a connection. A quantum adversary would need to perform real-time cryptanalysis during the connection lifetime -- a substantially weaker threat model than breaking stored ciphertext offline. Using classical key exchange at the transport layer keeps the handshake small (X25519 adds 32 bytes versus approximately 1,568 bytes for Kyber-1024) and fast (0.12 ms for X25519 key agreement).

**BLAKE3** provides integrity hashing throughout the protocol. It produces 32-byte digests, runs approximately 3 times faster than SHA-256, and is inherently parallelizable. Every chunk, every hash chain entry, every Merkle tree node, and every asset hash uses BLAKE3.

The cipher suite is not negotiable per-connection. All nodes run the same algorithms. This eliminates downgrade attacks and simplifies verification: a node never needs to check which algorithm was used, only whether the proof is valid.


## 9. Applications

**Distributed compute.** Catalog's execution delegation model places workloads on mesh nodes selected by BlockMatrix coordinates. The scheduler evaluates available nodes by distance to required data, current load, available resources, and trust context. Every execution requires Proof of State verification, and results are recorded in the executor's sovereign hash chain, producing an auditable, tamper-evident computation trail. Failover is automatic: when a node fails during computation, the workload migrates to the nearest equivalent node and resumes from the last checkpoint stored through the distribution pipeline.

**Enterprise private meshes.** An enterprise deploys HyperMesh within its own infrastructure, controlling the root CA, node membership, and trust boundaries. BlockMatrix's geospatial clustering enforces data residency -- shards are constrained to nodes within designated geographic zones. TrustChain's Certificate Transparency provides a complete audit trail. The Private privacy preset (Bounded + Tracked) delivers compliance readiness for frameworks including GDPR, HIPAA, and SOC 2. Gateway nodes enable selective peering with the public mesh or other private deployments under controlled policy.

**Sovereign infrastructure.** Any participant can contribute compute, storage, and bandwidth as discoverable mesh assets. Resources are registered through Catalog with capability declarations and availability parameters. The node's sovereign hash chain records all resource contributions and verifications, building a cryptographically attested operational history. Mesh positioning through BlockMatrix coordinates reflects actual network performance, not static configuration.

**Economic interop (Caesar).** Caesar is an optional economic interop bridge for value transfer across the mesh. The CAES token is a purely intermediary exchange medium -- not a store of value. Caesar bridges fiat and cryptocurrency systems through a normalized payment interface. For full protocol details, see the Caesar whitepaper.

**Execution & analytics (Engauge).** Engauge is a planned execution and analytics layer that distributes rewards for paid content hosting and provides privacy-preserving measurement of hosting quality and network performance.


## 10. Implementation

HyperMesh is implemented in Rust across nine crates with a shared canonical type system. The node ships as a single statically-linked binary with a single TOML configuration file. The project is open source.

The implementation is structured to reflect the layered architecture: STOQ provides QUIC transport with eBPF kernel integration and FALCON-1024 cryptography. TrustChain implements the federated certificate authority, Certificate Transparency, and DNS-over-QUIC resolution. BlockMatrix implements the 3D coordinate system, tensor operations, every-node blockchain, geospatial clustering, matrix persistence (write-ahead log with incremental snapshots), and the complete asset system with six resource adapters. Catalog provides the asset package registry with execution delegation. Caesar implements token economics and multi-chain bridging. The eBPF intelligence layer compiles C kernel programs (XDP, kprobe, tracepoint) and manages AF_XDP zero-copy sockets. Gateway provides the HTTP/3 entry point at `trust.hypermesh.online`.

The protocol specification and implementation continue to evolve. This document describes the architecture as designed and implemented in February 2026.


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
