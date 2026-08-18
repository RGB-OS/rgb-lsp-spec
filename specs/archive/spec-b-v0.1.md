# Specification B — Self-Custodial Virtual USDT Backed Directly by Canonical RGB-USDT

**Status:** Research Draft v0.1  
**Architecture:** Unilateral reserve exit  
**User custody model:** Self-custodial Lightning channel plus unilateral canonical-USDT recovery path  
**Backing model:** Shared canonical RGB-USDT reserve with pre-authorized user exit paths  
**Primary objective:** Remove discretionary federation custody from canonical-USDT redemption.

---

## 1. Purpose

This protocol extends the virtual-liquidity architecture so that user monetary claims are backed directly by canonical RGB-USDT rather than by a federation promise.

Users continue to use vUSDT over Lightning for normal payments.

However, every sufficiently settled user claim has a cryptographically enforceable path to canonical RGB-USDT held in a shared reserve.

No LSP or federation signature is required at the time of unilateral exit.

---

## 2. Target Security Property

The defining requirement is:

> If the LSP, federation, backend infrastructure, and all operators disappear or refuse cooperation, the user can still obtain the canonical RGB-USDT corresponding to their settled claim.

The emergency flow MUST ultimately be:

```text
User
 |
 | force close / exit
 v
valid settled claim
 |
 | pre-authorized Bitcoin/RGB transaction path
 v
shared canonical-USDT reserve
 |
 v
user-controlled canonical RGB-USDT output
```

No discretionary redemption approval occurs.

---

## 3. Core Architecture

```text
                         Canonical RGB-USDT
                              reserve
                                |
                                v
                       shared Bitcoin seal
                                |
                      virtual settlement tree
                     /          |           \
                    /           |            \
                   v            v             v
                Alice          Bob          Carol
                $1,000       $2,000          $500
```

Normal operation remains off-chain:

```text
Alice <---- vUSDT Lightning ----> LSP
```

The settlement tree exists as an emergency ownership path.

---

## 4. Assets

### 4.1 Canonical USDT

`USDT`

The actual RGB-USDT asset.

### 4.2 Virtual USDT

`vUSDT`

The synthetic channel-liquidity asset.

A large supply MAY exist.

Example:

```text
1,000,000,000 vUSDT
```

---

## 5. Economic State

The protocol retains:

```text
C = virtual capacity
R = redeemable user value
```

But unlike Specification A, a sufficiently settled `R` does not represent:

```text
federation owes user USDT
```

It represents:

```text
user owns an enforceable path
to canonical RGB-USDT
```

---

## 6. Solvency Invariant

Let:

```text
R_settled = total claims included in the latest unilateral reserve state

A_reserved = canonical USDT cryptographically committed
             to that settlement state
```

The fundamental invariant is:

```text
R_settled <= A_reserved
```

The huge vUSDT capacity supply remains unrelated:

```text
C_total >> A_reserved
```

is allowed.

---

## 7. Shared Reserve

Instead of assigning canonical USDT to every user Lightning channel, canonical USDT remains aggregated.

Example:

```text
Shared reserve:
10,000,000 canonical USDT
```

Rather than:

```text
Alice channel:  1,000 USDT
Bob channel:      200 USDT
Carol channel:    750 USDT
...
```

the system maintains:

```text
           10m shared canonical USDT
                     |
              settlement tree
             /      |       \
          Alice    Bob      Carol
```

This preserves capital pooling.

---

## 8. Settlement Epochs

Lightning state changes constantly.

The shared canonical reserve does not need to be reconstructed for every HTLC.

Instead, the protocol introduces **settlement epochs**.

Example:

```text
Epoch 100

Alice R = 100
Bob   R = 200
Carol R = 50
```

Later:

```text
Alice pays Bob 30
Bob pays Carol 20
```

Fast Lightning state becomes:

```text
Alice R = 70
Bob   R = 210
Carol R = 70
```

At the next settlement:

```text
Epoch 101

Alice = 70
Bob   = 210
Carol = 70
```

A new shared reserve commitment is produced.

---

## 9. Settled and Pending Claims

Because Lightning state may advance between reserve epochs, two claim states exist.

```text
R_current
```

Current Lightning economic balance.

```text
R_settled
```

Latest amount protected by a unilateral canonical-USDT exit path.

Example:

```text
Alice current balance:     700
Alice settled balance:     500
Pending settlement:        200
```

The strongest security guarantee applies to:

```text
R_settled
```

The protocol SHOULD minimize:

```text
R_current - R_settled
```

through frequent settlement.

---

## 10. Settlement Tree

For each epoch the reserve state is represented by a pre-authorized transaction tree.

Simplified Bitcoin view:

```text
                       Reserve UTXO
                           |
              +------------+------------+
              |                         |
           Branch A                  Branch B
          /        \                /        \
      Alice        Bob           Carol       Dave
```

Corresponding RGB state:

```text
               10m canonical USDT
                        |
              +---------+---------+
              |                   |
          Group A             Group B
          /    \              /    \
      Alice    Bob        Carol    Dave
```

Each user receives sufficient data to validate and broadcast the path from the reserve root to their user-controlled output.

---

## 11. User Exit Package

For each settled epoch Alice receives an **exit package** containing, conceptually:

```text
epoch_id
reserve_outpoint

user_pubkey
canonical_usdt_amount

required Bitcoin transactions
required signatures
RGB transition data
RGB consignments/proofs

timelock information
state replacement information
```

The package MUST contain everything Alice needs for unilateral execution, aside from normal Bitcoin block confirmation.

---

## 12. Normal Operation

Alice has:

```text
C = 5,000
R_current = 1,000
R_settled = 1,000
```

She continues using vUSDT Lightning.

The settlement transactions are not broadcast.

They remain as a fallback.

---

## 13. LSP Failure

The LSP disappears.

Alice has:

```text
R_settled = 1,000
```

She executes:

```text
reserve root
    |
    | broadcast pre-signed transaction
    v
intermediate branch
    |
    | broadcast next transaction
    v
Alice output
```

RGB state follows the same transaction chain.

Final result:

```text
Alice-controlled Bitcoin UTXO
+
1,000 canonical RGB-USDT allocation
```

No federation approval is required at exit time.

---

## 14. RGB Requirement

The Bitcoin transaction tree alone is insufficient.

Each branch MUST also preserve a valid RGB ownership transition.

Conceptually:

```text
Bitcoin:

Reserve output
    ->
Branch output
    ->
Alice output
```

must correspond to:

```text
RGB:

10m USDT reserve allocation
    ->
sub-allocation
    ->
Alice 1,000 USDT
```

The user's client MUST be able to validate this chain independently.

---

## 15. Virtual Seal Requirement

This architecture requires RGB state to follow outputs that may initially exist only inside pre-signed, not-yet-confirmed Bitcoin transactions.

Conceptually:

```text
real reserve seal
      |
      v
virtual transaction
      |
      v
virtual seal
      |
      v
virtual transaction
      |
      v
user seal
```

This requires an RGB construction capable of safely treating such pre-authorized virtual outputs as part of a client-validated ownership path.

This MAY require:

- an Ark-like RGB seal primitive;
- an RGB virtual-output seal;
- fallback seal extensions;
- another protocol-specific construction.

This is a major R&D dependency.

---

## 16. Stale Settlement Problem

The central difficulty is replacing old user exit rights.

Example:

```text
Epoch 100

Alice = 100
Bob   = 100
```

Alice later pays Bob 50.

New state:

```text
Epoch 101

Alice = 50
Bob   = 150
```

Alice still physically possesses her old Epoch 100 package:

```text
Alice -> 100
```

The protocol MUST prevent Alice from successfully enforcing both:

```text
old 100 claim
```

and:

```text
new 50 / payment state
```

---

## 17. Settlement Replacement

Every new epoch MUST either:

1. cryptographically invalidate the previous exit;
2. make use of the previous exit punishable;
3. consume the previous claim;
4. bind old-state use to a forfeit path;
5. use timelocks such that only one valid settlement can ultimately survive.

A system where users simply receive new pre-signed trees while retaining fully usable old trees is invalid.

---

## 18. Forfeit / Revocation Layer

A likely model is conceptually similar to:

```text
Old claim
    |
    | user enters new epoch
    v
forfeit/revocation transaction
    |
    +---- old exit becomes unsafe/unspendable
    |
    v
New claim
```

The exact mechanism is implementation-specific.

The protocol MUST define precisely:

- what makes a previous epoch obsolete;
- who can enforce obsolescence;
- which information users retain;
- what happens if some users refuse the new settlement.

---

## 19. Settlement Finality

The protocol distinguishes:

### Lightning Finality

Fast operational ownership:

```text
R_current
```

### Reserve Finality

Unilateral canonical-USDT ownership:

```text
R_settled
```

A user balance may therefore temporarily include:

```text
settled amount
+
pending amount
```

---

## 20. Receiving USDT

Alice receives 100 canonical USDT.

Fast flow:

```text
Payer
 |
 | 100 canonical USDT
 v
LSP

LSP
 |
 | 100 vUSDT
 v
Alice
```

Current state:

```text
Alice:
R_current += 100
```

At next settlement:

```text
Alice:
R_settled += 100
```

and her new reserve exit package includes the additional amount.

---

## 21. Sending USDT

Alice has:

```text
R_current = 500
R_settled = 500
```

She sends 100 externally.

Fast state:

```text
R_current = 400
```

The old settlement may still cryptographically represent 500 until replaced.

Therefore the payment protocol MUST ensure that sending funds interacts safely with the settlement revocation mechanism.

The user MUST NOT be able to send 100 and subsequently enforce the old 500 canonical-USDT reserve claim.

This is one of the protocol's most important security requirements.

---

## 22. User-to-User Payments

Alice pays Bob 100.

Before:

```text
Alice R_current = 500
Bob   R_current = 100
```

After:

```text
Alice R_current = 400
Bob   R_current = 200
```

At the next settlement, the reserve tree MUST reflect the new distribution.

No canonical USDT needs to move on-chain during ordinary payments.

---

## 23. Reserve Tree Capital Efficiency

Example:

```text
Canonical reserve = 10m
vUSDT capacity    = 1b
```

The reserve supports:

```text
10m of real user-owned monetary value
```

while the synthetic capacity asset provides:

```text
1b of potential inbound liquidity
```

without assigning 1b canonical USDT to Lightning channels.

---

## 24. Operator Role

In this architecture the LSP coordinates:

- payment routing;
- vUSDT channels;
- canonical-USDT routing;
- settlement epochs;
- construction of reserve trees.

However, the LSP MUST NOT possess a veto over valid settled unilateral exits.

---

## 25. Federation Role

A federation MAY still exist for:

- availability;
- co-signing settlement epochs;
- reserve management;
- recovery coordination;
- monitoring;
- settlement construction.

However, after an epoch becomes valid and settled:

```text
federation cooperation MUST NOT be required
for a user to execute their exit.
```

This is the key difference from Specification A.

---

## 26. Custody Model

Specification A:

```text
User R
 |
 | request
 v
Federation
 |
 | threshold signature
 v
canonical USDT
```

Specification B:

```text
User R_settled
 |
 | unilateral execution
 v
pre-authorized reserve path
 |
 v
canonical USDT
```

In Specification B the federation may help construct the path but does not retain discretionary control over a valid settled claim.

---

## 27. Failure Scenarios

### LSP Offline

User executes latest valid settled exit.

Expected result:

```text
canonical USDT recovered
```

### Federation Offline

User executes latest valid settled exit.

Expected result:

```text
canonical USDT recovered
```

### LSP and Federation Collude

They MUST NOT be able to invalidate an already-settled user claim unless the user voluntarily transitions into a later valid state.

### User Publishes Old Settlement

The protocol's revocation/forfeit mechanism MUST prevent profitable double redemption.

### Bitcoin Congestion

Exit may be delayed and expensive, but must remain possible.

Users MAY need to broadcast multiple transactions to unroll the settlement tree.

---

## 28. Security Invariants

The implementation MUST maintain:

```text
R_settled_total <= canonical USDT committed to settlement
```

A valid settled user claim MUST have:

```text
one and only one enforceable canonical-USDT exit
```

A new settlement MUST prevent profitable enforcement of superseded claims.

No operator signature MAY be required after a valid user's emergency exit path becomes settled.

---

## 29. Protocol Research Requirements

The following require formal design before production:

1. RGB state over pre-signed virtual transaction trees.
2. Virtual or Ark-like RGB single-use seals.
3. RGB consignment construction for unilateral tree unrolling.
4. Old-epoch invalidation.
5. Forfeit/revocation transaction design.
6. Settlement epoch transitions.
7. Partial-user participation.
8. Reserve-tree fee management.
9. Bitcoin transaction fee bumping.
10. Interaction with Lightning commitment revocation.
11. Backup and recovery of user exit packages.
12. Handling of canonical-USDT reserve fragmentation.

---

## 30. Security Classification

This architecture targets:

> **Self-custodial virtual USDT with direct unilateral claims against canonical RGB-USDT backing.**

The user does not merely own a claim against an issuer.

For the settled portion of their balance, they possess an enforceable transaction path to canonical RGB-USDT.

---

## 31. Product-Level Comparison

| Property | Specification A | Specification B |
|---|---|---|
| User channel | Self-custodial | Self-custodial |
| Inbound liquidity | Synthetic vUSDT | Synthetic vUSDT |
| User claim backing | Canonical USDT | Canonical USDT |
| Reserve control | Federation | Shared cryptographic reserve structure |
| Normal payments | Lightning | Lightning |
| Emergency close | User unilateral | User unilateral |
| USDT redemption | Federation required | User unilateral |
| Federation refusal | Can block USDT | Cannot block valid settled exit |
| Complexity | Moderate | High / research |
| Protocol changes | Limited | Likely RGB R&D |

---

## 32. Long-Term Goal

The ideal end state is:

```text
Huge vUSDT capacity
        +
Lightning payment UX
        +
pooled canonical-USDT capital
        +
full backing of actual user balances
        +
unilateral exit to canonical RGB-USDT
```

without requiring canonical USDT to be fragmented across every user Lightning channel.

That is the primary objective of this architecture.
