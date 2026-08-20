# PIP-01: Units, keysets, denominations, rotation

**Status: RFC (v0.1-draft)** · Reference implementation: [`@picocash/mint`](https://github.com/picocash/picocash/tree/main/packages/mint); rotation lifecycle still draft.

## Denominations

Powers of 2 in **USDC.e base units** (6 decimals): `1, 2, 4, …, 2^30` µUSDC.e (≈ $1073 max single denomination). All amounts are integers; a mint MAY support fewer denominations and MUST publish exactly which via `/v1/keys`.

## Unit

**The unit is the token contract.** A backed unit is the canonical string

```
tip20:<chain_id>:<token_address>     e.g. tip20:42431:0x20c0000000000000000000000000000000000000
```

with the address in lowercase hex. It MUST resolve to a deployed TIP-20 contract on that chain, and it MUST equal the mint's vault binding: a mint MUST verify at startup that `vault.token()` is exactly the unit's token address and refuse to run otherwise. Amounts are integers in that token's base units (TIP-20 stablecoins use 6 decimals); the display symbol comes from the token contract's `symbol()`, never from the protocol. A pathUSD mint and a pyUSD mint are therefore different units, different keysets, and different vaults — one vault per currency, so the solvency check stays one number against one number.

The dedicated **credits** instrument (zero-backed, service-redeemable only, never meltable) is deliberately NOT a TIP-20 unit: it uses the `credit:` namespace (e.g. `credit:a9n9.net`) and its own keyset, so backed money and promo credits can never be confused — structurally, not by convention.

## Key derivation

A keyset's per-denomination private keys derive from one 32-byte keyset seed:

```
k_d = first candidate in [1, n) of:
      candidate_i = HMAC-SHA256(seed, "picocash/keyset/v1/" || unit || "/" || decimal(d) || "/" || decimal(i))
      for i = 0, 1, 2, …
```

where `d` is the denomination. The seed is the only secret; it MUST be stored encrypted at rest and MUST NOT appear in code, manifests, or logs.

## Keyset id

```
id = "00" || first 7 bytes, hex, of SHA256( concat of 33-byte compressed pubkeys, ascending by denomination, || UTF-8(unit) )
```

16 lowercase hex chars total; leading `"00"` is a version byte (NUT-02-style). The unit string is folded into the hash, so the keyset id itself commits to the exact TIP-20 contract and chain backing it — two keysets over different tokens can never share an id even given identical keys. *(Open question narrowed: whether to also hash expiry, NUT-02-v2-style, before freeze.)*

## Rotation lifecycle (settled shape, details TBD)

New keyset per epoch. Old keysets degrade: **active** → **swap-only** (grace window; tokens refreshable to the new keyset) → **redeem-only** (melt/settle only) → **archived**. States and expiry timestamps are published in `/v1/keys`. Blind signatures are only ever issued under an **active** keyset; swap inputs are accepted from active and swap-only keysets.

Open questions tracked for the RFC:

1. Keyset id derivation — NUT-02 v2 alignment (above).
2. Grace-window durations — protocol constants or per-mint policy surfaced in `/v1/info`?
3. Maximum denomination / amount-splitting guidance for privacy (uniform token counts).
