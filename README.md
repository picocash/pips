# pips

> **Status: pre-alpha; every pip is under RFC.** Design feedback is the most valuable contribution right now.

A **pip** is a seed. It's also the dot on a die, and the smallest move a price can make. These are the pips of **[picocash](https://github.com/picocash/picocash)** — small specifications you can grow an implementation from.

picocash is private, instant eCash for machine payments: Chaumian blind-signature bearer tokens backed 1:1 by stablecoins on [Tempo](https://tempo.xyz), designed as a payment method for [MPP](https://mpp.dev). One deposit funds unlimited sub-100ms offline-verified payments; the mint can't link issuance to redemption; the vault's solvency is provable on-chain.

*(And no, PIP doesn't stand for anything. If you need it to, "Picocash Improvement Proposal" works fine.)*

## The pips

| PIP | Title | Status |
|---|---|---|
| [PIP-00](PIP-00.md) | Cryptography — BDHKE, hash-to-curve, DLEQ | RFC (v0.1) |
| [PIP-01](PIP-01.md) | Units, keysets, denominations, rotation | RFC (v0.1-draft) |
| [PIP-02](PIP-02.md) | Mint HTTP API | RFC (v0.1-draft) |
| [PIP-03](PIP-03.md) | Melt & fees — the exit guarantee | RFC (v0.1-draft) |
| [PIP-04](PIP-04.md) | Vault, factory & custody | RFC (v0.1-draft) |
| [PIP-05](PIP-05.md) | MPP payment method `picocash` | RFC (draft) |
| [PIP-06](PIP-06.md) | Token serialization (`picoA…`) | RFC (v0.1-draft) |
| [PIP-07](PIP-07.md) | Token links — encrypted relay (optional) | RFC (v0.1-draft) |
| [PIP-08](PIP-08.md) | Spending conditions — P2PK locks | RFC (v0.1-draft) |
| [vectors/](vectors/) | Published test vectors | v0.1 |

**Statuses**: `draft` (shape may change freely) → `RFC` (implemented in the reference stack, feedback actively sought) → `frozen` (breaking changes need a new PIP). Everything here is RFC or earlier — nothing is frozen yet, which is exactly why now is the cheap time to object.

## What's already baked in

The reference stack ([picocash](https://github.com/picocash/picocash) + [picocash-contracts](https://github.com/picocash/picocash-contracts)) implements every pip, live on Tempo's Moderato testnet. Highlights the specs pin down:

- **Offline verifiability** (PIP-00): every token carries a DLEQ proof, so anyone verifies it against a mint's published keys with no mint round-trip — measured ~45ms mean merchant-side verification in the MPP reference implementation (settlement at the mint follows, by default before success).
- **The unit is the token contract** (PIP-01): `tip20:<chain_id>:<address>` — keyset keys and ids derive from it, and the mint refuses to run unless the vault's on-chain token binding agrees.
- **Insert-before-sign / insert-before-pay** (PIP-02/03): no signature and no payout ever exists before the corresponding spend is durably recorded.
- **The exit guarantee** (PIP-03/04): payouts cannot be paused by the contract (operator liveness is still required); each melt pays out exactly once (`meltPaid`); the exit fee is capped by the vault's committed `maxMeltFee`; raising that cap takes the same public timelock as rotating custody keys.
- **Unilateral exit** (PIP-04): if a mint goes silent past its attestation interval plus a grace period, holders redeem at the vault directly — the vault verifies each token's DLEQ and P2PK lock on-chain with the mint's *public* key. Live on Moderato.
- **Attested liabilities with teeth** (PIP-04): outstanding supply is an operator attestation, published on-chain and checkable against the vault balance; vaults commit at deployment to a solvency-publication policy — miss the attestation interval and the vault stops accepting deposits until the mint publishes again.
- **Lockable tokens** (PIP-08): a proof can be locked to a public key (with locktime + refund), so a merchant can pay *this* agent only, and a human can fund an agent that can spend *only with this merchant* — NUT-11 wire-compatible.
- **Challenge-bound credentials** (PIP-05): payment secrets commit to the challenge nonce and realm (`PC-BIND`), so an intercepted credential replays nowhere.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Short version: open an issue to discuss, PR against a pip to propose text, implement against [vectors/](vectors/) to prove compatibility. Vector bugs are spec bugs — reporting a mismatch from your implementation is one of the most valuable contributions there is.

Security-sensitive findings: **security@picocash.dev** (see the reference repo's [SECURITY.md](https://github.com/picocash/picocash/blob/main/SECURITY.md)).

## License

[Apache-2.0](LICENSE)
