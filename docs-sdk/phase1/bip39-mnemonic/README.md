# Phase 1: BIP-39 Mnemonic Support

**Shipped in:** ✅ 0.0.5-dev  
**Files:** [lib/src/crypto/bip39/bip39_mnemonic.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/lib/src/crypto/bip39/bip39_mnemonic.dart), [lib/src/crypto/bip39/bip39_wordlist.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/lib/src/crypto/bip39/bip39_wordlist.dart), [test/src/crypto/bip39/bip39_mnemonic_test.dart](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/test/src/crypto/bip39/bip39_mnemonic_test.dart)

## Summary

`Bip39Mnemonic` implements the BIP-39 standard end to end: generating a new mnemonic from secure random entropy, restoring and validating an existing one (checksum included), and deriving the 64-byte seed that later becomes the input to SLIP-0010 key derivation in `0.0.6-dev`. This release does not yet turn that seed into a `SolanaKeypair`, it only produces the seed itself.

## Library Choice: Built From Scratch, No Third-Party BIP-39 Package

Confirmed principle for this SDK: minimize third-party dependencies. A general-purpose BIP-39 implementation is not needed when the algorithm itself is a short, fully specified encoding (entropy, a SHA-256 checksum, and a fixed 2048-word list), so it was implemented directly rather than pulling in a dedicated `bip39`-style package.

Only one new building block was needed beyond what the SDK already used: `package:cryptography`'s `Pbkdf2` class (`macAlgorithm: Hmac.sha512()`, `iterations: 2048`, `bits: 512`) for the seed derivation step. `Sha256`, from the same package, was already in use for the mnemonic checksum. No new package dependency was added.

## Design Decision: The Official English Wordlist, Verified

The 2048-word English wordlist is a fixed public constant defined by the BIP-39 specification itself, byte for byte. Rather than trusting a single unverified source, it was generated from the MIT-licensed `trezor/python-mnemonic` reference package's bundled wordlist file, then verified programmatically: exactly 2048 entries, all unique, strictly alphabetically sorted, first entry `abandon`, last entry `zoo`. This followed this SDK's standing practice of verifying against an authoritative source rather than memory, especially for a value where a single wrong entry silently breaks interoperability with every other BIP-39 wallet.

## Design Decision: Rejecting Non-ASCII Passphrases

The BIP-39 spec requires both the mnemonic sentence and the optional passphrase to be Unicode NFKD-normalized before seed derivation. Dart's standard library has no built-in NFKD implementation. The official English wordlist is pure ASCII, where NFKD is a no-op, so the mnemonic sentence itself is unaffected. The passphrase is user-supplied, free-form text, so it could contain non-ASCII characters where normalization would actually matter.

`toSeed()` rejects a non-ASCII passphrase with an `ArgumentError` rather than silently deriving a seed that would disagree with every other BIP-39 wallet given the same non-ASCII passphrase. Adding a full Unicode normalization dependency only to support that edge case was judged not worth it for this release; an ASCII-only passphrase is unaffected either way.

## Design Decision: PBKDF2 Seed Derivation, Verified Against the Reference Implementation

`toSeed()` follows the BIP-39 spec exactly: PBKDF2-HMAC-SHA512, 2048 iterations, password = the mnemonic sentence (UTF-8), salt = the literal string `"mnemonic"` concatenated with the passphrase (UTF-8), 64-byte output.

Before finalizing the implementation, the full algorithm (entropy to words, checksum validation, and PBKDF2 seed derivation) was independently reproduced in Python against the official BIP-39 all-zero-entropy test vector and the `python-mnemonic` reference library's own output, rather than trusting a remembered or assumed-correct value. That cross-check is also what the test suite below exercises directly.

## What 0.0.5-dev Tests Actually Check

Tests cover `generate()`, `fromPhrase()`, and `toSeed()` in `test/src/crypto/bip39/bip39_mnemonic_test.dart`:

1. `generate()` produces the requested word count, for every value in `[12, 15, 18, 21, 24]`, and rejects any other value.
2. Every generated word belongs to the official wordlist, and a generated mnemonic round-trips through `fromPhrase()`.
3. `fromPhrase()` rejects an invalid word count, a word outside the official wordlist, and a phrase with a broken checksum.
4. `toSeed()` matches the official BIP-39 test vector: the 12-word all-zero-entropy phrase with passphrase `"TREZOR"` derives the exact expected 64-byte seed.
5. `toSeed()` rejects a non-ASCII passphrase, and is deterministic: the same mnemonic and passphrase always derive the same seed.

## Out of Scope for 0.0.5-dev

- Deriving an actual `SolanaKeypair` from the seed (SLIP-0010 hierarchical derivation, path `m/44'/501'/0'/0'`): `0.0.6-dev`.
- Full Unicode NFKD normalization, needed only for a non-ASCII passphrase: not yet planned.
- Consolidated exception types across keypair generation, import, signing, and derivation: deferred to the Phase 1 error-handling pass.

## Related

- [Phase 1: Keypair Generation](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/phase1/keypair-generation/README.md), the 32-byte seed this mnemonic's derived seed eventually feeds into.
- [CHANGELOG.md, 0.0.5-dev entry](https://github.com/nemorixgroup/solana-flutter-sdk/blob/main/CHANGELOG.md), the release notes this document expands on.

## Sources

- [BIP-39: Mnemonic code for generating deterministic keys](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
- [trezor/python-mnemonic](https://github.com/trezor/python-mnemonic), reference implementation and English wordlist source
- [package:cryptography on pub.dev](https://pub.dev/packages/cryptography)
- [solana-keygen mnemonic defaults (clap-v3-utils source)](https://github.com/anza-xyz/agave/blob/master/clap-v3-utils/src/keygen/mnemonic.rs)
