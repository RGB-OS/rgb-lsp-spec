# Specification B v0.2 — Self-Custodial Virtual USDT Backed Directly by Canonical RGB-USDT

**Status:** Draft v0.2 — feature-complete, prepared for external audit
**Supersedes:** Research Draft v0.1 (`archive/spec-b-v0.1.md`)
**Incorporates:** All dispositions of the v0.1 Soundness Review (`archive/spec-b-v0.1-soundness-review.md`), findings F1–F12
**Architecture:** Per-epoch cohort outputs with pre-signed settlement trees, cosigned n-of-n roots, connector-bound forfeits, expiring claims
**User custody model:** Self-custodial Lightning channels plus unilateral canonical-USDT recovery path for cosigned, unexpired settled claims
**Primary objective:** Remove discretionary operator/federation custody from canonical-USDT redemption, within the enforcement boundary of pre-signed Bitcoin transactions (no covenant dependency)

---

## Table of contents

1. [Purpose and scope](#1-purpose-and-scope)
2. [Conformance language, notation, and definitions](#2-conformance-language-notation-and-definitions)
3. [Target security property (rescoped)](#3-target-security-property-rescoped)
4. [System model, assets, and adversary model](#4-system-model-assets-and-adversary-model)
5. [Economic state](#5-economic-state)
6. [Solvency invariants and structural non-inflation](#6-solvency-invariants-and-structural-non-inflation)
7. [Reserve and output model](#7-reserve-and-output-model)
8. [Epoch lifecycle and construction ceremony](#8-epoch-lifecycle-and-construction-ceremony)
9. [Epoch transaction anatomy](#9-epoch-transaction-anatomy)
10. [Settlement tree structure](#10-settlement-tree-structure)
11. [Leaf sub-channels](#11-leaf-sub-channels)
12. [In-flight HTLCs and quiescence at refresh](#12-in-flight-htlcs-and-quiescence-at-refresh)
13. [Forfeit / revocation layer](#13-forfeit--revocation-layer)
14. [Expiry and refresh](#14-expiry-and-refresh)
15. [RGB layer: virtual seals, atomic epochs, history growth, dust floor](#15-rgb-layer-virtual-seals-atomic-epochs-history-growth-dust-floor)
16. [Fees, anchors, and exit cost model](#16-fees-anchors-and-exit-cost-model)
17. [Failure scenarios and guarantee boundaries](#17-failure-scenarios-and-guarantee-boundaries)
18. [Security properties and proof sketches](#18-security-properties-and-proof-sketches)
19. [Protocol parameters](#19-protocol-parameters)
20. [Client (wallet) requirements](#20-client-wallet-requirements)
21. [Covenant upgrade path](#21-covenant-upgrade-path)
22. [Privacy considerations](#22-privacy-considerations)
23. [Open items and R&D register](#23-open-items-and-rd-register)
24. [Conformance checklist](#24-conformance-checklist)
25. [Changelog and findings disposition](#25-changelog-and-findings-disposition)
- [Appendix A — Non-normative Q&A](#appendix-a--non-normative-qa)
- [Appendix B — BTC instantiation (normative profile)](#appendix-b--btc-instantiation-normative-profile)

---

## 1. Purpose and scope

### 1.1 Purpose

This protocol extends the virtual-liquidity architecture (Specification A / virtual-liquidity spec V1.1, the *overlay spec*) so that settled user monetary claims are backed **directly** by canonical RGB-USDT rather than by an operator or federation promise.

Users transact in vUSDT over Lightning for normal payments. Every **cosigned, unexpired, settled** claim additionally carries a cryptographically enforceable path to canonical RGB-USDT held in a shared reserve. No operator or federation signature is required at the time of unilateral exit.

### 1.2 Scope

This document specifies:

- the on-chain output and key model of the shared reserve (§7);
- the epoch lifecycle, construction ceremony, and epoch transaction (§8–§9);
- the pre-signed settlement tree and per-user leaf sub-channels (§10–§11);
- the forfeit layer that makes epoch replacement safe (§13);
- claim expiry, refresh, and the associated liveness obligations (§14);
- the RGB-layer requirements, including the virtual-seal dependency contract (§15);
- fee, anchor, and exit-cost requirements (§16);
- the exact guarantee boundary, failure analysis, and proof sketches (§3, §17, §18).

### 1.3 Out of scope

- The vUSDT Lightning overlay itself (channel construction, HTLC forwarding, synthetic-asset issuance, operator exposure caps). It is normatively defined in the overlay spec; §4.3 and §5 restate the definitions this document depends on, so this document is self-contained for audit purposes.
- Canonical RGB-USDT issuance and issuer behavior (§4.5, assumption A2).
- Fiat redemption of canonical USDT by the issuer.
- A concrete instantiation of the RGB virtual-seal construction; §15.2 specifies the required interface as a formal dependency (D-VS) with acceptance criteria in §23.

### 1.4 Reading guide for auditors

All normative requirements carry stable identifiers `[B-nn]`. Invariants are `I1..I3`, guarantees `G1..G5`, assumptions `A1..A7`, dependencies `D-*`, open R&D items `R-1..R-6`, and ceremony validation checks `V1..V12`. §24 aggregates every MUST-level requirement. §25 maps every v0.1 review finding (F1–F12) to its resolution.

---

## 2. Conformance language, notation, and definitions

### 2.1 Conformance language

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

Conformance targets:

- **Operator** — the party (possibly federated internally, §4.2) that runs the LSP, constructs epochs, and provides liquidity.
- **Client** — user wallet software implementing this protocol.

### 2.2 Notation

| Symbol | Meaning |
|---|---|
| `U_i` | User *i* (key: `P_i`, an x-only secp256k1 public key) |
| `O` | Operator (key: `P_O`) |
| `N` | Epoch index (monotonically increasing integer) |
| `E_N` | Epoch transaction of epoch `N` (§9) |
| `O_N` | Cohort output of `E_N` (§7.2) |
| `S_N` | Cohort of epoch `N`: the set of users with a leaf in `E_N`'s tree |
| `S_j` | Participant set of tree node `j`: users whose leaves descend from `j` |
| `L_i^N` | Leaf output of user `i` in epoch `N` |
| `H_N` | Bitcoin block height at which `E_N` reaches `k` confirmations |
| `H_exp^N` | Absolute expiry height of epoch `N` (§14) |
| `C_i` | User *i*'s vUSDT channel capacity (overlay spec) |
| `R_current,i` | User *i*'s current economic balance (§5) |
| `R_settled,i` | User *i*'s settled (leaf-state) balance (§5) |
| `pending_i` | `R_current,i − R_settled,i` (§5) |
| `Δ_leaf` | Relative delay on leaf sub-channel commitment transactions (§13.3) |
| `Δ_rev` | Relative delay on the to-user output of leaf commitments (§11.4) |
| `r` | Settlement tree radix (§10) |
| `k` | Confirmation depth for epoch activation (§8.4) |
| `M` | Exit safety margin in blocks (§14.4) |
| `R_min` | Minimum settled leaf balance, USDT-denominated (§15.4) |
| `P_max` | Client policy cap on `pending_i` (§11.6) |

All amounts of canonical USDT are in the asset's atomic units. All Bitcoin amounts are in satoshis. All delays and expiries are in Bitcoin block heights (CSV for relative, CLTV for absolute).

### 2.3 Definitions

- **Canonical USDT** — the RGB fungible asset issued by the issuer under contract ID `USDT_CONTRACT_ID` (a deployment parameter, §19).
- **vUSDT** — the synthetic channel-liquidity asset of the overlay spec. vUSDT confers **no** claim by itself (§4.4).
- **Settled claim** — the balance `R_settled,i` recorded in the latest leaf sub-channel state of user `i`'s leaf in an active, unexpired epoch that the user cosigned.
- **Exit package** — the data set enabling unilateral exit (§20.2).
- **Refresh** — the act of a user joining epoch `N+1`'s cohort and forfeiting their epoch-`N` leaf (§14.2).
- **Unroll** — sequential broadcast of the pre-signed tree path from a cohort output down to a leaf (§16.3).
- **Cosigned** — signed by the user's own key as part of an n-of-n aggregate; used throughout to mark the participation precondition of guarantee G1.

---

## 3. Target security property (rescoped)

### 3.1 Guarantee

> **G1 (informal).** If the operator, federation, backend infrastructure, and all other users disappear or refuse cooperation, a user holding a **cosigned, unexpired** settled claim of value `R_settled` can still obtain at least `R_settled` canonical RGB-USDT, using only their exit package, their own keys, and Bitcoin block confirmation, within a bounded number of blocks.

The formal statement and proof sketch are in §18.

### 3.2 Boundary of the guarantee

Two weakenings relative to the v0.1 target property are **inherent to pre-signed transactions on Bitcoin without covenants** (review findings F1, F2, F10). They are scope limits, not implementation choices:

1. **Cosign participation (F1).** The unilateral-exit guarantee covers only claims the user personally cosigned into a settlement tree. A user who never participates in any epoch ceremony holds only overlay-spec credit.
2. **Expiry and refresh liveness (F2, F10).** Every settled claim expires at its epoch's `H_exp`. A user (or their delegate, R-5) MUST either refresh into a later epoch or exit on-chain before the exit deadline `D_exit` (§14.4). A user who does neither loses the settled claim's on-chain backing to the operator's expiry sweep; the debt reverts to overlay-spec credit.

If CTV/CSFS-class covenants activate on Bitcoin, both weakenings are removable (§21).

### 3.3 Explicit non-guarantees

The protocol does **not** guarantee:

- **NG1** — recovery of `pending_i` (unsettled deltas) when the operator disappears. Pending amounts are unsecured operator credit, bounded by client policy `P_max` (§11.6) and by the overlay spec's exposure caps.
- **NG2** — liveness of refresh, cooperative redemption, or vUSDT payments. The operator can always stall these (§17.4); the user's remedy is unilateral exit.
- **NG3** — privacy of settled allocations against cohort co-members (§22).
- **NG4** — value of vUSDT itself (§4.4).
- **NG5** — anything about claims whose epoch has expired (§14.5).

---

## 4. System model, assets, and adversary model

### 4.1 Actors

- **User** — holds their own channel and leaf keys; runs a Client.
- **Operator** — runs the LSP: routes payments, provides vUSDT and canonical liquidity, constructs epochs, watches the chain for forfeit enforcement (§13.6).
- **Federation (optional)** — a threshold group that MAY internally control the operator key `P_O` for availability and internal-compromise resistance. Externally the federation is indistinguishable from the operator; this document therefore uses "operator" for both. **[B-01]** The security analysis MUST NOT assign the federation any trust role beyond the operator role: after an epoch is active, no operator/federation action is required for exit (G1).
- **Issuer** — issues canonical RGB-USDT. Out of scope beyond assumption A2.

### 4.2 Keys

**[B-02]** Each user MUST use a dedicated x-only key `P_i` for reserve participation, derived from their wallet seed (BIP-340 compatible; derivation path is a deployment convention). **[B-03]** Loss of the wallet seed with no exit-package backup loses the claim (§20.2); loss of the exit package alone with the seed intact loses the claim as well — both MUST be backed up (§20.3).

### 4.3 Relationship to the overlay spec

The vUSDT overlay provides: user–operator Lightning channels carrying vUSDT (RGB-assets-in-LN), HTLC-based payments, and operator exposure caps limiting total unsecured credit. This document consumes from the overlay spec:

- the definition of `C_i` and `R_current,i`;
- the per-user and global exposure caps (restated as I3, §6.4);
- the RGB-Lightning channel mechanics reused verbatim for leaf sub-channels (§11.2).

### 4.4 Assets

**Canonical USDT** (`USDT`): the actual RGB asset. **vUSDT**: the synthetic liquidity asset; supply MAY be arbitrarily large (e.g. 1,000,000,000 vUSDT). **[B-04]** Clients and operators MUST treat vUSDT as valueless outside the claim structure of this document plus the overlay spec's credit accounting; in particular, vUSDT MUST NOT be represented to users as USDT, and displayed user balances MUST distinguish settled from pending value (§20.5).

### 4.5 Assumptions

- **A1 (Bitcoin).** Bitcoin consensus is safe and live; no reorganization exceeds depth `k`; transactions paying prevailing fee rates confirm within the confirmation target `T_conf` used to size `M` (§14.4, §16.4).
- **A2 (RGB + issuer).** RGB client-side validation is sound: a validated consignment chain from the USDT genesis to an allocation on a UTXO the user controls confers ownership of that USDT amount. The issuer does not reissue, freeze, or fork the canonical asset. (If the canonical contract carries issuer freeze/reissue rights, that is an asset-level risk disclosed under §17.7.)
- **A3 (D-VS).** An RGB virtual-seal construction satisfying the interface in §15.2 exists and is used. This is a formal dependency; the guarantee chain G1 is conditional on it.
- **A4 (User liveness, bounded).** The user's Client (or delegate) acts at least once per epoch lifetime before `D_exit`, and can watch the chain during the windows `Δ_leaf`/`Δ_rev` when it has broadcast an exit. No other liveness is assumed for safety of settled claims.
- **A5 (MuSig2).** MuSig2 as specified in BIP-327, with its nonce-handling requirements, is secure (EUF-CMA of the aggregate under honest-signer nonce discipline).
- **A6 (Fees).** During any window in which the user must confirm transactions (unroll, claim, penalty), the fee market permits confirmation via CPFP on anchors within the block budgets assumed in §16.4.
- **A7 (Operator watch, scoped).** While a superseded (forfeited) leaf's cohort output is unexpired, the operator (or its watchtower) observes the chain and can react within `Δ_leaf`. This assumption protects **the operator**, not users; its failure harms only the operator (§13.6). It never overlaps with the operator-disappearance scenario of G1, because when the operator is gone no epoch after the user's last cosigned epoch exists, hence the user's latest state is not superseded and no forfeit against them is outstanding.

### 4.6 Adversary model

The adversary may control, adaptively and in any combination: the operator (including the federation), any subset of other users, network scheduling within Bitcoin's synchrony bounds (A1), and all off-chain communication (may withhold, delay, reorder). The adversary cannot break secp256k1/BIP-340/BIP-327 cryptography, Bitcoin consensus beyond A1, or RGB validation soundness (A2). Guarantees are stated per honest user against this adversary.

---

## 5. Economic state

### 5.1 State variables

Per user `i`:

```text
C_i           vUSDT channel capacity (overlay spec; inbound liquidity)
R_current,i   current economic balance: the amount the operator owes user i,
              as recorded by the latest vUSDT Lightning channel state(s)
R_settled,i   latest leaf sub-channel state balance of user i in their
              (unique, §14.3) active cohort; 0 if the user has no
              active cosigned leaf
pending_i  =  R_current,i − R_settled,i
```

**[B-05]** The system MUST maintain `pending_i ≥ 0` for every user at every observable point, i.e. the settled leaf state MUST NOT exceed the current economic balance. Enforcement is the send-ordering rule [B-26] plus the receive rule [B-27].

### 5.2 Interpretation

- `R_settled,i` is **user-owned property**: an enforceable transaction path to canonical USDT (G1).
- `pending_i` is **operator credit**: it becomes property only at the next leaf-state raise or refresh. Non-guarantee NG1 applies.
- `C_total ≫ Σ_i R_settled,i` is allowed and expected: virtual capacity is not backed and needs no backing (§6.3).

### 5.3 Lifecycle of value

```text
receive vUSDT            leaf-state raise (§11.5)          refresh (§14.2)
        │                        │                                │
        ▼                        ▼                                ▼
   pending_i  ────────────►  R_settled,i  ───────────────►  R_settled,i
   (credit)     if within        (property,                 (property, new
                leaf capacity     current epoch)             epoch, new expiry)
```

Sends move value in the opposite direction and MUST decrement `R_settled,i` first ([B-26]).

---

## 6. Solvency invariants and structural non-inflation

### 6.1 Per-epoch invariant

Let `A_N` be the canonical USDT amount allocated to cohort output `O_N` by `E_N`'s root RGB transition, and let `leaf_alloc_i^N` be the USDT allocated to leaf `L_i^N` by the tree's transitions.

- **I1 (leaf conservation).** `Σ_{i ∈ S_N} leaf_alloc_i^N = A_N`, enforced structurally by RGB per-transition conservation across the tree's transitions (§15.1) — a tree violating this fails RGB validation and MUST be rejected at ceremony check V4.
- **I2 (settled coverage).** For every user and at all times, `R_settled,i ≤ leaf_alloc_i^N` where `N` is the user's active epoch. Enforced by leaf sub-channel construction: leaf states split exactly `leaf_alloc_i^N` between user and operator (§11.3); a state assigning the user more than the leaf holds cannot be constructed.

Together: `Σ_i R_settled,i ≤ Σ_active N A_N ≤` total canonical USDT held under reserve-descended allocations.

### 6.2 Structural non-inflation (G2, resolves F8)

**G2.** *At ceremony time, every cohort participant independently verifies — before contributing any signature — that the sum of all cohort leaf allocations does not exceed the canonical USDT consumed by the epoch transaction's root transition, by validating the full RGB consignment of `E_N` and the tree (checks V3–V5).* Non-inflation is therefore **proven to every participant at cosign time**, not audited operationally after the fact. Across epochs, double-backing is impossible because RGB allocations are bound to Bitcoin UTXOs (single-use seals): two epoch transactions cannot consume the same reserve allocation without a Bitcoin double-spend, excluded by A1.

### 6.3 What is *not* an invariant

`C_total ≤ anything` is not required. Virtual capacity is unbacked by design; the overlay spec's exposure caps (I3) bound the *credit* consequences, not the capacity number.

### 6.4 Exposure caps (restated from overlay spec)

- **I3 (credit caps).** `pending_i ≤ P_max` per user as client policy ([B-29]), and `Σ_i pending_i ≤ X_global` as operator policy, where `X_global` is the overlay spec's global exposure cap. Violation of I3 harms only the capped party's counterparty risk; it never affects G1.

---

## 7. Reserve and output model

*(Resolves F1: reserve control is fully defined; no key set other than ones including the user can conflict-spend the user's path.)*

### 7.1 Design decision

Of the three mainnet-available options for making a pre-signed tree enforceable —

1. per-epoch interactive n-of-n cosigning over the cohort (Ark model);
2. covenants (CTV/CSFS) — **not activated; MUST NOT be a dependency** (§21);
3. federation-controlled reserve — **contradicts the objective**;

— this protocol REQUIRES option 1. **[B-06]** Every output on the path from a cohort output to a user's leaf MUST be spendable before expiry only with an aggregate signature whose signer set includes that user.

### 7.2 Cohort output `O_N`

Pay-to-Taproot output of `E_N`:

- **Internal key:** `MuSig2_agg({P_i : i ∈ S_N} ∪ {P_O})` — n-of-n over the entire cohort plus the operator, aggregated per BIP-327 with all tweaks per BIP-341.
- **Script tree (single leaf):** `<H_exp^N> OP_CHECKLOCKTIMEVERIFY OP_DROP <P_O> OP_CHECKSIG` — the operator expiry-sweep path (§14.5).

**[B-07]** The key-path spend of `O_N` MUST only ever be exercised by the pre-signed root tree transaction(s) produced in the ceremony (§8). Participants enforce this by construction: the aggregate signature exists only for those transactions, and creating any other spend requires every participant — including each victim — to sign again. This is the load-bearing fact behind G1 (see §18.1, step 1).

### 7.3 Internal node outputs

Tree node `j` with participant set `S_j` (§10):

- **Internal key:** `MuSig2_agg({P_i : i ∈ S_j} ∪ {P_O})`.
- **Script tree:** operator expiry path as in §7.2, same `H_exp^N`.

Restricting the aggregate to the subtree's participants keeps ceremony signing cost O(N·log N) aggregate sessions instead of O(N²) (§8.3) while preserving [B-06]: every ancestor of user `i`'s leaf has `i ∈ S_j`.

### 7.4 Leaf outputs

Leaf `L_i^N`:

- **Internal key:** `MuSig2_agg({P_i, P_O})` (2-of-2).
- **Script tree:** operator expiry path, same `H_exp^N`.

Spend classes of a leaf (full mechanics in §11 and §13):

| Spend | Path | Delay | Signed when |
|---|---|---|---|
| Cooperative close | key path | none | on demand |
| Leaf commitment (user's or operator's unilateral) | key path, pre-signed | input `nSequence = Δ_leaf` | at each leaf-state update |
| Forfeit (refresh) | key path, pre-signed, second input = connector | none | at refresh |
| Expiry sweep | script path | `CLTV H_exp^N` | unilateral by operator |

**[B-08]** Every pre-signed leaf commitment transaction MUST set the input spending `L_i^N` with `nSequence = Δ_leaf`. The forfeit transaction MUST NOT carry that delay. This delta is what makes forfeits win races against stale commitments (§13.4).

### 7.5 Uniform expiry

**[B-09]** All outputs of epoch `N`'s tree (cohort output, internal nodes, leaves) MUST carry the same absolute expiry height `H_exp^N` in their operator script path, so that the user-facing exit deadline (§14.4) is a single number per epoch.

---

## 8. Epoch lifecycle and construction ceremony

*(Resolves F2 jointly with §9 and §14: epochs are partitioned per cohort output; old cohorts persist untouched until expiry.)*

### 8.1 Lifecycle

```text
 CONSTRUCTING ──► VALIDATED ──► TREE-SIGNED ──► PACKAGED ──► FORFEITS-IN ──► BROADCAST ──► ACTIVE ──► EXPIRING ──► EXPIRED
      │               │              │              │              │             (E_N confirmed      (height ≥      (height ≥ H_exp^N;
      └── abort ──────┴──────────────┴──────────────┴──────────────┘              k deep)          H_exp^N − M)     sweep §14.5)
                (abort at any pre-broadcast stage: discard all state;
                 no on-chain effect; all previous epochs unaffected)
```

**[B-10]** An epoch that aborts before `E_N` broadcast MUST leave no enforceable artifacts: this holds by construction because every new-epoch object (tree, forfeits) spends outputs of the unbroadcast `E_N` either directly or via its connector (§13.2), and `E_N` remains unsigned until stage 6 below.

### 8.2 Ceremony stages

**[B-11]** The ceremony MUST proceed in this order; a Client MUST NOT perform a stage before completing (and verifying) the prior one:

1. **Construct.** Operator builds the unsigned `E_N` (§9), the full tree (§10), all RGB transitions and per-participant consignments (§15). Because all inputs are SegWit (Taproot), the txid of `E_N` is fixed before signing; tree transactions are constructed against it.
2. **Validate.** Each participant runs checks V1–V12 (§8.3). Any failure ⇒ that participant aborts (does not sign); operator MAY reconstruct with a smaller cohort (restart at stage 1 with fresh nonces, [B-13]).
3. **Cosign tree.** MuSig2 signing sessions produce, for every tree transaction, the aggregate signature of its input's signer set. Each user participates in exactly the sessions for nodes on their root-to-leaf path, plus their initial leaf commitment (§11.3).
4. **Package.** Operator distributes to each participant their complete exit package (§20.2). **[B-12]** A Client MUST verify the completeness and validity of its exit package — including all aggregate signatures and the RGB consignment — against checks V1–V12 before proceeding.
5. **Forfeit signing.** Each refreshing participant signs their forfeit of the old leaf (§13.2). **[B-14]** A Client MUST NOT sign a forfeit before completing stage 4 verification. **[B-15]** The operator MUST NOT proceed to stage 6 until it holds valid forfeits from every refreshing participant; if any are missing, the epoch MUST be aborted (never broadcast).
6. **Sign and broadcast `E_N`.** The operator signs `E_N` (its inputs are operator-controlled) and broadcasts it.

**[B-13]** MuSig2 sessions MUST follow BIP-327 nonce rules: fresh nonces per session, no nonce reuse across ceremony restarts, and secure nonce state handling. A restarted ceremony is a new set of sessions.

**Why this order is safe.** If the operator broadcasts `E_N` early (before stage 5 completes for some user), that user has a fully signed new claim (stages 3–4 done) and an inactive forfeit — the operator, not the user, bears the double-claim risk, and [B-15] makes that operator misbehavior against itself. If the operator never broadcasts, forfeits are permanently invalid (connector never exists) and old claims stand. At no ordering can an honest user hold a signed-away old claim without an enforceable new one. (§18.3.)

### 8.3 Ceremony validation checks

**[B-16]** Before contributing any signature, a Client MUST verify:

- **V1** — `E_N` is well-formed per §9; its txid matches the txid all tree transactions spend.
- **V2** — every output script on the client's root-to-leaf path matches §7 exactly (keys, aggregate composition, uniform `H_exp^N` per [B-09]), and the client's own key is in every ancestor aggregate.
- **V3** — the RGB root transition of `E_N` consumes reserve-descended USDT allocations and allocates `A_N` to `O_N`'s seal; the consignment back to genesis validates under A2.
- **V4** — the tree's RGB transitions satisfy I1: leaf allocations sum to `A_N`; the client's own `leaf_alloc_i^N` equals the agreed refresh amount (§14.2).
- **V5** — the client's initial leaf state (§11.3) equals `R_settled,i` carried over per §14.2 and §12.
- **V6** — every tree transaction and pre-signed leaf transaction carries a compliant fee anchor and minimal embedded fee (§16.1–16.2).
- **V7** — leaf commitment inputs carry `nSequence = Δ_leaf` [B-08]; commitment outputs carry `Δ_rev` and revocation paths per §11.4.
- **V8** — `H_exp^N − H_broadcast-target ≥ W_exp` and parameters match the advertised deployment parameter set (§19).
- **V9** — the connector output for this client exists in `E_N`, is operator-spendable, and matches the forfeit template (§13.2) — checked by refreshing participants.
- **V10** — `leaf_alloc_i^N ≥ R_min` (§15.4).
- **V11** — cohort size `|S_N| ≤ N_max` and tree shape matches §10 (radix `r`, balanced within one level).
- **V12** — no transaction in the package is malleable in a way that changes txids the package depends on (all inputs SegWit; no legacy inputs anywhere in the tree).

Checks V3–V4 are what make G2 structural: they force full-tree visibility (with the privacy consequence recorded in §22).

### 8.4 Activation

**[B-17]** Clients and the operator MUST treat epoch `N` as ACTIVE only once `E_N` has `k` confirmations. **[B-18]** A Client MUST retain its previous epoch's exit package until the new epoch is ACTIVE, and SHOULD retain it until the old cohort's expiry (it is harmless after forfeit and useful evidence).

---

## 9. Epoch transaction anatomy

*(Resolves F9: all reserve flows happen here or in operator-only script paths.)*

### 9.1 Structure of `E_N`

```text
inputs:
  [0..a]  operator reserve inputs        — Taproot, operator-controlled; carry the
                                           canonical-USDT allocations being committed
  [a+1..b] operator BTC liquidity inputs — fund cohort BTC value, connectors, anchors
  [opt]   user onboarding inputs         — user-contributed canonical-USDT UTXOs
                                           (atomic onboarding, §9.5)

outputs:
  [0]     cohort output O_N              — BTC: Σ leaf BTC values + tree anchor budget
                                           RGB: A_N canonical USDT (root transition)
  [1..c]  connector outputs              — one per refreshing participant; minimal
                                           standard value (e.g. 330 sat P2TR); operator key
  [c+1]   operator USDT change           — RGB: reserve remainder allocation
  [c+2]   operator BTC change
  [c+3]   fee anchor                     — per §16.1
```

**[B-19]** All inputs of `E_N` MUST be SegWit (txid-stable before signing, [B-11] stage 1). **[B-20]** The root RGB transition MUST allocate exactly `A_N` to the `O_N` seal and the remainder to the operator change seal; no other USDT outputs may appear in `E_N`.

### 9.2 Reserve flows (exhaustive list)

**[B-21]** Canonical USDT under this protocol moves **only** via:

1. epoch transactions (commit `A_N`, return change);
2. expiry sweeps of expired cohort outputs and orphaned tree/leaf outputs (§14.5) — operator-only script paths;
3. forfeit transactions (§13) — reclaiming a superseded leaf;
4. user exits (tree unroll + leaf spend, §11.4/§16.3);
5. cooperative leaf closes (§11.7) and cooperative on-chain redemptions paid from operator change.

Canonical USDT received by the operator mid-epoch (e.g. inbound canonical routing, v0.1 §20) sits in ordinary operator UTXOs and touches the reserve structure only as inputs to a later `E_{N+1}`.

### 9.3 One transaction per epoch

The on-chain cost of the partitioned-epoch design (F2 disposition) is exactly one confirmed transaction per epoch plus, eventually, one expiry sweep per cohort — independent of cohort size. Epoch cadence is an operator policy trade-off between capital lockup (a cohort's `A_N` plus BTC is committed until its expiry) and settlement latency for pending balances; the resulting capital requirements are made explicit in §9.4.

### 9.4 Operator capital model

*(The offline-user guarantee (F2 disposition) is paid for in operator capital; this subsection makes that price explicit and auditable.)*

**Reserve decomposition: backing vs parked receivables.** Two quantities must not be conflated:

```text
gross USDT locked = 1 × TVL_settled      live backing: every settled claim in exactly
                                          one active leaf — invariant (G2/T5), no lever
                  + parked receivables    operator-owned capital in unexpired FORFEITED
                                          leaves, awaiting recovery — policy-dependent
```

**Backing is always exactly 1×.** No fractional reserve over settled balances is constructible (a violating epoch collects no signatures), and no policy below changes this. The multiplier question concerns only the second term: when a user refreshes, their claim moves to a freshly funded leaf while their forfeited old leaf — now an operator receivable, not user money — stays physically locked in the old cohort output until that cohort is reclaimed or expires (§14.5).

**Parked-receivable multiplier.** Per user, the number of parked copies of their balance is `min(T_reclaim, W_exp) / C`, where `C` is that user's refresh cadence and `T_reclaim` the operator's reclamation lag. The multiplier applies to the **actively refreshing** balance set only; dormant users sit at 1×. Worked points (with `W_exp ≈ 26` weeks):

| Refresh cadence `C` | Reclamation lag `T_reclaim` | Gross multiple of refreshing TVL |
|---|---|---|
| weekly | never (wait for expiry) | ~27× — the worst-case corner |
| weekly | ~monthly | ~5× |
| ~monthly | never | ~7.5× |
| ~monthly | ~monthly | ~2× |
| no refresh (dormant) | — | 1× |

**Whose money each term is.** The 1× live backing is economically **user-funded**: it entered as the users' own inbound canonical flow, held briefly as float and committed into their leaves — the operator's net position in it is ≈ 0 (plus any receive-headroom overprovision, which is operator capital). The parked receivables, by contrast, are entirely the **operator's own working capital**: each refresh fronts a fresh copy of the user's balance while the previous copy waits out the reclamation lag. Operator-own liquidity requirement:

```text
LSP_own ≈ (gross multiple − 1) × refreshing TVL  +  redemption float  +  headroom
```

**The capital dial.** The gross multiple is an operating point the operator chooses, not a property of the design. Per `$1,000` of refreshing user balance:

| Operating point | LSP own capital per $1,000 TVL |
|---|---|
| Aggressive reclamation (≈ per-epoch), or refresh throttled to ≈ reclamation cadence | ≈ $1,000 |
| Weekly refresh, monthly reclamation (the 5× row) | ≈ $4,000 |
| Lazy corner (weekly refresh, wait for expiry) | ≈ $26,000 — not a sane operating point |

Reclamation cost is on-chain fees only — unrolling a cohort is O(cohort size) transactions **shared across the whole cohort** (per-user cost: a few hundred to a few thousand sats), and reclaimed outputs feed directly as inputs into the next epoch transaction, so capital velocity equals reclamation cadence. For any deployment with meaningful TVL, fees are far cheaper than multiples of working capital, so the rational equilibrium sits near the 1–2× end.

**The structural floor.** The multiple cannot reach exactly 1×: during every refresh overlap there is necessarily a moment of 2× per refreshing user, because the new leaf must be funded before the old leaf becomes reclaimable. This floor is the direct capital cost of the offline-user guarantee ("old claims stay valid until replaced", the F2 disposition) and is irreducible without covenants (§21).

**Financing cost, not solvency risk.** Parked receivables are illiquid but certain: the forfeits are the operator's unilaterally enforceable pre-signed right, and the expiry sweep is consensus-guaranteed. The operator's true cost is therefore working-capital financing, `(multiple − 1) × refreshing-TVL × cost of capital`, traded against reclamation fees and against the `pending` growth that slower refresh cadence causes. The remaining levers: shorter `W_exp` (cheaper capital, heavier user liveness burden per §14.4) and per-user refresh throttling.

**Full-cohort rollover (optional optimization).** When **every** member of cohort `N` participates in the refresh into `N+1`, the transition MAY be executed as a *rollover*: the epoch transaction `E_{N+1}` takes the old cohort output `O_N` itself as an input, cosigned by the full old aggregate `S_N ∪ {O}` during the same ceremony. This recycles the old cohort's capital atomically — no duplication, no transient 2×, and **no forfeits or connectors are needed for that transition**, because every old claim's root input is spent by the very transaction that activates the new claims (supersession by conflict, enforced by consensus rather than by operator watch). If any member is absent, the transition MUST fall back to the standard partitioned epoch of §8–§9. Sharding cohorts by activity level makes the all-online case common for active users, taking their steady-state cost toward 1×.

**[B-62]** In a rollover, the signature over the `O_N` input plays the role of the forfeit and MUST be governed by the same ordering discipline: a Client MUST NOT sign the spend of `O_N` before completing stage-4 verification of its full `N+1` package ([B-12]/[B-14] applied verbatim), and the operator MUST NOT broadcast `E_{N+1}` without the complete old-aggregate signature (partial rollovers are forbidden — it is all of `S_N` or the §8–§9 fallback). Leaf commitments retain `Δ_leaf` unconditionally ([B-08]), since a cohort cannot know at construction time whether its *next* transition will qualify for rollover.

### 9.5 Atomic onboarding (user-contributed epoch inputs)

Without this option, a new deposit is unsecured operator credit (`pending`) from the moment the user pays the operator until their first epoch activates — an onboarding trust window equal to the entire deposit. Atomic onboarding eliminates it: the user contributes their own canonical-USDT UTXO **directly as an input to `E_N`**, cosigning it during the ceremony, with their leaf created by the same transaction. Either `E_N` confirms — and the deposit and the enforceable claim come into existence atomically — or it never confirms and the user still owns their original UTXO, since their input signature authorizes exactly `E_N` and nothing else. At no instant does the operator hold the deposit without the user holding an enforceable claim.

```text
Alice's 500-USDT UTXO ──┐
operator reserve inputs ─┼──► E_N ──► cohort output ──► tree ──► Alice's leaf (≥ 500)
operator BTC inputs ─────┘
```

**[B-63]** Onboarding inputs MUST be SegWit ([B-19] applies to them verbatim) and their RGB allocations MUST be consumed by `E_N`'s root transition, so that conservation checks V3–V4 cover them. A contributing Client MUST verify, before signing its deposit input, that its leaf allocation is at least its contribution plus any concurrently settled pending, and MUST apply the package-first discipline to that signature ([B-12]/[B-14] verbatim — no deposit-input signature before stage-4 verification of the complete exit package). The operator MUST NOT accept an onboarding contribution outside a ceremony, and an aborted epoch ([B-10]) leaves the contributed UTXO untouched by construction.

Deposits made outside a ceremony (ordinary receives, §11.5) remain supported and remain `pending` until first settlement; Clients SHOULD prefer atomic onboarding for amounts above `P_max` and MUST display the difference per [B-53].

**Early reclamation.** The operator MAY unilaterally unroll an old cohort's tree and broadcast the connector-bound forfeits of refreshed leaves to recover their capital before expiry. All transactions involved are pre-signed and conflict with no honest user's path (§10.4, formal companion L1/T6); unforfeited leaves remain untouchable until `H_exp` regardless. This trades O(cohort size) on-chain fees for released capital and is rational whenever the capital cost of waiting exceeds the fee cost.

**Redemption-side float.** Settled redemption via unilateral exit consumes **no** operator liquidity — the USDT is already in the cohort output, so settled balances are structurally run-proof: a simultaneous mass exit is linear in cohort size (§16.3) and cannot fail for lack of funds (T5/T6). Cooperative redemption (§11.7), by contrast, is paid from the operator's *unlocked* float, and what the operator receives in exchange — the increase of its in-leaf share `ω` — is a receivable locked until the user's exit, a cooperative close, early reclamation, or the cohort's expiry. The float must therefore cover, over the capital-recovery horizon:

```text
Float ≈ peak of [ cooperative redemptions
                + new-epoch commitments settling pending
                − inbound canonical receipts
                − recovered exits / forfeits / sweeps ]
```

Pending settlement (refresh) is backed operationally by the inbound canonical flow that created the pending, never structurally — exactly NG1, which is why `pending` is capped (I3, T7). If the float is exhausted, cooperative flows stall and users degrade to unilateral exit: fees and delay, never loss of settled principal (§17.6-style graceful degradation).

**[B-58]** A deployment MUST publish, alongside the [B-47] exit-cost model, an operator capital model stating: the backing/receivable decomposition above with its projected gross multiple (from the deployment's expected `C` and chosen `T_reclaim`), the redemption-float sizing and its recovery horizon, the reclamation schedule, and the caps `P_max`/`X_global` in force. The model MUST present live backing (invariantly 1×) separately from parked receivables so that the solvency statement and the financing statement cannot be conflated. This model is part of the audit surface.

---

## 10. Settlement tree structure

### 10.1 Shape

**[B-22]** The tree MUST be an `r`-ary tree over the cohort's leaves, balanced to within one level, rooted at a single root transaction spending `O_N`. Each tree transaction spends one parent output and creates up to `r` child outputs plus one fee anchor. Depth is `⌈log_r |S_N|⌉`.

```text
                E_N
                 │  (cohort output O_N, n-of-n cohort ∪ O)
             T_root
          ┌──────┴──────┐
        T_A             T_B          internal nodes: S_j ∪ O aggregates
      ┌───┴───┐       ┌───┴───┐
   L_Alice  L_Bob  L_Carol  L_Dave   leaves: 2-of-2 (U_i, O)
```

### 10.2 BTC values

**[B-23]** Each leaf MUST carry a fixed BTC amount `btc_leaf` (§19) sufficient to make every downstream output (commitment outputs at `Δ_rev`, anchors) standard and above dust; internal nodes carry the sum of their descendants plus their children's anchor budget. Embedded fees follow §16.2.

### 10.3 RGB values

Each tree transaction carries the RGB transition sub-allocating its input's USDT to its children (§15.1). Leaf `L_i^N` receives `leaf_alloc_i^N`.

### 10.4 Sibling independence

Transactions in disjoint subtrees do not conflict. One user's unroll broadcasts only their path; a shared prefix already broadcast by any co-member reduces every descendant's remaining exit cost (§16.3). No user action can invalidate another user's path — conflicting spends of shared ancestors cannot be created after the ceremony ([B-07], §7.3).

---

## 11. Leaf sub-channels

*(Resolves F4: per-payment security, not epoch-level.)*

### 11.1 Concept

Each leaf is a two-party (user, operator) payment-channel over `leaf_alloc_i^N` canonical USDT and `btc_leaf` sats, with LN-penalty state updates, **no HTLC outputs** (F5, §12), and a funding output (`L_i^N`) that is itself unconfirmed until an exit unrolls the tree. RGB-Lightning commitment mechanics are reused verbatim (overlay spec / D-VS property VS-4).

### 11.2 State

Leaf state `s = (n_state, user_amt, op_amt)` with `user_amt + op_amt = leaf_alloc_i^N`, `n_state` strictly increasing. `R_settled,i := user_amt` of the latest state. The initial state (signed at ceremony stage 3) sets `user_amt` per §14.2.

### 11.3 Update protocol

**[B-24]** Leaf-state updates MUST follow LN-penalty discipline: for each new state, both parties exchange new asymmetric commitment transactions (pre-signed key-path spends of `L_i^N`, `nSequence = Δ_leaf` per [B-08]) and then exchange revocation secrets for the prior state. An update is **complete** only when revocation of the prior state is received; Clients and operator MUST treat incomplete updates as not having occurred.

### 11.4 Commitment transaction layout

The holder-side commitment of state `s`:

```text
input:  L_i^N                      (nSequence = Δ_leaf)
outputs:
  to_holder:  holder key + CSV Δ_rev   OR   counterparty revocation key (penalty)
              [RGB: holder's USDT amount]
  to_counterparty: counterparty key (CSV 1)
              [RGB: counterparty's USDT amount]
  anchor:     per §16.1
```

**[B-25]** Publication of a revoked commitment forfeits the publisher's entire leaf amount to the counterparty via the revocation path, which MUST be exercisable throughout `Δ_rev`. The RGB transitions for both outputs and for the penalty spend are part of the pre-signed state data (D-VS property VS-4).

**[B-64] (leaf commitment format — BOLT3-inspired, deliberately not BOLT3-compatible).** Leaf sub-channels MUST NOT be implemented as BOLT3 channels; two incompatibilities are structural. First, BOLT3 encodes the obscured commitment number into the input `nSequence` with bit 31 set, which *disables* relative timelocks, whereas [B-08] requires an **active** `nSequence = Δ_leaf` on every leaf commitment — commitment numbering MUST therefore be carried in `nLockTime` (or equivalent out-of-band state) only. Second, the funding output (the leaf) is unconfirmed for the channel's entire normal life — beyond "zero-conf", it is expected never to confirm — so implementations MUST NOT gate channel operation on funding depth. Leaf commitments carry no HTLC outputs (§12). Interoperability with third-party Lightning nodes is not a goal for this channel: both endpoints are always the user's Client and the operator.

### 11.5 Send and receive rules

- **[B-26] (send ordering — closes the send-then-exit gap).** The operator MUST NOT irrevocably commit (forward or settle) any outgoing vUSDT HTLC of user `i` whose settlement would make `R_current,i < R_settled,i`. Concretely: any send that would take the user's balance below the current leaf state MUST be preceded or accompanied by a completed leaf-state decrease to at most the post-send balance. If the HTLC subsequently fails, the parties MAY complete a leaf-state raise back up (subject to [B-27]).
- **[B-27] (receive rule).** On receives, the leaf state MAY be raised, with operator cooperation, up to `leaf_alloc_i^N`. Amounts beyond leaf capacity MUST remain accounted as `pending_i` — never as settled — until refresh.
- **[B-28] (user-side mirror of B-26).** A Client MUST NOT release an outgoing HTLC preimage / complete an outgoing payment before it has completed the corresponding leaf-state decrease, except where the decrease and the channel update are part of one atomic update session with the operator.

Operator credit exposure is thereby zero on sends and equal to the pending-receive delta on receives — composing with the overlay spec's caps (I3).

### 11.6 Pending cap

**[B-29]** Clients MUST enforce a policy cap `pending_i ≤ P_max`: when a prospective receive would exceed it and the leaf is at capacity, the Client MUST either refuse the payment or obtain a refresh first. `P_max` is user-configurable with a RECOMMENDED default in §19.

### 11.7 Cooperative close and redemption

With both parties online, a leaf MAY be closed cooperatively at any time: a key-path spend of `L_i^N` (or, if the tree is unbroadcast, simply a refresh that pays the user's `R_settled` out as canonical USDT from operator change in `E_{N+1}`). Cooperative canonical-USDT redemption at any amount ≤ `R_current,i` is an overlay-level flow settled at the next refresh or paid directly from operator funds.

---

## 12. In-flight HTLCs and quiescence at refresh

*(Resolves F5.)*

**[B-30]** A refresh snapshot for user `i` MUST be taken under per-channel quiescence: all vUSDT channels between `i` and the operator MUST have no in-flight HTLCs at the moment the refresh amount (§14.2) is fixed, using the LN quiescence protocol (`option_quiescence` / BOLT quiescence draft) or an equivalent stop-and-drain handshake.

**[B-31]** In-flight amounts MUST be excluded from the settled leaf: mirroring HTLC success/timeout paths into the pre-signed tree is rejected by design (combinatorial pre-signing blowup). In-flight HTLCs resolve into `R_current,i` after the snapshot and settle at the next leaf-state update or refresh.

**[B-32]** Quiescence is per refreshing user and MUST NOT be required across the whole cohort simultaneously; the operator assembles per-user fixed amounts as users become quiescent during ceremony stage 1–2.

---

## 13. Forfeit / revocation layer

*(Resolves F3: connector-bound forfeits; rollback griefing is structurally narrowed to operator-vs-one-user.)*

### 13.1 What needs revoking

Under the partitioned-epoch model (§8–§9), old and new cohorts never conflict on-chain. The only stale enforceable object a refreshing user retains is **their own old leaf claim** (`L_i^N` path plus their latest old leaf commitment). Its only possible victim is the operator (it is operator capital that backs the already-refreshed value). Other users are unaffected by construction (§10.4) — this is what reduces v0.1's rollback-griefing surface to a bilateral problem.

### 13.2 Forfeit transaction

At refresh (ceremony stage 5), user `i` signs:

```text
forfeit_i^{N→N+1}:
  input 0: L_i^N            key path (MuSig2 2-of-2 U_i + O), no CSV delay
  input 1: connector_i^{N+1} output of E_{N+1}, operator key
  output:  operator          BTC: leaf BTC minus fee budget
                             RGB: full leaf_alloc_i^N to operator seal
  anchor:  per §16.1
```

**[B-33]** The forfeit MUST take the connector output of `E_{N+1}` as an input. This binds validity of the forfeit to confirmation of `E_{N+1}`: if the new epoch never confirms, the forfeit can never be valid, and the old claim stands intact. Atomicity of "old claim revoked ⟺ new claim exists" follows (§18.3).

**[B-34]** The operator MUST NOT spend `connector_i` other than in `forfeit_i`, until the old cohort `N` expires (after which connectors are swept, §14.5).

### 13.3 Why the forfeit wins the race

The user's only unilateral spends of `L_i^N` are pre-signed commitments carrying `nSequence = Δ_leaf` ([B-08]): after an unroll confirms `L_i^N`'s parent, the commitment is consensus-invalid for `Δ_leaf` blocks. The forfeit has no such delay. **[B-35]** Upon observing any spend that confirms `L_i^N`'s parent for a leaf it holds a forfeit for, the operator MUST broadcast the forfeit (with CPFP per §16.1) immediately; it has the full `Δ_leaf` window to confirm it ahead of any commitment. `Δ_leaf` MUST be sized for adversarial congestion (§19).

### 13.4 Scope of the operator-watch assumption

Forfeit enforcement is the only place operator liveness matters (A7), and it protects only the operator. If the operator's watch fails, the refreshed user collects twice and the operator eats the loss; no honest third party is affected. When the operator has disappeared, no forfeit against any user's **latest** state exists (their latest cosigned epoch was never superseded), so G1 is unaffected — the watch requirement and the disappearance scenario cannot overlap.

### 13.5 Leaf-level revocation

Within a live leaf, revoked commitment states are punished by the standard penalty path (§11.4, [B-25]) — this handles stale *states*, while the forfeit handles stale *leaves*. Both mechanisms are independent; either alone is insufficient (the penalty cannot reach across epochs; the forfeit cannot distinguish states).

---

## 14. Expiry and refresh

*(Resolves F2 and F10: expiry is chosen, with a long window, a mandated safety margin, and a stated liveness obligation.)*

### 14.1 Decision: claims expire

Perpetual claims would fragment the reserve unrecoverably (v0.1 R&D item 12) and lock operator capital forever absent per-user cooperation. **[B-36]** Every epoch MUST carry a uniform absolute expiry `H_exp^N` ([B-09]) with `H_exp^N − H_N ≥ W_exp` (§19). The cost is the refresh liveness obligation of §3.2(2), mitigated by the long window, mandated client alarms (§20.4), and delegated refresh (R-5).

### 14.2 Refresh semantics

A refresh of user `i` from epoch `N` into `N+1` fixes, under quiescence (§12):

```text
refresh_amount_i = R_current,i at snapshot          (in-flight excluded)
leaf_alloc_i^{N+1} ≥ refresh_amount_i               (operator MAY overprovision capacity)
initial leaf state user_amt = refresh_amount_i
```

so refresh simultaneously (a) converts all pending into settled, (b) re-arms expiry, and (c) MAY resize leaf capacity. The old leaf is forfeited per §13. **[B-37]** A Client MUST verify V4/V5 equality between the agreed `refresh_amount_i` and the new package before signing anything.

### 14.3 One active leaf per user

**[B-38]** A user MUST have at most one unforfeited leaf across all unexpired epochs, and the operator MUST refuse to construct an epoch violating this. (A user may transiently appear in old cohort `N` (forfeited) and new cohort `N+1`; the forfeit makes the old leaf operator property.)

### 14.4 Exit deadline and safety margin

The user-facing deadline for *starting* a unilateral exit from epoch `N` is:

```text
D_exit^N = H_exp^N − M
M ≥ depth · T_conf + Δ_leaf + Δ_rev + 2 · T_conf + B      (all in blocks)
```

with `depth = ⌈log_r |S_N|⌉` unroll confirmations at target `T_conf` each, the leaf commitment delay `Δ_leaf` followed by one commitment confirmation, the to-holder claim delay `Δ_rev` followed by one claim confirmation, and buffer `B` for fee spikes and reorgs (≥ `k`). (Deadline sufficiency is proven as Corollary T2.1 of the formal companion.) **[B-39]** Deployments MUST publish `M` (with the parameter set, §19) and Clients MUST treat `D_exit` as the hard deadline: a user who has not refreshed by `D_exit` MUST begin unilateral exit (subject to the user override in [B-49]).

### 14.5 Expiry sweep

From height `H_exp^N`, the operator MAY sweep via the script paths: the cohort output if never unrolled (one transaction reclaims the entire remaining cohort), any confirmed tree or leaf outputs of expired epochs, and residual connectors. **[B-40]** The operator MUST NOT have any spend capability on these outputs before `H_exp^N` other than the cosigned/pre-signed paths of §7.4 — enforced by consensus (CLTV), not policy. Value swept from expired unexercised claims re-enters the operator reserve; the corresponding user debt reverts to overlay-spec credit (NG5) — the operator SHOULD honor it cooperatively, but it is no longer property.

---

## 15. RGB layer: virtual seals, atomic epochs, history growth, dust floor

*(Resolves F7; makes the D-VS dependency a precise contract.)*

### 15.1 RGB structure of an epoch

One RGB transition per Bitcoin transaction, each conserving its input allocation (I1):

```text
E_N root transition:  reserve allocations → { A_N on O_N seal, change on operator seal }
tree transitions:     parent allocation   → { child allocations }          (per node)
leaf state:           leaf allocation     → { user_amt, op_amt }           (per commitment)
```

Every participant's consignment contains the root transition and their path's transitions (G2, V3–V4).

### 15.2 Dependency D-VS: required virtual-seal interface

This protocol REQUIRES an RGB construction with the following properties. **[B-41]** A deployment MUST NOT go to production before an instantiation of D-VS meeting all four properties has been specified and audited (acceptance criteria in §23, R-1).

- **VS-1 (virtual witnesses).** An RGB state transition may be witnessed by a *pre-signed, not-yet-confirmed* Bitcoin transaction, identified by exact txid; validity of the resulting state is conditional on eventual confirmation of exactly that transaction chain.
- **VS-2 (no alternative closing).** A seal defined on a virtual output can be closed **only** by the transition committed (via tapret) in the pre-signed transaction fixed at signing time. Because tapret commitments are inside the signed transaction, they are immutable once the ceremony completes; there is no operator or user capability to bind a different transition to the same chain.
- **VS-3 (unilateral consignments).** Consignments covering a root-to-leaf virtual path can be constructed at ceremony time and validated by third parties after the path confirms, with no data from the operator at exit time.
- **VS-4 (LN-state compatibility).** Leaf sub-channel state updates (asymmetric commitments, revocation/penalty spends) carry RGB transitions with the same conditional-validity semantics, as in RGB-Lightning channels, over a virtual funding output.

### 15.3 Atomic epochs (F7a)

Because tapret commitments are fixed at signing, **any** change to an epoch's allocation set — one participant added, one amount changed — changes transactions and therefore requires a full re-sign of the affected subtree's sessions. **[B-42]** Epochs MUST be treated as atomic construction ceremonies; incremental mutation of a signed tree is impossible and MUST NOT be attempted (abort and reconstruct instead, [B-10]).

### 15.4 Dust floor (F7c)

Bitcoin dust limits and RGB minimums bound how small a leaf can be. **[B-43]** `leaf_alloc_i ≥ R_min` and leaf BTC ≥ `btc_leaf` (§19); users whose `R_current` is below `R_min` remain entirely in pending credit (NG1 applies) and SHOULD be warned by their Client ([B-52]).

### 15.5 History growth (F7b)

Every epoch appends one transition to the operator-change lineage, and each exit path adds `depth + 1` transitions to the exiting allocation's history. Consignment size for a user exit is `O(epoch_history + depth)`; validation cost grows linearly in epochs elapsed. Mitigations: (a) reserve sharding across parallel UTXO lineages bounds per-lineage epoch count; (b) exited allocations terminate their branch; (c) checkpoint/pruning of validated history is an open RGB-level item — registered as R-3 with a quantified growth budget required before parameter freeze. **[B-44]** Deployments MUST publish the projected consignment-size envelope for their chosen parameters.

---

## 16. Fees, anchors, and exit cost model

*(Resolves F6.)*

### 16.1 Anchors everywhere

**[B-45]** Every pre-signed transaction in this protocol — tree transactions, leaf commitments, penalty spends, forfeits — MUST include a fee anchor output (P2A `OP_1 <4e73>` where policy support exists, else a keyed anchor spendable by the broadcasting party) enabling CPFP at broadcast time by the party that needs confirmation. Anchor spendability MUST NOT require the counterparty.

### 16.2 Embedded fees

**[B-46]** Embedded fees in pre-signed transactions SHOULD be minimal (at or near the relay floor); confirmation urgency is priced at broadcast time via CPFP, never at signing time. Pre-signed fee rates are otherwise guaranteed to go stale over an epoch lifetime.

### 16.3 Exit cost

A single user's worst-case unilateral exit broadcasts `depth = ⌈log_r |S_N|⌉` tree transactions plus one leaf commitment plus one claim spend, sequentially (each spends the prior's output), plus CPFP children. Shared prefixes already confirmed (by the operator's sweep preparations or by co-members' exits) reduce this; the first exiting user in a subtree pays for its prefix, siblings free-ride. Total chain load if *all* `|S_N|` users exit is `O(|S_N|)` transactions (each tree transaction appears once), i.e. amortized `O(1)` per user plus their leaf spends — a mass-exit is linear, not quadratic.

### 16.4 Cost model requirement

**[B-47]** Before freezing parameters (`r`, `W_exp`, `M`, `Δ_leaf`, anchor values, `btc_leaf`), a deployment MUST publish a quantified worst-case exit cost model: vbytes per path, assumed adversarial fee rate, resulting sat cost, and the minimum claim size for which exit remains rational; `R_min` MUST be at or above that rationality floor. This model is part of the audit surface.

---

## 17. Failure scenarios and guarantee boundaries

*(Resolves F11: the collusion claim is rescoped precisely.)*

### 17.1 Operator (LSP) offline / disappeared

User executes the latest cosigned exit: unroll, wait `Δ_leaf`, broadcast latest leaf commitment, wait `Δ_rev`, claim. Result: `R_settled` canonical USDT recovered (G1). `pending` is lost up to `P_max` (NG1). No forfeit can threaten the latest state (§13.4).

### 17.2 Federation offline

Identical to §17.1 — the federation has no distinct trust role ([B-01]).

### 17.3 Operator and federation collude (rescoped)

They **cannot**: take or invalidate a cosigned, unexpired settled claim — any conflicting spend of the user's path requires the user's key ([B-06]); the expiry path is consensus-locked until `H_exp` ([B-40]).

They **can** (liveness/credit harms, not custody of settled principal):

1. withhold new epochs — pending amounts stay unsecured credit indefinitely (NG1/NG2);
2. refuse a specific user's refresh — forcing that user to exit on-chain before `D_exit` (censorship-to-exit, cost bounded by §16.3);
3. stall cooperative flows (sends via [B-26] refusal, redemptions, closes) — remedy is exit.

### 17.4 User publishes a superseded (forfeited) leaf

Operator broadcasts the connector-bound forfeit within `Δ_leaf` and reclaims the entire leaf (§13). The publisher additionally pays their own unroll fees; the attack is strictly loss-making against a live operator. If the operator's watch fails (A7 violated), the loss is the operator's alone (§13.4).

### 17.5 User publishes a revoked leaf *state*

Penalty path forfeits the leaf to the counterparty (§11.4). Symmetric for a misbehaving operator.

### 17.6 Bitcoin congestion

Exit remains possible; cost and latency degrade. Anchors (§16.1) let the user pay market rates at broadcast; `M` (§14.4) and `Δ_leaf` (§19) are sized so that congestion within the A6 envelope cannot push an on-time exit past expiry or let a stale commitment beat a forfeit. Congestion beyond A6 is a stated assumption failure.

### 17.7 Assumption-failure table

| Assumption broken | Effect on honest users | Effect bounded by |
|---|---|---|
| A1 (deep reorg > k) | epoch/forfeit atomicity may transiently wobble; connector binding restores consistency at the surviving chain | [B-17], [B-33] |
| A2 (RGB/issuer) | asset-level failure; on-chain BTC skeleton unaffected but USDT value at risk | out of scope, disclosed |
| A3 (D-VS unavailable) | protocol MUST NOT launch | [B-41] |
| A4 (user misses `D_exit`) | that user's claim expires → credit (NG5); nobody else affected | §14.5, [B-49] |
| A5 (MuSig2) | forgery of aggregates breaks G1 | cryptographic assumption |
| A6 (fee envelope) | exits delayed/expensive; margin `M` may prove insufficient in extremis | [B-47] sizing |
| A7 (operator watch) | operator (only) loses forfeited amounts | §13.4 |

### 17.8 Ceremony aborts and partial failures

Any pre-broadcast abort leaves all prior epochs untouched ([B-10]). An operator crash mid-ceremony equals abort. A user crash mid-ceremony: if before stage 5, their old claim stands and the operator reconstructs without them; a user who completed stage 5 whose `E_N` never broadcasts also keeps their old claim ([B-33]).

---

## 18. Security properties and proof sketches

Statements hold against the §4.6 adversary under A1–A7 (with each property's actually-used subset noted). These are sketches for readability; the full mathematical model and complete proofs — including the spend-enumeration lemma, refresh-atomicity theorem, outcome-completeness theorem ("no unexpected reachable outcomes"), and realizability propositions — are in the formal companion, [`spec-b-v0.2-formal.md`](spec-b-v0.2-formal.md). Where sketch and companion differ, the companion governs.

### 18.1 G1 — Settled-claim safety (uses A1, A2, A3, A5, A6)

**Claim.** An honest user with a cosigned, unexpired settled claim `R_settled` in epoch `N` obtains ≥ `R_settled` canonical USDT within `depth·T_conf + Δ_leaf + Δ_rev + T_conf` blocks of starting exit at any height ≤ `D_exit^N`, with no cooperation from anyone.

**Proof sketch.**
1. *The path cannot be pre-empted.* Every output from `O_N` to `L_i^N` requires, before `H_exp^N`, an aggregate signature including `P_i` ([B-06], §7). The only such signatures in existence are on the ceremony's transactions (the user verifiably signed nothing else — V-checks — and MuSig2 partial signatures for other messages require the user's participation, A5). Hence no conflicting spend of any path output can be created; the pre-signed path is the unique spend until expiry.
2. *The path confirms.* Each path transaction is standard, fee-bumpable by the user alone ([B-45]), and its inputs become available sequentially; under A1/A6 each confirms within `T_conf`.
3. *The leaf resolves to the latest state.* The user broadcasts their latest commitment after `Δ_leaf`. No forfeit for this leaf exists (the user's latest epoch was never refreshed away — a forfeit exists only if the user signed one, which by [B-14] they did only upon holding a verified newer claim, contradicting "latest"). Older revoked commitments are the user's own and not broadcast; an operator-side revoked commitment is punishable within `Δ_rev` ([B-25]) yielding ≥ the full leaf — still ≥ `R_settled`.
4. *USDT follows.* The consignment held since the ceremony (V3–V4, VS-1/VS-2/VS-3) validates the allocation `user_amt = R_settled` onto the user's claimed output; under A2 this is ownership. ∎

### 18.2 G2 — Structural non-inflation (uses A1, A2)

Per §6.2: per-transition conservation (A2) forces I1 inside each epoch; single-use seals plus no-double-spend (A1) prevent cross-epoch double-backing; every participant checks the full chain at cosign time (V3–V4) so a violating epoch collects no signatures. ∎

### 18.3 G3 — No profitable rollback (uses A1, A7)

**Claim.** Publishing any superseded object yields the publisher ≤ 0 and harms no honest third party.

**Proof sketch.** Superseded objects are exactly: (a) forfeited leaves — reclaimed in full by the connector-bound forfeit within `Δ_leaf` ([B-08]/[B-33]/[B-35], A7), publisher pays fees; (b) revoked leaf states — penalized in full within `Δ_rev` ([B-25]). Third parties: disjoint-subtree independence (§10.4) plus epoch partitioning (§13.1) mean neither object conflicts with any other user's path. Atomicity: the forfeit is valid iff `E_{N+1}` confirmed ([B-33]), and the user signed it only while holding the verified new claim ([B-14]); ceremony ordering [B-11]/[B-15] excludes every interleaving in which an honest user has revoked the old claim without an enforceable new one (§8.2). ∎

### 18.4 G4 — Bounded operator credit (uses none beyond protocol rules)

Sends never create credit ([B-26]/[B-28]: settled is decremented first). Receives create at most `pending_i ≤ P_max` ([B-27]/[B-29]) and `Σ pending ≤ X_global` (I3). ∎

### 18.5 G5 — Expiry-bounded custody (uses A1)

Before `H_exp`, the operator's only capabilities on tree outputs are the cosigned pre-signed paths (§7, [B-40]); consensus (CLTV) — not operator policy — enforces the boundary. After `H_exp`, sweeps touch only expired claims, whose holders were subject to the published deadline `D_exit` and mandated alarms ([B-39], [B-48]). ∎

---

## 19. Protocol parameters

**[B-48]** Every deployment MUST publish a signed parameter set, and Clients MUST verify ceremony artifacts against it (V8). Recommended defaults (constraints normative, values RECOMMENDED):

| Parameter | Symbol | Constraint | Recommended default |
|---|---|---|---|
| Canonical contract | `USDT_CONTRACT_ID` | fixed at deployment | — |
| Tree radix | `r` | ≥ 2 | 4 |
| Max cohort size | `N_max` | ceremony completes within construction window; MuSig2 session count O(N·log N) | 1,024 per cohort (shard above) |
| Epoch expiry window | `W_exp` | `H_exp − H_N ≥ W_exp`; `W_exp > 2M` | 26,208 blocks (~6 months) |
| Exit safety margin | `M` | formula §14.4 | 2,016 blocks (~2 weeks) |
| Leaf commitment delay | `Δ_leaf` | `≥ max(k, T_conf)`; sized for adversarial congestion | 288 blocks |
| Revocation delay | `Δ_rev` | ≥ 144 | 144 blocks |
| Activation depth | `k` | ≥ 6 | 6 |
| Confirmation target | `T_conf` | per A6 envelope | 36 blocks |
| Min settled balance | `R_min` | ≥ §16.4 rationality floor and RGB minimums | deployment-computed |
| Leaf BTC value | `btc_leaf` | all downstream outputs standard | 10,000 sat |
| Connector value | — | ≥ dust for its type | 330 sat |
| Pending cap | `P_max` | client policy | 10% of `R_settled` or a fixed fiat-equivalent floor, whichever is greater |
| Refresh cadence | `W_refresh` | operator policy; SHOULD keep median `pending` small; with the reclamation lag, drives the §9.4 parked-receivable multiple `min(T_reclaim, W_exp) / C` | weekly epochs |

**[B-49]** Clients MUST allow the user to override policy values (`P_max`, alarm thresholds) but MUST NOT allow silently disabling the `D_exit` alarm; automatic exit at `D_exit` is RECOMMENDED as default-on, with explicit user override permitted.

---

## 20. Client (wallet) requirements

### 20.1 Roles

The Client is the user's line of defense; G1 assumes it performs V-checks (§8.3), keeps the exit package, meets `D_exit`, and watches during its own exit windows (A4).

### 20.2 Exit package contents

**[B-50]** After each ceremony the Client MUST hold, and verify it holds ([B-12]):

```text
epoch_id N, parameter set hash, H_exp^N
E_N txid; full path transactions O_N → L_i^N with aggregate signatures
latest leaf commitment (holder side) + signatures; revocation secrets
  received for all counterparty revoked states; own per-state secrets
RGB consignments: genesis → reserve → root transition → path transitions
  → leaf state transitions (VS-3)
anchor keys / spend info for every anchor on the path
```

plus, while refresh is in progress, the previous epoch's package ([B-18]).

### 20.3 Backup

**[B-51]** The exit package MUST be durably backed up after every state change (leaf update or refresh); backups MUST be recoverable with the wallet seed alone plus the backup blob (encrypt-to-seed-derived-key is RECOMMENDED). The operator MAY offer encrypted blob storage; the Client MUST NOT rely on it exclusively.

### 20.4 Alarms and automatic exit

**[B-52]** The Client MUST alarm the user (a) at `H_exp − 2M` (refresh strongly urged), (b) at `D_exit` (exit now; auto-exit per [B-49]), (c) when `pending_i` exceeds `P_max` ([B-29]), and (d) when `R_current,i < R_min` (value entirely in credit, §15.4).

### 20.5 Display honesty

**[B-53]** Displayed balances MUST distinguish settled (property) from pending (credit) value, and MUST NOT label vUSDT as USDT ([B-04]).

### 20.6 Watch duties

**[B-54]** While the Client has broadcast any stage of an exit, it MUST watch until final claim confirmation and be prepared to exercise penalty paths within `Δ_rev`. Delegation to a watchtower is permitted (the penalty/claim transactions are delegable without key handover for the anchor-CPFP and penalty-broadcast roles).

---

## 21. Covenant upgrade path

### 21.1 Consensus covenants

If CTV/CSFS-class covenants activate, both §3.2 weakenings are removable with the same economic design:

- **Cosign removal.** Tree outputs become covenant-committed (`OP_CTV` template hashes) instead of n-of-n aggregates: users no longer need to participate in ceremonies to gain enforceable leaves; offline users can be *included* in new epochs.
- **Expiry relaxation.** With non-interactive inclusion, refresh no longer requires the user, so expiry can lengthen substantially or be replaced by covenant-encoded rollover; the residual reason for expiry is operator capital recovery, a parameter choice rather than an enforcement boundary.

**[B-55]** This document MUST NOT depend on covenant activation (per §7.1); this section records the upgrade so the deployment's migration story is auditable. Migration itself (coexistence of cosigned and covenant cohorts) is R-6.

### 21.2 Committee mode: covenant emulation, deployable today (optional; closes R-5)

The base design already emulates covenants by n-of-n pre-signing — with the *user* in every aggregate, which is what makes G1 self-enforcing and what creates both §3.2 weakenings. Committee mode substitutes, for leaves that opt into it, a **covenant-emulation committee** `C = {P_c1, …, P_cN}` of independent signers who cosign the tree in the user's stead and are trusted to sign nothing conflicting (equivalently: to have discarded their signing capability for those outputs). This is the recognized emulation-federation pattern (cf. Babylon's covenant committee, Liquid functionaries, BitVM signer sets), specified here as a per-user *opt-in* mode.

**Parked leaves.** A user opts in at a refresh by converting their balance into a **parked (static) leaf**: no sub-channel, balance frozen, condition

```text
Key(U_i)  ∨  Agg({U_i, P_O})  ∨  CLTV(H_exp, Agg(C ∪ {P_O}))
```

— directly user-spendable after unroll (no commitment transaction, hence no user pre-signing needed at future epochs), cooperatively closable, and sweepable at expiry only by committee-plus-operator. Every ancestor aggregate and every expiry path on a parked path includes `C`. Sends from a parked leaf are impossible (no channel); receives accrue as `pending` (NG1) until the user returns and unparks into a normal leaf.

**Roll-forward instead of forfeit-and-expire.** A parked claim never needs the user again: at the cohort's expiry, the operator and committee sweep the expired cohort output and, **in the same transaction**, re-commit every unexercised parked leaf at unchanged balance into the new cohort. Atomicity is by construction (one transaction spends the old root and funds the new); no forfeits exist for parked leaves because their balances cannot change. A dormant user therefore remains safe indefinitely — their exit package updates are unnecessary because the old package remains the enforceable one until a conforming roll-forward replaces it on-chain, at which point the committee (any honest member) withholds signatures from any sweep that fails to re-commit them.

**Guarantee rescope (G1′).** For a parked claim, G1's "no key set excluding the user can conflict-spend" becomes: *a conflicting spend or a non-conforming sweep requires the unanimous collusion of all `N` committee members and the operator.* This is existential honesty — one honest committee member suffices — but it **is** a trust assumption where the base mode has none, and it rests on unverifiable non-signing (key/nonce discard cannot be proven). Wallets and deployments must present G1 and G1′ balances as distinct classes.

**What committee mode buys and costs:**

| | Base (self-covered) | Parked (committee-covered) |
|---|---|---|
| Trust | none — own key required | 1-of-N committee honest (unanimous collusion + operator to break) |
| User liveness | cosign each refresh; `D_exit` deadline | none; indefinite dormancy |
| Capital (§9.4) | 1× + parked receivables, 2× refresh floor | 1× exactly — roll-forward recycles the swept cohort directly |
| Spendability | full (channel) | frozen until unpark; receives → pending |

**Unplanned dormancy is not covered:** a self-covered user who silently disappears still faces the §14.4 deadline, because a live sub-channel balance cannot be given a static direct-spend path without enabling stale-state enforcement. Parking is for *planned* dormancy; the mitigation for unplanned loss of liveness remains the [B-52] alarms and auto-exit.

**Capital efficiency of committee mode.** Parked value costs the operator essentially zero own capital: the 1× backing is the user's own frozen balance, the roll-forward recycles the swept cohort in one transaction (no fronting, no receivables, no transient 2×), and the operator's only cost is the roll-forward fee once per `W_exp`, amortized across the parked cohort. Active (self-covered) value is deliberately untouched — its §9.4 economics apply unchanged. The blended requirement is therefore:

```text
LSP_own ≈ (multiple − 1) × active TVL  +  0 × parked TVL  +  redemption float + headroom
```

| Population mix | LSP own capital vs total TVL |
|---|---|
| 100% active at the 2× dial (base mode) | ~1.0× |
| 50% parked / 50% active at 2× | ~0.5× |
| 80% parked / 20% active at 1.5× | ~0.10× |
| Consensus covenants (§21.1), everything | ~0 (fees + float only) |

Since real balance populations are dormancy-heavy, a committee-mode deployment operates in the ~0.1–0.5× regime — matching covenant-based Ark-class economics for the dormant majority, with active users paying the working-capital premium only while transacting. Second-order gain: because expiry is invisible to parked users and active users refresh frequently anyway, `W_exp` can be shortened without user burden, which caps the active side's `min(T_reclaim, W_exp)` receivable window and compresses the active multiple further. Combined with full-cohort rollover for active cohorts (§9.4), steady-state capital approaches 1× of TVL system-wide, nearly all of it user-funded backing. The [B-58] capital model MUST reflect the parked/active split when committee mode is offered.

**[B-59]** Committee mode is OPTIONAL and strictly per-user, per-refresh opt-in. A deployment offering it MUST disclose the committee's membership, size `N`, and the G1′ trust model; Clients MUST display parked (G1′) and self-covered (G1) value as distinct classes and MUST NOT park a balance without explicit user consent.

**[B-60]** Parked leaves MUST be grouped in dedicated cohorts or subtrees so that committee-inclusive expiry paths never govern self-covered value. A roll-forward sweep MUST spend the expired cohort output and re-commit every unexercised parked balance, unchanged, in the same transaction; committee members MUST sign no spend of committee-covered outputs other than ceremony trees and conforming roll-forwards.

**[B-61]** The committee aggregate MUST be n-of-n (threshold schemes reduce theft to `t`-collusion and are forbidden for this role). A wedged committee halts new epochs and roll-forwards only: users retain their pre-signed unilateral exit paths, and deployments MUST size committee-rotation and exit arrangements so that a wedge degrades to mass exit, never to loss.

---

## 22. Privacy considerations

*(Records F12; no fix is claimed.)*

- Ceremony validation (V3–V4) requires each participant to see **all** cohort leaf allocations with amounts; a user's consignment and pre-signed path additionally reveal sibling allocations at each level to anyone they show it to. Settled balances are therefore cohort-public.
- Mitigations available without protocol change: cohort sharding (smaller `|S_N|` per tree bounds the blast radius, at the cost of more epoch transactions); operator-side decoy leaves (operator-owned allocations indistinguishable from user leaves); amount bucketing.
- On-chain, an unrolled path reveals the tree shape and the exiting leaf's amounts.
- Lightning-layer privacy is inherited from the overlay spec and unchanged here.

**[B-56]** Deployments MUST disclose the cohort-visibility property to users. Stronger cryptographic hiding (committed amounts with range proofs inside RGB validation) is not required by this version and is not registered as a blocker.

---

## 23. Open items and R&D register

Every item below is **scoped and bounded**: each has an owner-facing acceptance criterion, and none silently weakens §3. Items R-1 and R-2 are launch blockers ([B-41]); the rest are operational-quality items.

| ID | Item | Blocking? | Acceptance criterion |
|---|---|---|---|
| R-1 | D-VS instantiation (virtual seals) | **Launch blocker** | A concrete RGB construction with a security argument that VS-1..VS-4 hold; independent audit; interop test: full ceremony + adversarial exit on signet |
| R-2 | Exit cost and capital models per deployment | **Parameter-freeze blocker** | Published models per [B-47] and [B-58]; `R_min` derived from the former; reserve multiple `D` and redemption float from the latter |
| R-3 | Consignment growth / checkpointing | No (bounded by [B-44] disclosure) | Published size envelope; pruning design or sharding schedule keeping worst-case exit validation under a stated budget |
| R-4 | Ceremony transport & session protocol | No (any transport meeting [B-11]/[B-13] ordering works) | Message-level spec with replay/DoS handling; restart-with-fresh-nonces verified |
| R-5 | Delegated refresh | **Closed — resolved by §21.2** | Committee-mode parking refreshes (rolls forward) a dormant user's claim with no user participation; no individual delegate gains spending capability (breaking a parked claim requires unanimous committee + operator collusion, [B-61]); failure mode is no roll-forward with unilateral exit preserved. Residual: unplanned dormancy of self-covered users remains subject to the §14.4 deadline, mitigated by [B-52] alarms/auto-exit |
| R-6 | Covenant migration | No | Coexistence plan per §21 |

**R-4 scope note — Lightning integration requirements.** Nothing in this specification changes Bitcoin consensus or the network-facing BOLTs: the operator's outward channels are ordinary Lightning channels, and every deviation is confined to the user↔operator link. The R-4 deliverable therefore decomposes into three tiers. *Tier 0 (reuse):* the RGB-Lightning channel extension set for the overlay (asset-carrying commitments, asset-amount HTLC TLVs, asset invoices, funding consignment exchange), HTLC interception for [B-26] enforcement, and the BOLT quiescence protocol (or the equivalent handshake [B-34] permits). *Tier 1 (new, bilateral, non-BOLT):* the leaf sub-channel implementation per [B-64]; the atomic leaf-update-plus-HTLC session of [B-28]; the ceremony transport itself (nonce rounds, package delivery, forfeit collection) as custom peer messages; and watchtower extensions for the user-side `Δ_rev` penalty watch ([B-54]) and the operator-side forfeit watch (A7). *Tier 2 (out of scope):* multi-hop asset routing and standardized asset fields in BOLT11/12 — needed only if payments ever route beyond the operator hub.

**[B-57]** A deployment MUST NOT represent itself as conforming to this specification while R-1 or R-2 is unmet.

---

## 24. Conformance checklist

All MUST-level requirements by conformance target. Requirement B-46 carries only SHOULD force and is therefore intentionally absent here; it remains normative guidance in §16.2.

**Joint (both targets):** B-04, B-05, B-11, B-13, B-17, B-24, B-25, B-27, B-30, B-38, B-39, B-48, B-59 (operator: disclosure; client: display and consent), B-62 (client: package-first signing; operator: all-or-fallback broadcast), B-63 (client: package-first deposit signing and leaf verification; operator: ceremony-only acceptance), B-64 (both endpoints implement the leaf commitment format), B-65 (BTC profile: operator disclosure, client display).

**BTC profile (Appendix B) additionally:** B-66 (operator: explicit profile claim; R-1 exempt, R-2 not).

**Operator:** B-01, B-06, B-07, B-08, B-09, B-10, B-15, B-19, B-20, B-21, B-22, B-23, B-26, B-31, B-32, B-33, B-34, B-35, B-36, B-40, B-41, B-42, B-43, B-44, B-45, B-47, B-55, B-56, B-57, B-58, B-60, B-61 (B-60/B-61 additionally bind committee members in deployments offering §21.2).

**Client:** B-02, B-03, B-12, B-14, B-16, B-18, B-28, B-29, B-37, B-49, B-50, B-51, B-52, B-53, B-54.

An implementation claiming conformance MUST list any deviation from this checklist explicitly.

---

## 25. Changelog and findings disposition

### v0.1 → v0.2

| Finding | Severity | Resolution in v0.2 |
|---|---|---|
| F1 — reserve control undefined | Critical | §7: cohort n-of-n MuSig2 + operator-only CLTV expiry path; [B-06]/[B-07]; guarantee rescoped in §3.2(1) |
| F2 — offline users vs epoch replacement | Critical | §8–§9: per-epoch cohort outputs from operator liquidity; old cohorts persist to expiry; §14 refresh/expiry |
| F3 — rollback griefing | Critical | §13: connector-bound forfeits, `Δ_leaf` priority, operator watch scoped to A7; blast radius reduced to bilateral (§13.1) |
| F4 — send-then-exit gap | High | §11: leaf sub-channels with LN-penalty; send-ordering [B-26]/[B-28]; receive rule [B-27] |
| F5 — HTLCs at epoch cut | High | §12: per-user quiescence; in-flight excluded from settled |
| F6 — fee staleness | High | §16: anchors on every pre-signed tx, minimal embedded fees, mandatory exit-cost model [B-47] |
| F7 — virtual-seal knock-ons | High | §15: D-VS interface VS-1..VS-4; atomic epochs [B-42]; history budget [B-44]; dust floor [B-43] |
| F8 — solvency operational only | Medium | §6.2: structural non-inflation G2, proven at cosign time (V3–V4) |
| F9 — reserve flows unspecified | Medium | §9: exhaustive flow list [B-21]; epoch tx anatomy |
| F10 — expiry undecided | Medium | §14: expiry chosen; `W_exp`, `M`, deadlines, alarms; delegated refresh R-5 |
| F11 — overbroad collusion claim | Medium | §17.3: rescoped can/cannot lists; §3.3 non-guarantees |
| F12 — tree privacy | Medium | §22: disclosed, mitigations listed, no fix claimed |

### Honest weakenings vs v0.1 §2 (restated)

1. Unilateral exit covers only cosigned claims (§3.2.1).
2. Claims expire; refresh-or-exit liveness before `D_exit` is required (§3.2.2).

Both are the boundary of pre-signed-transaction enforcement on Bitcoin without covenants; §21 records their removal path.

---

## Appendix A — Non-normative Q&A

*This appendix is explanatory only. It introduces no requirements; where it appears to conflict with §1–§25 or the formal companion, those govern. Each answer cites the normative text it summarizes.*

### A.1 — "I hold a pre-signed exit. If my Lightning balance changes, doesn't the stale exit let me double-spend?"

It would, if the exit were a static pre-signed amount — that is exactly v0.1's send-then-exit gap (review finding F4). v0.2 closes it with three layers:

1. **The redemption right is a channel, not a number (§11).** The pre-signed tree only reaches your *leaf output*, a 2-of-2 sub-channel with the operator. What you can exit with is the **latest signed leaf state**, updated with LN-penalty mechanics, not the leaf's face value.
2. **Lightning state and leaf state move in lockstep, per payment ([B-26]/[B-28]).** The operator does not irrevocably commit your outgoing HTLC until you have completed a leaf-state decrease to your post-send balance. By the time a send is final, your redemption right has already shrunk. Receives are the mirror image: the leaf state is raised afterward, and until then the delta is `pending` — your credit exposure, not a double-spend opportunity.
3. **Publishing stale state loses money (§11.4, §13; Theorem T4).** A revoked leaf *state* is swept in full by the counterparty's penalty path within `Δ_rev`; a forfeited whole *leaf* from a refreshed epoch is taken in full by the operator's connector-bound forfeit, which wins the race because commitments carry a `Δ_leaf` delay the forfeit lacks ([B-08]).

Caveat, stated rather than hidden: layers 2–3 protect the **operator** only while it (or its watchtower) watches the chain (A7). A failed watch enriches the double-spender at the operator's expense only — never another user's (§10.4, T4(i)) — and never weakens your own exit, since a disappeared operator cannot have superseded your latest state (§13.4).

### A.2 — "How do I redeem? Do I bring vUSDT to get canonical USDT?"

vUSDT is not a redemption ticket; the right lives in your cosigned leaf state, not in the token ([B-04], NG4). Two paths:

- **Cooperative (operator online, §11.7).** Works like an ordinary Lightning payment *to* the operator: you pay `x` vUSDT back across your channel (leaf state decreasing in lockstep, [B-26]) and the operator delivers `x` canonical USDT — an on-chain RGB transfer to your invoice, or canonical USDT routed over Lightning. Nothing is "burned"; the balance simply moves back to the operator's side. Cooperative redemption can cover your full `R_current`, including pending.
- **Unilateral (operator gone or refusing; §16.3, Theorem T2).** You bring nothing and ask no one: broadcast the exit package — unroll, wait `Δ_leaf`, latest commitment, wait `Δ_rev`, claim — and receive exactly `R_settled` canonical USDT. No vUSDT changes hands because the accounting already happened continuously at each send.

Leftover vUSDT in your channel after an exit is inert: any further send would require a leaf-state decrease ([B-26]) against a leaf that is now spent on-chain, so it cannot be double-used. Force-closing the overlay channel recovers its BTC; the vUSDT allocation itself carries no claim beyond the one just exercised. The `R_settled` vs `R_current` asymmetry is why clients keep `pending` small ([B-29]).

### A.3 — "What happens to my pending balance if the operator disappears mid-refresh?"

Never worse than the moment before the refresh began — this is Theorem T3 (refresh atomicity) traced through the ceremony stages (§8.2):

| Operator vanishes… | Your position |
|---|---|
| during stages 1–3 (construct/validate/tree-sign) | Nothing enforceable was created ([B-10]); old claim intact, exit with old `R_settled`; pending lost (NG1, as always) |
| after stage 4 (you hold the verified new package) but before your forfeit | New package's epoch will never confirm — worthless but harmless; old claim intact |
| after your forfeit (stage 5) but before `E_{N+1}` broadcast | The forfeit spends `E_{N+1}`'s connector ([B-33]); with `E_{N+1}` unconfirmed it is permanently unenforceable — old claim intact |
| after `E_{N+1}` confirms | The refresh **succeeded**: your new initial leaf state equals your full snapshot `R_current` (§14.2), so pending became settled; exit the new leaf with everything |

The amount at risk mid-refresh is therefore exactly your pending as of the *old* state — the same exposure as before the refresh started. A refresh can only succeed or leave you where you were; there is no interleaving in which you surrendered the old claim without holding an enforceable new one.

### A.4 — "What limits leaf state transitions? In Lightning I can update indefinitely."

Update *count* is unlimited here too — leaf updates are ordinary off-chain LN-penalty updates (§11.3). The bounds are structural, not numerical:

- **Amplitude.** `u` moves freely within `[0, ℓ(i)]`, but leaf capacity is frozen per epoch: tapret commitments are fixed at signing, so a leaf cannot be resized in place (§15.3). Receives beyond capacity accumulate as pending until refresh; resizing means joining the next epoch.
- **Lifetime.** Every state dies at `H_exp`; refresh-or-exit by `D_exit` (§14.4). Vanilla LN channels live indefinitely; leaves do not — this is the honest weakening §3.2(2).
- **Shape.** Balance-only, no HTLC outputs (§12), so a leaf cannot route; all HTLC mechanics live in the overlay vUSDT channel, coupled to the leaf by [B-26] at one extra half-round per send.
- **Cooperation.** Every update needs the operator's signature, exactly as every LN update needs your peer's; refusal is a liveness risk (NG2) whose remedy is unilateral exit with the last completed state.

Mental model: a leaf is a Lightning channel in every off-chain respect, mounted on an unconfirmed funding output with a fixed size and an expiry date, delegating routing to the channel above it.

### A.5 — "Walk me through the money lifecycle: in, around, out."

**In (onboarding).** A new user's balance climbs a ladder: *deposit → pending → settled*.

1. **Channel:** the user gets a vUSDT Lightning channel from the LSP; inbound capacity is synthetic and costs the LSP nothing (§6.3). State: `R_current = 0`, no leaf.
2. **Deposit:** canonical USDT reaches the LSP by any route — the user's own transfer, a third-party payment, an on-ramp — and the LSP forwards the amount as vUSDT. Now `R_current = 500, R_settled = 0`: the whole deposit is `pending`, i.e. operator credit (NG1), while the LSP holds the canonical in float.
3. **First epoch:** at the next ceremony the user's snapshot fixes 500 ([B-34]), they run V1–V12, cosign their path and initial leaf state, verify their exit package ([B-12]) — no forfeit, nothing to supersede — and when `E_N` is `k`-deep, `R_settled = 500`: property, backed by the very canonical they deposited (§9.4, "whose money").
4. **Or skip the trust window entirely:** with atomic onboarding (§9.5) the user contributes their USDT UTXO as an input to `E_N` itself — the deposit and the enforceable claim are created by one transaction, or neither exists.

**Around (payments).** Two coupled layers move on every payment; the order is the security mechanism.

*Send 100 (balance 500):* (1) leaf-state decrease first — one LN-penalty update to `u = 400` ([B-26]/[B-28]); (2) then a standard vUSDT HTLC on the overlay channel, routed by the LSP, which bridges at its edge (vUSDT to a same-LSP recipient; its own canonical USDT to an external one); (3) settlement brings `R_current` to 400. A failed HTLC leaves the user briefly *under*-settled (never over — [B-05]), fixed by a cooperative raise.

*Receive 100:* payer delivers to the LSP, LSP forwards a vUSDT HTLC (`R_current` +100), then the leaf is raised up to capacity ([B-27]); any excess is `pending` until refresh, capped by `P_max` ([B-29]). Sends extend zero credit; receives extend bounded credit — the deliberate asymmetry.

No payment ever touches the chain, the reserve, or the tree: those move only at epochs, exits, forfeits, and sweeps (§9.2). Parked users (§21.2) cannot send until they unpark; refresh snapshots exclude in-flight HTLCs ([B-34]/[B-35]).

**Out (redemption).** Per A.2: cooperatively, pay vUSDT back and receive canonical (covers up to `R_current`); unilaterally, broadcast the exit package and take `R_settled` with no one's cooperation. The ladder runs in reverse: settled property leaves through a path that was pre-signed the day it was created.

---

## Appendix B — BTC instantiation (normative profile)

*This appendix defines a conformance profile of this specification in which the settled asset is bitcoin itself. A deployment implementing this profile is a "Spec B/BTC" deployment. Everything not modified here applies verbatim.*

### B.1 Construction

- **Settled asset = sats.** The epoch transaction, tree, and leaves carry plain bitcoin: a leaf's satoshi value *is* the user's settled balance (superseding `btc_leaf`'s anchor-budget role; leaf BTC = `R_settled,i` plus output overhead). Leaf sub-channel states split sats between user and operator with the unchanged §11/[B-64] machinery. Unilateral exit (§16.3, T2) delivers bitcoin directly.
- **Overlay asset = vBTC.** The operator issues an RGB asset `vBTC` and provisions it as overlay-channel capacity, exactly as vUSDT: arbitrarily large, synthetic, carrying no claim by itself. The overlay channels are ordinary RGB-Lightning channels (R-4 Tier 0); payments, coupling ([B-26]/[B-28]), quiescence (§12), and the pending mechanism ([B-27]/[B-29]) apply unchanged with `vBTC` in place of `vUSDT` and sats as the settled unit.
- **Atomic onboarding (§9.5)** takes a plain BTC UTXO as the user-contributed epoch input; [B-63] applies with the RGB-allocation clause vacuous.

### B.2 What becomes vacuous

Because no RGB state rides the tree or the exit path:

- **A3/AX-VS and dependency D-VS are not used**; **[B-45] does not apply** and **R-1 is not applicable** to this profile — it is not a launch blocker here.
- §15.1–§15.3 and §15.5 (tree transitions, virtual-seal interface, consignment growth, R-3) are vacuous; §15.4's floor reduces to Bitcoin dust and exit-cost rationality ([B-47]), i.e. `R_min` is set by [B-51]'s model alone.
- Ceremony checks V3–V4 reduce to verifying the tree's satoshi values (leaf sums and cohort output value), which Bitcoin consensus then enforces on every spend; G2 (non-inflation) holds by transaction validity alone.

### B.3 What strengthens

Every guarantee's asset half becomes unconditional: G1/T2 (and G1′ in committee mode) rest on AX-FIN/AX-CONF/AX-SIG only. The §3.2 boundary statement loses its research caveat — for settled balances this profile is Lightning-grade self-custody of bitcoin, with the expiry and ceremony-participation obligations as the only deltas from a vanilla channel (both softened by §21.2 as usual).

### B.4 What is retained unchanged

The output and key model (§7), ceremony (§8), epoch anatomy and capital model (§9, including the dial, rollover [B-62], and [B-58] disclosure — denominated in BTC), tree (§10), leaf sub-channels (§11, [B-64]), forfeit layer (§13), expiry/refresh (§14), fees and exit-cost model (§16), failure analysis (§17), parameters (§19, minus RGB-specific rows), client requirements (§20), committee mode (§21.2), and privacy properties (§22). R-2 and R-4 remain as stated (R-4's only RGB surface is the overlay channels).

### B.5 Profile requirements

**[B-65]** A Spec B/BTC deployment MUST apply [B-04] to vBTC verbatim: vBTC MUST NOT be represented to users as bitcoin, and displayed balances MUST distinguish settled sats (property) from pending vBTC (operator credit). vBTC issuance is unconstrained in size but MUST be disclosed as synthetic in the [B-58] capital model.

**[B-66]** A deployment claiming this profile MUST state so explicitly (conformance is to this appendix, with §B.2's exemptions and no others) and MUST still satisfy [B-57] with respect to R-2; R-1 is exempt per §B.2.

### B.6 Two-phase deployment path

This profile exercises every novel component of the specification — ceremonies, trees, leaf channels, forfeits, capital management, committee mode — with zero dependence on the open research item. The intended sequencing for a USDT deployment is therefore: ship and audit the BTC profile first (the unconditional core), then upgrade the settled asset to canonical RGB-USDT when a D-VS instantiation passes R-1's acceptance criteria — at which point the audit increment is exactly §15 plus the asset halves of the formal companion's theorems, nothing else.
