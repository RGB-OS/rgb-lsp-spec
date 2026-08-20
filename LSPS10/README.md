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
practice this fails twice over. First, every LSP's hot Lightning
infrastructure would become a direct custodian of canonical
stablecoin with no systemic response available when — not if — an LSP
is compromised. Second, Lightning economics demand that an LSP place
large amounts of *inbound liquidity* on its side of user channels so
users can receive payments; requiring that idle receiving capacity to
be fully collateralized canonical USDT makes inbound liquidity
prohibitively expensive and concentrates enormous canonical value in
hot channel state.

This specification therefore separates three things that a naive
design conflates:

* **The reserve** — canonical USDT locked in cold Taproot UTXOs
  controlled by a k-of-n **Redemption Federation**, whose members
  hold key shares inside Trusted Execution Environments (TEEs).
* **The transport layer** — **vUSDT**, a single federation-issued
  RGB20 asset that moves through Lightning channels. vUSDT tokens
  are deliberately **unbacked transport**: the LSP obtains them
  without depositing collateral (within published caps) and uses
  them as channel liquidity, including the inbound capacity on its
  side of every user channel. Total vUSDT in existence exceeds the
  reserve **by design**.
* **The backing layer** — the federation's **attribution ledger**,
  which records who actually owns reserve-backed value. Backing
  enters the ledger only against confirmed canonical deposits, moves
  between identities only through federation-signed **attribution
  locks** that travel with Lightning payments, and leaves only
  through redemption or withdrawal. The attribution ledger — not
  token possession — is what the reserve owes.

A unit of vUSDT in a user's channel is money exactly when the user's
attribution balance covers it. Tokens without attribution — including
the LSP's entire liquidity float, and anything a hacked LSP pushes to
accomplices — are **not redeemable and are refused at the point of
receipt by compliant wallets**. Any holder of tokens *with*
attribution can redeem into canonical USDT at the federation in a
single atomic Bitcoin transaction, **without the cooperation of any
LSP**.

The federation is a **validated signer, not a blind signer**: every
member independently re-validates every fact — Bitcoin chain state,
RGB consignments, ledger invariants, incident policy — before
contributing its signature share to any operation, so a compromised
LSP colluding with up to k−1 federation members gains nothing.

### Why tokens and backing are separate layers

> **Rationale**
> 1. **Inbound liquidity becomes free of collateral.** The LSP's
>    receiving capacity is transport, not money; provisioning it
>    requires no locked canonical USDT. This is the economic reason
>    vUSDT exists at all.
> 2. **Cold reserves, hot transport.** Only valueless-until-attributed
>    tokens are hot in channels; canonical USDT stays in k-of-n cold
>    custody.
> 3. **A hacked LSP cannot manufacture money.** Because Lightning
>    balance shifts leave no lineage, no token-level mechanism can
>    distinguish a legitimately received token from one a hacked LSP
>    pushed to a mule. Putting backing in a ledger the federation
>    validates — and requiring wallets to verify backing *before*
>    accepting payment — is what makes fake vUSDT both unredeemable
>    and undeceptive.
> 4. **Issuer-action insulation.** Issuer-side interventions on the
>    canonical contract hit the reserve interface, handled by
>    published federation policy, not individual user channels.

## Design Goals

1. **Self-custody.** The user's channel balance MUST be spendable
   and recoverable unilaterally (standard Lightning penalty and
   force-close mechanics), and the user's backed value MUST survive
   the disappearance of their LSP.
2. **Full reserving of backed value, provable by anyone.** At all
   times `reserves ≥ Σ attribution balances + Σ active locks`, with
   **both sides** of the inequality derivable from Bitcoin-anchored
   data (§ [Proof of Solvency](#transparency-and-proof-of-solvency)) —
   not merely attested. Transport supply is intentionally NOT bounded
   by reserves; it is bounded by published per-LSP caps that are
   equally auditable.
3. **LSP-independent redemption.** Any holder of vUSDT tokens with
   matching attribution MUST be able to redeem into canonical USDT
   with no LSP cooperation, atomically in a single Bitcoin
   transaction.
4. **Receive-time protection.** A compliant wallet MUST be able to
   determine, *before releasing the payment preimage*, whether an
   incoming payment carries backing — so unbacked value is refused at
   receipt, not discovered at redemption.
5. **Validated signing.** No federation member ever signs a message
   it has not independently derived from locally validated state.
6. **No single point of theft.** Compromise of an LSP **plus up to
   k−1 federation members** MUST NOT enable theft of reserves,
   attribution not derived from deposits, or redemption of unbacked
   tokens.
7. **Quantified LSP blast radius.** Total LSP compromise MUST be
   bounded by the LSP's own attribution (dominated by its
   velocity-limited hot grant budget), never by its transport float
   and never by user backed balances
   (§ [Compromise Containment](#compromise-containment)).
8. **Explicit degradation.** With up to n−k members unavailable,
   all federation operations continue through the k-of-n fallback
   path. Beyond that (n−k+1 or more offline), user custody and
   in-channel token movement MUST remain unaffected; backed-payment
   finality degrades along the explicit policy of
   § [Liveness](#liveness-and-degraded-operation), and issuance and
   redemption pause.

## Definitions

### Data types

This specification uses data types defined in
[LSPS0 Common Schemas][LSPS0.common_schemas].

###### Link: LSPS10.uusdt

Amounts of USDT, vUSDT, and attribution MUST be expressed in
**micro-USDT** (`uusdt`, 10⁻⁶ USDT), encoded as a JSON string
containing the decimal representation, with field names suffixed
`_uusdt`. For example, 250 USDT is `"250000000"`.

The vUSDT contract MUST use precision 6 so one contract unit equals
one micro-USDT. The canonical USDT contract's precision is fixed by
its issuer and outside this specification's control: **precision 6 on
the canonical contract is a deployment prerequisite** — conformant
deployments MUST verify it against the canonical contract's genesis
and MUST NOT deploy against a canonical contract with a different
precision unless an explicit unit-conversion factor is added to every
cross-contract amount comparison in this specification.

### Actors

* **User (Client)** — holds vUSDT in a Lightning channel with an LSP
  and backed value in the attribution ledger; the API consumer.
* **LSP** — Lightning Service Provider. Provides vUSDT channels and
  liquidity (per [LSPS1][]). Holds vUSDT transport float (channel
  liquidity plus on-chain float — unbacked by definition), its own
  attribution balance at the federation, transient canonical USDT
  UTXOs being deposited via Flow D, and USDT on exchanges for market
  operations and fast redemptions. The LSP is *not* a custodian of
  user funds and is *not* trusted for redemption or backing.
* **Federation** — n independent members, threshold k, custodying
  the reserve, exclusively issuing vUSDT, and operating the
  attribution ledger. Trusted collectively (k-of-n), never
  individually.
* **Federation Member** — one operator running the member stack
  (Bitcoin + RGB validators, replicated ledger, policy signer)
  inside a TEE.

### Assets and accounts

* **Canonical USDT** — issuer-native USDT as an RGB asset on
  Bitcoin. The reserve asset. Never placed in channels.
* **vUSDT** — the single federation-issued RGB20 transport asset
  shared by all registered LSPs. All vUSDT is issuance-class
  *transport*: unbacked, capped per LSP, and worthless without
  attribution.
* **Reserve** — federation-controlled Taproot UTXOs holding
  canonical USDT allocations.
* **Attribution balance** `A[i]` — the ledger balance of identity
  `i`: reserve-backed value owned by `i`. Identities are
  secp256k1 public keys; wallets SHOULD use channel-scoped
  pseudonymous keys derived from channel keys
  (§ [Privacy](#privacy-considerations),
  § [User Data Availability](#user-data-availability)).
* **Attribution lock** — a federation-signed, TTL-bounded
  commitment of part of a sender's attribution to a specific
  payment hash and beneficiary key
  (§ [Flow P](#flow-p--backed-payment-over-lightning)).
* **Hot grant budget** — the velocity-limited sub-account of an
  LSP's attribution from which its operational infrastructure may
  create locks; refilled from the LSP's main attribution only by
  cold signature (§ [LSP Registration](#lsp-registration-and-authority-model)).
* **Transport outstanding** `T[LSP]` — vUSDT issued to an LSP minus
  vUSDT of its returned float burned; bounded by `transport_cap[LSP]`.
* **Pending burn** — vUSDT received at federation seals (redeemed or
  swept) and not yet batch-burned.

## Architecture Overview

```
              Lightning Network (vUSDT transport in channels)
   ┌──────┐        LN         ┌───────────┐
   │ User │◄─────────────────►│    LSP    │──── USDT on exchanges
   └──┬───┘  payments carry   └─────┬─────┘     (bridging, Flow R2)
      │      attribution locks      │
      │                             │ transport issuance (Flow T)
      │ atomic redemption (R1):     │ deposits (Flow D)
      │ tokens + attribution        │ float sweep/burn (Flow S)
      ▼                             ▼
 ┌────────────────────────────────────────────────────────┐
 │                Bitcoin + RGB (one layer)               │
 │                                                        │
 │  vUSDT contract:            canonical USDT:            │
 │  transport tokens,          locked in Reserve UTXOs    │
 │  linear issuance/burn       (Taproot MuSig2 key path,  │
 │  lineages, per-LSP caps     k-of-n script fallback)    │
 │                                                        │
 │  Attribution ledger epoch roots: anchored via a        │
 │  single-use federation seal chain                      │
 └───────────────────────────┬────────────────────────────┘
                             │ validated k-of-n signing
 ┌───────────────────────────┴────────────────────────────┐
 │                 Redemption Federation                  │
 │   Member 1 … Member n — each runs, inside a TEE:       │
 │    • Bitcoin full node + RGB validator                 │
 │    • Replicated ledger: attributions, locks,           │
 │      transport caps, incident registry                 │
 │    • Policy-enforcing threshold signer                 │
 └────────────────────────────────────────────────────────┘
```

Trust structure:

* Users and LSPs trust the **federation collectively** (k-of-n,
  TEE-hardened, publicly attested, velocity-limited, Bitcoin-audited).
* The federation extends the LSP **no redemption trust and no
  backing trust**. Its only exposures to an LSP are (a) transport
  float — unredeemable by construction — and (b) the LSP's hot grant
  budget, which is the LSP's *own* attributed money.
* A hacked LSP can push tokens anywhere; it cannot move backing it
  does not own, and compliant recipients refuse token flows that
  arrive without backing.

## Token Layer (vUSDT)

vUSDT MUST use an RGB schema supporting capped secondary issuance and
provable burn (the IFA-style schemas of RGB v0.11+).

* **Genesis** creates no supply and MUST set the maximum supply to
  the schema maximum (2⁶⁴ − 1 units), since burn-and-reissue cycles
  permanently consume headroom.
* **Issuance** (Flow T) consumes the single-use **inflation-right
  seal** on a federation UTXO and reassigns the remaining allowance
  to a fresh federation seal, forming a *linear, Bitcoin-anchored
  lineage*: closing a seal twice is a Bitcoin double-spend, so the
  lineage records every issuance exactly once. Every issuance
  operation MUST embed (in anchored operation metadata) a reference
  to its **transport authorization**: the ledger entry naming the
  beneficiary LSP, amount, and that LSP's post-issuance
  `T[LSP] ≤ transport_cap[LSP]`. The federation MUST publish the
  transport ledger and caps, so auditors can verify **every**
  issuance against a published, cap-bounded authorization — a rogue
  quorum minting outside the caps is publicly visible.
* **Publication duties.** Hidden issuance is impossible (the seal
  either moved or it did not), but an *unpublished* issuance is only
  detectable if watched: the federation MUST publish each lineage
  extension within a fixed SLA of confirmation, every rights-bearing
  transition MUST reveal **all** of its outputs (no concealed
  allowance splits; allowance conservation MUST be checkable across
  each published step; the set of allowed inflation shards is fixed
  and published), and third-party monitors MUST track the current
  inflation- and burn-right head UTXOs and alarm on any spend
  without timely publication.
* **Burns** (Flow S, Flow R1 residue) are executed in epoch batches
  over vUSDT accumulated at federation seals, consuming the
  burn-right seal. For each burn epoch the federation MUST publish
  the burn anchor txid and the exact set of pending-burn UTXOs
  consumed; monitors MUST verify that pending-burn UTXOs are only
  ever spent in published burn transactions.
* **Confirmation depth.** Members MUST NOT treat any witness
  transaction in a presented consignment, deposit, or sweep as final
  below a normative minimum depth (RECOMMENDED 6), and MUST validate
  presented consignments across their full DAG to the vUSDT genesis
  using the locally held rights lineages.

Transport caps SHOULD be priced (a fee or bond proportional to
`transport_cap`, sized against demonstrated receive volume) so caps
track genuine liquidity need rather than defaulting to the maximum.

> **Rationale** Transport tokens being unbacked makes their theft
> uninteresting — but *free and unlimited* transport would still let
> a hacked LSP flood the ecosystem with tokens that non-compliant
> wallets might accept. Caps bound that pollution; pricing keeps the
> caps honest.

## Attribution Layer

The attribution ledger is the money. It is replicated at every
member, mutated only by the closed operation set below, checked
against the solvency invariant on every entry, and anchored to
Bitcoin every epoch.

**Operations** (each with its validation checklist in
§ [Federation Security Model](#federation-security-model)):

| Op | Effect | Authorized by |
|---|---|---|
| `deposit` | +A[depositor] | Confirmed canonical USDT into the Reserve (Flow D) |
| `lock` | A[sender] → lock escrow | Sender identity key (LSPs: hot grant budget rules) |
| `claim` | lock escrow → +A[beneficiary] | Beneficiary signature + payment preimage |
| `expire` | lock escrow → sender **re-lockable credit** | Lock TTL passing (with tolling) |
| `redeem` | −A[identity], reserve outflow | Flow R1 (atomic swap) |
| `withdraw` | −A[LSP], reserve outflow | LSP **cold** signature, allowlisted destination |
| `refill` | A[LSP] main → hot grant budget | LSP **cold** signature |
| `settle` | A[LSP₁] → A[LSP₂] | Both LSPs' cold signatures (explicit inter-LSP settlement) |

Expired-lock value returns as **re-lockable credit** (spendable only
via new locks, not withdrawable or redeemable directly) to limit the
accumulation of free-floating attribution surplus
(§ [Compromise Containment](#compromise-containment), residual 3).

**Epoch anchoring.** Each epoch (RECOMMENDED order of minutes), the
federation commits the ledger: a Merkle-sum tree over
`(identity, A[identity])` plus lock-escrow and re-lockable-credit
totals, with range-proofed sums (no negative or overflowing
balances). The epoch root is committed on Bitcoin by closing a
single-use **ledger seal** chained exactly like the inflation seal —
so publishing two roots for one epoch is a visible double-spend, and
an inclusion proof from any past epoch convicts any later shaving of
that balance. Every member co-signs the root; wallets MUST verify
inclusion of their own balances each epoch and alarm loudly on
failure or omission.

**Conservation.** Across epochs,
`Δ(Σ A + escrow + credits) = deposits − redemptions − withdrawals` —
and every term on the right is an on-chain observable event. An
auditor therefore verifies the ledger total without trusting the
federation: fabricated attribution breaks publicly checkable
conservation.

**Throughput.** Ledger operations are validated by every member but
threshold-signed with preprocessed low-latency signing (e.g. FROST
with precomputed nonce pairs); per-payment lock issuance is a ledger
operation, not an on-chain ceremony. Deployments MUST publish a
throughput budget, per-identity rate limits, and a minimum
certifiable delta: wallets aggregate settled dust below it into
periodic claims (displayed as "pending aggregation", not unbacked).

## Reserve Custody

### Reserve and rights UTXOs

Reserve UTXOs (canonical USDT), the vUSDT inflation- and burn-right
UTXOs, and the ledger-seal UTXO MUST be Taproot outputs with:

* **Key path**: MuSig2 ([BIP327][]) n-of-n aggregate of all member
  keys — normal cooperative operation.
* **Script paths**: a k-of-n `OP_CHECKSIGADD` tapscript as the
  liveness fallback, and OPTIONALLY a timelocked
  (`OP_CHECKSEQUENCEVERIFY`) recovery path to a designated recovery
  quorum. A FROST-style threshold key path MAY replace this
  construction once implementations mature.

Normal operation signs through the key path with all n members; when
up to n−k members are unavailable, k-quorums sign through the
tapscript fallback (latency may increase, operations continue).
Ledger operations — locks, claims, epoch roots — are threshold-signed
at k throughout.

The public reserve lower bound (§ Proof of Solvency) is conditional
on the canonical contract's issuer rights: if the canonical schema
supports freeze or clawback via issuer-side global state, an
allocation can be impaired without its UTXO being spent. Members and
auditors MUST track canonical-contract issuer operations affecting
Reserve allocations; resolving the canonical schema's exact rights is
a deployment prerequisite (see
[Open Questions](#open-questions)).

### TEE requirements

Each member MUST run its validators, ledger, and signer inside a
TEE, with remote attestation mutually verified at DKG time and on
every upgrade (evidence published via
`lsps10.fed.get_federation_info`), key shares generated by dealer-less
DKG inside the enclave and never exported, and enclave measurements
corresponding to publicly reproducible builds of open-source member
software. The member stack validates only Bitcoin and RGB — a
single-chain TCB small enough to audit credibly.

TEEs are **defense in depth, not the security model**: the protocol
MUST remain secure assuming any single TEE can be fully broken.
Members SHOULD diversify TEE vendors, hosting, and jurisdictions.

### Membership parameters

n MUST be ≥ 4; n = 7 with k = 5 is RECOMMENDED. Theft requires k
compromised members; liveness loss requires n−k+1 failures. k MUST
exceed n/2 so two disjoint quorums cannot exist.

### Federation identity anchor

The federation's identity MUST NOT be learned from an LSP or a
front-end — otherwise a malicious LSP could point wallets at a sham
federation whose forged locks make counterfeit vUSDT appear backed,
defeating Flow P at the root. The vUSDT contract genesis MUST commit
the initial member key set, n and k, the lock-verification key, and
the genesis seals of the rights and ledger chains; every membership
or key rotation thereafter is an ADMIN operation anchored through
the ledger seal chain. A wallet that has validated the vUSDT asset
therefore derives the current federation keys from Bitcoin-anchored
history alone: it MUST pin the asset id, resolve the member set from
genesis plus anchored rotations, and treat every endpoint — including
`federation_endpoint` from `lsps10.get_info` — as untrusted transport
whose responses are checked against the derived keys.

## LSP Registration and Authority Model

Every LSP registers before receiving transport issuance. Registration
binds two authorities:

* **Hot identity** — keys on operational infrastructure (node key,
  API keys, lock-signing key for the hot grant budget). Assumed
  stealable.
* **Cold authority** — a key or officer-key quorum enrolled in an
  offline ceremony, never on operational servers.

The governing principle is a **one-way valve with a metered spout**:

* Toward the federation, or risk-reducing: hot-authorized — sweeps
  (fixed destination), incident *proposals*.
* Out of federation control: cold-gated — `withdraw` (allowlisted
  destinations registered cold), `refill` of the hot grant budget,
  `settle`. A destination addition takes effect only after a
  mandatory delay during which it is **published in the signed event
  log and delivered to federation ADMIN** — a delay nobody can act
  on is decorative — and any member MAY propose an incident against
  a contested addition, freezing the LSP's `withdraw` until
  resolved. `withdraw` MUST additionally be delayed by at least the
  maximum lock validity, so in-flight grants land before balance
  leaves.
* The metered spout: day-to-day grants to customers come from the
  **hot grant budget** — capped in size, velocity-limited per epoch,
  and refillable only cold. A total hot compromise can drain at most
  this budget per epoch until an incident lands, and it is the LSP's
  own money.

Registration fixes, per LSP: `transport_cap_uusdt` (and its
pricing), `hot_grant_budget_uusdt` and its per-epoch velocity limit,
`sweep_trigger_uusdt` and the sweep baseline (§ Flow S),
`min_lsp_to_self_delay` / `max_user_to_self_delay`
(§ [Channel Requirements](#lightning-channel-requirements)), the
allowlist of withdrawal destinations, and — **mandatory for any LSP
advertising fast redemption** — a bond sized against its advertised
maximum in-flight R2 exposure, slashable on adjudicated R2 fraud.

## Protocol Flows

### Flow D — Deposit

The LSP (or any registered identity) transfers canonical USDT to a
fresh federation Reserve seal (RGB invoice from
`lsps10.fed.create_deposit`). After the normative confirmation depth,
each member independently verifies the consignment and credits
`A[depositor]`. Canonical USDT change from the depositor's inputs
returns to a depositor-controlled seal — checklists check **value
conservation on the federation-bound amount**, never "all change to
federation seals". Deposits MAY be funded directly from exchange
withdrawals; LSP-held canonical UTXOs pending deposit SHOULD be
minimized and MAY be cold.

The federation extends zero trust here; the depositor's exposure is
custody of the locked reserve — the k-of-n TEE story.

### Flow T — Transport issuance

The LSP requests vUSDT transport (no deposit). Members verify the
beneficiary is registered and unsuspended, the authorization is
entered in the published transport ledger, and post-state
`T[LSP] ≤ transport_cap[LSP]`; a quorum then executes the inflation
transition to the LSP's seal. Multiple authorizations MAY be batched
into one issuance transaction; the checklist applies
**per-beneficiary** (each beneficiary's issued amount equals its
authorization), not merely in sum.

The LSP deploys transport into channels as `lsp_balance` — users'
inbound capacity — per [LSPS1][]. Channel-side transport float is
bounded at channel open by capping `lsp_balance`, not by sweeping
live channels (see Flow S).

### Flow P — Backed payment over Lightning

The heart of the protocol: how backing moves with a payment, and how
a recipient knows — *before releasing the preimage* — that a payment
is real.

1. **Lock.** The sender's wallet requests an attribution lock:
   `{lock_id, amount, payment_hash, beneficiary_key, expiry}`,
   threshold-signed by the federation, debiting `A[sender]` (for an
   LSP sender: its hot grant budget) into escrow. `expiry` MUST be at
   least the final-hop CLTV deadline plus a fixed margin, and each
   payment attempt MUST use a fresh hash and lock. The
   `beneficiary_key` is taken from the invoice (an attribution key
   the recipient controls).
2. **Attach.** The sender attaches the lock proof (the signed lock
   statement) to the payment's final-hop TLV payload.
3. **Verify, then settle.** The recipient's wallet verifies the lock
   offline against the federation keys derived from the identity
   anchor (§ [Federation identity anchor](#federation-identity-anchor)):
   signature valid,
   amount ≥ invoice amount, payment hash matches, beneficiary key is
   its own, expiry leaves at least a safety window. A compliant
   wallet **MUST NOT release the preimage** — i.e. MUST fail the
   HTLC — if any check fails or the lock proof is absent, and MUST
   NOT release the preimage inside the safety window of lock expiry.
   vUSDT received outside this flow (keysend, non-compliant senders,
   bare on-chain transfers without § re-attribution) carries **no
   redemption right** and MUST be flagged as unbacked.
4. **Claim.** After settlement the recipient claims the lock with
   `{lock_id, preimage, signature by beneficiary_key}` — valid
   **independent of channel liveness**, so a force-close between
   settlement and claim cannot strand an honest recipient. Claims
   are recorded against a nullifier set (no replay) and credited to
   `A[beneficiary]` in the next epoch delta, with an inclusion proof
   at the epoch anchor. Unclaimed locks expire to the sender's
   re-lockable credit; expiries **toll** (pause) during any interval
   in which published log-heads show the federation below quorum.

Routing hops move only transport tokens; **no intermediary can steal
the backing** — the lock names the beneficiary key, and the preimage
alone claims nothing. Routed payments never touch any LSP's
attribution: the inter-LSP channel shift is unbacked float moving,
so no per-payment netting exists (explicit `settle` operations,
cold-signed, handle inter-LSP rebalancing). Routing fees on vUSDT
channels SHOULD be denominated in sats so the forwarded vUSDT amount
is conserved end-to-end and equals the lock amount.

**On-chain transfers** between users reuse the same machinery: the
recipient's wallet issues its RGB invoice only after verifying a
federation lock bound to its attribution key and the anticipated
transfer, claimable by the confirmed transfer's anchor txid instead
of a preimage.

> **Rationale — locks before payment, not receipts after.** A
> preimage is revealed to every routing hop at settlement, so any
> "claim with preimage" scheme without a named beneficiary is
> stealable by intermediaries; and any scheme where backing is
> checked *after* settlement leaves the recipient holding
> irrevocably-received tokens while racing the sender's remaining
> balance. Locking before payment makes the recipient's verification
> instant, offline, and race-free — and makes "provisional funds" an
> exceptional state rather than the routine one, so the unbacked
> warning stays credible.

### Flow S — Float sweep and retirement

Sweeps bound the LSP's *on-chain* transport float and retire
transport supply. When on-chain float exceeds `sweep_trigger_uusdt`,
or at least every 24 hours (reducing float to the registered
baseline), the LSP MUST transfer the excess to the federation sweep
seal (hot-callable; destination fixed). Swept tokens join the
pending-burn pool and reduce `T[LSP]` at the next burn epoch.
**Sweeps never credit attribution**: token possession is not money,
so there is nothing to credit — the mis-crediting channel (sweep →
credit → withdraw) is closed structurally.

Channel-side float is NOT swept from live channels — no RGB channel
splicing exists today, and sweeping users' inbound capacity would
destroy their ability to receive. Channel float is bounded at open
(`lsp_balance` caps) and retired on channel close (closed-out LSP
float MUST be swept). An LSP that misses its sweep obligations is
refused further transport issuance and accumulates automatic
incident evidence.

### Flow R1 — Atomic on-chain redemption (base path, LSP-independent)

Available to any identity holding vUSDT tokens **and** matching
attribution — with no LSP involvement.

1. The user obtains an on-chain vUSDT allocation, normally by
   closing their channel. (Unilateral exit latency is the user-side
   `to_self_delay` plus confirmations, plus second-stage HTLC
   resolution — see § Channel Requirements — and requires the RGB
   stash, see § User Data Availability.)
2. The user calls `lsps10.fed.create_redemption` with the amount,
   their token inputs + consignments, their attribution identity
   (signed request), and a canonical USDT beneficiary — a
   witness-output seal of the redemption transaction itself is
   permitted, so a freshly force-closed user needs no other UTXO.
   The federation places a TTL-bounded hold on the attribution and
   on Reserve UTXOs (rate-limited against DoS) and returns the
   transaction template: user vUSDT → federation redemption seal
   (with vUSDT change back to a user change seal); canonical USDT
   (amount − fee) → the user's seal; canonical change → a fresh
   Reserve seal.
3. The user signs their inputs and submits. Each member
   independently reconstructs and validates the transaction under
   the [REDEEM checklist](#redeem) — including
   `A[identity] ≥ amount` with no conflicting hold — then a quorum
   completes the reserve-input signature and the transaction is
   broadcast. The swap is atomic; `A[identity]` is debited; the
   returned tokens await batch burn.

Presentations without matching attribution are refused with the
distinct status `UNBACKED` (published, appealable) — this is the
standing rule that makes fake vUSDT worthless, active at all times,
incident or no incident. An identity whose attribution is intact but
whose tokens were destroyed by a third party's fault MAY be made
whole through the dispute process (ADMIN): honoring attribution
without tokens is always reserve-safe, since solvency counts only
attribution.

### Flow R2 — Fast redemption over Lightning (LSP-fronted)

For instant exit to exchange rails without an on-chain footprint.
Ordering closes the three-leg race:

1. User calls `lsps10.create_fast_redemption` on the LSP with amount
   and payout address (any network the LSP supports).
2. The LSP returns a vUSDT hold invoice; the user's wallet obtains
   an attribution lock bound to the invoice hash with the LSP as
   beneficiary (this is the lock-verification step of Flow P with
   roles reversed).
3. The LSP verifies the lock, executes the payout, settles the HTLC
   after the payout is submitted/observable on the payout network,
   then claims the lock — so the attribution debit can never be
   front-run and the LSP is never unbacked-and-out-of-pocket.
4. The LSP's accumulated attribution is later withdrawn (cold) or
   re-granted; its accumulated tokens are swept per Flow S.

The federation is not on the payout path and extends no trust to the
LSP. The user's exposure is one in-flight HTLC. LSP fraud (settled
HTLC, no payout) is adjudicated by the federation ADMIN threshold on
submitted evidence of the foreign-chain state — explicitly outside
the signing TCB — and slashes the mandatory R2 bond; per-LSP
advertised R2 volume caps bound aggregate in-flight exposure. The
minimum invoice CLTV MUST exceed the payout-broadcast SLA, and a
missed SLA MUST fail the HTLC (refunding the user; status
`REFUNDED`).

A federation-operated Lightning gateway for in-Lightning redemption
remains **deferred** (§ [Deferred](#deferred-lightning-gateway)).

## Lightning Channel Requirements

Normative behavior for LSPS10 channels, stated in BOLT terms:

* **CSV delays.** `to_self_delay` is chosen by the counterparty. The
  client MUST propose `to_self_delay ≥ min_lsp_to_self_delay`
  (RECOMMENDED ≥ 144) for the LSP's outputs and MUST verify it in
  the resulting commitment scripts; the LSP MUST accept proposals up
  to a published maximum. Symmetrically, the client MUST reject an
  LSP-proposed user-side `to_self_delay` above
  `max_user_to_self_delay` (RECOMMENDED ≤ 2016), since that delay is
  the user's unilateral-exit latency.
* **Close discipline.** A hacked LSP can obtain immediately
  spendable outputs by co-op closing (BOLT2 `shutdown` handling is
  automatic) or by provoking the client's force-close. LSPS10
  clients MUST NOT auto-complete cooperative-close negotiation
  (rate-limit and require explicit consent or a cooldown) and MUST
  NOT auto-force-close on receipt of `error` on LSPS10 channels.
  With the attribution model these protections are defense in depth —
  tokens the attacker extracts are unredeemable — but they preserve
  users' receiving capacity and orderly exits.
* **Payment TLV.** Invoices carry the recipient's attribution key;
  final-hop TLV carries the lock proof; compliant recipients enforce
  Flow P step 3 unconditionally.
* **Fees.** Routing fees on vUSDT channels SHOULD be sats-denominated
  (see Flow P).

## Federation Security Model

### The validated-signer principle

Each member runs a policy-enforcing signer whose only input is its
co-located validator stack: a Bitcoin full node (SHOULD; at minimum
full validation of everything touching federation UTXOs and
presented consignments), an RGB validator holding the complete
rights lineages of both contracts, and the replicated ledger with
per-entry invariant checking and strict per-account serialization
(check-and-hold semantics; TTL-bounded holds from R1 requests and R2
quotes; no operation admitted against a conflicting hold).

**No member ever signs a hash it received from another party.** Each
member constructs the transaction or ledger entry to be signed from
its own validated state and signs only its own construction; the
threshold protocol fails harmlessly on divergence.

> **Rationale** This is what the headline guarantee rests on: a
> compromised LSP colluding with up to k−1 members cannot mint
> attribution, redeem unbacked tokens, or move reserves, because
> every honest member of any quorum re-derives and re-checks
> everything. A blind-signing federation collapses to a single point
> of failure the moment one coordinator or enclave breaks.

### Validation checklists (normative)

#### DEPOSIT
Consignment valid to canonical genesis; witness depth ≥ minimum;
full deposit amount to a federation-derived Reserve seal;
depositor-bound change permitted (value conservation on the
federation-bound amount); not previously credited; credit equals the
deposited amount.

#### TRANSPORT-ISSUE
Beneficiary registered and unsuspended; published authorization entry
exists; per-beneficiary issued amount equals its authorization;
inflation transition correctly consumes the current head seal with
full output reveal and allowance conservation; post-state
`T[LSP] ≤ transport_cap[LSP]`.

#### LOCK
Sender signature valid; `A[sender]` (or hot grant budget, within its
epoch velocity limit) covers the amount with no conflicting hold;
expiry within policy bounds; identity within rate limits; no active
incident freeze on the sender's grant path.

#### CLAIM
Lock exists, unexpired (with tolling), un-nullified; preimage matches
the lock's payment hash; signature verifies against the lock's
beneficiary key. (Channel state is irrelevant by design.)

#### REDEEM
Request unexpired, unpaid, within per-request and per-epoch velocity
caps; token consignments valid across the full DAG to genesis, inputs
unspent, witness depths ≥ minimum; `A[identity] ≥ amount` with no
conflicting hold; the member's own reconstruction pays exactly
`amount` in vUSDT to the redemption seal (user change permitted to a
user seal) and exactly `amount − fee` in canonical USDT to the
beneficiary bound at request time (reserve change to a fresh Reserve
seal); post-state solvency invariant holds. Attribution shortfall →
status `UNBACKED`, published, appealable.

#### BURN
Consumes the burn-right head correctly; burns only federation-seal
pending-burn allocations; publishes the anchor txid and exact
consumed UTXO set; reduces `T[LSP]` accounting exactly.

#### RESERVE-SPEND
Any other transaction spending federation UTXOs (consolidation, fee
management, key-rotation roll-overs, epoch ledger-seal closures)
returns every output and RGB allocation to federation-derived seals.

#### CREDIT-OUT
`withdraw`: cold signature valid; destination on the cold-registered
allowlist, past its addition delay and uncontested; amount within `A[LSP]` net of
escrow and holds; the mandatory delay ≥ maximum lock validity has
elapsed; post-state solvency invariant holds; no active incident for
the LSP. `refill` and `settle`: cold signatures; amounts within
balances; same invariant.

#### ADMIN
Membership changes, DKG resharing, enclave-measurement approvals,
parameter and cap changes, LSP registration/revocation, incident
declaration/renewal/lift, bond slashing, and dispute resolutions
require an administrative threshold of at least k (n−1 RECOMMENDED
for membership and measurement changes), each member validating the
proposal against published governance rules.

### Transaction construction

For every co-signed Bitcoin transaction (Flows D/T/S/R1, burns,
reserve spends), the specification of record MUST fix: the DBC
commitment scheme and host output (RECOMMENDED: Tapret on the
federation change output), deterministic input and output ordering,
the fee policy, and the exact PSBT and MuSig2 message sequence across
the `create_*`/`submit_*` calls — the multi-protocol commitment is
finalized **before** any signature is produced, so independent member
reconstructions are bit-identical by construction. Consignment
delivery to a beneficiary concludes with an explicit acceptance
acknowledgment (re-fetchable by `*_id`).

### Replicated ledger

Deterministic and replicated: any two honest members with the same
entries derive the same state. Byzantine agreement is not required —
signing is the only consequential act and already demands k
independent validations, so divergence safely halts signing. Members
MUST publish signed log-head commitments at least once per epoch;
divergent heads raise a public alarm. The nullifier set is
epoch-scoped with explicit tolling records and pruned after expiry
plus the dispute window; per-identity caps bound outstanding locks.

### Key generation and rotation

All threshold keys from dealer-less DKG inside enclaves. On
membership change or scheduled rotation the federation reshares and
rolls federation UTXOs under RESERVE-SPEND. A member with lapsed
attestation or divergent behavior is excluded from quorums until
re-attested.

### Watchtower duty

Federation members MUST jointly operate watchtowers for registered
LSPs' channels — the offline-user protection claims of this
specification are conditional on this coverage. Because a vUSDT
justice transaction must carry RGB state transitions for the revoked
outputs' allocations, clients MUST upload, per commitment update, an
encrypted justice blob including the RGB witness data needed to
construct the asset transition (the User Data Availability
replication feeds this). The same monitoring feeds incident
detection: a wave of unilateral closes from one LSP is loud,
on-chain, and automatic grounds for an incident proposal.

### Incident response

The standing defense against fake vUSDT is the attribution gate — it
is always on and needs no declaration. Incidents handle the
compromise of an LSP's *own* authority:

* **Declaration.** Hot credentials can only *propose* (a proposal
  must not freeze anything — otherwise it would be a DoS lever for
  the attacker). An incident takes effect on the LSP's cold
  signature or the federation ADMIN threshold, on evidence: mass
  unilateral closes, hot-grant velocity anomalies, attempted
  non-allowlisted withdrawals, missed sweeps.
* **Effects.** The LSP's hot grant budget, `refill`, `withdraw`,
  `settle`, and transport issuance freeze. Locks created **before**
  the incident timestamp are honored unconditionally — the lock
  record is an unforgeable pre-incident timestamp, so honest
  recipients paid before the hack are never stranded. Grants inside
  the incident window MAY be clawed back if the deployment enables a
  grant-finality delay; wallets then distinguish "certified
  (finalizing)" from "final". Sweeps remain allowed.
* **Scope.** Incident effects operate on **ledger state only** —
  never on token lineage. (Lineage cannot attribute channel-close
  outputs: every user's close output traces to the LSP's funding
  exactly like the attacker's float, so lineage screening would
  freeze the honest users of the very LSP whose customers most need
  Flow R1. The attribution gate makes it unnecessary.) Every
  incident action is published in the signed log with an appeal
  path, time-boxed with ADMIN renewal.

### Recovery

The timelocked recovery path allows a designated quorum to move
federation funds after a long CSV delay if the federation permanently
loses k-liveness. Because `OP_CSV` is relative to each UTXO's
confirmation, routine RESERVE-SPEND activity (key-rotation rolls at
least once per recovery period) automatically resets the recovery
clock: the path ripens only if the federation has been genuinely
unable to move funds for the entire window, so a DoS that merely
partitions members cannot open it while any k-quorum still functions.
Third-party monitors SHOULD alarm as any federation UTXO approaches
recovery ripeness. Recovery-quorum composition MUST be diverse and
independent of the member set (details:
[Open Questions](#open-questions)). If reserves are impaired below the invariant (e.g.
issuer action), the federation MUST suspend deposits and locks and
switch redemption to a published **pro-rata mode over attribution
balances** (never over token holdings).

## Compromise Containment

Assume the LSP is fully hacked: node keys, API keys, servers, stash,
lock-signing key. What can the attacker monetize?

| # | Attack path | Defense | Residual loss |
|---|---|---|---|
| 1 | Withdraw / refill / settle with stolen credentials | Cold-gated + allowlist + delay ≥ max lock validity | **0** |
| 2 | Push transport float to mules over LN and redeem | Mules without locks hold unredeemable tokens (standing attribution gate); compliant wallets refuse the payments outright at receipt | 0 from reserves; ecosystem pollution bounded by `transport_cap` |
| 3 | Grant attribution to mules from the hot budget, mules redeem | It is the LSP's own money; bounded by hot-budget size × epoch velocity × epochs until incident; monitoring on grant anomalies | ≤ hot grant budget per epoch until incident |
| 4 | Close channels (co-op or provoked) and take outputs on-chain | Outputs are transport tokens — unredeemable without attribution; close-discipline rules and `to_self_delay` slow the mechanics as defense in depth | ≈ 0 from reserves |
| 5 | R2: settle user HTLCs without paying out | Lock-before-settle ordering removes the attribution race; remaining fraud is bounded by advertised in-flight volume and recovered from the mandatory bond | ≤ in-flight R2 exposure, bond-covered |
| 6 | Broadcast revoked states against offline users | Penalty enforcement via mandatory federation watchtowers with RGB justice blobs | ≈ 0 given coverage |
| 7 | Redirect an in-flight redemption or claim | Payout beneficiary bound at request time; claims need the beneficiary key | 0 |

**Security statement.** With the one-way valve, lock-before-payment,
the standing attribution gate, mandatory watchtower coverage, and
close discipline, the maximum extractable value from a total LSP
compromise is the LSP's **own** attributed funds reachable through
the hot grant budget (velocity-limited per epoch until an incident
lands) plus bonded in-flight R2 exposure. User backed balances and
the reserve are at zero direct exposure. Everything beyond that
requires compromising k federation members.

**Honest residuals.** (1) Recipients on *non-compliant* wallets can
still accept worthless tokens — the boundary of the guarantee is the
compliant ecosystem, which is why bare vUSDT receipt outside Flow P
carries no redemption right and why asset-registry metadata should
carry that warning. (2) Attribution and tokens are separately
transferable, so holders of attribution surplus could buy stolen
float at a discount and marry it to their own backing at R1 — the
system stays solvent (their backing is consumed), but theft retains
a discounted resale value; re-lockable-credit rules minimize surplus
creation. (3) During the detection window, honest users of the
hacked LSP experience frozen grants and paused issuance, not lost
funds.

Wider failure analysis:

| Scenario | Backed holders | Reserve | Notes |
|---|---|---|---|
| LSP + up to k−1 members compromised | Safe | **Safe — no quorum can sign an invalid operation** | The design goal; degraded liveness at worst |
| n−k+1 members offline | Funds safe; token payments continue; backed finality per § Liveness | Frozen until recovery | Deposits, locks, claims, redemptions pause; lock expiries toll |
| k members compromised (full quorum) | At risk, rate-limited | At risk, rate-limited | Epoch velocity caps; and every theft channel is publicly visible: issuance outside the published caps, attribution breaking on-chain conservation, epoch-root equivocation as a Bitcoin double-spend |
| Issuer acts against Reserve allocations | Pro-rata mode over attributions | Partially impaired | The concentrated surface the transport/backing split buys; policy published in advance |
| User loses stash / attribution key | That user cannot redeem | Unaffected | Mitigated by § User Data Availability |

## Liveness and Degraded Operation

Federation liveness is on the backed-payment finality path — this
specification says so plainly rather than pretending otherwise.
During an outage (published log-heads below quorum):

* Token movement in channels continues; custody is unaffected.
* New locks cannot be issued. Wallets MAY spend **pre-fetched
  sender-bound blank locks** (issued in advance against the sender's
  balance, bound to the sender, with the payment binding
  countersigned by the sender at pay time). Blank locks carry a
  stated residual risk — a malicious sender can double-bind one lock
  until the federation returns — so per-sender blank-lock exposure
  is capped and recipients treat blank-lock payments as
  strong-provisional.
* Lock and claim validity **tolls** across the outage with a grace
  window on resumption.
* Deposits, claims, redemptions, and withdrawals queue until quorum
  returns.

## User Data Availability

Flow R1 is the guarantee the design leans on; it needs three things
at exit time, all of which MUST be covered by the mandatory
client-side-encrypted backup: the **RGB stash**, the **attribution
identity keys** (RECOMMENDED derived from channel keys so channel
recovery implies attribution recovery), and **outstanding lock
state**. LSPs MUST store clients' encrypted backups and MUST
replicate them to the federation proactively on every update or on a
fixed schedule — an on-request duty replicates nothing when the LSP
dies first. The federation exposes an authenticated resync query
(balance, inclusion proof, outstanding locks per identity) so a
restored wallet cannot over-commit its attribution. Per-commitment
encrypted justice blobs (§ Watchtower duty) ride the same channel.

## Transparency and Proof of Solvency

Public invariant: `reserves ≥ Σ attributions + escrow + credits`,
with both sides Bitcoin-derived:

* **Reserve (lower bound):** published Reserve UTXO list with
  consignments proving each allocation; anyone verifies validity
  client-side and unspent-ness against Bitcoin. Completeness is
  unnecessary (hidden reserves only help). Conditional on the
  canonical contract's issuer rights (§ Reserve Custody).
* **Ledger total (exact, conserved):** the epoch root chain anchored
  through the single-use ledger seal — equivocation is a visible
  double-spend, history is immutable, prior inclusion proofs convict
  later shaving — plus the conservation identity
  `Δtotal = deposits − redemptions − withdrawals`, whose right side
  consists of on-chain events. Wallets verify their own inclusion
  every epoch.
* **Transport supply (complete):** the published inflation lineage
  with full-reveal and allowance-conservation rules; every issuance
  matched to a published cap-bounded authorization. Burns via
  published anchor txids and consumed-UTXO sets; pending burn via
  the same UTXO+consignment mechanism as the reserve. Disclosure
  policy is uniform: where consignment publication would leak user
  histories, member-co-signed attestations with consignments
  escrowed to designated auditors are used instead, and the affected
  term is stated as auditor-verifiable rather than
  anyone-verifiable. Note transport supply is an integrity metric,
  not a solvency term — solvency is carried entirely by the
  attribution ledger.

`lsps10.fed.get_federation_info` MUST expose: member identities and
keys, n/k, enclave measurements and attestation evidence, reserve
UTXO list, epoch root chain head, transport ledger and caps, all
parameters and fees, per-LSP certification latency and R2 caps, the
incident registry, and per-member-signed log-heads. Responses MUST
be verifiable against member signatures — never trusted from a
front-end. Third-party monitors SHOULD track all of it continuously,
including the rights-seal and ledger-seal head UTXOs.

## Privacy Considerations

The attribution ledger gives the federation a view of backed
balances and payment deltas per attribution identity. This
specification reduces — but does not eliminate — the resulting
surveillance surface: identities are channel-scoped pseudonymous
keys, unlinkable across channels by construction; the federation
sees lock amounts and timing but not invoice content or the
token-level route; and every incident-related ledger action is
public. The honest statement is that vUSDT offers Lightning-grade
privacy toward the world and pseudonymous-ledger privacy toward the
federation. A blind-credential attribution design (fixed-denomination
notes with nullifiers) would remove the per-identity view but is
mutually exclusive with per-identity inclusion proofs; it remains an
open question.

## API (Draft)

Transport follows [LSPS0][]: REST JSON APIs, with the method names
below mapping to endpoint paths (e.g. `POST /lsps10/get_info`).
Methods are namespaced `lsps10.*`
(client ↔ LSP) and `lsps10.fed.*` (anyone ↔ federation). All
`*_uusdt` fields are [<LSPS10.uusdt>][]; all `*_at` fields are
[<LSPS0.datetime>][]; `psbt` and `*_consignment*` fields are
[<LSPS0.binary_blob>][]; `*_inputs` fields are lists of
[<LSPS0.outpoint>][].

### lsps10.get_info (client ↔ LSP)

| Method     | lsps10.get_info |
|------------|-----------------|
| Idempotent | Yes             |

```json
{
  "vusdt_asset_id": "rgb:2dkSTbr-jFhznbPmo-TQafzswCN-av4gTsJjX-ttx6CNou5-M9juwLv",
  "canonical_usdt_asset_id": "rgb:7bqNPmr-kQhzobRma-WRbfzsxDM-bv5hUtKkY-uux7DOpv6-N0kvxMw",
  "federation_endpoint": "https://fed.example.org",
  "fast_redemption": {
    "supported": true,
    "min_uusdt": "10000000",
    "max_uusdt": "5000000000",
    "max_inflight_uusdt": "20000000000",
    "fee_ppm": 3000,
    "payout_networks": ["rgb", "tron", "ethereum"],
    "payout_sla_seconds": 600
  }
}
```

- `fee_ppm` [<LSPS0.ppm>][]

### lsps10.create_fast_redemption / lsps10.get_fast_redemption (client ↔ LSP, Flow R2)

`create_fast_redemption` request/response as below;
`get_fast_redemption` polls by `redemption_id` and is where the full
status enum lives:
`AWAITING_LOCK → AWAITING_PAYMENT → HELD → PAID_OUT → SETTLED`,
with `EXPIRED` and `REFUNDED` (payout SLA missed or lock invalid —
HTLC failed) as terminal failures.

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
  "lsp_attribution_key": "02c9d1afd383fb651d93975d2a2e727aca5e6dd4fa2f090fded50102a149969ce8",
  "fee_uusdt": "750000",
  "expires_at": "2026-08-20T12:19:06.991Z",
  "status": "AWAITING_LOCK"
}
```

- `lsp_attribution_key` [<LSPS0.pubkey>][] — beneficiary key for the
  user's attribution lock (Flow R2 step 2).

### lsps10.fed.get_federation_info

The transparency bundle
(§ [Transparency](#transparency-and-proof-of-solvency)).

### lsps10.fed.create_deposit (Flow D)

Request `{ "amount_uusdt", "depositor_identity" }` → response
`{ "deposit_id", "reserve_rgb_invoice", "min_confirmations",
"expires_at" }`. The depositor transfers canonical USDT to the
invoice; attribution is credited after validation.

### lsps10.fed.create_transport_issuance / submit_transport_issuance (Flow T)

Request `{ "amount_uusdt", "beneficiary_rgb_invoice" }` (LSP
hot-authenticated) → PSBT template → countersigned, broadcast,
consignment returned with acceptance acknowledgment. Authorization
entries appear in the published transport ledger.

### lsps10.fed.create_lock / claim_lock / get_attribution (Flow P)

`create_lock` request:

```json
{
  "amount_uusdt": "250000000",
  "payment_hash": "3b1f4a…",
  "beneficiary_key": "03a1b2…",
  "expiry_at": "2026-08-20T12:49:06.991Z"
}
```

Response: the threshold-signed lock statement (the lock proof the
sender attaches to the payment TLV). `claim_lock` takes
`{ "lock_id", "preimage", "beneficiary_signature" }`.
`get_attribution` (authenticated) returns the identity's balance,
latest epoch inclusion proof, re-lockable credit, and outstanding
locks — the restore/resync endpoint. `prefetch_locks` issues
sender-bound blank locks per § Liveness.

### lsps10.fed.create_redemption / submit_redemption / get_redemption (Flow R1)

`create_redemption` request:

```json
{
  "amount_uusdt": "250000000",
  "identity_signature": "d29f…",
  "vusdt_inputs": ["B0570984EA35E417A20327D72414CDA0EB8200418772FA3E1A28D76EF4977CF2:1"],
  "vusdt_consignments": ["<consignment>"],
  "vusdt_change_seal": "utxob:…",
  "usdt_beneficiary": "wout:0"
}
```

- `usdt_beneficiary` — an RGB invoice or a witness-output seal of
  the redemption transaction itself (no pre-existing UTXO needed).

Response: `{ "redemption_id", "psbt", "fee_uusdt", "expires_at" }`.
`get_redemption` status:
`AWAITING_SIGNATURE → VALIDATING → BROADCAST → COMPLETE`, terminal
failures `EXPIRED`, `REJECTED`, and `UNBACKED` (attribution
shortfall — distinct, published, appealable).

### lsps10.fed.create_sweep (Flow S)

Hot-callable; destination fixed to the federation sweep seal; retires
transport at the next burn epoch; never credits attribution.

### LSP account and admin methods

`lsps10.fed.withdraw`, `lsps10.fed.refill`, `lsps10.fed.settle`
(cold-signed per CREDIT-OUT); `lsps10.fed.register_destination`
(cold-signed, delayed); `lsps10.fed.propose_incident` (hot) /
incident effect on cold signature or ADMIN; `lsps10.fed.register_lsp`
(cold ceremony, out-of-band components).

## Deferred: Lightning Gateway

A federation-operated Lightning node (TEE-held keys, VLS-style
policy signer) receiving redemptions in-Lightning was designed and
deliberately deferred: with attribution locks, Flow R2 already
achieves in-Lightning exit without giving any TEE a direct loss
path. The `lsps10.fed.create_ln_redemption` namespace is reserved.

## Open Questions

1. **Canonical contract rights** — whether the issuer-native USDT
   schema carries freeze/clawback via global state, the agreed
   Reserve policy when exercised, and any coordination with the
   issuer. A deployment prerequisite.
2. **Blind attribution credentials** — replacing per-identity
   balances with fixed-denomination blinded notes (nullifier-based)
   to remove the federation's per-identity view, at the cost of
   inclusion proofs and recoverability. Mutually exclusive with the
   Merkle-sum design; choose per deployment.
3. **Lock-signing architecture and throughput budget** — FROST
   preprocessing depth, target locks/second, minimum certifiable
   delta, and blank-lock exposure caps for degraded mode.
4. **Grant-finality delay** — whether to enable incident clawback of
   in-window grants (stronger containment) at the cost of a
   "finalizing" wallet state.
5. **Transport cap pricing** — bond vs fee, sizing against receive
   volume, and cap-adjustment cadence.
6. **Ecosystem boundary** — asset-registry metadata and wallet
   guidance so non-LSPS10 wallets either implement Flow P or refuse
   the asset; measurement of the discounted-float residual.
7. **Dispute and adjudication procedure** — evidence formats for R2
   fraud (foreign-chain proofs), token-loss make-whole claims, and
   incident appeals; SLAs and publication.
8. **Fee schedule** — deposit/lock/claim/redemption fees, transport
   pricing, sats-vs-vUSDT routing-fee guidance; concrete numbers.
9. **FROST migration** — replacing MuSig2 + tapscript fallback with
   a threshold key path for federation UTXOs.
10. **Governance** — membership admission/removal, recovery-quorum
    composition and a challenge/veto procedure for recovery attempts
    during induced liveness failures, parameter-change procedure
    (v1: static membership, ADMIN thresholds as specified).

## References

* [LSPS0][], [LSPS1][] — this repository.
* [BIP327][] — MuSig2.
* [RGB](https://rgb.tech) / RGB20 fungible interfaces — asset layer;
  IFA-style schemas for capped secondary issuance and burn.
* [BOLT specifications](https://github.com/lightning/bolts) —
  `to_self_delay`, shutdown/closing semantics, TLV payloads.
* [VLS](https://vls.tech/) — prior art for policy-enforcing
  (non-blind) signers.
* [Fedimint](https://fedimint.org/) — prior art for federated
  custody; differs in that LSPS10 keeps balances self-custodial on
  Lightning and uses the federation for reserves, transport
  issuance, and the attribution ledger.
* Maxwell-style proof-of-liabilities trees — prior art and known
  pitfalls for the epoch Merkle-sum commitment.

[LSPS0]: ../LSPS0/README.md
[LSPS1]: ../LSPS1/README.md
[LSPS0.common_schemas]: ../LSPS0/common-schemas.md
[BIP327]: https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
[<LSPS0.ppm>]: ../LSPS0/common-schemas.md#link-lsps0ppm
[<LSPS0.pubkey>]: ../LSPS0/common-schemas.md#link-lsps0pubkey
[<LSPS0.binary_blob>]: ../LSPS0/common-schemas.md#link-lsps0binary_blob
[<LSPS0.datetime>]: ../LSPS0/common-schemas.md#link-lsps0datetime
[<LSPS0.outpoint>]: ../LSPS0/common-schemas.md#link-lsps0outpoint
[<LSPS10.uusdt>]: #link-lsps10uusdt
