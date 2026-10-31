# PIP-09: Nostr identity binding and the engram wallet

**Status: RFC (v0.1-draft)** · Depends on [PIP-06](PIP-06.md) (tokens), [PIP-08](PIP-08.md) (P2PK), and the Nostr NIPs cited inline · Reference implementation: [`@picocash/sdk`](https://github.com/picocash/picocash/tree/main/packages/sdk) (`nostr.ts`, `store.ts`, `nip44.ts`, `relay.ts`, `engram-store.ts`) and [`@picocash/mcp`](https://github.com/picocash/picocash/tree/main/packages/mcp) (the agent-facing wallet).

## Why

Agent platforms built on Nostr — [Buzz](https://github.com/block/buzz) is the motivating one — give every agent a secp256k1 keypair as its identity, an owner who attests that key (NIP-OA), and encrypted agent memory the owner can read (NIP-AE). They give it no money. picocash P2PK locks ([PIP-08](PIP-08.md)) are BIP-340 over secp256k1 — the same curve and signature a Nostr key already produces. This PIP specifies how an agent's existing Nostr identity **holds, spends, and is funded with** picocash, with no new key material, and how its wallet lives as an owner-readable memory record on its own relay.

The product it enables, in one line: an owner funds an agent with a **budget** — tokens locked to the agent's npub, or to a specific merchant, refundable to the owner after a locktime — instead of handing the agent an unbounded vendor API key.

## 1. Key binding

### 1.1 From an npub to a lock key

A Nostr public key is the 32-byte x-only coordinate; NIP-19 encodes it as `npub1…`. A P2PK lock `data` ([PIP-08](PIP-08.md)) is a 33-byte compressed point. The lock key for a Nostr identity is:

```
lock_key = "02" || hex(x_only)
```

The parity byte is fixed to `02` by convention. BIP-340 verification is x-only, so a signature by the key verifies regardless of the point's actual parity; the choice is immaterial to spending and MUST NOT be interpreted as a claim about parity.

Implementations MUST accept an identity as `npub1…` (bech32), as 64-hex x-only, or as 66-hex compressed (either parity), and derive the same `lock_key` from each.

### 1.2 Signing — the parity rule

About half of all secp256k1 keys have an odd-Y public point whose true compressed encoding begins `03`. A signer that decides *which proofs are its own* by comparing a lock's `data` to its own compressed key **for full-string equality will silently fail to sign half of all npub-derived locks**, even though the resulting signature would verify.

A conforming signer MUST match proofs to its key by the **x-only coordinate** (the trailing 64 hex characters of `data`, of each `pubkeys` member, and of each `refund` member), never by the parity byte. The reference `signerFromNsec` does this; the SDK's older `signProofs` does not and MUST NOT be used with npub-derived locks.

### 1.3 Private key input

Implementations MUST accept the secret as `nsec1…` (NIP-19) or 64-hex. Agent harnesses that already inject the key (Buzz: `BUZZ_PRIVATE_KEY`) need no additional configuration.

## 2. The wallet record

### 2.1 Placement

The wallet is a NIP-AE memory record: a `kind:30174` event authored by the agent key, addressed at slug

```
mem/picocash/wallet
```

with the `d` tag, `p` tag, and NIP-44 v2 encryption under the owner⇄agent conversation key exactly as NIP-AE specifies. Nothing here departs from NIP-AE; a generic NIP-AE client sees an opaque memory entry.

Consequences, which are the point:

- the relay stores ciphertext — balances and history are opaque to it;
- the **owner** decrypts the same record with their own key: the owner's accounting view requires no agent cooperation and no API;
- the wallet follows the agent across harnesses and restarts, so a remote agent that is reaped does not take its in-flight state with it.

### 2.2 Body

The NIP-AE memory body's `value` is the JSON serialization of a **snapshot**:

```jsonc
{
  "proofs":   [ /* PIP-06 proofs, spendable set */ ],
  "reserved": { "<reservation-id>": [ /* proofs in an uncommitted spend */ ] },
  "history":  [ { "at": 1700000000, "kind": "pay", "delta": -1500, "ref": "llm.example" } ],
  "seq":      12
}
```

- `proofs`: proofs not currently part of a spend. Identity of a proof is `Y = hash_to_curve(secret)`; a snapshot MUST NOT contain two proofs with the same `Y` across `proofs` and `reserved`.
- `reserved`: spends in flight (§3). Proofs here are not spendable until committed or released.
- `history`: append-only ledger. `kind ∈ { receive, send, pay, mint, melt, refund }`; `delta` is signed base units; `ref` is free-form correlation.
- `seq`: strictly increases on every write (§3.3).

Readers MUST ignore unknown fields. The serialized body MUST NOT exceed NIP-44's 65,535-byte plaintext limit; a writer that would exceed it MUST refuse the write with an error telling the holder to consolidate (swap small denominations into fewer proofs) rather than truncate.

## 3. Writing: reserve → commit | release

The failure that loses money is a crash between "proofs handed to a mint or merchant" and "change written back". A conforming wallet therefore records a spend in three steps, persisting each:

1. **reserve** — move the proofs about to be presented from `proofs` into `reserved[id]` and persist, **before** anything leaves the wallet.
2. **spend** — swap, pay, or melt.
3. **commit** (drop `reserved[id]`, add change, append history) or **release** (move them back). A spend that may have reached the mint but whose outcome is unknown MUST be left reserved for reconciliation via the mint's `checkstate`, never released.

### 3.1 Event timing

Per NIP-AE §Writing: `created_at = max(now, head.created_at + 1)`. A head whose `created_at` is implausibly far ahead of wall-clock (reference: 24 h) MUST be treated as clock-poisoned and the write surfaced as a conflict.

### 3.2 Head selection

Per NIP-AE §Head selection, including signature verification before decryption, re-derivation of `d` from the decrypted slug, and greatest-`created_at` / lowest-`id` ordering. A tombstone (`value: null`) is a valid head meaning "no wallet".

### 3.3 Concurrency

Two sessions of one agent are the realistic race. Before every mutation a writer MUST re-read the head and MUST refuse (`ConflictError`) if the head's `seq` differs from the `seq` it last observed. After publishing it SHOULD re-read the head and surface a conflict if its own event is not the head. The loser of a race gets an error and reloads; it never silently overwrites the other session's spend.

## 4. Funding an agent

### 4.1 Locked to the agent

An owner funds an agent with a token whose proofs are P2PK-locked to the agent's `lock_key` (§1.1), optionally with `locktime` and `refund = [owner lock_key]`. Such a token is safe to transmit over any channel. The agent claims it by signing (§1.2) and swapping to fresh proofs; the owner reclaims any unspent remainder after the locktime with the refund key, with no agent cooperation.

### 4.2 Locked to a merchant — budgets

An owner may instead lock proofs to a **merchant's** advertised key (a service's PIP-05 `pubkey`). The agent then holds a budget it can spend only at that merchant: it presents such proofs as-is in answer to that merchant's challenge ([PIP-05](PIP-05.md), P2PK binding), and they are worthless anywhere else.

Two consequences implementations MUST handle:

- **No re-denomination.** The agent cannot swap merchant-locked proofs (the mint requires the merchant's witness), so a merchant budget pays only prices that are an **exact subset-sum of its denominations**. A wallet MUST present a merchant budget only when such a subset exists for the challenged amount and otherwise fall back to its open balance rather than fail the payment. Owners SHOULD fund merchant budgets in units of the merchant's price points. The protocol-level fix is an open question in [PIP-08](PIP-08.md) (lock-preserving re-denomination).
- **Receive without swap.** A merchant-locked token cannot be claimed by swapping. A wallet receiving one MUST verify every proof offline (DLEQ), SHOULD confirm via `checkstate` that the mint has not seen them spent, and then hold them **unswapped**, reported separately from the open balance. Until spent, the sender still holds identical proofs: a merchant-locked token is a claim accepted from a trusted party (the owner), not bearer cash. Wallets MUST say so.

### 4.3 What the owner can see

Because the wallet is an engram (§2), the owner reads: open balance, in-flight reservations, each merchant budget and its remaining amount, and the ledger — with their own key, from the relay. The reference `picocash-owner` command is this view.

## 5. Agent-facing surface (informative)

The reference wallet is an MCP server with six tools: `balance`, `fetch_paid` (answers a picocash 402 from the wallet; settle-first), `send` (locked to a recipient npub; **refuses** an unlocked bearer token unless explicitly opted in, because a token pasted into a channel is public), `receive`, `melt`, `history`. It takes the agent key from the platform's existing environment and selects the engram store when a relay and owner key are configured.

## 6. Security considerations

- **Key = identity = wallet.** Whoever holds the agent's secret spends its balance — including a remote-execution provider the platform hands the key to by design. The mitigation is the lock model: small, time-boxed, merchant-restricted budgets. Owners SHOULD fund no more than they would accept losing to the agent's host.
- **The relay sees ciphertext and sizes.** Event lengths and write timing leak activity patterns, not amounts or counterparties.
- **Unlocked tokens in public channels** are cash to the first reader. Reference wallets default to locked sends and require an explicit opt-in for unlocked ones.
- **Merchant-locked tokens are claims**, not cash, until spent (§4.2).
- **Owner visibility is total.** The owner can read every proof in the agent's wallet and could spend any proof locked to their own refund key. This is the intended trust relationship, and agents SHOULD be told.

## 7. Test vectors

- Key binding: the NIP-19 vector `nsec1vl029mgpspedva04g90vltkh6fvh240zqtv9k0t9af8935ke9laqsnlfe5` ⇄ `67dea2ed018072d675f5415ecfaed7d2597555e202d85b3d65ea4e58d2d92ffa`; conforming signers MUST sign-and-verify for keys of **both** parities (the reference test forces one of each).
- Record format: the reference implementation reproduces Buzz's pinned NIP-AE vectors byte-exactly — conversation key, `d` tags, every ciphertext (with pinned nonces), and every event id and signature (zero aux). Any NIP-44/NIP-AE implementation that does likewise interoperates.

## Open questions for RFC

- Should `history` be capped or rotated out to a second slug so a busy agent never hits the body limit?
- Owner → agent **policy** (max per call, per day, allowed merchants) as a separate owner-authored engram the wallet enforces locally — advisory only, since the real bound is the lock on the tokens.
- A standard discovery of a merchant's lock key and price units, so owners can fund budgets in the right denominations without reading each merchant's docs.
