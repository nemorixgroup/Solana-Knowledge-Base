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

```
Seed: 9d61b19deffd5a60ba844af492ec2cc44449c5697b326919703bac031cae7f60
Public key: d75a980182b10ab7d54bfed3c964073a0ee172f3daa62325af021a68f707511a
```

## What 0.0.3-dev Tests Actually Check

Eight new tests cover this release, added to `test/src/crypto/solana_keypair_test.dart` alongside `0.0.2-dev`'s four:

1. `fromSeed` recreates the exact same keypair (public and private key) from the same seed.
2. Two different seeds produce two different keypairs.
3. `fromSeed` rejects a seed shorter than 32 bytes with `ArgumentError`.
4. `fromSeed` rejects a seed longer than 32 bytes with `ArgumentError`.
5. `fromSeed` derives the correct public key for RFC 8032 Test 1's seed (known-answer test).
6. `fromSecretKey` correctly imports a valid 64-byte `secretKey`, recovering the original seed and public key.
7. `fromSecretKey` rejects a `secretKey` that is not exactly 64 bytes with `ArgumentError`.
8. `fromSecretKey` rejects a `secretKey` whose embedded public key does not match the one derived from its embedded seed, with `ArgumentError`.

## Out of Scope for 0.0.3-dev

- Message and transaction signing/verification: `0.0.4-dev`.
- BIP39 mnemonic support: `0.0.5-dev`.
- SLIP-0010 derivation (path `m/44'/501'/0'/0'`): `0.0.6-dev`.
- Base58 encoding of `publicKey` into a displayable Solana address: `Phase 2`.

## Related

- [Phase 1: Keypair Generation](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/phase1/keypair-generation/README.md), the `0.0.2-dev` document this one continues.
- [CHANGELOG.md, 0.0.3-dev entry](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/CHANGELOG.md#003-dev), the release notes this document expands on.

## Sources

- [package:cryptography on pub.dev](https://pub.dev/packages/cryptography)
- [solana-web3.js Keypair class](https://solana-foundation.github.io/solana-web3.js/classes/Keypair.html)
- [RFC 8032, Section 7.1: Test Vectors](https://www.rfc-editor.org/rfc/rfc8032#section-7.1)
