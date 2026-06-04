# HyperMesh Substrate — Development Plan

The master plan for building the Substrate layer. It ties together the
architecture (`papers/SUBSTRATE.md`), the per-phase design specs
(`PHASE_{A,B,C}.md` in this directory), the crate (`core/base/`), and the live
PDL tracking tree. Read the whitepaper for *why*; read this for *how and in what
order*; read the phase specs for *exactly what*.

---

## Status snapshot (2026-06-03)

| Stage | State |
|---|---|
| Spec + scaffold | **done** — `papers/SUBSTRATE.md`, `core/base/` crate (traits, types, 4 adapter stubs), R15/R16 added to `HYPERMESH.md` §3, workspace wired, `crate-status.toml` at `planning` |
| Phase A — Sovereign addressing (R15) | planned — spec: `PHASE_A.md` |
| Phase B — Link-sovereign interface mgmt (R16) | planned — spec: `PHASE_B.md` |
| Phase C — Physical/radio | deferred R&D — spec: `PHASE_C.md` |

**PDL roadmap:** `roadmap_1770012690055_8wbx3t4f2` (repurposed in place — see note below).

| Phase | PDL phase ID | Sprints |
|---|---|---|
| A | `phase_1780516747187_b552ct4j9` | A1 `sprint_1780516767429_atjrgyld5`, A2 `sprint_1780516769266_eteljp4mi` |
| B | `phase_1780516754686_skv43jxrm` | B1 `sprint_1780516771396_u0uz2xz80`, B2 `sprint_1780516776798_7qjgfomus` |
| C | `phase_1780516760023_t84xzxrbl` | C1 `sprint_1780516778602_ge3i2d0dx` |

> **PDL note.** The PDL `roadmap_update` tool does not persist `status` changes,
> so the prior "Code Quality & Architecture Cleanup" roadmap (its phases
> completed) could not be closed, and a new roadmap could not be created. Per
> decision on 2026-06-03 the roadmap was **repurposed in place**: its
> vision/objectives/metrics now describe the Substrate. The roadmap's *name*
> field is stale ("Code Quality…"); treat the vision text as authoritative.

---

## Sequencing & dependencies

```
Scaffold (done)
   │
   ▼
Phase A ── A1 derive/verify (pure, testable) ──► A2 reachability + STOQ injection
   │                                                      │
   │ (A provides the derived address B assigns)           │
   ▼                                                      ▼
Phase B ── B1 enumerate + carrier + musl gate ──► B2 assign + self-heal + migration
   │
   │ (A is link-agnostic & reused; B's carrier model informs C)
   ▼
Phase C ── C1 feasibility memo + build/no-build (pilot-funded)
```

- **A before B** — B assigns the address A derives; building B first would assign nothing meaningful.
- **A1 before A2** — the pure derivation is the testable core; wire injection only once it is proven.
- **B1 before B2** — you must enumerate/read carrier before you can assign-and-heal; B1 also clears the musl gate that B2's production use depends on.
- **C after A+B stable** — C reuses A's identity→address and learns from B's carrier model; it must not block them and must not start mid-flight.

---

## Cross-cutting invariants (hold in every phase)

1. **No upward dependency.** `base` depends only on `hypermesh-lib`. STOQ never depends on `base`. CI guard: `cargo tree -p stoq | grep -q base && exit 1`.
2. **Injection at the binary.** All Substrate-derived values (`bind_address`, `public_ipv6`, `ebpf_interface`) and signals (migration) flow into STOQ via the `blockmatrix` orchestration layer, never by STOQ reaching down.
3. **Graceful degradation, never hard-fail.** netlink → sysfs → tier-3 fallback. A node missing capability degrades; it does not lose the network.
4. **Identity is the root of address.** Every address is `f(NodeId)` and peer-verifiable. This holds across all phases, including radio.
5. **musl-deployable.** Anything enabled in production builds musl static-pie, or ships behind a documented glibc/musl split.

---

## Per-phase summary

### Phase A — Sovereign Addressing (R15) · spec `PHASE_A.md`
Derive a verifiable `fd48:4d00::/32` from `NodeId`; discover reachability via a reflector observed-address echo; inject derived values into STOQ's `TransportConfig` at the construction sites in `blockmatrix`. Pure-math core + wiring; the only networked risk is the reflector echo (kept minimal — not STUN/ICE). **Acceptance:** property-tested derivation, node boots with derived address bound, layering guards green.

### Phase B — Link-Sovereign Interface Management (R16) · spec `PHASE_B.md`
Implement the `rtnetlink_linux` backend (enumerate / carrier / lease-free assign / link-event stream) and the `sysfs_fallback` read-only tier; self-heal on link flap and signal STOQ's existing connection-migration machinery; replace `detect_outbound_interface()`. **Hard gate:** confirm netlink deps build under musl. **Acceptance:** netns flap test (re-select → re-assign → migration signal), tier degradation verified, musl build (or documented split).

### Phase C — Physical/Radio (deferred) · spec `PHASE_C.md`
Research-only. Answer six open questions on zero-ISP device-to-device connectivity; produce a feasibility memo and a funding-gated build/no-build decision. No R-number until concrete and testable. `radio_mesh` stays a stub. If it proceeds, it becomes a new roadmap.

---

## Definition of done (roadmap level)

- R15 and R16 acceptance criteria (in `PHASE_A.md` §6 / `PHASE_B.md` §7) all pass.
- `core/base/crate-status.toml` Substrate.a + Substrate.b items are `working`; `core/CLAUDE.md` R15/R16 lines reflect implemented status.
- A node on real hardware boots with an identity-derived address on a self-selected, carrier-checked interface, survives a link flap via migration, and depends on no external network daemon for any of it.
- Phase C has a documented decision (not necessarily a "yes").
- The website status sync (`core/scripts/sync-status.sh`) reflects `base` progress (run by the maintainer / pre-push hook, not as an automated side effect of this work).
