# PIP-04: Vault, factory & custody

**Status: RFC (v0.1-draft)** · Reference implementation: [picocash/picocash-contracts](https://github.com/picocash/picocash-contracts) (`PicocashVault.sol` + `PicocashVaultFactory.sol`), built after the mint ran against a fake vault so the interface was dictated by a running consumer. Current Tempo **Moderato testnet** deployments: factory `0xbcaa0658103C88B30c7028d2f28964403AEf0BFe`, dev-mint vault `0xd409D3c16F3472bD75fb86eF3f2D69d602F3cfA3` (deployed through the factory; token pathUSD `0x20c0…0000`, 2-day rotation timelock).

**Factory & on-chain discovery.** Vaults are deployed via `PicocashVaultFactory.deployVault(token, operator, timelock, publishThresholdBps, publishIntervalBlocks, maxMeltFee, name, mintUrl)` — permissionless, retaining no authority over deployed vaults (`isVault()` proves canonical bytecode; `VaultDeployed` events enumerate the ecosystem). Each vault exposes a read-only `info()`: the stable subset of the mint's `/v1/info` ([PIP-02](PIP-02.md)) (name, mint API URL, token, operator, active keyset) plus live custody figures (backing balance, last published outstanding supply + timestamp, publication policy and due-state) — a client can go from factory to vault to mint URL to keys with no off-chain registry, judging solvency on the way. The operator maintains the metadata via `setMintInfo` / `setActiveKeyset`.

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

## Emergency redemption — unilateral holder exit (proposed, v0.2)

**Status: design + verified prototype, not deployed.** Everything above makes an abandoned mint *visible* (deposits halt when the attestation is overdue) but not *survivable*: `ecashMelt` is operator-only, so if the operator disappears the backing is un-pausable, fully visible, and stuck. This section closes that gap. The key observation is that every picocash proof already carries a DLEQ payload ([PIP-00](PIP-00.md) §3) that can be verified with nothing but the mint's **public** key — so the vault can check tokens itself.

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

### Prototype and cost

`picocash-contracts/src/emergency/` contains `Secp256k1.sol` (Jacobian arithmetic, Shamir double-scalar multiplication, SEC1 decompression via the modexp precompile) and `EcashProofVerifier.sol` (`hash_to_curve` per PIP-00, `hashE` over lowercase uncompressed hex per NUT-12, and `verifyProof(secret, C, K, e, s, r) → (valid, Y)`). It is tested against vectors generated by the reference TypeScript crypto: three real mint signatures (plain, `PC-BIND`, and `P2PK` secrets) verify; wrong secret, wrong `r`, wrong `e`, wrong `C`, zero scalars, and off-curve points are rejected.

Measured: **≈1.75 M gas per proof** (four double-scalar multiplications, dominated by ~1,000 field multiplications each). On Tempo that is on the order of a cent per proof — acceptable for an exit path used once, and a redeemer holding many small denominations can batch. Obvious optimisations, none needed for correctness: fold `s·G − e·K` into the `ecrecover` trick (saves one multiplication — but the *point* is needed for `hashE`, not just its address, so only `R1` can use it), wNAF windows, and a hypothetical secp256k1 precompile.

### Interface sketch

```solidity
function registerKeyset(bytes8 keysetId, uint256[] calldata amounts, bytes[] calldata pubkeys) external onlyOperator;
function emergencyGraceBlocks() external view returns (uint256);     // immutable, deploy-time
function emergencyMode() external view returns (bool);               // overdue for > grace, no one flips it
function emergencyRedeem(Proof[] calldata proofs, address to) external;  // anyone, only in emergencyMode
function emergencySpent(bytes32 y) external view returns (bool);
function emergencyRedeemed(bytes8 keysetId) external view returns (uint256); // running total vs cap
```

Adding this to the vault is a **new contract version** (new factory, new bytecode hash) — existing testnet vaults will not gain it. Open for review: the grace period default, whether the cap tolerance should be zero, and whether the spent-set commitment should be mandatory for orderly shutdown.
