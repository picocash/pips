# PIP-00: Cryptography — BDHKE, hash-to-curve, DLEQ

**Status: RFC (v0.1)** · Reference implementation: [`@picocash/crypto`](https://github.com/picocash/picocash/tree/main/packages/crypto) · Test vectors: [`vectors/crypto-v0.1.json`](vectors/crypto-v0.1.json)

Curve: **secp256k1**. `G` is the generator, `n` the group order. Scalars are 32-byte big-endian integers mod `n`. Points serialize as 33-byte SEC1 compressed, except inside `hash_e` (below) where uncompressed serialization is used, matching Cashu NUT-12.

## 1. hash_to_curve

Maps an arbitrary byte string to a curve point with unknown discrete log. Identical to Cashu NUT-00:

```
DOMAIN_SEPARATOR = "Secp256k1_HashToCurve_Cashu_"   (UTF-8, no trailing null)
msg_hash = SHA256(DOMAIN_SEPARATOR || message)
for counter in 0, 1, 2, ... (uint32, little-endian, 4 bytes):
    candidate = 0x02 || SHA256(msg_hash || counter)
    if candidate parses as a valid compressed point: return that point
```

Implementations MUST fail after 2^16 iterations (never reached in practice; each iteration succeeds with p ≈ 0.5).

## 2. Blind Diffie–Hellman Key Exchange (BDHKE)

Roles: **Client** (token holder), **Mint** (holds private key `k` per denomination, publishes `K = kG`).

1. Client picks a **secret** `x` (arbitrary bytes; see [PIP-05](PIP-05.md) for structured secrets) and computes `Y = hash_to_curve(x)`.
2. Client picks blinding factor `r ←$ [1, n)` and sends the **blinded message** `B_ = Y + rG`.
3. Mint returns the **blind signature** `C_ = k·B_` (plus a DLEQ proof, §3 — REQUIRED in picocash).
4. Client **unblinds**: `C = C_ − r·K`. Note `C = k·Y`.

The **proof** (bearer token unit) is `(x, C)` against keyset key `K`. The mint verifies a spend by checking `k·hash_to_curve(x) == C`, and records `Y` in the spent-secret ledger. The mint never sees `x` or `Y` at signing time, so issuance and redemption are unlinkable.

## 3. DLEQ proof

Proves `C_ = k·B_` was produced with the same `k` such that `K = k·G` — i.e. discrete-log equality — without revealing `k`. This lets **anyone** verify a token against a mint's published keyset **offline**, which is what makes accept-then-settle and agent-to-agent transfer sound. picocash mints MUST attach a DLEQ proof to every blind signature (Cashu treats this as optional; picocash does not).

Identical to Cashu NUT-12:

```
hash_e(P1, P2, ...) = SHA256( UTF8( hex(uncompressed(P1)) || hex(uncompressed(P2)) || ... ) )
```

**Mint (prover)**, with nonce `w ←$ [1, n)`:

```
R1 = w·G
R2 = w·B_
e  = hash_e(R1, R2, K, C_)
s  = w + e·k   (mod n)
→ proof {e, s}
```

**Verifier holding B_ and C_** (client at issuance):

```
R1 = s·G − e·K
R2 = s·B_ − e·C_
valid iff e == hash_e(R1, R2, K, C_)
```

**Verifier holding only the proof `(x, C)`** (a service accepting a token) additionally needs the blinding factor `r`, which the token carries in its DLEQ payload `{e, s, r}`:

```
Y  = hash_to_curve(x)
B_ = Y + r·G
C_ = C + r·K
→ run the verification above
```

Carrying `r` in the token is safe: `r` only links `B_`/`C_` to `Y`/`C` for someone who already holds the token; the mint never sees the token's DLEQ payload before redemption.

## 4. Requirements

- Blinding factors and DLEQ nonces MUST come from a CSPRNG and MUST NOT be reused.
- Verifiers MUST reject points at infinity and non-canonical scalar encodings (`0` or `≥ n`).
- All hash-to-curve, blind/unblind, and DLEQ operations MUST reproduce the published test vectors, which include the upstream Cashu NUT-00/NUT-12 vectors.
