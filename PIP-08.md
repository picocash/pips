# PIP-08: Spending conditions — Pay-to-Public-Key (P2PK)

**Status: RFC (v0.1-draft)** · Reference implementation: [`@picocash/crypto`](https://github.com/picocash/picocash/tree/main/packages/crypto) (`p2pk.ts`), enforced by [`@picocash/mint`](https://github.com/picocash/picocash/tree/main/packages/mint), used by [`@picocash/sdk`](https://github.com/picocash/picocash/tree/main/packages/sdk) and [`@picocash/mppx-method`](https://github.com/picocash/picocash/tree/main/packages/mppx-method). Wire-compatible with Cashu NUT-10 (well-known secrets) and NUT-11 (P2PK) so existing wallet code and auditors recognise it.

## Why

A bare eCash proof is a bearer instrument: whoever holds `(secret, C)` can spend it. For machine payments that is sometimes exactly wrong:

- A **merchant paying an agent** (a refund, a rebate, a bounty) wants only *that* agent to be able to redeem — a token in transit or in a log file should be worthless to anyone else.
- A **human funding an agent** wants to bound what the agent can do with the money: *"spend this only with merchant M, and if M never redeems it, I get it back."* The agent carries the tokens but cannot divert them.

P2PK makes the secret itself carry the condition. The mint enforces it at spend time; anyone can read it offline. No new cryptography is involved beyond a Schnorr signature.

## Secret format

A P2PK secret is a *well-known secret*: UTF-8 JSON, hex-encoded, used as the `secret` of an ordinary proof (so `Y = hash_to_curve(secret)` and blind signing are unchanged — the mint never sees the condition at issuance).

```jsonc
["P2PK", {
  "nonce": "<32 random bytes, hex>",          // uniqueness; two locks to the same key are different secrets
  "data":  "<33-byte compressed secp256k1 pubkey, hex>",   // the lock key
  "tags":  [                                 // OPTIONAL
    ["locktime", "1755720000"],              // unix seconds; the lock expires here
    ["refund",   "<pubkey>", "…"],           // after locktime, any of these keys may spend
    ["pubkeys",  "<pubkey>", "…"],           // additional lock keys
    ["n_sigs",   "2"],                       // k-of-(data + pubkeys) required before locktime (default 1)
    ["sigflag",  "SIG_INPUTS"]               // the only flag in v0.1 (default)
  ]
}]
```

Key order `nonce → data → tags` and no whitespace is the canonical encoding producers MUST emit; parsers MUST accept any valid JSON (the secret is opaque bytes to the mint's ledger either way). Unknown tags MUST be ignored. Secrets are capped at 1024 bytes ([PIP-02](PIP-02.md)).

## Witness

To spend a P2PK proof the spender attaches a witness to the proof object (swap and melt inputs, [PIP-02](PIP-02.md)):

```jsonc
{ "amount": 8, "keyset_id": "…", "secret": "…", "C": "…",
  "witness": "{\"signatures\":[\"<64-byte BIP-340 Schnorr signature, hex>\"]}" }
```

Each signature is BIP-340 Schnorr over **`sha256(secret bytes)`**, verified against the x-only form of a 33-byte key in the secret (`SIG_INPUTS`). Signatures that verify against no permitted key are ignored; over-providing is not an error. Duplicate signatures from the same key count once.

## Spend rules

At time `now` (the mint's clock, unix seconds):

| State | Who may spend |
|---|---|
| no `locktime`, or `now < locktime` | at least `n_sigs` distinct keys among `data` ∪ `pubkeys` have signed |
| `now ≥ locktime`, `refund` present | any one `refund` key has signed |
| `now ≥ locktime`, no `refund` | **anyone** — the lock has simply expired |

A secret that starts with `["P2PK"` but fails to parse (bad key length, non-numeric locktime, `n_sigs` > keys, unsupported `sigflag`) is **unspendable forever**; the mint MUST reject it with `SPENDING_CONDITION_FAILED`. The mint does not validate secrets at issuance (it cannot see them) — producers own this risk, so SDKs MUST validate before blinding.

Mints MUST enforce these rules on `/v1/swap` and `/v1/melt`. `/v1/checkstate` is unaffected. A mint advertises support as `"spending_conditions": ["P2PK"]` in `GET /v1/info`; a wallet MUST NOT lock tokens at a mint that does not advertise it (the tokens would be unspendable until a mint upgrade).

## Errors

`403 SPENDING_CONDITION_FAILED` — the message names the failing input and why (missing signature, wrong key, lock expired and refund required, malformed condition). Recovery text tells the client what to sign.

## Use in the MPP method ([PIP-05](PIP-05.md))

A service MAY publish a lock key as `pubkey` in its challenge. Proofs **P2PK-locked to that key** are then accepted as bound to the service in place of `PC-BIND` secrets — this is how an agent spends tokens a human pre-locked to a merchant: the agent cannot swap them (it lacks the key), so it presents an exact-amount subset as-is, and the service signs them at settlement. Services MUST refuse locked proofs whose lock expires within their settlement window (an expired lock is an unconditional proof and binds to nobody); the reference acceptor requires ≥ 60 s of remaining lock. Multisig locks (`n_sigs > 1`) are not accepted as a binding.

Because such proofs are bound to the *service* rather than to the *challenge*, replay protection across challenges at the same service rests on the duplicate-`Y` guard (PIP-05 check 6), which is required anyway.

## Token serialization ([PIP-06](PIP-06.md))

`picoA` proofs carry an optional `w` field holding the witness string, so a signed proof survives a round trip through a token or link.

## Offline verification

`lockOf(proof)` (SDK) parses the condition from any proof; a receiver can see who can spend a token before claiming it, and `Wallet.receive(token, { unlockKey })` signs and swaps in one step. DLEQ verification ([PIP-00](PIP-00.md)) is unaffected — P2PK restricts *who* can spend, DLEQ proves the mint *did* sign.

## Security notes

- **Locking is not escrow.** The mint enforces the condition; it does not adjudicate. The human's safety net is the `refund` tag, not the merchant's goodwill.
- **Locktime needs a clock.** Mints MUST use a monotonic, NTP-disciplined clock; a producer SHOULD leave slack (minutes) around a locktime so that spend/refund do not race the mint's view of time.
- **Signing is per secret.** A signature over one proof's secret is worthless for another; key holders sign each proof they spend.
- **Privacy.** A P2PK proof reveals the lock key to the mint at spend time. Locking is a deliberate trade of unlinkability for control; producers SHOULD use fresh lock keys per relationship where that matters.

## Open questions for RFC

- `SIG_ALL` (signatures also cover the outputs, so a captured witnessed proof cannot be re-targeted to the captor's outputs in a swap) — needed for untrusted transports; deferred to v0.2.
- HTLC (hash-locked) conditions for atomic multi-party flows — would follow NUT-14's shape if adopted.
- Whether `/v1/checkstate` should return the lock for a `Y` (it cannot — it has only `Y`, by design).
