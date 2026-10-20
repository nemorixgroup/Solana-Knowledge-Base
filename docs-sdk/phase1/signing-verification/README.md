# Phase 1: Signing & Verification

**Shipped in:** ✅ 0.0.4-dev  
**Files:** [lib/src/crypto/solana_keypair.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/lib/src/crypto/solana_keypair.dart), [test/src/crypto/solana_keypair_test.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/test/src/crypto/solana_keypair_test.dart), [example/phase1/keypair_signing_example.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/example/phase1/keypair_signing_example.dart)

## Summary

`SolanaKeypair.sign()` and `SolanaKeypair.verifySignature()` add the core Ed25519 signing primitives on top of `0.0.2-dev`'s generation and `0.0.3-dev`'s import. Every required signer on a Solana transaction provides a signature of this exact form over the serialized message; building and signing an actual transaction message is a later phase, this release covers the underlying cryptographic primitive only.

## Design Decision: verifySignature as a Static Method

`sign()` is an instance method: it needs `this.privateKey`, so it only makes sense on a `SolanaKeypair` that holds one.

Verification is different: checking a signature only ever needs the signer's public key, never a private key. The common case is verifying a signature produced by someone else's keypair, for which this SDK never has a `SolanaKeypair` instance in the first place, only their public key bytes. Making `verifySignature` a static method taking `publicKey` directly avoids forcing callers to construct a throwaway keypair-shaped object just to validate a signature.

## Design Decision: Upfront Length Validation

Both `publicKey` (32 bytes) and `signature` (64 bytes) are validated before calling into `package:cryptography`, throwing `ArgumentError` that names the expected length. This matches the pattern already established in `0.0.3-dev` for `fromSeed`/`fromSecretKey`.

## Design Decision: Extending the RFC 8032 Known-Answer Test

`0.0.3-dev` verified `fromSeed`'s seed-to-public-key derivation against RFC 8032 Section 7.1's Test 1 vector. That same official vector also publishes the expected signature for signing the empty message with that seed, so `0.0.4-dev` extends the known-answer coverage to `sign` and `verifySignature` using it.

The expected signature was independently cross-checked with Python's `cryptography` library (OpenSSL-backed Ed25519) before being added to the test suite, following this SDK's standing practice of verifying test vectors outside the implementation under test.

```
Seed: 9d61b19deffd5a60ba844af492ec2cc44449c5697b326919703bac031cae7f60
Public key: d75a980182b10ab7d54bfed3c964073a0ee172f3daa62325af021a68f707511a
Message: (empty)
Signature: e5564300c360ac729086e2cc806e828a84877f1eb8e5d974d873e065224901555fb8821590a33bacc61e39701cf9b46bd25bf5f0595bbe24655141438e7a100b
```


## What 0.0.4-dev Tests Actually Check

Eleven new tests cover this release, added to `test/src/crypto/solana_keypair_test.dart` alongside `0.0.2-dev`'s four and `0.0.3-dev`'s eight:

1. `sign` produces a 64-byte signature.
2. `sign` is deterministic: signing the same message twice with the same keypair produces the same signature.
3. `sign` produces different signatures for different messages.
4. `sign` matches the RFC 8032 Test 1 known-answer signature for the empty message.
5. `verifySignature` returns `true` for a signature produced by `sign` over the original message and public key.
6. `verifySignature` returns `false` when the message was tampered with after signing.
7. `verifySignature` returns `false` when the signature bytes were tampered with.
8. `verifySignature` returns `false` when checked against a different keypair's public key.
9. `verifySignature` rejects a `publicKey` that is not exactly 32 bytes with `ArgumentError`.
10. `verifySignature` rejects a `signature` that is not exactly 64 bytes with `ArgumentError`.
11. `verifySignature` matches the RFC 8032 Test 1 known-answer test (returns `true` for the official seed/message/signature combination).

## Out of Scope for 0.0.4-dev

- BIP39 mnemonic support: `0.0.5-dev`.
- SLIP-0010 derivation (path `m/44'/501'/0'/0'`): `0.0.6-dev`.
- The on-chain Ed25519 Program precompile instruction format (`Ed25519SigVerify111111111111111111111111111`), used to verify arbitrary signed messages inside a Solana transaction: this is a transaction-instruction format, not a cryptographic primitive, and belongs to a later phase (Phase 4, Transaction Construction, or a dedicated gap item).

## Related

- [Phase 1: Keypair Import](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/phase1/keypair-import/README.md), the `0.0.3-dev` document this one continues.
- [CHANGELOG.md, 0.0.4-dev entry](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/CHANGELOG.md), the release notes this document expands on.

## Sources

- [package:cryptography on pub.dev](https://pub.dev/packages/cryptography)
- [Solana Docs: Transactions](https://solana.com/docs/core/transactions)
- [Solana Docs: Precompiles](https://solana.com/docs/core/programs/precompiles)
- [RFC 8032, Section 7.1: Test Vectors](https://www.rfc-editor.org/rfc/rfc8032#section-7.1)
