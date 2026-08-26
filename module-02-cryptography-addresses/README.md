# Module 2: Cryptography & Addresses
 
Ed25519 keypairs, Base58 encoding, and address derivation (PDA and Associated Token Accounts) - the addressing foundation used by every later module and every phase of `solana_flutter_sdk`. Sourced exclusively from [official Solana documentation](https://solana.com/docs) and official GitHub repositories (`anza-xyz`, `solana-program`).
 
## 1. Ed25519 Keypairs
 
A regular Solana address is a 32-byte Ed25519 public key; the full keypair (public + private) is what `SolanaKeypair.generate()` (`0.0.2-dev`) produces. Every required signer on a transaction provides a 64-byte Ed25519 signature over the serialized message.
 
| Element | Size |
|---|---|
| Public key / address | 32 bytes |
| Raw private key (seed) | 32 bytes |
| Solana's on-disk Keypair format (`id.json`, `solana-keygen`) | 64 bytes (32-byte seed + 32-byte public key, concatenated) |
| Signature | 64 bytes |
 
This confirms a decision already made in `docs-sdk/phase1`: `SolanaKeypair.privateKey` stores the raw 32-byte seed, not the 64-byte concatenated format `solana-keygen` uses - official source now backs that decision directly at the code level (`anza-xyz/solana-sdk`, `keypair/src/lib.rs`).
 
Beyond transaction signing (implicit - every required signer signs the message), Solana exposes a native **Ed25519 Program** precompile for verifying arbitrary Ed25519 signatures on-chain - useful when a program needs to validate an off-chain-signed message. It runs as native validator code rather than inside the sBPF VM, because cryptographic operations there would be too slow.
 
- Program ID: `Ed25519SigVerify111111111111111111111111111`
## 2. Base58 Encoding
 
Every 32-byte address is displayed as a Base58-encoded string, up to 44 characters. This is the only fact official docs confirm explicitly about Base58 on Solana.
 
**Honesty note**: official Solana documentation does **not** explain why Base58 was chosen, nor does it specify the exact alphabet or which characters it excludes. Only at the implementation level (the `five8` and `bs58` crates - third-party dependencies, not Solana-org repositories) can it be inferred that ambiguous characters (`0`, `O`, `I`, `l`) are excluded. This is not asserted as an official Solana fact - it's flagged as circumstantial, dependency-level evidence, not an official source.
 
## 3. Program Derived Addresses (PDA) - Full Derivation Algorithm
 
Module 1 introduced the concept; here is the exact algorithm, as documented at `solana.com/docs/core/pda/pda-derivation`:
 
1. Validate that the number of seeds does not exceed `MAX_SEEDS` (16) and no individual seed exceeds `MAX_SEED_LEN` (32 bytes).
2. SHA-256 hash all seeds, the `program_id`, and the string `"ProgramDerivedAddress"` together, producing a 32-byte result.
3. Check whether the result is a valid point on the Ed25519 curve.
4. If the result **is** on the curve → error (`InvalidSeeds`): that address would have a corresponding private key, violating the PDA security property.
5. If the result is **not** on the curve → return it as the PDA.
**How the "off-curve" check works technically** (code level, `anza-xyz/solana-sdk`, `pubkey/src/lib.rs`): the resulting 32 bytes are interpreted as a compressed Edwards y-coordinate (`CompressedEdwardsY`), and decompression onto the curve is attempted. If decompression succeeds, a valid point exists - the address is "on-curve" and rejected as a PDA. If it fails, no valid point exists for those bytes - the address is "off-curve" and safe to use, since no private key can exist for it.
 
`find_program_address` searches the bump seed **from 255 down to 1** (not down to 0 - see correction note below), calling `create_program_address` with each value until the result falls off the curve. The first value that succeeds is the canonical bump; using a non-canonical bump creates a second valid address for the same seeds, which can lead to vulnerabilities.
 
> **Correction to Module 1**: Module 1 states the bump seed is "tried from 255 down to 0." The official source (`solana.com/docs/core/pda/pda-derivation`) states it searches **"from 255 down to 1"** - 0 is never tried. Module 1 should be updated to match.
 
| Parameter | Official value |
|---|---|
| `create_program_address` compute cost | 1,500 CU |
| Additional cost per failed bump attempt in `find_program_address` | 1,500 CU |
| Max seeds | 16 |
| Max bytes per seed | 32 |
| Max PDA signers per CPI | 16 |
| Hash marker string | `"ProgramDerivedAddress"` (21 bytes) |
 
## 4. Associated Token Accounts (ATA)
 
An ATA is a PDA derived specifically by the **Associated Token Account Program**, using these three seeds, in this exact order:
 
```
seeds = [wallet_address, token_program_id, mint_address]
program_id = ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
```
 
The `token_program_id` seed matters because the Token Program and Token-2022 are distinct programs:
 
| Program | Program ID |
|---|---|
| Associated Token Account Program | `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL` |
| SPL Token Program (legacy) | `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` |
| Token-2022 (Token Extensions) Program | `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` |
 
Important distinction: the ATA's **address** is derived by the Associated Token Account Program, but the resulting **account** is owned by the Token Program or Token-2022 - not by the Associated Token Account Program itself. For any given wallet + token program + mint combination, exactly one ATA exists.
 
## Summary
 
Ed25519 (32/64-byte keypairs, 64-byte signatures), Base58 (address encoding, with no officially documented rationale), and address derivation (PDA via SHA-256 + curve check, ATA as a specific PDA case) are the addressing foundation for everything that follows - especially SDK Phase 2 (`docs-sdk/phase-2`, PDA and ATA derivation) and every later program module.
 
## Sources
 
- [Solana Docs: Terminology](https://solana.com/docs/references/terminology)
- [Solana Docs: Account Structure](https://solana.com/docs/core/accounts/account-structure)
- [Solana Docs: Transactions](https://solana.com/docs/core/transactions)
- [Solana Docs: Precompiles](https://solana.com/docs/core/programs/precompiles)
- [Solana Docs: PDA Derivation](https://solana.com/docs/core/pda/pda-derivation)
- [Solana Docs: Create Token Account](https://solana.com/docs/tokens/basics/create-token-account)
- [anza-xyz/solana-sdk - pubkey/src/lib.rs](https://github.com/anza-xyz/solana-sdk/blob/master/pubkey/src/lib.rs)
- [anza-xyz/solana-sdk - keypair/src/lib.rs](https://github.com/anza-xyz/solana-sdk/blob/master/keypair/src/lib.rs)
- [solana-program/associated-token-account - address.rs](https://github.com/solana-program/associated-token-account/blob/main/interface/src/address.rs)
- [solana-program/token - lib.rs](https://github.com/solana-program/token/blob/main/interface/src/lib.rs)
- [solana-program/token-2022 - lib.rs](https://github.com/solana-program/token-2022/blob/main/interface/src/lib.rs)
## Related
 
- [solana_flutter_sdk](https://github.com/nemorixgroup/solana-flutter-sdk) - the SDK this knowledge base supports
- [docs-sdk](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/README.md) - SDK implementation decisions
- [Module 1: Architecture & Core Concepts](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/module-01-architecture-core-concepts/README.md)
