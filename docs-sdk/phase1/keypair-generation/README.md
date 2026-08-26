# Phase 1: Keypair Generation

**Shipped in:** ✅ 0.0.2-dev  
**Files:** [lib/src/crypto/solana_keypair.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/lib/src/crypto/solana_keypair.dart), [test/src/crypto/solana_keypair_test.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/test/src/crypto/solana_keypair_test.dart)

## Summary

`SolanaKeypair.generate()` produces a new, random Ed25519 keypair: a 32-byte public key and a 32-byte private key seed, via `package:cryptography`'s `Ed25519` implementation. This is the first real implementation on top of the 0.0.1-dev scaffold, and the foundation every later phase builds on.

## Library Choice: package:cryptography

Confirmed principle for this SDK, matching Nemorix Group's other native SDKs: minimize third-party dependencies, and prefer `package:cryptography`'s general-purpose Ed25519 implementation over niche or unmaintained alternatives.

Two alternatives were considered and rejected before starting Phase 1:

- `ed25519_edwards`: synchronous API, but unmaintained for roughly 4 years at the time of writing.
- `edwards25519`: actively maintained, but a low-level curve-only library, using it would mean implementing key derivation from scratch instead of relying on a tested, audited implementation.

`package:cryptography`'s `Ed25519` class exposes `newKeyPair()`, `extractPublicKey()`, and `extractPrivateKeyBytes()`, everything 0.0.2-dev needed, with an async API. This mirrors the same choice already made for XRPL's Ed25519 key derivation.

## Design Decision: 32-byte Seed, Not the 64-byte secretKey Format

`SolanaKeypair.privateKey` exposes the raw 32-byte Ed25519 seed produced by `package:cryptography`, not the 64-byte `secretKey` format used by some Solana tooling, for example the Solana CLI's `id.json`, or `@solana/web3.js`'s `Keypair.secretKey`, which concatenates the seed with the public key.

This was confirmed against the official `@solana/web3.js` `Keypair` class documentation before implementing: it exposes two distinct constructors, `fromSeed(seed: Uint8Array)`, documented as taking "a 32 byte seed", and `fromSecretKey(secretKey: Uint8Array, options?)`. That distinction is a strong signal that `secretKey` is a different, larger format than the raw seed, consistent with the TweetNaCl convention used by Solana tooling: seed concatenated with the public key, 64 bytes total.

`0.0.2-dev` only generates a fresh random keypair, it does not yet need to interoperate with externally generated keys, so this distinction did not block the release. Supporting the 64-byte format, reading an `id.json`, or a `Keypair.secretKey` byte array from another tool, is deferred to `0.0.3-dev`, when `fromSeed` and `fromSecretKey` are implemented.

## Design Decision: No Fixed Test Vector Yet

`generate()` produces random output on every call, so there is no single expected result to check the implementation against, independent official test vectors do not apply to pure randomness.

This SDK's standing practice, established across Nemorix Group's other SDKs, is to independently verify test vectors via Python (`hashlib`, `pynacl`, and similar) before finalizing tests, rather than trusting only internal round-trip tests. That practice resumes in `0.0.3-dev`: once `fromSeed` accepts a fixed, known seed, an RFC 8032 known-answer test vector, independently reproduced in Python via `pynacl`, will validate the derived public key.

## What 0.0.2-dev Tests Actually Check

Four tests cover this release, all in `test/src/crypto/solana_keypair_test.dart`:

1. `publicKey` and `privateKey` are each exactly 32 bytes.
2. Two calls to `generate()` produce different keys, confirming the random source is actually being used.
3. The public key is internally consistent with its own private key seed, re-derived independently via `Ed25519().newKeyPairFromSeed(...)`, not just trusted from the same code path that produced it.
4. `toString()` never includes the raw private key bytes, only its byte length.

## Out of Scope for 0.0.2-dev

- Deriving a keypair from an existing seed or secret key (`fromSeed`, `fromSecretKey`): `0.0.3-dev`.
- Message and transaction signing/verification: `0.0.4-dev`.
- BIP39 mnemonic support: `0.0.5-dev`.
- SLIP-0010 derivation: `0.0.6-dev`.

## Related

- [Module 1: Architecture & Core Concepts](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/module-01-architecture-core-concepts/README.md), general background on Solana's account model and addresses.
- [CHANGELOG.md, 0.0.2-dev entry](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/CHANGELOG.md), the release notes this document expands on.

## Sources

- [package:cryptography on pub.dev](https://pub.dev/packages/cryptography)
- [solana-web3.js Keypair class](https://solana-foundation.github.io/solana-web3.js/classes/Keypair.html)
- [Solana Docs: Accounts](https://solana.com/docs/core/accounts)
