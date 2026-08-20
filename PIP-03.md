# PIP-03: Melt & fees

**Status: RFC (v0.1-draft)** · Reference implementation: [`@picocash/mint`](https://github.com/picocash/picocash/tree/main/packages/mint) + [`PicocashVault`](https://github.com/picocash/picocash-contracts). Endpoints live under the mint API ([PIP-02](PIP-02.md)); this PIP specifies their semantics.

Melt is the exit: proofs are burned at the mint and the vault pays the unit's TIP-20 token to an on-chain address. Its design center is one ordering rule — **insert-before-pay** — and one economic rule — **the melter pays for the payout**.

Melt burns proofs and pays out the unit's TIP-20 token from the vault.

`POST /v1/melt/quote` request `{ "amount": 500000, "unit": "tip20:42431:0x20c0...0000", "to": "0x…" }` → response `{ "melt_id": "<32 bytes hex>", "amount", "fee", "total", "unit", "to", "state": "UNPAID", "expires_at" }`, where `amount` is the on-chain payout, `fee` is the mint's melt fee (advertised in `/v1/info` as `fees.melt`), and `total = amount + fee` is what the inputs must sum to. The melt id is 32 bytes and is passed on-chain as the vault's `bytes32 meltId` — the vault enforces **one payout per melt id, forever**.

**Why the fee**: the payout is an on-chain transaction the operator signs and pays gas for. Without a fee, melting is a griefing vector — mint once, melt in minimum-size pieces, and bleed the operator one payout-tx at a time. With it, the melter's burned tokens cover the payout: the vault pays out `amount` while `amount + fee` of liability is burned, so the fee accrues **inside the vault** as surplus (balance − outstanding grows by `fee` per melt), reimbursing the operator without any extra transfer. Every melt strictly *improves* the solvency margin.

`POST /v1/melt` request `{ "melt_id": "…", "inputs": [Proof…] }`. Rules:

- Every input verifies like a swap input; `sum(inputs) == total` exactly (`AMOUNT_MISMATCH`; use `/v1/swap` for change first).
- **Insert-before-pay**: input `Y`s enter the spent-secret ledger inside one DB transaction *before* any on-chain payout is attempted. A conflict aborts with `TOKEN_ALREADY_SPENT` and nothing is paid.
- Then the mint calls `vault.ecashMelt(to, amount, meltId)` as operator. On success → `{ "state": "PAID", "tx_hash": "0x…" }`.
- If the chain call fails, the melt is recorded as `OWED` (`PAYOUT_FAILED`, HTTP 502): the tokens are consumed and the debt is durable. **Retry by re-POSTing the same `melt_id` with the same inputs** — the mint verifies the input set matches (hash), skips re-spending, and re-attempts the payout. The vault's per-meltId guard makes double-payout impossible even across retries.
- A `PAID` melt replayed with the same inputs returns the same result idempotently; different inputs → `MELT_ALREADY_PAID`.

State machine: `UNPAID → PENDING → PAID`, with `OWED` as the retryable failure branch. `GET /v1/melt/quote/{id}` polls state.


## The flow, end to end

1. **Quote**: `POST /v1/melt/quote {amount, unit, to}` → 32-byte `melt_id` (later the vault's `bytes32 meltId`), state `UNPAID`, expiry. Nothing is committed.
2. **Burn (insert-before-pay)**: `POST /v1/melt {melt_id, inputs}` verifies every proof, then — inside one DB transaction — locks the melt row, inserts every input's `Y` into the spent-secret ledger (a conflict rolls everything back: `TOKEN_ALREADY_SPENT`), records the input-set hash, and moves to `PENDING`. The proofs are dead and the debt durable **before any chain call**.
3. **Payout**: the mint calls `vault.ecashMelt(to, amount, meltId)` as operator. The vault enforces **one payout per melt id, forever** (`meltPaid`). Success → `PAID` + tx hash.
4. **Failure → retry**: chain failure records `OWED` (`PAYOUT_FAILED`, HTTP 502) — tokens consumed, debt recorded. Retrying with the same `melt_id` and the same inputs skips the burn and re-attempts the payout; if an earlier attempt secretly landed, the vault's `MeltAlreadyPaid` revert proves it and the melt is marked `PAID`. Exactly one payout, no matter how many retries or crashes.
5. Anyone can check `vault.meltPaid(meltId)` on-chain rather than trusting the mint's word.

## Open questions for RFC

1. Flat fee vs. gas-indexed fee — the flat fee under-recovers if chain fees spike past it (operator retunes via config, bounded by the vault ceiling).
2. Should `OWED` melts be enumerable via a public endpoint (an on-time-payout track record for mints)?
