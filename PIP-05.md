# PIP-05: MPP payment method `picocash`

**Status: RFC (draft)** · Reference implementation: [`@picocash/mppx-method`](https://github.com/picocash/picocash/tree/main/packages/mppx-method) (includes the mppx binding). This document is the picocash differentiator: a binding of Chaumian ecash to MPP's challenge → credential → receipt flow, giving services **sub-100ms offline-verified payment acceptance** (measured ~45ms mean in the reference implementation) with no on-chain transaction per call and no payer identity exposure. Field names and exact envelope framing will track the mppx custom-method interface (see [mpp.dev](https://mpp.dev)); the shapes below are the proposal.

## Flow overview

```
Agent                         Service                        Mint
  |  -- request ----------------> |                            |
  |  <- 402 + Challenge --------- |                            |
  |  select proofs, bind          |                            |
  |     to challenge nonce        |                            |
  |  -- Credential -------------> |                            |
  |                               |  verify OFFLINE (DLEQ,     |
  |                               |  amount, binding, allow-   |
  |                               |  list, expiry)  <100ms     |
  |  <- 200 + Receipt ----------- |                            |
  |                               |  -- swap/redeem (async) -> |
  |                               |  <- new proofs / ack ----- |
```

The service verifies offline, then settles with the mint by swapping the proofs for ones only it knows. Two settlement modes exist; **settle-first is the default**: the service MUST NOT report the payment as successful until the swap has landed. A service MAY opt into **accept-then-settle** — respond on offline verification and settle afterwards — in which case it knowingly carries double-spend risk for the settlement window, bounded by per-challenge amount caps it sets itself, and MUST label such receipts `settlement: "pending"`.

## Challenge

Issued by the service (in the MPP challenge envelope, `method: "picocash"`):

```jsonc
{
  "method": "picocash",
  "realm": "api.example.dev",            // service identity / scope
  "challenge_id": "chal_9f2c…",           // unique, single-use
  "nonce": "b7e2…",                       // 32-byte hex, binds the credential
  "amount": 50000,                        // integer base units
  "unit": "tip20:42431:0x20c0...0000",
  "mints": [                              // allowlist the service will accept
    { "url": "https://mint.picocash.app", "keyset_ids": ["…"] }
  ],
  "expiry": 1755630000                    // unix seconds
}
```

## Credential

The agent selects proofs summing to **exactly** `amount` (using `/v1/swap` for change beforehand if needed) and binds them to the challenge:

```jsonc
{
  "method": "picocash",
  "challenge_id": "chal_9f2c…",
  "mint": "https://mint.picocash.app",
  "keyset_id": "…",
  "proofs": [
    { "amount": 32768, "secret": "<structured secret>", "C": "02…", "dleq": { "e": "…", "s": "…", "r": "…" } },
    …
  ]
}
```

**Challenge binding (anti-replay).** Proof secrets are structured so the secret itself commits to the challenge: the agent mints (or swaps into) proofs whose secret is

```
["PC-BIND", { "nonce": "<challenge nonce>", "realm": "<realm>", "salt": "<random 32B hex>" }]
```

serialized as canonical JSON. A credential intercepted in transit is then worthless anywhere else: the secret only satisfies *this* challenge at *this* realm, and the verifier MUST check the commitment before accepting. Unbound (plain random-secret) proofs MUST be rejected by verifiers. *(Open question 1: P2PK-style spending conditions instead of secret-format commitment — see below.)*

## Verification (service side, offline)

In order, all MUST pass:

1. `challenge_id` known, unexpired, never previously accepted (single-use).
2. Mint URL and `keyset_id` in the service's allowlist; keyset keys already cached from `/v1/keys`; and the **keyset's `unit` equals the challenge's `unit`** — an equal base-unit number in a different TIP-20 token is not payment.
3. Each proof's secret parses as `PC-BIND` and commits to this challenge's `nonce` and `realm`.
4. Amounts sum to exactly `amount`; each amount is a valid denomination of the keyset.
5. **DLEQ verifies** for each proof against the cached keyset key for its denomination ([PIP-00](PIP-00.md) §3).
6. No duplicate `Y = hash_to_curve(secret)` within the credential or against previously accepted credentials.

No network round-trip: this is the sub-100ms **verification** path (≈45 ms measured in the reference implementation). It is not the end-to-end latency in settle-first mode, which adds one mint round-trip.

**Replay state.** Checks 1 and 6 are only as good as the store behind them. Marking a challenge paid and recording its `Y`s MUST be a single atomic operation, and in any deployment with more than one service instance that store MUST be shared (a database, not process memory). A process-local cache is acceptable only for a single instance, and it MUST be bounded (evict expired challenges; retain `Y`s at least until the mint has settled them).

## Receipt & settlement

Settlement is a call to the mint's `/v1/swap` (exchanging the received proofs for fresh ones the service owns — this is the double-spend check and the moment of finality) or `/v1/melt`. In settle-first mode the receipt is returned after that call with `settlement: "settled"`; in accept-then-settle mode it is returned immediately with `settlement: "pending"` and updated later. Receipt shape:

```jsonc
{
  "method": "picocash",
  "challenge_id": "chal_9f2c…",
  "accepted_at": 1755629912,
  "amount": 50000,
  "settlement": "pending",               // -> "settled" | "double-spent"
  "checkstate_ref": "…"                  // mint checkstate reference once settled
}
```

In settle-first mode a double-spend is simply a rejected payment. In accept-then-settle mode, if settlement later fails the service's recourse is service-level (revoke the API result, ban the realm session); the economics — bounded per-call amounts — make that fraud window smaller than a card chargeback by orders of magnitude, but it is the service's risk to take, not the protocol's promise.

## Open questions for RFC

1. **Binding mechanism**: secret-format commitment (above, simple, works today) vs. P2PK-style spending conditions (NUT-10/11-adjacent — richer, allows locking to a service pubkey, but heavier). Leaning secret-format for v0; input wanted.
2. Should the receipt be mint-cosigned (checkstate attestation) so agents can prove payment to third parties?
3. Overpayment handling: exact-amount requirement (above) vs. service-returned change token.
4. Multi-mint credentials in one payment — worth the verifier complexity?
