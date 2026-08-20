# PIP-06: Token serialization

**Status: RFC (v0.1-draft)** · Reference implementation: [`@picocash/sdk`](https://github.com/picocash/picocash/tree/main/packages/sdk) (`serializeToken` / `parseToken`)

A **token** is a bundle of proofs handed from one holder to another — over a chat message, a QR code, a URL, a clipboard. It needs one canonical string form so every wallet can produce and accept it. This pip defines that form.

## Format

```
"pico" || <version letter> || base64url( UTF-8( JSON ) )
```

- The literal prefix `pico` followed by a single uppercase **version letter**. This document defines version **`A`**. Parsers MUST reject unknown versions rather than guess.
- **base64url** per RFC 4648 §5, **without padding** — the result is safe in URLs, QR codes, and anywhere `+`, `/`, `=` would be mangled.
- The payload is compact JSON (no insignificant whitespace) with short keys, because these strings get pasted by hand and shown in UIs.

## Version `A` payload

```jsonc
{
  "m": "https://mint.example",          // mint URL the proofs belong to
  "u": "tip20:42431:0x20c0…0000",       // unit (PIP-01)
  "t": [                                // proofs grouped by keyset
    {
      "i": "00260deaaf7e6868",          // keyset id
      "p": [
        {
          "a": 524288,                  // amount (base units)
          "s": "<secret, hex>",
          "c": "<C, 33-byte compressed point, hex>",
          "d": { "e": "…", "s": "…", "r": "…" }   // DLEQ payload (PIP-00 §3) — REQUIRED
        }
      ]
    }
  ],
  "memo": "thanks for the coffee"       // OPTIONAL, free text, ≤ 256 bytes
}
```

Field semantics map one-to-one onto the Proof object of [PIP-02](PIP-02.md); the short keys exist only to keep the encoded string small.

### Requirements

- Every proof MUST carry its `d` (DLEQ payload including the blinding factor `r`). A token without DLEQs cannot be verified offline, and a receiver MUST reject it — offline verifiability before claiming is the whole point of a transferable token.
- `m` and `u` MUST be present. A receiver MUST refuse tokens whose unit or mint it does not serve rather than attempt cross-mint acceptance.
- Amounts are integers in base units. No floats, ever.
- A receiver SHOULD verify every DLEQ offline, then **claim** the token by swapping every proof for fresh ones it alone knows (PIP-02 `/v1/swap`). Until the swap lands, the sender still holds spendable copies — possession of the string is not ownership; the swap is.
- Producers SHOULD emit proofs in descending amount order within a keyset; parsers MUST NOT rely on order.

### Limits

A token string is untrusted input. Parsers MUST enforce hard limits before and during decoding, rejecting (not truncating) anything over them. The reference limits: 1 MiB of input, 1024 proofs, 512-character memo, 512-character mint URL; `m` MUST be an `http(s)` URL, `u` MUST match `tip20:<chain_id>:<40-hex address>`, keyset ids MUST be 16 lowercase hex characters, and every hex field MUST be exactly its expected length. Decoding MUST use strict UTF-8 and strict base64url (no padding, no whitespace).

## Example

A one-proof token for 64 base units on the Moderato dev mint (secret and keys are test values; do not reuse):

```
picoAeyJtIjoiaHR0cDovL2xvY2FsaG9zdDozMzM4IiwidSI6InRpcDIwOjQyNDMxOjB4MjBjMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInQiOlt7ImkiOiIwMDI2MGRlYWFmN2U2ODY4IiwicCI6W3siYSI6NjQsInMiOiIwMCIsImMiOiIwMiIsImQiOnsiZSI6IjAwIiwicyI6IjAwIiwiciI6IjAwIn19XX1dfQ
```

decodes to

```json
{"m":"http://localhost:3338","u":"tip20:42431:0x20c0000000000000000000000000000000000000","t":[{"i":"00260deaaf7e6868","p":[{"a":64,"s":"00","c":"02","d":{"e":"00","s":"00","r":"00"}}]}]}
```

(Field values here are placeholders to show the shape; a real token's `s`, `c`, `d` are full-length hex as in PIP-02.)

## Open questions for RFC

1. **Version `B` (CBOR)**: base64url-JSON tokens for a typical $1 (7 proofs with DLEQ) run ~3 KB — fine for paste, heavy for QR. A CBOR version with binary fields would roughly halve it. Worth defining now, or when a QR flow actually exists?
2. **URI scheme**: `picocash:picoA…` or `web+picocash://` for click-to-receive links. Needs a wallet registration story first.
3. Should `memo` be authenticated (bound into a secret) or stay advisory? Advisory seems right — it is not money.
