# Specification B v0.1 — Soundness Review

**Reviewed document:** spec-b-direct-canonical-usdt-backing.md (Research Draft v0.1)
**Scope:** Bitcoin-layer enforceability, epoch replacement, RGB-layer feasibility, economic invariants.
**Verdict:** The objective is achievable, but not as written. v0.1 maps onto a known construction class — Ark-style pre-signed rounds, channel-factory leaves, RGB client-side validation over unconfirmed transactions — and inherits both the strengths and the hard constraints of that class. Three findings are critical. Each is fixable, but the fixes force two honest weakenings of the Section 2 target property: unilateral exit only covers claims the user personally cosigned, and claims must carry an expiry with a refresh liveness requirement. v0.2 incorporates these.

---

## Framing

v0.1 is, structurally:

```text
shared reserve UTXO
  + pre-signed transaction tree      -> Ark round / channel factory
  + per-user leaf claims             -> sub-channels
  + RGB state over unconfirmed txs   -> LN-RGB virtual seals, generalized
  + vUSDT Lightning overlay          -> unchanged from Spec A
```

Every soundness question in v0.1 has a known answer or a known impossibility in this class. The review below applies those results.

---

## Critical findings

### F1. Reserve UTXO control is undefined; without it the target property fails trivially

v0.1 never states who can spend the reserve outpoint. This is the load-bearing question. A pre-signed transaction tree is only enforceable if no key set can produce a conflicting spend of its root. If the reserve is controlled by an operator or federation key, the federation double-spends the root, the entire tree becomes unenforceable, and RGB state follows Bitcoin — the canonical USDT goes wherever the conflicting spend sends it. Section 27's collusion claim is then false.

On mainnet today there are exactly three options:

1. Per-epoch interactive n-of-n cosigning: every user with a claim in the epoch is part of a MuSig2 aggregate on the reserve output. A conflicting spend requires the user's own signature. This is the Ark model.
2. Covenants (CTV/CSFS): the tree is consensus-enforced, no interaction needed. Not activated; cannot be a dependency.
3. Federation-controlled reserve: contradicts the specification's stated objective.

Only (1) is available. Consequence: the unilateral-exit guarantee extends only to claims the user cosigned into a tree. This MUST be stated as a scope limit, not hidden.

**Disposition:** v0.2 §7 defines the output model explicitly; §3 rescopes the guarantee.

### F2. Offline users are inconsistent with epoch replacement as specified

Sections 16–18 require each epoch to invalidate the previous one. Section 29 item 7 requires handling partial participation. These conflict under a single shared reserve outpoint:

- If new epochs consume or conflict with the old root, a user offline during epoch N+1 loses their only enforceable package — its root input is gone — and cannot receive a replacement. The honest system destroys the guarantee it exists to provide.
- If old trees remain valid for non-participants, the non-participant's old root and the participants' new root are conflicting spends of the same outpoint. Whichever confirms first destroys the other cohort's claims.

There is no arrangement of pre-signed transactions over one persistent outpoint that resolves this. The known resolution partitions by epoch: each epoch produces a fresh on-chain cohort output funded from operator liquidity; refreshing users forfeit their old leaf (conditionally, see F3) and receive a new one; old cohort outputs persist untouched until an expiry, after which the operator sweeps the remainder. Non-participants keep their old package valid until expiry.

Consequences accepted in v0.2: one on-chain transaction per epoch; claims expire; users (or their delegates) must refresh once per expiry window; operator capital is committed for up to one expiry window per cohort.

**Disposition:** v0.2 §8–§9 (epoch lifecycle), §14 (expiry/refresh).

### F3. Stale-state rollback griefing is worse than double redemption

Sections 16–18 treat old-state publication as a double-spend by the publishing user. The more damaging case: any holder of an old package broadcasts it at a loss to themselves, rolling every co-participant back to old balances. Leaf-level punishment of the publisher does not restore other users' post-rollback deltas.

Under the F2 partitioned model this narrows: old and new cohorts do not conflict on-chain, so the only stale object is the user's own old leaf, and the only victim of its enforcement is the operator. The standard fix applies: at refresh, the user signs a forfeit of the old leaf to the operator, bound by a connector output to confirmation of the new epoch transaction (atomicity: the forfeit is invalid unless the new claim exists). If the user later unrolls the old tree, the operator broadcasts the forfeit and reclaims the leaf. Enforcement requires operator chain-watching while the system is live; when the operator is gone, no new epochs exist and the last settled state is final, so the watch requirement and the disappearance scenario never overlap.

**Disposition:** v0.2 §13 (forfeit layer), §17 (failure scenarios).

---

## High findings

### F4. Send-then-exit gap (v0.1 §21) needs a per-payment mechanism, not an epoch-level one

Between epochs, a user who sends over Lightning still holds the old, larger leaf claim. Waiting for the next epoch to reconcile leaves the operator extending unsecured credit equal to each user's in-epoch net send volume. Fix: make each leaf a two-party sub-channel (user + operator 2-of-2) with LN-penalty state updates. Any vUSDT send that would take the user's balance below the current leaf state MUST be preceded or accompanied by a signed leaf-state decrease. Receives MAY raise the leaf state up to leaf capacity; beyond that they remain pending until refresh. This bounds operator exposure per user to zero on sends and to the pending-receive delta on receives, and composes with the exposure caps already defined in the virtual-liquidity spec V1.1.

### F5. In-flight HTLCs at the epoch cut are unspecified

A settlement snapshot taken while HTLCs are in flight is ambiguous. Mirroring HTLC success/timeout paths into the tree multiplies pre-signed state combinatorially; reject. v0.2 requires per-channel quiescence for the refreshing user and excludes in-flight amounts from the settled leaf; they resolve into R_current and settle at the next refresh.

### F6. Fee and anchor policy for pre-signed transactions is unspecified

Pre-signed fee rates go stale over an epoch's lifetime. Every tree transaction, claim transaction, and forfeit MUST carry a fee anchor enabling CPFP at broadcast time; embedded fees SHOULD be minimal. Exit cost is O(log N) sequential transactions per user; a shared prefix broadcast by one user reduces cost for siblings. Worst-case exit cost under congestion MUST be quantified before parameters (tree radix, expiry) are fixed.

### F7. The RGB virtual-seal dependency is acknowledged but its knock-on constraints are not

Named in v0.1 §15/§29 as R&D; correct. Missing constraints: (a) tapret commitments in pre-signed transactions are fixed at signing, so any change to an epoch's allocation set is a full tree re-sign — epochs are atomic construction ceremonies; (b) each epoch appends transitions that every future holder of reserve-descended USDT must validate — consignment growth needs a pruning or checkpoint strategy; (c) leaves below Bitcoin dust or RGB minimums cannot exist — a minimum settled balance is a protocol parameter, with sub-minimum value remaining pending credit.

---

## Medium findings

### F8. The solvency invariant can be strengthened from operational to structural

The epoch transaction's root RGB transition allocates the cohort total from the operator reserve seal. Every participant's consignment contains this transition, and RGB validation enforces per-transition conservation. Each user therefore independently verifies that the sum of all cohort allocations does not exceed the reserve input — non-inflation is proven to every participant at cosign time, not audited after the fact. v0.1 does not claim this property; v0.2 does (§6, G2).

### F9. Reserve flows are unspecified

Canonical USDT received by the LSP mid-epoch (v0.1 §20) does not touch any reserve structure until the next epoch transaction, which takes operator liquidity as inputs and produces operator change. Top-up, withdrawal, expiry sweeps, and cooperative redemptions all occur only at epoch transactions or operator-only script paths. v0.2 §9 specifies the epoch transaction anatomy.

### F10. Expiry versus perpetual claims is a forced decision, not an open one

Perpetual claims imply the reserve fragmentation problem v0.1 lists as R&D item 12 and make operator capital unrecoverable without per-user cooperation. Expiry resolves both at the cost of a refresh liveness requirement. v0.2 chooses expiry with a long window and a mandated exit safety margin; delegated refresh is an R&D item.

### F11. Section 27's collusion claim must be rescoped

Correct form: a colluding LSP and federation cannot take a cosigned, unexpired settled claim (they lack the user's key for any conflicting spend). They CAN withhold new epochs (pending amounts stay unsecured credit), refuse a user's refresh (forcing exit before expiry), and stall cooperative flows. These are liveness and credit risks, not custody of settled principal. v0.2 §17 states all three.

### F12. Privacy disclosure of the tree

A user's consignment and pre-signed path reveal sibling allocations at each level of their path, with explicit amounts. Cohort sharding reduces the blast radius. Noted as a consideration, not fixed.

---

## Summary of required changes

| # | Severity | v0.1 gap | v0.2 resolution |
|---|---|---|---|
| F1 | Critical | Reserve key control undefined | Cohort n-of-n MuSig2 + operator expiry path (§7) |
| F2 | Critical | Offline users vs epoch replacement | Per-epoch cohort outputs, expiry, refresh (§8–9, 14) |
| F3 | Critical | Rollback griefing | Connector-bound forfeits, operator watch (§13) |
| F4 | High | Send-then-exit credit gap | Leaf sub-channels, per-send state updates (§11) |
| F5 | High | HTLCs at epoch cut | Quiescence + exclusion (§12) |
| F6 | High | Fee staleness | Anchors on all pre-signed txs, exit cost model (§16) |
| F7 | High | Virtual-seal knock-ons | Atomic epochs, history growth, dust floor (§15) |
| F8 | Medium | Solvency operational only | Structural non-inflation proof (§6) |
| F9 | Medium | Reserve flows | Epoch tx anatomy (§9) |
| F10 | Medium | Expiry undecided | Expiry chosen, parameters (§14, 19) |
| F11 | Medium | Overbroad collusion claim | Rescoped guarantees (§3, 17) |
| F12 | Medium | Tree privacy | Noted (§22) |

The two weakenings relative to v0.1 §2 — cosign participation and expiry — are not design choices. They are the boundary of what pre-signed transactions can enforce on Bitcoin without covenants. If CTV/CSFS activates, both weakenings are removable; v0.2 §21 records the upgrade path.
