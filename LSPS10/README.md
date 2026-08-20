# LSPS10 Self-Custodial Stablecoin Channels with Federated Redemption

| Name    | `federated_redemption` |
|---------|------------------------|
| Version | 0                      |
| Status  | Draft                  |

> **Note** The LSPS10 number is provisional and was chosen to avoid
> collision with upstream [BitcoinAndLightningLayerSpecs/lsp](https://github.com/BitcoinAndLightningLayerSpecs/lsp)
> numbers (LSPS0–LSPS7), which this repository may adapt in the future.

## Motivation

Users want a dollar-denominated balance that is:

1. **Self-custodial** — held in their own Lightning channel, not on an
   exchange or in an LSP's database.
2. **Instant and cheap to transact** — payable over the Lightning
   Network.
3. **Redeemable** — convertible at par into *canonical USDT* (USDT on
   its native settlement networks, e.g. Ethereum or Tron) without
   having to trust the LSP that serves their channel.

RGB makes (1) and (2) possible today: an RGB20 asset pegged 1:1 to
USDT (called **vUSDT** in this specification) can be allocated inside
Lightning channels and routed over the Lightning Network. What is
missing is (3): a synthetic dollar is only as good as its redemption
path. If redemption depends on the honesty and solvency of a single
LSP, the user has merely traded exchange custody risk for LSP custody
risk.

This specification introduces a **Redemption Federation**: a k-of-n
group of independent operators that

* custodies the canonical USDT reserves backing the entire vUSDT
  supply, in threshold-controlled accounts whose key shares live
  inside Trusted Execution Environments (TEEs);
* is the only entity able to release vUSDT into circulation, and only
  against verified reserve deposits;
* redeems vUSDT into canonical USDT for any bearer, **without the
  LSP's cooperation** if necessary;
* acts as a **validated signer, not a blind signer**: every federation
  member independently re-validates every fact (Bitcoin chain state,
  RGB consignments, canonical-chain deposits, supply invariants)
  before contributing its signature share to any operation.

The specification also bounds the damage of a compromised LSP by
capping the LSP's hot vUSDT exposure and periodically sweeping excess
channel funds back into federation custody.

## Design Goals

1. **Self-custody.** The user's vUSDT channel balance MUST be
   spendable and recoverable by the user unilaterally (standard
   Lightning penalty/force-close mechanics apply).
2. **Full collateralization.** At all times, canonical USDT reserves
   under federation custody MUST be greater than or equal to the
   circulating vUSDT supply. Issuance and redemption MUST preserve
   this invariant atomically from the federation's point of view.
3. **LSP-independent redemption.** A user MUST be able to redeem
   vUSDT into canonical USDT with no cooperation from any LSP, by
   closing their channel and presenting the resulting on-chain vUSDT
   to the federation.
4. **Validated signing.** No federation member ever signs a message
   it has not independently derived from locally validated state. A
   federation member's signing key MUST be unable to produce a
   signature that its co-located validator has not approved.
5. **No single point of theft.** Compromise of an LSP **plus up to
   k−1 federation members** MUST NOT enable theft of reserves,
   unbacked issuance of vUSDT, or fraudulent redemption. This is the
   central security requirement of this specification.
6. **Bounded LSP blast radius.** A fully compromised LSP MUST NOT be
   able to cause losses beyond (a) its own capped hot channel float,
   (b) its posted collateral, and (c) user channel balances whose
   owners are offline beyond the watchtower horizon.
7. **Safe degradation.** Loss of federation liveness (up to n−k
   members offline) MUST NOT affect Lightning payments or user
   custody; only issuance and redemption pause.

## Definitions

### Data types

This specification uses data types defined in
[LSPS0 Common Schemas][LSPS0.common_schemas].

Additionally:

###### Link: LSPS10.uusdt

Amounts of USDT and vUSDT MUST be expressed in **micro-USDT**
(`uusdt`, 10⁻⁶ USDT — the minimal unit of canonical USDT), encoded as
a JSON string containing the decimal representation, with field names
suffixed `_uusdt`. For example, 250 USDT is `"250000000"`.

> **Rationale** This mirrors the `_sat`/`_msat` convention of LSPS0
> and avoids floating point. The vUSDT RGB contract MUST use
> precision 6 so that one contract unit equals one micro-USDT.

### Actors

* **User (Client)** — holds vUSDT in a Lightning channel with an LSP;
  the API consumer.
* **LSP** — Lightning Service Provider. Provides vUSDT channels and
  liquidity (per [LSPS1][]). Holds only **vUSDT** (as Lightning
  liquidity and on-chain float) and **canonical USDT on exchanges**
  (for market operations and fast redemptions). The LSP is *not* a
  custodian of user funds and is *not* trusted for redemption.
* **Federation** — n independent members, threshold k, custodying
  reserves and controlling vUSDT issuance. Trusted collectively
  (k-of-n), never individually.
* **Federation Member** — one operator running the member stack
  (validators + signer) inside a TEE.
* **Gateway** *(OPTIONAL)* — a Lightning node operated under
  federation policy control whose keys live in a TEE, used for the
  optional in-Lightning redemption path (§ [Flow R2b](#r2b-lightning-redemption-via-federation-gateway-optional)).

### Assets

* **Canonical USDT** — issuer-native USDT on a supported settlement
  network (e.g. Ethereum, Tron). The reserve asset.
* **vUSDT** — an RGB20 fungible asset on Bitcoin, pegged 1:1 to
  canonical USDT, transferable on-chain (witness transactions) and
  off-chain (Lightning channels).
* **Treasury** — the set of federation-controlled Bitcoin UTXOs
  holding non-circulating vUSDT allocations, plus the
  federation-controlled canonical-chain reserve accounts.
* **Circulating supply** — total vUSDT issued minus vUSDT held in the
  Treasury.

## Architecture Overview

```
          Lightning Network (vUSDT channels)
 ┌──────┐   LN    ┌───────┐    LN     ┌─────────────┐
 │ User │◄───────►│  LSP  │◄─────────►│  Gateway    │  (optional)
 └──┬───┘         └──┬────┘           │ (TEE, fed-  │
    │                │                │  policy)    │
    │ close +        │ periodic      └──────┬──────┘
    │ on-chain       │ sweeps                │ sweeps
    │ redemption     │ (splice-out)          │
    ▼                ▼                       ▼
 ┌─────────────────────────────────────────────────────┐
 │            Bitcoin: vUSDT (RGB) layer               │
 │   Treasury UTXOs: Taproot, MuSig2 / k-of-n script   │
 └──────────────────────────┬──────────────────────────┘
                            │ validated issuance /
                            │ redemption (k-of-n, TEE)
 ┌──────────────────────────┴──────────────────────────┐
 │                 Redemption Federation               │
 │  Member 1 … Member n — each runs, inside a TEE:     │
 │   • Bitcoin full node + RGB validator               │
 │   • Canonical-chain validator                       │
 │   • Replicated event log (issuance/redemption)      │
 │   • Policy-enforcing threshold signer               │
 └──────────────────────────┬──────────────────────────┘
                            │ threshold-controlled
                            ▼ reserve accounts
 ┌─────────────────────────────────────────────────────┐
 │     Canonical chain (e.g. Ethereum/Tron): USDT      │
 │     Reserves ≥ circulating vUSDT at all times       │
 └─────────────────────────────────────────────────────┘
```

Trust flows in exactly one direction:

* Users and LSPs trust the **federation collectively** (k-of-n, TEE
  hardened, publicly attested, velocity-limited).
* The federation trusts **no one**: not the LSP, not the user, not
  any single member, and not the Gateway beyond an explicit,
  collateralized exposure cap.

## Asset Model

The vUSDT contract SHOULD be an RGB20 fungible asset with:

* **Precision** 6 (one unit = 1 uusdt).
* **Single genesis issuance** of the maximum supply to a
  Treasury-controlled seal (the *treasury model*). "Issuance into
  circulation" is a transfer out of the Treasury; "redemption" is a
  transfer back into the Treasury. Circulating supply is therefore
  `genesis_supply − treasury_balance`.

> **Rationale** The treasury model makes issuance and redemption
> symmetric RGB transfers, avoids repeated secondary-issuance
> operations, and lets every federation member compute circulating
> supply from state it already validates. A burn-based model
> (secondary issuance on deposit, provable burn on redemption) is an
> acceptable alternative and is listed under
> [Open Questions](#open-questions).

All Treasury seals MUST be controlled by federation
threshold-controlled Bitcoin UTXOs (§ [Reserve Custody](#reserve-custody)).
Consequently no vUSDT can enter or leave circulation without a
federation threshold signature, and — by the validated-signing rule —
without every contributing member having independently validated the
corresponding reserve movement.

## Reserve Custody

### Bitcoin side (Treasury UTXOs)

Treasury UTXOs MUST be Taproot outputs with:

* **Key path**: MuSig2 ([BIP327][]) n-of-n aggregate of all member
  keys — used for normal cooperative operation.
* **Script path(s)**: a k-of-n `OP_CHECKSIGADD` tapscript as a
  liveness fallback when up to n−k members are unavailable, and
  OPTIONALLY a timelocked (`OP_CHECKSEQUENCEVERIFY`) recovery path to
  a designated recovery quorum, for disaster recovery after prolonged
  federation failure.

A FROST-style threshold Schnorr key path MAY be used instead of
MuSig2 + script fallback once implementations mature; this choice is
transparent to the rest of the protocol.

### Canonical-chain side (reserve accounts)

Reserve accounts on the canonical chain MUST be controlled by a
threshold-of-members mechanism appropriate to that chain: threshold
ECDSA (e.g. CGGMP21) for externally-owned accounts, or chain-native
multisig / smart-contract accounts (e.g. a Safe) with the same k
threshold. The same validated-signing rule applies to every
canonical-chain signature share.

### TEE requirements

Each member MUST run its validator set, event log, and signer inside
a TEE, with:

* **Remote attestation** — members mutually verify attestation of
  each other's enclave measurement at DKG time and on every software
  upgrade; attestation reports MUST be published so users and LSPs
  can audit them (`lsps10.fed.get_federation_info`).
* **Sealed key shares** — signing key shares MUST be generated inside
  the enclave via DKG and MUST never exist in plaintext outside it.
* **Reproducible builds** — the attested enclave measurement MUST
  correspond to a publicly reproducible build of open-source member
  software.

TEEs are **defense in depth, not the security model**. The protocol
MUST remain secure under the assumption that any single TEE can be
fully broken (see [Trust and Failure Analysis](#trust-and-failure-analysis)):
this is precisely why the federation is a k-of-n *validated* signer
rather than a single attested blind signer. Members SHOULD diversify
TEE vendors, hosting providers, and jurisdictions.

### Membership parameters

* n MUST be ≥ 4; n = 7 with k = 5 is RECOMMENDED as a starting point.
* Theft of reserves requires compromising k members; loss of liveness
  requires n−k+1 members to fail. Operators SHOULD choose k such that
  k > n/2 (so two disjoint quorums cannot exist) and n−k is large
  enough to tolerate routine downtime.

## Protocol Flows

### Flow I — Deposit and issuance (canonical USDT → vUSDT)

Direction of trust: the LSP trusts the federation with the deposited
reserves. The federation extends **zero trust** to the LSP: vUSDT is
released only against a confirmed, locally-verified deposit.

1. The LSP calls `lsps10.fed.create_deposit`, indicating the amount
   and the Bitcoin UTXO / blinded seal that should receive the vUSDT.
   The federation returns a unique deposit address on the canonical
   chain (derived from the reserve account structure) and a
   `deposit_id`.
2. The LSP transfers canonical USDT to the deposit address.
3. Each federation member independently observes the deposit on its
   own canonical-chain validator and waits for
   `min_canonical_confirmations`.
4. When a signing quorum's members have each locally verified the
   deposit, the federation co-signs an RGB transfer from the Treasury
   to the LSP's seal (witness transaction on Bitcoin), applying the
   [ISSUE validation checklist](#issue--treasury-release-to-an-lsp),
   and delivers the consignment to the LSP.
5. The LSP validates the consignment client-side. The LSP now holds
   vUSDT it can deploy into channels (its own liquidity per [LSPS1][],
   or pushed to users who paid the LSP out-of-band).

Users normally acquire vUSDT *from* an LSP (paying the LSP with
canonical USDT, fiat, or BTC out-of-band, or receiving vUSDT over
Lightning from anyone). Users MAY also use Flow I directly if they
can receive an on-chain RGB transfer.

### Flow S — Periodic sweep (bounding LSP hot exposure)

A compromised LSP loses whatever vUSDT sits on its side of its
channels. To keep this loss bounded and to keep the systemic float
small, LSP hot exposure is capped and periodically swept back into
federation custody.

* `max_lsp_hot_uusdt` — the maximum vUSDT the LSP SHOULD keep across
  its channel balances and on-chain float. Advertised by the
  federation per LSP (it MAY depend on the LSP's collateral, § R2).
* When the LSP's float exceeds `sweep_trigger_uusdt`, or at least
  every `sweep_interval_blocks`, the LSP MUST move the excess to the
  Treasury:
  1. LSP calls `lsps10.fed.create_sweep`; the federation returns a
     Treasury RGB invoice (blinded seal) and a `sweep_id`.
  2. The LSP splices out of channels (or spends its on-chain float)
     into an RGB transfer paying that Treasury seal, and delivers the
     consignment.
  3. Each member validates the consignment and confirmation depth,
     then credits the LSP's **federation balance** in the replicated
     event log.
  4. The LSP MAY later withdraw its federation balance as canonical
     USDT (a redemption under Flow R1, to the LSP's registered
     canonical address) or re-issue it into channels (Flow I step 4
     without a new deposit, debiting the balance).

> **Rationale** Splice-out keeps channels open while de-risking the
> LSP. The federation balance makes sweeps cheap and reversible for
> the LSP without touching canonical-chain reserves, while keeping
> the full-collateralization invariant intact: swept vUSDT is in the
> Treasury (out of circulation) and the LSP's claim on it is a
> liability fully matched by reserves.

User-side channel balances are **not** swept — they are the user's
self-custodial funds. User protection against a compromised LSP
broadcasting revoked states comes from standard LN penalty
enforcement; the federation members SHOULD collectively operate a
watchtower service for channels of registered LSPs
(§ [Watchtower duty](#watchtower-duty)).

### Flow R1 — On-chain redemption (base path, LSP-independent)

This path MUST always be available to any vUSDT bearer and requires
no LSP cooperation. This is what makes vUSDT redeemable even if the
user's LSP is malicious, insolvent, or gone.

1. The user obtains an on-chain vUSDT allocation — normally by
   cooperatively or force-closing their channel (standard RGB-LN
   close mechanics allocate the user's vUSDT to a UTXO they control
   after any timelocks).
2. The user calls `lsps10.fed.create_redemption` with:
   * `amount_uusdt`,
   * `payout` — canonical chain id and payout address.
   The federation returns `redemption_id`, an **RGB invoice** with a
   unique Treasury blinded seal, `expires_at`, and the fee.
3. The user constructs an RGB transfer of the exact amount to that
   seal, broadcasts the witness transaction, and submits the
   consignment via `lsps10.fed.submit_redemption_consignment`.
4. Each member independently runs the
   [PAYOUT validation checklist](#payout--canonical-usdt-release-for-a-redemption).
   Only when its own checks pass does it contribute its signature
   share to the canonical-chain payout transaction.
5. The federation broadcasts the payout of
   `amount_uusdt − redemption_fee` to the payout address registered
   in step 2 and marks the redemption complete in the event log.

The payout address is bound to the blinded seal at request time, so
an attacker who learns the RGB invoice gains nothing by paying it,
and a man-in-the-middle cannot redirect the payout. The user MAY
authenticate the request with a public key to allow later payout
address changes; otherwise the address is immutable for that
`redemption_id`.

> **Note — ordering.** The user's vUSDT reaches the Treasury one or
> more Bitcoin confirmations before the canonical payout is released.
> During this window the user holds a claim against the federation.
> This is inherent to cross-chain redemption without a common HTLC
> layer and is exactly the exposure the federation's k-of-n + TEE +
> transparency design exists to make acceptable.

### Flow R2a — Fast redemption over Lightning, LSP-fronted (default fast path)

The LSP acts as a market maker between its own vUSDT and its
canonical USDT held on exchanges:

1. User calls `lsps10.create_fast_redemption` on the **LSP** with
   amount and canonical payout address.
2. LSP returns a vUSDT Lightning **hold invoice** and a quote
   (`fee_uusdt`, `expires_at`).
3. User pays the invoice; the LSP holds the HTLC, sends canonical
   USDT to the user's payout address from its exchange balance, and
   settles the HTLC once the canonical transfer is broadcast,
   revealing the preimage as a receipt.
4. The LSP's accumulated vUSDT float is later swept (Flow S) or
   redeemed (Flow R1) at the federation.

Trust analysis: the **federation is not involved and extends no
trust to the LSP**. The user's exposure to the LSP is one in-flight
redemption for the duration of one HTLC; a user who does not accept
even that MUST use Flow R1. A misbehaving LSP that takes the HTLC
without paying out is detectable and provable by the user (invoice +
preimage + absence of canonical payout), and SHOULD result in the
federation revoking the LSP's registration.

### Flow R2b — Lightning redemption via federation Gateway (OPTIONAL)

For redemption *inside* Lightning without depending on the LSP's
exchange balance, the federation MAY operate a Gateway: a Lightning
node with vUSDT channels to registered LSPs, whose node/channel keys
live in a TEE and whose signer enforces a VLS-style policy (it signs
only channel states consistent with a declared invoice ledger —
validated, not blind, at the Gateway level too).

1. User calls `lsps10.fed.create_ln_redemption`; the Gateway returns
   a vUSDT invoice bound to `redemption_id` + payout address.
2. User pays over Lightning (routed through their LSP into the
   Gateway's channel).
3. The Gateway's TEE produces a signed, attested **receipt**: the
   settled HTLC, the new channel state co-signed by the LSP (the
   channel peer), and the running unswept balance for that channel.
4. Federation members validate the receipt against the
   [PAYOUT checklist](#payout--canonical-usdt-release-for-a-redemption),
   LN-path additions, and release the canonical payout.
5. The Gateway sweeps its channel balances to the Treasury frequently
   (Flow S mechanics: splice-out / cooperative close), at which point
   the received vUSDT stops being channel exposure and becomes
   validated on-chain Treasury state; any discrepancy between
   attested receipts and swept amounts is detected and attributed.

Because a settled-but-unswept HTLC is *channel state*, not Bitcoin
state, federation members cannot fully re-derive it from their own
chain validators — they rely on the Gateway TEE attestation plus the
LSP's counter-signature. This residual trust MUST be bounded, not
eliminated:

* `max_gateway_unswept_uusdt` — global cap on unswept Gateway
  balance; payouts pause when reached until a sweep confirms.
* Per-LSP cap: LN-path payouts routed via a given LSP MUST NOT
  exceed that LSP's **posted collateral** (canonical USDT or swept
  vUSDT held at the federation as a bond) plus its confirmed swept
  balance.
* Large redemptions (above `max_ln_redemption_uusdt`) MUST use
  Flow R1.

Worst case — Gateway TEE **and** LSP both compromised, colluding —
the fabricated-receipt loss is capped by
`min(max_gateway_unswept_uusdt, LSP collateral)`, and the LSP's
collateral is slashed to cover it. A compromised LSP alone cannot
fabricate receipts (it cannot make the Gateway sign), and a
compromised Gateway alone is constrained by its policy signer, its
attestation, and the cap.

## Federation Security Model

### The validated-signer principle

Every federation member runs a **policy-enforcing signer** whose only
interface to the outside world is its co-located validator stack. The
signer MUST refuse to produce a signature share for any message that
its local validator has not derived from a fully validated state
transition. Concretely, each member independently maintains:

* a Bitcoin full node (SHOULD; at minimum a header-verifying client
  with full validation of all transactions relevant to Treasury UTXOs
  and submitted consignments — full node strongly RECOMMENDED);
* an RGB validator with the complete contract history of the vUSDT
  contract's Treasury lineage;
* a validating client of each supported canonical chain;
* the replicated event log of all deposits, issuances, sweeps,
  redemptions, and collateral movements, with the running invariant
  `reserves ≥ circulating` checked on every entry.

**No member ever signs a hash it received from another party** — not
from the LSP, not from a coordinator, not from another member. Each
member constructs the transaction/state-transition to be signed from
its own validated state and signs only its own construction (the
threshold protocol then fails harmlessly if constructions diverge).

> **Rationale** This is the property the user-facing guarantee rests
> on: a compromised LSP colluding with up to k−1 compromised members
> cannot produce a valid theft, because every *honest* member in any
> signing quorum re-derives and re-checks everything and will not
> contribute its share. A blind-signing federation (members sign
> whatever a coordinator or "the enclave" asks) collapses to
> single-point-of-failure the moment one coordinator or one enclave
> is broken.

### Validation checklists (normative)

#### ISSUE — Treasury release to an LSP

A member MUST verify, before contributing a share:

1. The referenced deposit exists on the canonical chain with ≥
   `min_canonical_confirmations`, pays a federation-derived deposit
   address, and its amount ≥ the issuance amount.
2. The deposit has not been credited before (event log).
3. The beneficiary seal belongs to a registered LSP (or valid direct
   depositor) as recorded at `create_deposit` time.
4. The RGB state transition is valid, moves exactly the issuance
   amount out of the Treasury, and sends all change back to a
   federation-derived Treasury seal on a federation-derived Taproot
   output.
5. Post-state invariant: `reserves ≥ circulating` still holds.

#### PAYOUT — canonical USDT release for a redemption

A member MUST verify:

1. The redemption request exists, is unexpired, and is unpaid.
2. **R1 path:** the submitted consignment is valid under RGB
   consensus rules, terminates in the Treasury seal bound to this
   `redemption_id`, transfers exactly `amount_uusdt`, and its witness
   transaction has ≥ `min_bitcoin_confirmations` on the member's own
   Bitcoin view.
3. **R2b path:** the Gateway receipt carries a valid TEE attestation
   of the approved Gateway build; the channel state is co-signed by
   the registered LSP; and the payout respects
   `max_ln_redemption_uusdt`, `max_gateway_unswept_uusdt`, and the
   LSP's collateral-based cap.
4. The payout transaction the member itself constructs pays exactly
   `amount_uusdt − redemption_fee` to the payout address bound at
   request time, with change only to federation reserve accounts.
5. Post-state invariant: `reserves − payout ≥ circulating` after
   accounting for the vUSDT returned to the Treasury.
6. Velocity limits: the payout does not breach per-request
   (`max_redemption_uusdt`) or per-epoch
   (`max_redeemed_per_epoch_uusdt`) caps.

> **Rationale — velocity limits.** Even if k members were somehow
> simultaneously compromised, epoch caps convert an instantaneous
> catastrophic drain into a slow leak that public monitoring
> (§ [Transparency](#transparency-and-proof-of-reserves)) can catch
> and the remaining members can halt.

#### TREASURY-SPEND — Bitcoin-side Treasury operations

For consolidations, fee management, or sweep-change handling, a
member MUST verify that every output and every RGB allocation of the
transaction it signs is again federation-derived, i.e. Treasury
operations can never leak value out of federation control.

#### ADMIN — membership, resharing, parameter changes

Membership changes, DKG resharing, enclave-measurement approvals
(software upgrades), cap/fee parameter changes, and LSP
registration/revocation MUST require an administrative threshold of
at least k members (a higher threshold, e.g. n−1, is RECOMMENDED for
membership and measurement changes), with each member validating the
proposal against published governance rules before signing.
Governance details are out of scope for this draft.

### Replicated event log

The event log is the federation's shared source of ordering for
everything that is not directly on-chain (deposit registrations,
redemption requests, LSP balances, collateral). It MUST be
deterministic and replicated such that any two honest members that
have seen the same events derive the same state; a BFT log is
acceptable but not required — because *signing* is the only
consequential act and signing already requires k independent
validations, the log needs consistency, not Byzantine agreement, and
divergence merely (and safely) halts signing. Each member SHOULD
periodically publish a signed commitment to its log head; diverging
heads MUST raise a public alarm.

### Key generation and rotation

* All threshold keys MUST be created by distributed key generation
  inside the members' enclaves; no trusted dealer.
* On membership change or scheduled rotation, the federation MUST
  reshare (proactive secret sharing) and, for Bitcoin Treasury
  UTXOs, roll funds to outputs under the new key set with the
  TREASURY-SPEND checklist.
* A member whose enclave attestation lapses or whose behavior
  diverges MUST be excluded from quorums until re-attested.

### Watchtower duty

Federation members SHOULD jointly operate watchtowers for the
channels of registered LSPs, so that a compromised LSP broadcasting
revoked states against offline users triggers penalty enforcement.
This closes the main avenue by which an LSP compromise could touch
*user* (rather than LSP) funds.

### Recovery

The OPTIONAL timelocked recovery script path
(§ [Reserve Custody](#reserve-custody)) allows a designated recovery
quorum to move Treasury funds after a long `OP_CSV` delay if the
federation is permanently unable to reach k. Canonical-chain reserves
SHOULD have an analogous dead-man recovery arrangement. If reserves
ever fall below circulating supply (e.g. issuer freeze of a reserve
account — a real risk with canonical USDT), the federation MUST
suspend issuance and switch redemption to a published pro-rata mode.

## Trust and Failure Analysis

| Scenario | vUSDT holders | Reserves | Notes |
|---|---|---|---|
| LSP compromised | User channel balances safe (penalty + watchtowers); in-flight R2a swaps at risk, one HTLC each | Safe | Loss limited to LSP's own hot float (≤ `max_lsp_hot_uusdt`) and its collateral |
| LSP + up to k−1 members compromised | Safe | **Safe — no quorum can sign an invalid operation; every honest member in any quorum revalidates everything** | The design goal; degraded liveness at worst |
| Gateway TEE compromised | Safe | Safe | Policy signer + attestation + caps; LN redemptions pause |
| Gateway TEE + LSP colluding | Safe | Loss ≤ min(`max_gateway_unswept_uusdt`, LSP collateral); collateral slashed | The only place TEE failure costs anything, and it is pre-funded by the LSP |
| n−k+1 members offline | Funds safe; LN payments unaffected | Frozen until recovery | Issuance/redemption pause; timelocked recovery as last resort |
| k members compromised (full quorum) | At risk | At risk, rate-limited by epoch caps | Outside the threat model; mitigated by TEE diversity, attestation, velocity limits, public monitoring |
| Canonical issuer freezes a reserve account | Pro-rata mode | Partially frozen | Diversify reserve accounts/chains; disclosed in `get_federation_info` |
| User loses RGB stash / consignment history | That user cannot prove ownership | Unaffected | Wallets MUST back up RGB state alongside channel state |

## Transparency and Proof of Reserves

* `lsps10.fed.get_federation_info` MUST expose: member identities and
  public keys, current enclave measurements + attestation evidence,
  n/k, all reserve account addresses, all caps/fees, and the signed
  current supply statement (`circulating_uusdt`,
  `reserves_uusdt`, log-head commitment, per-member signatures).
* Reserve addresses are publicly auditable on the canonical chain.
  Circulating vUSDT is not globally observable (RGB is client-side
  validated), so the supply statement is an n-member co-signed
  attestation; any member refusing to co-sign a false statement makes
  the discrepancy public. Third-party monitors SHOULD track reserve
  addresses against published supply statements continuously.

## API (Draft)

Transport follows [LSPS0][]. Methods are namespaced `lsps10.*`
(client ↔ LSP) and `lsps10.fed.*` (anyone ↔ federation; served by
each member or a shared front-end — responses MUST be verifiable
against member signatures, never trusted from the front-end alone).

### lsps10.get_info (client ↔ LSP)

| JSON-RPC Method | lsps10.get_info |
|-----------------|-----------------|
| Idempotent      | Yes             |

**Response**

```json
{
  "asset_id": "rgb:2dkSTbr-jFhznbPmo-TQafzswCN-av4gTsJjX-ttx6CNou5-M9juwLv",
  "federation_endpoint": "https://fed.example.org",
  "fast_redemption": {
    "supported": true,
    "min_uusdt": "10000000",
    "max_uusdt": "5000000000",
    "fee_ppm": 3000,
    "payout_networks": ["ethereum", "tron"]
  }
}
```

- `fee_ppm` [<LSPS0.ppm>][]

### lsps10.create_fast_redemption (client ↔ LSP, Flow R2a)

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

`status` transitions: `AWAITING_PAYMENT → HELD → PAID_OUT → SETTLED`
(or `EXPIRED` / `REFUNDED`). The LSP MUST NOT settle the HTLC before
broadcasting the canonical payout.

### lsps10.fed.get_federation_info

Returns the transparency bundle described in
[Transparency](#transparency-and-proof-of-reserves).

### lsps10.fed.create_deposit (Flow I)

**Request**

```json
{
  "amount_uusdt": "100000000000",
  "network": "ethereum",
  "beneficiary_rgb_invoice": "rgb:2dkSTbr-...:utxob:..."
}
```

**Response**

```json
{
  "deposit_id": "d3f0…",
  "deposit_address": "0x5FbDB2315678afecb367f032d93F642f64180aa3",
  "min_canonical_confirmations": 12,
  "expires_at": "2026-08-21T09:00:00.000Z"
}
```

### lsps10.fed.create_redemption / submit_redemption_consignment / get_redemption (Flow R1)

**create_redemption request**

```json
{
  "amount_uusdt": "250000000",
  "payout": { "network": "ethereum", "address": "0xAb58…" },
  "auth_pubkey": "02c9d1afd383fb651d93975d2a2e727aca5e6dd4fa2f090fded50102a149969ce8"
}
```

**create_redemption response**

```json
{
  "redemption_id": "7c1b…",
  "rgb_invoice": "rgb:2dkSTbr-...:utxob:...",
  "fee_uusdt": "750000",
  "min_bitcoin_confirmations": 6,
  "expires_at": "2026-08-21T09:00:00.000Z"
}
```

`submit_redemption_consignment` carries the consignment as
[<LSPS0.binary_blob>][]. `get_redemption` returns
`status ∈ {AWAITING_TRANSFER, CONFIRMING, VALIDATING, PAYING_OUT,
COMPLETE, EXPIRED, REJECTED}` plus, once paying out, the canonical
transaction id.

### lsps10.fed.create_sweep (LSP ↔ federation, Flow S)

As `create_redemption`, but the credited amount goes to the LSP's
federation balance instead of a canonical payout. Companion methods
`lsps10.fed.get_balance`, `lsps10.fed.withdraw` (canonical payout of
the balance) and `lsps10.fed.reissue` (back into vUSDT, Flow I
step 4) are defined identically to their building blocks above.

### lsps10.fed.create_ln_redemption (Flow R2b, OPTIONAL)

As `lsps10.create_fast_redemption`, but the invoice is issued by the
federation Gateway and payout is made by the federation after
validated receipt, per the PAYOUT checklist.

## Open Questions

1. **Treasury model vs burn model** — treasury-return (this draft) or
   secondary-issuance + provable RGB burn? Burn makes supply changes
   more externally legible; treasury makes them cheaper and
   symmetric.
2. **Gateway (R2b): include in v1 or defer?** R2a (LSP-fronted) plus
   R1 may be sufficient initially; R2b adds the only TEE-critical
   loss path and meaningful complexity.
3. **Threshold scheme on Bitcoin** — MuSig2 key path + k-of-n
   tapscript fallback (this draft) vs FROST key path.
4. **Collateral sizing** — fixed bond vs proportional to
   `max_lsp_hot_uusdt` vs proportional to trailing LN-redemption
   volume; slashing procedure and adjudication.
5. **Sweep cadence economics** — splice frequency vs on-chain fees vs
   exposure window; who pays.
6. **Canonical chain set** — which USDT networks to support at launch
   and how reserve balances are allocated across them; issuer-freeze
   risk management.
7. **User data availability** — standardize encrypted RGB stash
   backup with the LSP or federation so channel-close redemption
   never fails for lack of consignment history.
8. **Governance** — member admission/removal, parameter change
   process, and the recovery-quorum composition.
9. **Fee schedule** — issuance/redemption/sweep fees and how
   federation operating costs are funded.

## References

* [LSPS0][], [LSPS1][] — this repository.
* [BIP327][] — MuSig2.
* [RGB20](https://github.com/RGB-WG/rgb-interfaces) — fungible asset
  interface.
* [VLS](https://vls.tech/) — Validating Lightning Signer, prior art
  for policy-enforcing (non-blind) signers, applied here to the
  Gateway.
* [Fedimint](https://fedimint.org/) — prior art for federated
  custody; differs in that LSPS10 keeps balances self-custodial on
  Lightning and uses the federation only for reserves and redemption.

[LSPS0]: ../LSPS0/README.md
[LSPS1]: ../LSPS1/README.md
[LSPS0.common_schemas]: ../LSPS0/common-schemas.md
[BIP327]: https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
[<LSPS0.ppm>]: ../LSPS0/common-schemas.md#link-lsps0ppm
[<LSPS0.binary_blob>]: ../LSPS0/common-schemas.md#link-lsps0binary_blob
