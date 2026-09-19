# Phase 1: Keypair Import

**Shipped in:** ✅ 0.0.3-dev  
**Files:** [lib/src/crypto/solana_keypair.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/lib/src/crypto/solana_keypair.dart), [test/src/crypto/solana_keypair_test.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/test/src/crypto/solana_keypair_test.dart), [example/phase1/keypair_import_example.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/example/phase1/keypair_import_example.dart)

## Summary

`SolanaKeypair.fromSeed()` and `SolanaKeypair.fromSecretKey()` add deterministic keypair reconstruction on top of `0.0.2-dev`'s random `generate()`: restoring a previously generated keypair from its 32-byte seed, and importing a keypair created by external Solana tooling from its 64-byte `secretKey` format.

## Design Decision: Cross-Checking fromSecretKey's Embedded Public Key

As noted in the `0.0.2-dev` documentation, Solana's 64-byte `secretKey` format (used by the Solana CLI's `id.json` and `@solana/web3.js`'s `Keypair.secretKey`) concatenates a 32-byte seed with its 32-byte public key, following the TweetNaCl convention.

`fromSecretKey` splits the input at byte 32, re-derives the public key from the seed half via `fromSeed`, and compares it against the embedded public key half (bytes 32-63). If they don't match, it throws `ArgumentError` instead of silently returning a keypair whose exposed `publicKey` disagrees with what the original 64-byte value implied. Official Solana documentation does not explicitly require this check, but it is a defensive measure against a corrupted or hand-edited key file going undetected until on-chain use fails.

## Design Decision: Upfront Length Validation

Both `fromSeed` (32 bytes) and `fromSecretKey` (64 bytes) validate their input length before calling into `package:cryptography`, throwing `ArgumentError` that names the expected length. This surfaces a wrong-length seed or secretKey immediately and explicitly, rather than as an opaque failure several layers down inside the Ed25519 implementation.

## Design Decision: Closing the RFC 8032 Known-Answer Test

`0.0.2-dev` deferred independent test-vector verification because `generate()` produces random output with no fixed expected result to check against. With `fromSeed` now implemented, this became possible: `0.0.3-dev` tests `fromSeed` against RFC 8032 Section 7.1's Test 1 vector, the official published seed-to-public-key pair.

The vector was independently cross-checked with Python's `cryptography` library (OpenSSL-backed Ed25519) before being added to the test suite, following this SDK's standing practice of verifying test vectors outside the implementation under test. `pynacl` was the originally planned tool per the `0.0.2-dev` docs; `cryptography` was used instead as the independent Python implementation available in this environment. Both wrap mature, independently audited Ed25519 implementations, so the cross-check carries the same weight.

## Related

- [Module 1: Architecture & Core Concepts](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/module-01-architecture-core-concepts/README.md), general background on Solana's account model and addresses.
- [Module 2: Cryptography & Addresses](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/module-02-cryptography-addresses/README.md), Ed25519 keypairs, Base58 encoding, address derivation
- [CHANGELOG.md, 0.0.3-dev entry](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/CHANGELOG.md#003-dev), the release notes this document expands on.
