# Substrate Phase C — Physical/Radio (Deferred R&D)

**Status:** roadmap-only research · **PDL phase:** `phase_1780516760023_t84xzxrbl`
**Implements:** nothing yet — intentionally **not** assigned a protocol R-number
**Canonical context:** `papers/SUBSTRATE.md` §8

---

## 1. What Phase C is, and is not

Phase C is the endgame of substrate sovereignty: links that exist with **no incumbent infrastructure at all** — wireless mesh radio, opportunistic device-to-device peering, store-carry-forward. The network exists before any ISP, router, or DHCP server does.

It is **not** designed in this document, and it is **not** built. It is a research effort scoped to produce a *decision*, not a feature. It is deliberately excluded from the R-number system: per `core/CLAUDE.md`, requirements must be "concrete and testable," and radio is not yet either. When/if Phase C produces a concrete, testable contract, *that* becomes a new R-number and likely a follow-on roadmap — not an expansion of this one.

The `radio_mesh` adapter (`base/src/adapters/radio_mesh.rs`) stays a no-capability stub until then. It exists so the question has a home in the registry, not so the question is answered.

---

## 2. Open research questions (the actual deliverable is answers to these)

1. **Radio hardware abstraction.** What is the minimal device/driver surface the Substrate must target? 802.11s mesh mode, Wi-Fi Direct, raw monitor-mode injection, LoRa, Bluetooth mesh, or a pluggable radio HAL spanning several? Which are reachable from userspace on commodity hardware without custom kernel modules?

2. **Neighbor discovery without a router.** How does a node find peers when there is no DHCP, no router advertisement, no DNS? Beaconing? A well-known multicast/broadcast rendezvous on the radio link? How does this compose with TrustChain identity so discovery is authenticated, not open?

3. **Association & link establishment.** What replaces "carrier" (Phase B's central concept) on a radio link that has no kernel netdev? Is there a `LinkState` analog, and does `Substrate.b`'s self-healing model transfer, or does radio need its own?

4. **Addressing on a non-netdev link.** Phase A derives `fd48:4d00::/32` from identity — that still holds (identity is link-agnostic). But how is that address *carried* over a radio link the kernel does not present as an interface? Is there a shim that presents the radio as a netdev (TUN-like), or does STOQ need a non-socket datapath?

5. **Opportunistic peering & store-carry-forward.** Links are intermittent. Does the Substrate buffer and forward when a peer reappears? How does this interact with STOQ's connection model and BlockMatrix's instruction-based retrieval (which already assumes shards can come from multiple positions)?

6. **Spectrum, regulatory, power.** What are the legal constraints per region (licensed vs unlicensed bands)? What is the power/battery cost on mobile devices, and does R13 (minimum device spec) survive a radio substrate?

---

## 3. Method

- A **feasibility memo** answering §2 with evidence (existing mesh-radio projects — e.g. the design lineage of 802.11s, Yggdrasil, cjdns, Briar's transports — surveyed for what transfers and what does not).
- A **build / no-build / prototype recommendation**, gated explicitly on **pilot funding**. No engineering commitment is made without it.
- If "prototype": a tightly scoped spike (single radio type, two devices, authenticated discovery + one derived-address exchange), tracked as a **new roadmap**, not folded into the Substrate roadmap.

---

## 4. Acceptance criteria

Phase C is "done" (as a research phase) when:
1. The six §2 questions have documented answers / evidence.
2. A build/no-build decision exists with its funding gate stated.
3. The `radio_mesh` adapter's status is explicitly recorded (still-stub vs promoted-to-prototype).

No code-level acceptance — by design.

---

## 5. Dependencies

Phase C leans on Phase A (identity→address is link-agnostic and reusable) and learns from Phase B (whether the carrier/self-healing model generalizes). It must not start before A and B are stable, and must not block them.
