# Specification B — Self-Custodial Virtual USDT Backed by Canonical RGB-USDT

This repository contains the protocol specification for a self-custodial virtual-USDT (vUSDT) system whose settled user claims are backed **directly** by canonical RGB-USDT through unilateral, pre-authorized exit paths — no discretionary operator or federation approval at redemption time.

## Documents

| Document | Status | Description |
|---|---|---|
| [`specs/spec-b-v0.2.md`](specs/spec-b-v0.2.md) | **Draft v0.2 — prepared for external audit** | The current specification. Feature-complete: output/key model, epoch ceremony, settlement trees, leaf sub-channels, forfeit layer, expiry/refresh, RGB virtual-seal dependency contract, fee/exit-cost requirements, formal invariants and proof sketches, parameters, conformance checklist. |
| [`specs/archive/spec-b-v0.1.md`](specs/archive/spec-b-v0.1.md) | Superseded | Research Draft v0.1 (original architecture sketch). |
| [`specs/archive/spec-b-v0.1-soundness-review.md`](specs/archive/spec-b-v0.1-soundness-review.md) | Archival | Soundness review of v0.1 (findings F1–F12). Every finding's disposition is implemented in v0.2; see v0.2 §25 for the mapping. |

## Reading order for auditors

1. v0.2 §1–§3 — scope, notation, and the exact (rescoped) guarantee with its stated boundary.
2. v0.2 §7–§14 — the mechanism: outputs, ceremony, tree, leaf sub-channels, forfeits, expiry.
3. v0.2 §18 — security properties G1–G5 with proof sketches and the assumptions each uses.
4. v0.2 §23 — the R&D register; items R-1 (RGB virtual seals) and R-2 (exit cost model) are explicit launch/parameter-freeze blockers and are the only open dependencies.
5. The v0.1 review, cross-checking that each finding's disposition (v0.2 §25) is in fact discharged.

## Guarantee in one paragraph

For every **cosigned, unexpired, settled** claim, the user holds a pre-signed Bitcoin transaction path plus RGB consignments sufficient to unilaterally obtain their canonical RGB-USDT, enforceable even if the operator, federation, and all infrastructure disappear (v0.2 §3, G1). The two honest weakenings relative to the original v0.1 goal — cosign participation and claim expiry with a refresh liveness requirement — are the boundary of what pre-signed transactions can enforce on Bitcoin without covenants, not design choices; the covenant upgrade path that removes both is recorded in v0.2 §21.

## Status of this repository

The earlier RGB-LSP transport/channel-request drafts (LSPS0/LSPS1) were removed from the working tree in favor of this specification set; they remain available in git history.

## Responsible disclosure

Please report suspected protocol or specification vulnerabilities privately via GitHub's private vulnerability reporting on this repository (Security → Report a vulnerability) rather than public issues. Reports will be acknowledged, and reporters credited in the changelog unless anonymity is requested.
