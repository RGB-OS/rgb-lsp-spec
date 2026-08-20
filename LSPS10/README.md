# LSPS10 Self-Custodial Stablecoin Channels with Federated Redemption

| Name    | `federated_redemption` |
|---------|------------------------|
| Version | 0                      |
| Status  | Draft                  |

> **Note** The LSPS10 number is provisional and was chosen to avoid
> collision with upstream [BitcoinAndLightningLayerSpecs/lsp](https://github.com/BitcoinAndLightningLayerSpecs/lsp)
> numbers (LSPS0–LSPS7), which this repository may adapt in the future.

## Motivation

Canonical USDT exists as an issuer-native RGB asset on Bitcoin. In
principle it could be placed directly into Lightning channels; in
practice, doing so would make every LSP's hot Lightning
infrastructure a direct custodian of canonical stablecoin, with no
systemic response available when — not if — an LSP is compromised.

This specification instead separates the **reserve asset** from the
**working asset**:

* **Canonical USDT** (issuer-native, on RGB) is locked in cold
  reserve UTXOs controlled by a k-of-n **Redemption Federation**,
  whose members hold key shares inside Trusted Execution
  Environments (TEEs).
* **vUSDT** — a single federation-issued RGB20 asset — is the
  working asset of the LSP layer: it is what LSPs deploy as channel
  liquidity and what users hold, self-custodially, in their
  channels. Every unit of vUSDT is issued 1:1 against canonical
  USDT locked in the reserve, atomically and verifiably.
* Any bearer of vUSDT can redeem it into canonical USDT at the
  federation **without the cooperation of any LSP**, in a single
  atomic Bitcoin transaction.

The federation is a **validated signer, not a blind signer**: every
member independently re-validates every fact — Bitcoin chain state,
RGB consignments on both contracts, supply invariants, incident
policy — before contributing its signature share to any operation.
Compromise of an LSP plus a minority of federation members yields
the attacker nothing.

### Why a separate working asset

> **Rationale** Putting canonical USDT directly into channels was
> considered and rejected:
>
> 1. **Cold reserves, hot float.** Channel funds are necessarily
>    hot. With a working asset, only the capped operational float is
>    hot; the canonical asset stays in k-of-n cold custody.
> 2. **A redemption chokepoint exists.** Because vUSDT only becomes
>    canonical USDT by passing through federation validation, stolen
>    funds can be screened and frozen at redemption during a
>    declared incident (§ [Incident response](#incident-response)).
>    Stolen canonical USDT could only be stopped by the issuer.
> 3. **Issuer-action insulation.** Issuer-side interventions on the
>    canonical contract (e.g. freezes) hit the reserve interface,
>    where they are handled by published federation policy, rather
>    than hitting individual user channels.
> 4. **Enforceable exposure caps.** Since every unit of vUSDT enters
>    the LSP layer through federation issuance, the federation can
>    *structurally* enforce a ceiling on any LSP's hot float
>    (§ [Flow S](#flow-s--sweeps-and-the-one-way-valve)).

## Design Goals

1. **Self-custody.** The user's vUSDT channel balance MUST be
   spendable and recoverable unilaterally (standard Lightning
   penalty/force-close mechanics).
2. **Full collateralization, provable by anyone.** At all times:
   `reserves ≥ issued − burned − pending_burn`, where every term is
   verifiable against Bitcoin by an external auditor
   (§ [Proof of Solvency](#transparency-and-proof-of-solvency)) —
   not merely attested.
3. **LSP-independent, atomic redemption.** Any vUSDT bearer MUST be
   able to redeem into canonical USDT with no LSP cooperation, via a
   single Bitcoin transaction in which the vUSDT hand-over and the
   canonical USDT payout are atomic.
4. **Validated signing.** No federation member ever signs a message
   it has not independently derived from locally validated state.
5. **No single point of theft.** Compromise of an LSP **plus up to
   k−1 federation members** MUST NOT enable theft of reserves,
   unbacked issuance, or fraudulent redemption.
6. **Quantified LSP blast radius.** Total LSP compromise MUST be
   bounded by the LSP's federation-enforced hot cap, with user
   balances and the LSP's swept federation balance at zero exposure
   (§ [Compromise Containment](#compromise-containment)).
7. **Safe degradation.** Loss of federation liveness (up to n−k
   members offline) MUST NOT affect Lightning payments or user
   custody; only issuance and redemption pause.

## Definitions

### Data types

This specification uses data types defined in
[LSPS0 Common Schemas][LSPS0.common_schemas].

###### Link: LSPS10.uusdt

Amounts of USDT and vUSDT MUST be expressed in **micro-USDT**
(`uusdt`, 10⁻⁶ USDT), encoded as a JSON string containing the
decimal representation, with field names suffixed `_uusdt`. For
example, 250 USDT is `"250000000"`. Both RGB contracts MUST use
precision 6 so one contract unit equals one micro-USDT.

### Actors

* **User (Client)** — holds vUSDT in a Lightning channel with an
  LSP; the API consumer.
* **LSP** — Lightning Service Provider. Provides vUSDT channels and
  liquidity (per [LSPS1][]). Holds only **vUSDT** (channel liquidity
  plus a capped on-chain float) and **USDT on exchanges** (for
  market operations and fast redemptions, § [Flow R2](#flow-r2--fast-redemption-over-lightning-lsp-fronted)).
  The LSP is *not* a custodian of user funds and is *not* trusted
  for redemption.
* **Federation** — n independent members, threshold k, custodying
  the reserve and exclusively controlling vUSDT issuance. Trusted
  collectively (k-of-n), never individually.
* **Federation Member** — one operator running the member stack
  (Bitcoin + RGB validators, event log, policy signer) inside a TEE.

### Assets

* **Canonical USDT** — issuer-native USDT as an RGB asset on
  Bitcoin. The reserve asset. Never placed in channels.
* **vUSDT** — a single RGB20 fungible asset shared by all
  registered LSPs, whose issuance and burn rights are held
  exclusively by the federation threshold key. The working asset.
* **Reserve** — the set of federation-controlled Taproot UTXOs
  holding canonical USDT allocations.
* **Issued** — total vUSDT ever issued (provable via the inflation
  lineage, § [Asset Model](#asset-model)).
* **Pending burn** — vUSDT received at federation seals (redeemed or
  swept) and not yet batch-burned.
* **Circulating** — issued − burned − pending burn.

## Architecture Overview

```
            Lightning Network (vUSDT channels)
   ┌──────┐    LN     ┌───────────┐
   │ User │◄─────────►│    LSP    │──── USDT on exchanges
   └──┬───┘           └─────┬─────┘     (bridging, Flow R2)
      │                     │
      │ channel close       │ atomic deposit+issuance (Flow I)
      │ + atomic            │ sweeps: burn-and-credit (Flow S)
      │ redemption (R1)     │
      ▼                     ▼
 ┌───────────────────────────────────────────────────┐
 │              Bitcoin + RGB (one layer)            │
 │                                                   │
 │   vUSDT contract              canonical USDT      │
 │   federation-issued,          issuer-native,      │
 │   linear inflation/burn       locked in Reserve   │
 │   lineages                    UTXOs               │
 │                                                   │
 │   Reserve UTXOs: Taproot — MuSig2 key path,       │
 │   k-of-n tapscript fallback, timelocked recovery  │
 └────────────────────────┬──────────────────────────┘
                          │ validated k-of-n signing
 ┌────────────────────────┴──────────────────────────┐
 │              Redemption Federation                │
 │   Member 1 … Member n — each runs, inside a TEE:  │
 │    • Bitcoin full node + RGB validator            │
 │    • Replicated event log + incident registry     │
 │    • Policy-enforcing threshold signer            │
 └───────────────────────────────────────────────────┘
```

Trust flows in exactly one direction: users and LSPs trust the
**federation collectively** (k-of-n, TEE-hardened, publicly
attested, velocity-limited, publicly auditable). The federation
trusts **no one** — not the LSP, not the user, not any single
member. Because both assets live on Bitcoin, every value movement
between them is a single co-signed transaction: there is no
cross-chain window in which either side trusts the other's follow-
through.

## Asset Model

vUSDT MUST use an RGB schema supporting **capped secondary
issuance** and **provable burn** (the IFA-style schemas of
RGB v0.11+):

* **Genesis** creates no circulating supply. The genesis max supply
  MUST be set far above any plausible cumulative issuance (e.g. the
  full 2⁶⁴-unit range), since burn-and-reissue cycles permanently
  consume headroom.
* **Issuance** consumes the single-use **inflation-right seal**,
  held on a federation-controlled UTXO, and reassigns the remaining
  allowance to a fresh federation seal. The inflation right
  therefore has a *linear, Bitcoin-anchored lineage*: a hidden
  issuance would require double-spending the inflation seal's UTXO,
  which Bitcoin prevents. Publishing this lineage makes **total
  issued supply provable and complete**, not merely claimed.
* Every issuance operation MUST be **co-anchored in the same witness
  transaction** as the canonical USDT deposit that backs it
  (§ [Flow I](#flow-i--atomic-deposit-and-issuance)). Auditors
  thereby verify 1:1 deposit↔issuance matching directly from the
  published lineages: an unbacked issuance is not just forbidden —
  it is *publicly visible the moment it happens*, even under full
  quorum compromise.
* **Burns** are executed by the federation in **epoch batches** over
  vUSDT accumulated at its redemption and sweep seals, consuming the
  burn-right seal — likewise a linear, publishable lineage. Between
  receipt and batch burn, such vUSDT is accounted as *pending burn*.

> **Rationale — burn model over treasury model.** A pre-minted
> treasury cannot be externally audited: RGB has no global ledger,
> so no outside observer can upper-bound what has left a treasury.
> Linear issuance and burn lineages anchor supply accounting to
> Bitcoin's double-spend prevention, making the solvency statement
> verifiable end-to-end (§ [Proof of Solvency](#transparency-and-proof-of-solvency)).
> The cost — issuance serialization through one seal — is handled by
> epoch batching, or by a small fixed number of parallel inflation
> shards if volume demands it.

## Reserve Custody

### Reserve UTXOs

Reserve UTXOs (canonical USDT allocations) and the vUSDT
inflation-right and burn-right UTXOs MUST be Taproot outputs with:

* **Key path**: MuSig2 ([BIP327][]) n-of-n aggregate of all member
  keys — normal cooperative operation.
* **Script paths**: a k-of-n `OP_CHECKSIGADD` tapscript as the
  liveness fallback when up to n−k members are unavailable, and
  OPTIONALLY a timelocked (`OP_CHECKSEQUENCEVERIFY`) recovery path
  to a designated recovery quorum for disaster recovery after
  prolonged federation failure. A FROST-style threshold key path MAY
  replace this construction once implementations mature.

### TEE requirements

Each member MUST run its validators, event log, and signer inside a
TEE, with:

* **Remote attestation** — mutually verified at DKG time and on
  every software upgrade; attestation evidence MUST be published
  (`lsps10.fed.get_federation_info`).
* **Sealed key shares** — generated inside the enclave via DKG;
  never in plaintext outside it.
* **Reproducible builds** — attested measurements MUST correspond
  to publicly reproducible builds of open-source member software.

Because the federation validates only Bitcoin and RGB, the entire
member stack — full node, RGB validator, event log, signer — is a
single-chain trusted computing base, small enough to audit and
reproduce credibly.

TEEs are **defense in depth, not the security model**. The protocol
MUST remain secure assuming any single TEE can be fully broken; this
is precisely why the federation is a k-of-n *validated* signer
rather than one attested blind signer. Members SHOULD diversify TEE
vendors, hosting, and jurisdictions.

### Membership parameters

n MUST be ≥ 4; n = 7 with k = 5 is RECOMMENDED. Theft requires k
compromised members; liveness loss requires n−k+1 failures. k MUST
exceed n/2 so two disjoint quorums cannot exist.

## LSP Registration and Authority Model

Every LSP registers with the federation before receiving issuance.
Registration binds two distinct authorities:

* **Hot identity** — the keys on the LSP's operational
  infrastructure (node key, API keys). Assumed stealable.
* **Cold authority** — a key (or officer-key quorum) enrolled in an
  offline ceremony, which MUST never touch operational servers.

The governing principle is a **one-way valve**:

* Operations that move value **toward** the federation, or that only
  reduce risk, are hot-authorized: sweeps (fixed destination),
  incident *declaration*.
* Operations that move value **out** of federation control are
  cold-gated: withdrawals of the LSP's federation balance MUST carry
  a cold signature and MUST pay only to destinations pre-registered
  cold; adding a destination requires a cold signature plus a
  mandatory delay with notification.
* Re-issuance of the LSP's federation balance into hot vUSDT MAY be
  hot-initiated, but ONLY within the LSP's hot-cap headroom
  (§ [Flow S](#flow-s--sweeps-and-the-one-way-valve)) — so even
  this cannot raise the attacker's ceiling.

Registration also fixes:

* `max_lsp_hot_uusdt` — the LSP's hot-float ceiling,
  **federation-enforced at issuance time** from event-log
  accounting (`issued_to_lsp − swept_by_lsp ≤ cap`). Since vUSDT
  only enters the LSP layer through federation issuance, this bound
  is structural, not advisory.
* `min_lsp_to_self_delay` — LSPS10 channels MUST set the CSV delay
  on LSP-side outputs to at least this value (RECOMMENDED ≥ 144
  blocks). See [Compromise Containment](#compromise-containment)
  for why.
* OPTIONALLY a small registration bond, slashable on provable
  fast-redemption fraud (§ [Flow R2](#flow-r2--fast-redemption-over-lightning-lsp-fronted)).

## Protocol Flows

### Flow I — Atomic deposit and issuance

A single Bitcoin transaction carries both contracts' state
transitions under one anchor:

* **Inputs**: the LSP's UTXO(s) holding canonical USDT
  (LSP-signed); the federation's inflation-right UTXO
  (threshold-signed).
* **RGB transitions**: canonical USDT → a fresh Reserve seal; vUSDT
  issuance of the same amount → the LSP's seal; remaining inflation
  allowance → a fresh federation seal.

Procedure: the LSP calls `lsps10.fed.create_issuance` with the
amount, its USDT inputs + consignments, and its vUSDT beneficiary
invoice; each federation member independently reconstructs the
transaction, runs the [ISSUE checklist](#issue), and contributes its
share; the completed transaction is broadcast and the vUSDT
consignment returned. Multiple deposits MAY be batched into one
issuance transaction per epoch.

Either the transaction confirms — deposit locked *and* vUSDT
issued — or nothing happens. The LSP's residual trust in the
federation is custody of the locked reserve only; the federation
extends **zero trust** to the LSP.

### Flow S — Sweeps and the one-way valve

A compromised LSP loses whatever vUSDT is hot. To keep that bounded:

1. When the LSP's hot float exceeds `sweep_trigger_uusdt`, or at
   least every 24 hours, the LSP MUST sweep the excess: splice out
   of channels (or spend on-chain float) into an RGB transfer to the
   federation **sweep seal** (fixed destination — hot-callable,
   since "abuse" only protects the funds).
2. After confirmation and validation, members credit the LSP's
   **federation balance** in the event log; the swept vUSDT joins
   the pending-burn pool and is destroyed at the next burn epoch.
3. The LSP re-deploys via `lsps10.fed.reissue` — a fresh Flow I
   issuance debiting the balance instead of requiring a new deposit,
   allowed only within hot-cap headroom — or withdraws canonical
   USDT via `lsps10.fed.withdraw` (cold-gated, allowlisted).

> **Rationale — burn-and-credit.** Sweeping into a burn rather than
> a custody pool keeps the audit picture exact: circulating tokens
> are precisely the tokens in the wild, and the LSP's parked claim
> is an event-log liability fully matched by reserves. Combined with
> the one-way valve, sweeping is *strictly* de-risking: server
> compromise, however total, cannot move a swept balance.

User channel balances are never swept — they are self-custodial.
User protection against a compromised LSP is penalty enforcement
plus the federation watchtower duty
(§ [Watchtower duty](#watchtower-duty)).

### Flow R1 — Atomic on-chain redemption (base path, LSP-independent)

Available to **any** vUSDT bearer, with no LSP involvement — this is
what makes vUSDT money even if the bearer's LSP is malicious,
insolvent, or gone.

1. The user obtains an on-chain vUSDT allocation — normally by
   cooperatively or force-closing their channel.
2. The user calls `lsps10.fed.create_redemption` with the amount,
   their vUSDT inputs + consignments, and an RGB invoice for
   receiving canonical USDT. The federation reserves Reserve UTXOs
   for the request (TTL-bounded, rate-limited against DoS) and
   returns a transaction template: user vUSDT → federation
   redemption seal; canonical USDT (amount − fee) → the user's
   seal; reserve change → a fresh Reserve seal.
3. The user signs their inputs and submits. Each member
   independently reconstructs and validates the full transaction
   under the [REDEEM checklist](#redeem); a quorum completes the
   reserve-input signature; the transaction is broadcast.

The swap is **atomic**: the user's vUSDT and the federation's
canonical USDT move in the same transaction, so at no point does
either side hold an unsecured claim on the other. The redeemed vUSDT
sits at the redemption seal as pending burn until the next burn
epoch.

### Flow R2 — Fast redemption over Lightning (LSP-fronted)

For instant exit without an on-chain footprint, the LSP acts as
market maker between vUSDT and its exchange-held USDT:

1. User calls `lsps10.create_fast_redemption` on the **LSP** with
   amount and a payout address on any network the LSP supports
   (exchange withdrawal rails — e.g. Tron, Ethereum — or canonical
   RGB USDT).
2. LSP returns a vUSDT Lightning **hold invoice** and a quote.
3. User pays; the LSP holds the HTLC, executes the payout, and
   settles the HTLC on broadcast, revealing the preimage as a
   receipt.
4. The LSP's accumulated vUSDT is later swept (Flow S) or redeemed
   (Flow R1).

The federation is not involved and extends no trust to the LSP. The
user's exposure is one in-flight HTLC; users who reject even that
use Flow R1. LSP fraud here is cleanly provable (invoice + preimage
+ absent payout) and SHOULD slash the registration bond and revoke
registration.

A federation-operated Lightning gateway for in-Lightning redemption
was considered and is **deferred** — see
[Deferred: Lightning Gateway](#deferred-lightning-gateway).

## Federation Security Model

### The validated-signer principle

Each member runs a policy-enforcing signer whose only input is its
co-located validator stack:

* a Bitcoin full node (SHOULD; at minimum full validation of all
  transactions touching federation UTXOs and submitted
  consignments);
* an RGB validator holding the complete lineages of the vUSDT
  contract's rights and the Reserve's canonical USDT allocations;
* the replicated event log: issuances, sweeps, redemptions, burns,
  LSP balances, incident registry — with the solvency invariant
  checked on every entry.

**No member ever signs a hash it received from another party** — not
from an LSP, a coordinator, or another member. Each member
constructs the transaction to be signed from its own validated state
and signs only its own construction; the threshold protocol fails
harmlessly if constructions diverge.

> **Rationale** This is what the headline guarantee rests on: a
> compromised LSP colluding with up to k−1 compromised members
> cannot produce a valid theft, because every honest member of any
> quorum re-derives and re-checks everything and withholds its
> share. A blind-signing federation collapses to a single point of
> failure the moment one coordinator or one enclave breaks.

### Validation checklists (normative)

#### ISSUE

Before contributing a share to a deposit+issuance transaction, a
member MUST verify:

1. The canonical USDT input consignments are valid and the inputs
   are unspent on its own Bitcoin view.
2. The USDT transition pays the full deposit amount to a
   federation-derived Reserve seal on a federation-derived output.
3. The issuance amount equals the co-anchored deposit amount; the
   inflation transition correctly consumes the current
   inflation-right seal and returns the remainder to a
   federation-derived seal.
4. The beneficiary is a registered LSP (or valid direct depositor);
   for `reissue`, the debit is within the LSP's balance **and** the
   result respects `issued_to_lsp − swept_by_lsp ≤
   max_lsp_hot_uusdt`.
5. No active incident suspends the beneficiary.
6. Post-state solvency invariant holds.

#### REDEEM

Before contributing a share to a redemption transaction:

1. The request exists, is unexpired, unpaid, and within per-request
   and per-epoch velocity caps.
2. The presented vUSDT consignments are valid back to their
   issuance events; inputs unspent on the member's own view.
3. **Incident screening**: no active incident freeze applies to the
   presented allocations' lineage
   (§ [Incident response](#incident-response)).
4. The transaction the member itself constructs pays vUSDT to the
   federation redemption seal and exactly `amount − fee` of
   canonical USDT to the invoice bound at request time; all change
   returns to federation-derived seals.
5. Post-state solvency invariant holds, counting the received vUSDT
   as pending burn.

#### BURN

Epoch burns MUST consume the burn-right seal correctly, burn only
vUSDT held at federation redemption/sweep seals, and reduce
pending-burn accounting exactly.

#### RESERVE-SPEND

Any other transaction spending federation UTXOs (consolidation, fee
management, key rotation roll-overs) MUST send every output and
every RGB allocation back to federation-derived seals — Reserve
operations can never leak value.

#### CREDIT-OUT

Withdrawals of an LSP federation balance MUST carry a valid cold
signature, pay only cold-registered destinations, respect the
allowlist-change delay, and be blocked entirely under an active
incident for that LSP.

#### ADMIN

Membership changes, DKG resharing, enclave-measurement approvals,
parameter changes, LSP registration/revocation, and incident
lift/renewal MUST require an administrative threshold of at least k
(n−1 RECOMMENDED for membership and measurement changes), each
member validating the proposal against published governance rules.

### Replicated event log

The log MUST be deterministic and replicated so that any two honest
members with the same events derive the same state. Byzantine
agreement is not required: signing is the only consequential act and
already demands k independent validations, so divergence safely
halts signing rather than corrupting state. Members MUST
periodically publish signed log-head commitments; divergent heads
raise a public alarm.

### Key generation and rotation

All threshold keys MUST come from dealer-less DKG inside the
enclaves. On membership change or scheduled rotation, the federation
MUST reshare and roll federation UTXOs to the new key set under
RESERVE-SPEND. A member with lapsed attestation or divergent
behavior MUST be excluded from quorums until re-attested.

### Watchtower duty

Federation members SHOULD jointly operate watchtowers over all
registered LSPs' channels: penalty enforcement for revoked states
protects offline users, and the same monitoring feeds incident
detection — a wave of unilateral closes from one LSP is loud,
on-chain, and automatic grounds to propose an incident.

### Incident response

The redemption chokepoint doubles as the systemic circuit breaker.

* **Declaration.** An incident for an LSP is declared by (a) the
  LSP's cold authority (hot-callable *proposal* is also allowed —
  declaring can only protect), or (b) federation ADMIN threshold on
  evidence: mass unilateral closes, anomalous outbound flows,
  attempted non-allowlisted withdrawals, velocity anomalies.
* **Effects.** Immediately: the LSP's federation balance and
  registration freeze (no issuance, no CREDIT-OUT); redemption
  screening activates for allocations whose lineage traces to the
  LSP's holdings as of declaration time; sweeps *into* the
  federation remain allowed.
* **Scope limits.** Screening MUST be incident-scoped: only during a
  declared incident, only for allocations provably attributable to
  the LSP at declaration, time-boxed with ADMIN-threshold renewal,
  every freeze and lift published in the signed log, with an appeal
  path for false positives.

> **Rationale** Unbounded freezing power would destroy vUSDT's
> bearer credibility — the entire point is redemption without
> anyone's permission. Screening is incident response, not standing
> surveillance; its every use is public.

### Recovery

The timelocked recovery script path allows a designated recovery
quorum to move federation funds after a long CSV delay if the
federation permanently loses k-liveness. If reserves ever fall below
the invariant (e.g. issuer action against Reserve allocations), the
federation MUST suspend issuance and switch redemption to a
published pro-rata mode.

## Compromise Containment

Assume the LSP is fully hacked: node keys, API keys, servers, RGB
stash. What can the attacker present to the federation?

| # | Attack path | Defense | Residual loss |
|---|---|---|---|
| 1 | Withdraw/reissue the LSP's federation balance with stolen credentials | One-way valve: CREDIT-OUT is cold-gated + allowlisted; reissue capped by hot-cap headroom | **0** |
| 2 | Force-close channels, redeem the LSP float on-chain (R1) | Co-op close needs the user's signature, so all closes are unilateral → LSP outputs locked `to_self_delay` blocks; mass closes auto-trigger an incident; lineage screening blocks redemption of those allocations before they are even spendable | ≈ 0 given `to_self_delay` ≥ incident-response SLA |
| 3 | Launder float over Lightning to mules who redeem clean-lineage tokens | Untraceable by design (LN transfers leave no lineage); bounded by the **federation-enforced** hot cap, throttled by per-epoch velocity caps, alarmed by outbound-flow anomalies | ≤ `max_lsp_hot_uusdt`, minus what fast incident response catches |
| 4 | Broadcast revoked states against offline users | Penalty enforcement + federation watchtowers | ≈ 0 |
| 5 | Redirect an in-flight redemption payout | Payout invoice bound at request creation | 0 |

**Security statement.** Given the one-way valve, destination
allowlisting, `to_self_delay ≥` the incident-response SLA,
watchtower coverage, and incident screening, the maximum extractable
value from a total LSP compromise is the Lightning-launderable
fraction of the hot float — at most `max_lsp_hot_uusdt`, a parameter
the federation enforces structurally at issuance time — with user
balances and the swept federation balance at zero exposure.
Everything beyond that requires compromising k federation members.

Wider failure analysis:

| Scenario | vUSDT holders | Reserve | Notes |
|---|---|---|---|
| LSP + up to k−1 members compromised | Safe | **Safe — no quorum can sign an invalid operation** | The design goal; degraded liveness at worst |
| n−k+1 members offline | Funds safe; LN unaffected | Frozen until recovery | Issuance/redemption pause; timelocked recovery as last resort |
| k members compromised (full quorum) | At risk | At risk, but rate-limited by epoch caps — and any unbacked issuance is *publicly visible* as an inflation-lineage event with no co-anchored deposit, so markets and LSPs can react immediately | Outside the threat model; mitigated by TEE diversity, attestation, velocity limits, public auditability |
| Issuer acts against Reserve allocations | Pro-rata mode | Partially frozen | The concentrated surface the working-asset design buys; policy published in advance |
| User loses RGB stash | That user cannot redeem | Unaffected | Mitigated by mandatory stash backup, below |

## User Data Availability

Flow R1 is the guarantee the whole design leans on, and it fails if
the user's RGB stash is gone when they arrive post-close. Client
wallets MUST maintain an encrypted stash backup (client-side key)
alongside channel-state backup; LSPs MUST offer stash-backup storage
and MUST replicate it to the federation on request, so the death of
an LSP does not take its users' redemption evidence with it.

## Transparency and Proof of Solvency

Every term of the solvency statement is verifiable against Bitcoin:

* **Issued (upper bound, complete):** the published inflation-right
  lineage — hidden issuance is a Bitcoin double-spend. Each
  issuance's backing deposit is co-anchored and cross-checkable.
* **Reserve (lower bound):** the published list of Reserve UTXOs
  with consignments proving each holds its stated canonical USDT;
  anyone verifies validity client-side and unspent-ness against
  Bitcoin. Completeness is unnecessary — undisclosed extra reserves
  only strengthen the position.
* **Pending burn (lower bound):** same mechanism over the
  federation's redemption/sweep seal UTXOs.
* **Burned:** the burn-right lineage. Public burn proofs reveal the
  burned tokens' transaction history, so the default is
  member-co-signed burn attestations published (amount + anchor
  txid) with full consignments escrowed to designated auditors;
  since burns only *reduce* circulating supply, this choice never
  weakens the public bound `reserves + pending_burn ≥ issued −
  attested_burns`.

`lsps10.fed.get_federation_info` MUST expose: member identities and
keys, n/k, enclave measurements + attestation evidence, all
published lineages and UTXO lists above, all caps and fees, the
incident registry, and per-member-signed log-head commitments.
Responses MUST be verifiable against member signatures — never
trusted from a front-end. Third-party monitors SHOULD track all of
it continuously.

## API (Draft)

Transport follows [LSPS0][]. Methods are namespaced `lsps10.*`
(client ↔ LSP) and `lsps10.fed.*` (anyone ↔ federation).

### lsps10.get_info (client ↔ LSP)

| JSON-RPC Method | lsps10.get_info |
|-----------------|-----------------|
| Idempotent      | Yes             |

```json
{
  "vusdt_asset_id": "rgb:2dkSTbr-jFhznbPmo-TQafzswCN-av4gTsJjX-ttx6CNou5-M9juwLv",
  "canonical_usdt_asset_id": "rgb:7bqNPmr-kQhzobRma-WRbfzsxDM-bv5hUtKkY-uux7DOpv6-N0kvxMw",
  "federation_endpoint": "https://fed.example.org",
  "fast_redemption": {
    "supported": true,
    "min_uusdt": "10000000",
    "max_uusdt": "5000000000",
    "fee_ppm": 3000,
    "payout_networks": ["rgb", "tron", "ethereum"]
  }
}
```

- `fee_ppm` [<LSPS0.ppm>][]

### lsps10.create_fast_redemption (client ↔ LSP, Flow R2)

| JSON-RPC Method | lsps10.create_fast_redemption |
|-----------------|-------------------------------|
| Idempotent      | No                            |

**Request**

```json
{
  "amount_uusdt": "250000000",
  "payout": { "network": "tron", "address": "TWd4WrZ9wn84f5x1hZhL4DHvk738ns5jwb" }
}
```

**Response**

```json
{
  "redemption_id": "9f1c7a2e-40cd-4c9e-9d2f-1c2a5b7e8d90",
  "invoice": "lnbcrt...",
  "fee_uusdt": "750000",
  "expires_at": "2026-08-20T12:19:06.991Z",
  "status": "AWAITING_PAYMENT"
}
```

`status ∈ {AWAITING_PAYMENT, HELD, PAID_OUT, SETTLED, EXPIRED,
REFUNDED}`. The LSP MUST NOT settle the HTLC before broadcasting the
payout.

### lsps10.fed.get_federation_info

Returns the transparency bundle
(§ [Transparency](#transparency-and-proof-of-solvency)).

### lsps10.fed.create_issuance / submit_issuance (Flow I)

**create_issuance request**

```json
{
  "amount_uusdt": "100000000000",
  "usdt_inputs": ["F27C97F46ED7281A3EFA7287410082EBA0CD1424D72703A217E435EA840957B0:0"],
  "usdt_consignments": ["<LSPS0.binary_blob>"],
  "vusdt_beneficiary_invoice": "rgb:2dkSTbr-...:utxob:..."
}
```

**create_issuance response**

```json
{
  "issuance_id": "a41f…",
  "psbt": "<LSPS0.binary_blob>",
  "fee_uusdt": "0",
  "expires_at": "2026-08-20T13:00:00.000Z"
}
```

`submit_issuance` carries the LSP-signed PSBT; the federation
validates (ISSUE checklist), countersigns, broadcasts, and returns
the txid and the vUSDT consignment.

### lsps10.fed.create_redemption / submit_redemption / get_redemption (Flow R1)

**create_redemption request**

```json
{
  "amount_uusdt": "250000000",
  "vusdt_inputs": ["B0570984EA35E417A20327D72414CDA0EB8200418772FA3E1A28D76EF4977CF2:1"],
  "vusdt_consignments": ["<LSPS0.binary_blob>"],
  "usdt_beneficiary_invoice": "rgb:7bqNPmr-...:utxob:..."
}
```

**create_redemption response**

```json
{
  "redemption_id": "7c1b…",
  "psbt": "<LSPS0.binary_blob>",
  "fee_uusdt": "750000",
  "expires_at": "2026-08-20T13:00:00.000Z"
}
```

`submit_redemption` carries the user-signed PSBT.
`get_redemption` returns `status ∈ {AWAITING_SIGNATURE, VALIDATING,
BROADCAST, COMPLETE, EXPIRED, REJECTED, SCREENED}` plus the txid
once broadcast. Reserve-UTXO reservations are TTL-bounded and
rate-limited (anti-DoS).

### lsps10.fed.create_sweep (LSP ↔ federation, Flow S)

Hot-callable; destination fixed to the federation sweep seal;
credits the LSP's federation balance on confirmation. Companions:
`lsps10.fed.get_balance`; `lsps10.fed.reissue` (hot, within hot-cap
headroom — a Flow I issuance debiting the balance);
`lsps10.fed.withdraw` (cold-signed, allowlisted destinations only);
`lsps10.fed.register_destination` (cold-signed, delayed);
`lsps10.fed.declare_incident` (cold-signed, or hot-proposed).

## Deferred: Lightning Gateway

A federation-operated Lightning node (TEE-held keys, VLS-style
policy signer) receiving redemptions in-Lightning was designed and
deliberately deferred: it is the only component whose TEE compromise
would carry direct loss, and it drags in collateral and slashing
machinery. Flows R1 + R2 cover the product surface. The
`lsps10.fed.create_ln_redemption` namespace is reserved.

## Open Questions

1. **Burn-proof publication scope** — public full consignments
   (maximum verifiability, leaks redeemed tokens' histories) vs
   co-signed attestations + designated auditors (this draft's
   default).
2. **Issuer rights interaction** — whether the canonical USDT RGB
   contract carries freeze/clawback over Reserve allocations, and
   the agreed policy (and any coordination with the issuer) when
   exercised.
3. **Reissue destination policy** — hot reissue direct to node seals
   within headroom (this draft: loss ceiling = hot cap, smooth
   operations) vs cold-registered destinations only (lower ceiling,
   heavier ceremony).
4. **Epoch and cap parameters** — issuance/burn batching cadence,
   velocity caps, `max_lsp_hot_uusdt` sizing, incident-response SLA
   vs `min_lsp_to_self_delay`.
5. **FROST migration** — timing for replacing MuSig2 + tapscript
   fallback with a threshold key path.
6. **Governance** — membership admission/removal process,
   recovery-quorum composition, parameter-change procedure (v1:
   static membership, ADMIN thresholds as specified).
7. **Fee schedule** — issuance free or at-cost; redemption fee as
   fixed + ppm; sweep costs borne by the LSP; concrete numbers TBD.
8. **DoS hardening** of Reserve-UTXO reservation during atomic
   redemption construction (request bonds vs rate limits).

## References

* [LSPS0][], [LSPS1][] — this repository.
* [BIP327][] — MuSig2.
* [RGB](https://rgb.tech) / RGB20 fungible interfaces — asset layer;
  IFA-style schemas for capped secondary issuance and burn.
* [VLS](https://vls.tech/) — Validating Lightning Signer; prior art
  for policy-enforcing (non-blind) signers.
* [Fedimint](https://fedimint.org/) — prior art for federated
  custody; differs in that LSPS10 keeps balances self-custodial on
  Lightning and uses the federation only for reserves, issuance,
  and redemption.

[LSPS0]: ../LSPS0/README.md
[LSPS1]: ../LSPS1/README.md
[LSPS0.common_schemas]: ../LSPS0/common-schemas.md
[BIP327]: https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
[<LSPS0.ppm>]: ../LSPS0/common-schemas.md#link-lsps0ppm
[<LSPS0.binary_blob>]: ../LSPS0/common-schemas.md#link-lsps0binary_blob
