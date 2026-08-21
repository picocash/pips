# PIP-07: Token links (encrypted relay) — OPTIONAL

**Status: RFC (v0.1-draft)** · Optional capability · Reference implementation: [`@picocash/mint`](https://github.com/picocash/picocash/tree/main/packages/mint) (relay) + [`@picocash/sdk`](https://github.com/picocash/picocash/tree/main/packages/sdk) (client)

A serialized token ([PIP-06](PIP-06.md)) is ~3 KB — fine for machines, hostile to chat messages, QR codes, and humans. A **token link** is a ~60-character URL that resolves to the full token. This pip defines how to do that **without the relay ever being able to read, spend, or link the token**.

## Threat model, stated first

A token is `(secret, C)`: whoever holds the bytes can spend. So any service that stores tokens in transit in plaintext becomes a custodian — it could spend them itself, and it learns `Y` before redemption, which lets it link sender to receiver and destroys the unlinkability blind signatures exist to provide. **A relay MUST therefore only ever hold ciphertext it cannot decrypt.** The decryption key lives exclusively in the URL fragment, which browsers and HTTP clients never transmit to servers.

The link itself remains a bearer credential, like the token. Burn-after-read and a short TTL shrink the window in which a leaked link is worth anything.

## Link format

```
<relay-origin>/t/<id>#<key>
```

- `<relay-origin>`: the relay's HTTPS origin (a mint MAY run a relay, but the relay is a separate capability — wallets MAY use any relay, and a mint is not required to run one).
- `<id>`: 16 random bytes, base64url without padding (22 chars). Unguessable; the relay MUST generate it, never the client.
- `<relay-origin>` MUST be `https://`. The fragment never leaves the client, but the path (and thus the `id`) does, and a plaintext-HTTP relay lets an on-path attacker race the recipient to the burn-after-read fetch. Wallets MUST refuse to create or resolve `http://` links, with a single exception for loopback hosts (`localhost`, `127.0.0.1`, `[::1]`) during development.
- `<key>`: 32 random bytes, base64url without padding (43 chars) — the AES-256-GCM key, **in the fragment only**.

## Encryption (client side)

```
key  = random(32 bytes)
iv   = random(12 bytes)
ct   = AES-256-GCM( key, iv, plaintext = UTF-8(token string), aad = none )
blob = base64url( iv || ct )          // ct includes the 16-byte GCM tag
```

The plaintext is the full PIP-06 token string (e.g. `picoA…`). The relay receives `blob` and nothing else.

## Relay API

A relay advertises itself in the mint's `GET /v1/info` as `relay: { "enabled": true, "max_bytes": 16384, "ttl_seconds": 86400 }`. If `relay` is absent or `enabled` is false, wallets MUST fall back to sharing the token string itself.

### `POST /v1/relay`

Request `{ "ct": "<blob>" }`. Rules: `ct` is base64url; decoded size ≤ `max_bytes` (`PAYLOAD_TOO_LARGE` otherwise). Response:

```jsonc
{ "id": "<22 chars>", "url": "https://relay.example/t/<id>", "expires_at": 1755720000 }
```

The wallet appends `#<key>` to `url` locally to form the link. The relay never sees the key.

### `GET /v1/relay/{id}`

Response `{ "ct": "<blob>" }` **exactly once**: the relay MUST delete the blob on first successful read (burn-after-read) and MUST delete it at `expires_at` if never read. Subsequent reads → 404 `RELAY_NOT_FOUND`. This is why senders MUST keep their own copy of the token until the recipient confirms receipt — a link lost in transit loses nothing.

### `GET /t/{id}` (browser handling, OPTIONAL)

When a person opens a link in a browser, the relay SHOULD serve a **reveal page** (content-negotiated on `Accept: text/html`; other clients get a JSON pointer `{ "link", "resolve" }`). The page:

- MUST NOT fetch `/v1/relay/{id}` on load. The blob is read only after an explicit user action ("Reveal"). This is what keeps link-preview bots, mail scanners, and accidental opens from burning a one-time link — a preview fetch sees an inert HTML page and nothing is consumed.
- MUST decrypt client-side with the key from the fragment (the browser never sends the fragment), render the token as text (never markup), offer a copy action, and tell the user the link is now used up.
- SHOULD remove the fragment from the address bar and history after reveal (`history.replaceState`), be served with `Cache-Control: no-store`, `Referrer-Policy: no-referrer`, `noindex`, and a CSP that permits only same-origin connections.
- MAY offer "Open in wallet": before reveal, by handing the *unconsumed* link (`<wallet>?link=<pointer>#<key>`) to a configured wallet UI; after reveal, by passing the token itself in a fragment (`<wallet>#token=<token>`). The reference mint configures this with `PICOCASH_RELAY_UI`.

The reference relay serves a self-contained page (no external assets), so any mint's links work without a website.

## Client behavior

1. Receiving a link: parse origin + id from the path, key from the fragment; `GET <origin>/v1/relay/<id>`; decrypt; the result MUST be a valid PIP-06 token (`pico` + version); then proceed exactly as for a pasted token — offline DLEQ verification, claim by swap.
2. Wallets MUST reject links whose key is missing/malformed or whose decrypted payload is not a token, and MUST treat decryption failure as "tampered or wrong link", never as a retryable relay error.
3. Wallets SHOULD display the relay origin before fetching, so a user can notice a link pointing at an unexpected host.

## What a relay learns

Upload size, upload time, and the IP addresses of uploader and downloader. Nothing about amounts, secrets, keysets, or the mint. Relays SHOULD be treated as untrusted and MAY be run by anyone; the reference mint runs one for convenience.

## Open questions for RFC

1. Should links carry the relay capability version (`/t/` vs `/t2/`) so the cipher suite can evolve without breaking parsers?
2. Multi-read links (e.g. N recipients, group payouts) — useful, but each read after the first widens the bearer window; likely a separate mechanism.
3. A `picocash:` URI scheme wrapping either a token or a link, for click-to-receive in native wallets.
