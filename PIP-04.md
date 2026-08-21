# PIP-04: Vault, factory & custody

**Status: RFC (v0.1-draft)** · Reference implementation: [picocash/picocash-contracts](https://github.com/picocash/picocash-contracts) (`PicocashVault.sol` + `PicocashVaultFactory.sol`), built after the mint ran against a fake vault so the interface was dictated by a running consumer. Current Tempo **Moderato testnet** deployments (vault v3: emergency redemption + withdrawal breaker): factory `0xE49A8fEA32448bd7cBFF7Aa0A3509e473D4CC377`, shared emergency verifier `0x7b64972Dd8027f64a2186E5831272774e2f0eC84`, hosted-mint vault `0x4380094eeEF8AB12B868bFBB46c7e7B90a713a83` (`mint.picocash.dev`), dev-mint vault `0xA46E150426959dbd40A3bAD372C8ABbBE57b8396` — all Sourcify exact-match; token pathUSD `0x20c0…0000`, 2-day rotation timelock, 6000-block attestation interval, 967,680-block (~7-day) emergency grace, breaker 50 % of backing per 5400 blocks (~1 h; a loose testnet setting — production mints should size it to a small multiple of honest daily melt volume). The v1 (`0xbcaa…0BFe`) and v2 (`0xBFEB…ba99`) factories and their vaults are drained and retired.

**Factory & on-chain discovery.** Vaults are deployed via `PicocashVaultFactory.deployVault(token, operator, timelock, publishThresholdBps, publishIntervalBlocks, maxMeltFee, name, mintUrl, emergencyGraceBlocks, meltLimitBps, meltEpochBlocks)` — permissionless, retaining no authority over deployed vaults (`isVault()` proves canonical bytecode; `VaultDeployed` events enumerate the ecosystem). Each vault exposes a read-only `info()`: the stable subset of the mint's `/v1/info` ([PIP-02](PIP-02.md)) (name, mint API URL, token, operator, active keyset) plus live custody figures (backing balance, last published outstanding supply + timestamp, publication policy and due-state) — a client can go from factory to vault to mint URL to keys with no off-chain registry, judging solvency on the way. The operator maintains the metadata via `setMintInfo` / `setActiveKeyset`.

**Publication policy (deploy-time commitment).** Every vault MUST set at least one publication rule at deployment; deploying with none reverts. The rules:

1. **Threshold** (`publishThresholdBps`): a publication is *due* when the backing balance has drifted more than this many basis points since the last publication (balance drift is the on-chain-observable proxy for outstanding drift). A soft trigger — the operator's publish job polls `isPublicationDue()` and publishes when it fires; nothing is blocked.
2. **Interval** (`publishIntervalBlocks`): more than this many blocks without a publication (including never having published) is a *breach* — `isPublicationOverdue()` turns true, and the vault **refuses allowance deposits (`ecashMint`) while overdue**: a mint that stops attesting stops taking new money. Withdrawals are never affected (exit is sacred), and memo-path deposits are plain token transfers the vault cannot intercept — the mint server enforces the same rule when issuing quotes.

The policy turns the solvency heartbeat from a courtesy into a machine-checkable commitment: silence has a defined meaning (`isPublicationOverdue()`), and publications happen exactly when they carry information. `publishOutstandingSupply` records balance-at-publish and block number to make both rules evaluable on-chain.

**Melt-fee ceiling (`maxMeltFee`) — the exit-tax cap.** Withdrawals being never-pausable is not enough on its own: an unbounded melt fee would be a backdoor exit tax that soft-freezes redemptions through pricing. Every vault therefore commits at deployment to a ceiling on the melt fee. A mint MUST NOT quote `fee` above its vault's `maxMeltFee` (the reference mint refuses to even start in that state), and wallets SHOULD check `vault.info().maxMeltFee` against `/v1/info fees.melt` ([PIP-03](PIP-03.md)) before depositing. Ceiling changes are asymmetric by design: **decreases are instant** (`decreaseMaxMeltFee`, user-favorable), **increases go through the rotation timelock** (`proposeMaxMeltFeeIncrease` → `applyMaxMeltFeeIncrease`) — raising the cost of exit requires the same public notice as changing who controls custody. The complete exit guarantee a holder gets at deposit time: *the contract cannot pause your payout, and the maximum it can ever cost you to leave is known on-chain, with any increase visible for the full timelock in advance.*

**One vault per currency.** The vault is bound to exactly one TIP-20 token at deployment; `vault.token()` is the on-chain authority for the mint's unit binding (`tip20:<chain_id>:<token_address>`, [PIP-01](PIP-01.md)), and the mint refuses to start if its configured unit disagrees. A mint supporting multiple currencies runs one vault per unit — solvency stays one number against one number, and a fault in one currency's custody cannot touch another's.

The primary deposit flow needs no vault call at all: a TIP-20 `transferWithMemo(vault, amount, quoteId)` credits the vault and emits the memo event the mint watches (memo is indexed on Tempo's TIP-20). `ecashMint(amount, mintQuoteId)` exists as an allowance-based fallback. Melt payouts go through `ecashMelt(to, amount, meltId)` — operator-only, **one payout per meltId enforced on-chain**, and with no pause check anywhere on the path.

Settled design constraints the implementation honors:

- Holds **the unit's TIP-20 stablecoin** (e.g. USDC.e on mainnet; pathUSD on Moderato testnet) backing outstanding tokens 1:1 — the token is an immutable constructor parameter, one vault per token.
- `ecashMint(amount, mintQuoteId)` — pulls the backing token via the `transferWithMemo` pattern; the memo binds the deposit to a mint quote; emits the event the mint watches before releasing blind signatures.
- `ecashMelt(to, amount, meltId)` — mint-operator-signed release for melts.
- **Solvency invariant**: vault balance ≥ outstanding token supply per keyset. The balance is on-chain truth. The outstanding supply is an **operator attestation** published on-chain under the vault's committed policy — anyone can compare the two, and a false attestation is publicly timestamped, but it is not a cryptographic proof of liabilities. (A holder-verifiable proof-of-liabilities scheme is an open RFC item.)
- Operator key rotation is **timelocked** (length is a per-deployment constructor argument, readable from `info()`). Deposits are pausable; **payouts are never contract-pausable** — there is no switch that stops `ecashMelt`. Liveness still depends on the operator signing melts in the current version; the publication policy and public timelocks make abandonment visible early, and **§Emergency redemption** below proposes the unilateral exit that cures it.
- Foundry; deployed to Tempo testnet first (build step 5), mainnet reference deployment behind hard caps (per-wallet and global outstanding limits) only at build step 9.

## Withdrawal breaker — bounding an active operator

**Status: implemented in vault v3, live on Moderato, exercised on-chain.** Emergency redemption covers a *silent* operator. An *active* one — the operator key paying itself — is the other half of the custody risk, and it cannot be prevented: `ecashMelt` is operator-signed and the vault cannot distinguish a real melt from theft (and an operator that holds the keyset key can sign itself any token it likes, so proof-verified melts would not help). What the contract can do is make theft **slow, visible, bounded, and escapable**:

- **Bounded.** Melt volume per `meltEpochBlocks` is capped at `meltLimitBps` of the backing balance at the epoch's start. An over-limit melt reverts (`MeltLimitExceeded(remaining)`). Both parameters are deploy-time commitments; `meltLimitBps` may be **decreased instantly** and **increased only through the rotation timelock** (`proposeMeltLimitIncrease` → `applyMeltLimitIncrease`).
- **Escapable.** The melt that consumes the allowance **latches** the breaker (`breakerTrippedAt`). While latched: `ecashMelt` and `ecashMint` revert, and `emergencyMode()` is true **immediately — no grace period** — so every holder can redeem at the vault with their proofs. The operator's path closes at the same moment the holders' path opens; "payouts are never pausable" is preserved for holders, not for the key.
- **Visible.** `breakerInfo()` exposes limit, epoch, baseline, allowance, melted-so-far, and the trip block; `BreakerEpoch` / `BreakerTrippedEvent` are emitted. The reference mint's status page shows utilisation and turns amber above 50 %.
- **Slow.** Un-latching takes the rotation timelock (`proposeBreakerReset` → `applyBreakerReset`) — public notice, during which holders keep exiting.

**Residual, stated plainly.** An operator can drain *just under* the allowance every epoch without ever tripping the breaker — `meltLimitBps` per epoch, indefinitely, in full view. Wallets and services SHOULD treat sustained high utilisation like a missed attestation and leave through the normal melt path (which keeps working). An operator who lies in the attestation is, as before, the remaining trust assumption; the breaker bounds the *rate* of what that trust can cost. `0/0` disables the breaker; wallets SHOULD read `breakerInfo()` before depositing and treat a disabled breaker as a material weakness.

Measured on Moderato (rogue-operator drill, 20 %/1000-block breaker on a $1 vault): a $0.50 payout reverted, a $0.20 payout succeeded and latched, a further $0.000001 reverted, and a holder redeemed $0.05 from the tripped vault with no mint involved in the same minute. Gas: the breaker adds ≈ 25 k to an epoch-rolling melt, ≈ 8 k otherwise.

## Emergency redemption — unilateral holder exit

**Status: implemented in vault v2, live on Moderato (see deployments above), exercised end-to-end on-chain.** Everything above makes an abandoned mint *visible* (deposits halt when the attestation is overdue) but not *survivable*: `ecashMelt` is operator-only, so if the operator disappears the backing is un-pausable, fully visible, and stuck. This section closes that gap. The key observation is that every picocash proof already carries a DLEQ payload ([PIP-00](PIP-00.md) §3) that can be verified with nothing but the mint's **public** key — so the vault can check tokens itself.

### Mechanism

1. **Keyset registry.** The operator registers each keyset's public keys per denomination on the vault (`registerKeyset(keysetId, amounts[], pubkeys[])`), idempotently and append-only. These are the same keys `GET /v1/keys` serves; publishing them on-chain costs the operator nothing in privacy and makes the vault self-sufficient. A keyset that is not registered cannot be emergency-redeemed — wallets MUST treat an unregistered active keyset as a red flag, exactly like a missed attestation.
2. **Trigger.** `emergencyMode()` is true once the publication policy has been overdue for longer than a deploy-time **grace period** (`emergencyGraceBlocks`, e.g. ~7 days). Nothing is switched by anyone: it is a pure function of the last publication and the clock. A returning operator clears it by publishing again; holders are never locked out of the normal path in the meantime — `ecashMelt` keeps working throughout.
3. **Redeem.** In emergency mode, anyone may call `emergencyRedeem(proofs[], to)` where each proof is `(amount, keysetId, secret, C, e, s, r)`. For each, the vault:
   - recomputes `Y = hash_to_curve(secret)` and verifies the DLEQ against the registered key for `(keysetId, amount)` — this proves the mint signed exactly this token;
   - checks `Y` against the vault's own **on-chain spent set** and records it;
   - enforces the spending condition if the secret is `P2PK` ([PIP-08](PIP-08.md)): the witness signature(s) are verified on-chain too, so a locked token stays locked in an emergency;
   - pays `amount` of the backing token to `to`. No fee: the melt-fee model exists to price operator gas, and here the redeemer pays their own.
4. **Cap.** Total emergency payouts per keyset MUST NOT exceed the last published `outstanding` for that keyset (plus a small tolerance for issuance after the last publication, bounded by the policy's threshold). This is what the attestation is *for*: it turns the operator's last honest statement into a ceiling on what a post-mortem attacker can extract.

### The hard part, stated honestly

The mint's spent-secret ledger is off-chain. A token that was swapped or melted at the mint before it went dark is indistinguishable, on-chain, from a live one — so an adversary who kept copies of tokens they already spent can present them in emergency mode, and the vault cannot know. Mitigations, in decreasing strength:

- **The cap** bounds the total loss to roughly `outstanding` at the last attestation: honest holders and the attacker compete for the same bounded pool, but the vault can never be drained below what was owed.
- **A final spent-set commitment.** An operator that is winding down *deliberately* (rather than vanishing) publishes a Merkle root of its spent `Y` set before the grace period ends; emergency redemptions must then carry a non-membership proof. This turns an orderly shutdown into a clean exit and leaves only the disorderly case exposed.
- **Short attestation intervals** shrink the window between the last known-good `outstanding` and the failure.
- **Wallet hygiene**: wallets SHOULD swap received tokens promptly (they already must, to own them), which keeps the live set small relative to history.

What this does **not** solve: a malicious operator who lies in the last attestation. That is the same trust assumption as today, now bounded in time and amount rather than open-ended. A holder-verifiable proof of liabilities remains the open RFC item.

### Implementation and cost

`picocash-contracts/src/emergency/` contains `Secp256k1.sol` (Jacobian arithmetic, Shamir double-scalar multiplication, SEC1 decompression via the modexp precompile), `EcashProofVerifier.sol` (`hash_to_curve` per PIP-00, `hashE` over lowercase uncompressed hex per NUT-12, `verifyProof(secret, C, K, e, s, r) → (valid, Y)`), and `PicocashEmergencyVerifier.sol` — a stateless contract the factory deploys once and every vault shares, which adds the PIP-08 evaluation: the caller supplies the P2PK conditions as a struct, the contract re-serialises them canonically and requires a byte-exact match with the secret (so the conditions cannot be misstated), then verifies BIP-340 Schnorr witnesses for the lock key before `locktime` or a `refund` key after. `pubkeys`/`n_sigs > 1` multisig is refused in v2. Tested against vectors generated by the reference TypeScript crypto, including real Schnorr witnesses from the lock, refund, and a stranger key.

Measured (Foundry): **≈2.1 M gas per plain proof, ≈2.35 M per P2PK-locked proof** (the Schnorr check adds one more double-scalar multiplication). On Moderato a six-proof redemption cost ≈ $0.017 in total — acceptable for an exit path used once, and a redeemer holding many small denominations batches them in one call. Tempo's receipts report a higher gas figure than the EVM reference (its own state pricing); the fee is what matters. Obvious optimisations, none needed for correctness: fold `s·G − e·K` into the `ecrecover` trick (saves one multiplication — but the *point* is needed for `hashE`, not just its address, so only `R1` can use it), wNAF windows, and a hypothetical secp256k1 precompile.

### Interface (vault v2+, current v3)

```solidity
function registerKeyset(bytes8 keysetId, uint256[] calldata amounts, bytes[] calldata pubkeys) external; // operator, append-only
function keysetKey(bytes8 keysetId, uint256 amount) external view returns (bytes memory);
function emergencyGraceBlocks() external view returns (uint64);      // immutable, deploy-time (0 = disabled; requires the interval rule)
function emergencyMode() external view returns (bool);               // block > (lastPublished or deploy) + interval + grace
function emergencyCap() external view returns (uint256);             // last attested outstanding; balance if never attested
function emergencyRedeem(PicocashEmergencyVerifier.Proof[] calldata proofs, address to) external; // anyone, only in emergencyMode
function emergencySpent(bytes32 y) external view returns (bool);
function emergencyRedeemed() external view returns (uint256);        // running total vs cap
function emergencyInfo() external view returns (bool mode, uint64 graceBlocks, uint256 redeemed, uint256 cap, address verifier);
// vault v3 — withdrawal breaker
function breakerInfo() external view returns (uint16 limitBps, uint64 epochBlocks, uint256 epochStart, uint256 baseline, uint256 allowance, uint256 melted, uint256 trippedAt);
function decreaseMeltLimit(uint16 newBps) external;            // operator, instant
function proposeMeltLimitIncrease(uint16 newBps) external;     // operator, then applyMeltLimitIncrease() after the timelock
function proposeBreakerReset() external;                        // operator, then applyBreakerReset() after the timelock
```

The reference mint ships `scripts/register-keyset.ts` (operator: publish the keys `GET /v1/keys` serves) and `scripts/emergency-redeem.ts` (holder: redeem a `picoA` token or proof list at the vault with no mint involved; `--unlock-key` signs P2PK-locked proofs). Wallets SHOULD read `emergencyInfo()` and `keysetKey()` for the active keyset before depositing and warn when the grace period is 0 or the keyset is unregistered.

Open for review: the grace period default, the cap tolerance (v2 uses zero), the never-attested fallback to balance, and whether the orderly-shutdown spent-set commitment should be mandatory.
