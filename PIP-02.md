# PIP-02: Mint HTTP API

**Status: RFC (v0.1-draft)** · Reference implementation: [`@picocash/mint`](https://github.com/picocash/picocash/tree/main/packages/mint), live against the deployed vault on Tempo Moderato testnet ([PIP-04](PIP-04.md)). Melt semantics and fees are specified in [PIP-03](PIP-03.md).

Base path `/v1`, JSON bodies. All amounts are integer base units. Byte strings (secrets, points, `Y` values) are lowercase hex; points are 33-byte SEC1 compressed. Secrets are **raw bytes**, hex-encoded on the wire (a structured `PC-BIND` secret is the hex of its canonical-JSON UTF-8 bytes).

The `unit` is the canonical TIP-20 identifier `tip20:<chain_id>:<token_address>` ([PIP-01](PIP-01.md)): the unit *is* the token contract backing it. A mint MUST verify at startup that the token is a deployed TIP-20 and that its vault's `token()` equals the unit's address, and MUST refuse to serve otherwise.

## Errors

Every error is:

```jsonc
{ "error": { "code": "TOKEN_ALREADY_SPENT", "message": "…", "recovery": "…" } }
```

`recovery` is mandatory and tells the calling agent what to do next (a9n9 convention). Codes used below: `INVALID_REQUEST`, `QUOTE_NOT_FOUND`, `QUOTE_EXPIRED`, `PAYMENT_REQUIRED`, `QUOTE_ALREADY_ISSUED`, `TOKEN_ALREADY_SPENT`, `OUTPUT_ALREADY_SIGNED`, `KEYSET_UNKNOWN`, `KEYSET_INACTIVE`, `AMOUNT_MISMATCH`, `INVALID_PROOF`, `AMOUNT_LIMIT`, `MELT_ALREADY_PAID`, `PAYOUT_FAILED`, `NOT_IMPLEMENTED`.

## Objects

**Proof** (a spendable token unit):

```jsonc
{ "amount": 4096, "keyset_id": "00a1…", "secret": "<hex>", "C": "02…" }
```

(The DLEQ payload `{e, s, r}` travels with tokens between agents/services but the mint does not require it — the mint verifies with its private key.)

**BlindedMessage** / **BlindSignature**:

```jsonc
{ "amount": 4096, "keyset_id": "00a1…", "B_": "02…" }
{ "amount": 4096, "keyset_id": "00a1…", "C_": "02…", "dleq": { "e": "<hex32>", "s": "<hex32>" } }
```

DLEQ on issuance is REQUIRED ([PIP-00](PIP-00.md) §3).

## Endpoints

### `GET /v1/info`

Mint metadata: `name`, `version`, `unit`, `keysets` (ids + state), `limits` (`max_mint_amount`), `fees` (`{ melt }`, base units — the flat per-melt fee), `melt` (whether a payout executor is configured), `relay` (PIP-07 token-link relay capability: `{ enabled, max_bytes, ttl_seconds }`), `vault` (`{ method, chain_id, token, deposit_address }`, or `"fake"` in dev), `contact`.

### `GET /v1/keys` · `GET /v1/keys/{keyset_id}`

```jsonc
{ "keysets": [ { "id": "00a1…", "unit": "tip20:42431:0x20c0...0000", "state": "active", "keys": { "1": "02…", "2": "03…", … } } ] }
```

### `POST /v1/mint/quote` → deposit instructions

Request `{ "amount": 1000000, "unit": "tip20:42431:0x20c0...0000" }`. Response:

```jsonc
{
  "quote_id": "<32 bytes, hex>",        // doubles as the bytes32 deposit memo
  "amount": 1000000, "unit": "tip20:42431:0x20c0...0000",
  "state": "UNPAID",                    // UNPAID → PAID → ISSUED
  "deposit": {
    "method": "tempo",                  // or "fake-vault" in dev
    "chain_id": 42431,
    "token": "0x20c0…0000",             // TIP-20 token contract
    "to": "0x…",                        // the vault contract ([PIP-04](PIP-04.md))
    "memo": "0x<quote_id>",
    "note": "call transferWithMemo(to, amount, memo) on the token contract; the memo binds the deposit to this quote"
  },
  "expires_at": 1755630000
}
```

The quote id is 32 random bytes so it fits a TIP-20 `bytes32` memo exactly, with no padding convention needed. The mint observes `TransferWithMemo(address indexed from, address indexed to, uint256 amount, bytes32 indexed memo)` events (memo is indexed on Tempo's TIP-20) and credits the quote whose id matches the memo; deposits may span multiple transfers. Note the sender's fee is charged separately in the same token — the deposited `amount` arrives intact.

`GET /v1/mint/quote/{quote_id}` polls state.

### `POST /v1/mint` → blind signatures

Request `{ "quote_id": "…", "outputs": [BlindedMessage…] }`. Rules:

- Quote must be **PAID** (the mint checks the deposit oracle on demand); otherwise `PAYMENT_REQUIRED` with deposit instructions in `recovery`.
- `sum(outputs.amount) == quote.amount`, every amount a valid denomination of an **active** keyset.
- **Idempotent**: repeating the call with the identical output set returns the identical signatures. A different output set for an ISSUED quote → `QUOTE_ALREADY_ISSUED`.
- Each `B_` is globally single-use (`OUTPUT_ALREADY_SIGNED` on reuse) — signatures are recorded before they are returned, inside one DB transaction with the quote-state change.

Response `{ "signatures": [BlindSignature…] }`, ordered as the request.

### `POST /v1/swap` → change-making / refresh

Request `{ "inputs": [Proof…], "outputs": [BlindedMessage…] }`. Rules:

- Every input proof verifies (`k·hash_to_curve(secret) == C`) against its keyset (active or swap-only); `INVALID_PROOF` otherwise.
- `sum(inputs) == sum(outputs)` (`AMOUNT_MISMATCH`; no fees in v0.1).
- No duplicate `Y` among inputs, no duplicate `B_` among outputs.
- **Insert-before-sign**: input `Y`s are inserted into the spent-secret ledger (PRIMARY KEY on `Y`) and outputs recorded *inside one DB transaction*; a conflict aborts everything with `TOKEN_ALREADY_SPENT` and no signature is released. Concurrent redemptions of the same proof: exactly one succeeds.

Response `{ "signatures": [BlindSignature…] }`.

### `GET /v1/solvency`

The liability side of proof of liabilities, public: `{ "keyset_id", "unit", "outstanding", "vault": { "chain_id", "address", "token" } }` where `outstanding = Σ issued blind signatures − Σ spent secrets` (swaps are neutral; mints issue; melts spend). Anyone can compare `outstanding` to the vault's on-chain token balance; the mint's operator publishes the same figure on-chain per epoch via `vault.publishOutstandingSupply`. A mint MAY cap outstanding supply (`AMOUNT_LIMIT` at quote time, reference mints MUST).

### `POST /v1/checkstate`

Request `{ "Ys": ["02…", …] }` — `Y = hash_to_curve(secret)` values, never secrets. Response `{ "states": [ { "y": "02…", "state": "UNSPENT" | "SPENT" } ] }` (`PENDING` reserved for melt, [PIP-03](PIP-03.md)).

### `POST /v1/melt/quote` · `GET /v1/melt/quote/{id}` · `POST /v1/melt`

Melt burns proofs and pays out on-chain from the vault. Endpoint shapes, the fee model, the `UNPAID → PENDING → PAID/OWED` state machine, and retry semantics are specified in [PIP-03](PIP-03.md).

## Fake vault (dev mode)

When `PICOCASH_VAULT` is not `tempo`, the mint runs an in-memory deposit oracle plus a fake payout executor, and exposes `POST /dev/deposit { "quote_id", "amount" }` to simulate an on-chain deposit. Same API, no chain — the real oracle and payout implement the same interfaces.

## Open questions for RFC

Quote expiry semantics (currently 15 min, unpaid quotes garbage-collected), fee schedule surface, checkstate batching limits, idempotency-key convention for `/v1/swap`.
