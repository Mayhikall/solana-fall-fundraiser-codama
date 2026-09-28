## Versions

anchor-cli 1.1.2 · node 22.21.1 · @codama/cli 1.6.3 · @codama/renderers-js 2.5.0 · @solana/kit 8.3.0

## TODO 3

Required: `fundraiser`, `vault` (plus `contributor`, `mintToRaise`, `amount`).
Optional: `contributorAccount`, `contributorAta`, `tokenProgram`, `systemProgram`.

`contribute` seeds the `fundraiser` PDA on `fundraiser.maker`, which is an internal field of the account being derived rather than an instruction input, creating a circular dependency where a finder would need the account to fetch its address. In contrast, `initialize` seeds `fundraiser` directly on the `maker` signer account which the caller supplies, allowing Codama to derive it automatically. Similarly, `vault` seeds on `fundraiser.mint_to_raise` and `fundraiser` authority, which depends on the unresolved `fundraiser` state. Meanwhile, `contributorAccount` seeds on `fundraiser` and `contributor` (both known in inputs), and `contributorAta` is an ATA for `contributor` and `mintToRaise` (both provided), so Codama can resolve them effortlessly.

## Bonus

Attempted and verified on-chain:
- **The Protocol Border**: Codama builds instructions adhering to the `@solana/kit` specification (`{ programAddress, accounts: [{ address, role }], data }`), whereas Anchor's `AnchorProvider` operates on `@solana/web3.js` v1 (`{ programId, keys: [{ pubkey, isSigner, isWritable }], data }`).
- **Adapter Mechanism**: `tests/helpers/kit-adapter.ts` bridges this gap in 20 lines by mapping Kit's `AccountRole` helpers (`isSignerRole`, `isWritableRole`) into web3.js `AccountMeta` flags and converting typed `Address` strings into `PublicKey` instances.
- **Signer Execution**: We passed `createNoopSigner(address(provider.publicKey.toBase58()))` to satisfy Kit's `InstructionSignerInput` without needing private key access at instruction generation time. When the converted instruction is submitted through `provider.sendAndConfirm`, Anchor's wallet detects its own public key marked as a signer and signs the transaction before sending.
- **On-chain Invariant**: Contributor contributions are capped at 10% of `TARGET` (3,000,000 units). Since the test setup already contributed 1,000,000 units, executing this second Codama-built contribution of 1,000,000 units remained within the cap and successfully incremented the vault balance by exactly `AMOUNT`.

## One thing that surprised me

How clean and zero-dependency the Codama-generated client feels compared to the dynamic Anchor runtime client, but also how heavily client-side DX depends on on-chain seed design: omitting `maker` from `contribute` saved an account in the transaction at the cost of breaking client-side PDA auto-derivation.
