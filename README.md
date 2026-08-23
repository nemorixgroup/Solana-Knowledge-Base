# Solana Knowledge Base

> A comprehensive, in-depth technical guide to the Solana blockchain - covering its account model, transaction structure, RPC interface, token programs, and payment primitives (Solana Pay, Subscriptions & Allowances), sourced exclusively from official Solana documentation and repositories.

## About This Repository

This repository is the companion knowledge base for [solana_flutter_sdk](https://github.com/nemorixgroup/solana-flutter-sdk), a native Flutter/Dart SDK for Solana payments. It is written for three audiences: contributors to the SDK, Flutter developers integrating Solana payments into their apps, and anyone studying Solana's architecture from official sources.

Every module is grounded exclusively in official Solana documentation [Solana Docs](https://solana.com/docs) and official GitHub repositories (solana-foundation, solana-program, anza-xyz). No third-party tutorials, blogs, or unofficial explainers are used as technical sources.

## Why Solana?

Solana's account model separates executable code (programs) from state (accounts), and every account is owned by exactly one program that alone can modify its data or debit its lamports. This, combined with Proof of History as a verifiable clock, is the architectural basis for the parallel transaction processing Solana is known for. For a payments SDK specifically, two native primitives make Solana distinctive: Solana Pay, a URL-based payment request standard usable without a backend, and Subscriptions & Allowances, a native on-chain recurring-payment primitive (live on mainnet since June 2026) that removes the need for custom escrow or delegation contracts to support recurring transfers.

## Important Distinctions

- **Solana vs. SOL**: Solana is the network/protocol; SOL is its native token.
- **Solana Foundation vs. Anza**: the Solana Foundation (solana.org) is the non-profit steward of the ecosystem (grants, governance support); Anza (anza-xyz) is the organization maintaining the validator client software, successor to Solana Labs' client maintenance role.
- **solana-foundation vs. solana-program (GitHub)**: `solana-foundation` hosts ecosystem/community projects (e.g. Explorer, Solana Pay); `solana-program` hosts the official on-chain programs themselves (e.g. Subscriptions & Allowances, Token-2022).

## Repository Structure

```
solana-knowledge-base/
  README.md
  module-01-architecture-core-concepts/
    README.md
  module-02-cryptography-addresses/
    README.md
  module-03-transactions-rpc/
    README.md
  module-04-programs-system-token/
    README.md
  module-05-solana-pay/
    README.md
  module-06-subscriptions-allowances/
    README.md
  module-07-development-ecosystem/
    README.md
```

## Modules

| # | Module | Topics covered |
|---|--------|-----------------|
| 01 | Architecture & Core Concepts - 🔄 Next | Account model, Program Derived Addresses (PDA), Cross-Program Invocation (CPI), rent-exemption, fees, compute budget |
| 02 | Cryptography & Addresses - ⏳ Pending | Ed25519 keypairs, Base58 encoding, address derivation |
| 03 | Transactions & RPC - ⏳ Pending | Legacy and versioned (v0) transactions, Address Lookup Tables, JSON-RPC HTTP and WebSocket API |
| 04 | Programs: System, Token & Token-2022 - ⏳ Pending | System Program, SPL Token Program, Token Extensions (Token-2022), Associated Token Account |
| 05 | Solana Pay - ⏳ Pending | Transfer Request and Transaction Request specification (v1.1) |
| 06 | Subscriptions & Allowances - ⏳ Pending | Fixed/recurring delegations, subscription plans, native recurring-payment primitive |
| 07 | Development Ecosystem - ⏳ Pending | Official SDKs, GitHub organizations, grants and funding |

## SDK Technical Decisions

This repository also documents the implementation decisions behind **solana_flutter_sdk** - a native Flutter/Dart SDK for Solana blockchain. 
Every implementation decision is grounded in the official sources documented in this repository - no third-party references,
no unverified code.

Current status: **Phase 1 in progress**

| Phase | Status |
|-------|--------|
| Phase 1 - Cryptographic Fundamentals | 🔄 In progress | 
| Phase 2 - Addresses & Encoding | ⏳ Pending |
| Phase 3 - RPC Connection Layer | ⏳ Pending |
| Phase 4 - Transaction Construction & Signing | ⏳ Pending |
| Phase 5 - System Program & Core Token Operations | ⏳ Pending |
| Phase 6 - Solana Pay: Transfer Request | ⏳ Pending |
| Phase 7 - Solana Pay: Transaction Request | ⏳ Pending |
| Phase 8 - Subscriptions & Allowances: Delegations | ⏳ Pending |
| Phase 9 - Subscriptions & Allowances: Plans | ⏳ Pending |
| Phase 10 - Closing Audit | ⏳ Pending |

See [docs-sdk/](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/README.md) for the full documentation by phase.

## Official Resources

### Core

| Resource | URL |
|----------|-----|
| Official documentation | https://solana.com/docs |
| Solana Foundation | https://solana.org |
| Terminology reference | https://solana.com/docs/references/terminology |

### Developer Tools

| Resource | URL |
|----------|-----|
| Solana Explorer | https://explorer.solana.com |
| Explorer source (official) | https://github.com/solana-foundation/explorer |

### SDKs (Official, verified so far)

| Language | Package |
|----------|---------|
| TypeScript | `@solana/kit` (referenced by `@solana/pay` and `@solana/subscriptions`) |
| Rust | `subscriptions` (generated client), `solana-sdk` |

> Dart/Flutter has no official SDK - that's the gap `solana_flutter_sdk` fills.

### GitHub Organizations

| Organization | URL |
|---------------|-----|
| Solana Foundation | https://github.com/solana-foundation |
| Official on-chain programs | https://github.com/solana-program |
| Validator client maintainers (successor to solana-labs) | https://github.com/anza-xyz |
| Anchor framework | https://github.com/coral-xyz |

### Community & News

| Resource | URL |
|----------|-----|
| Official news and ecosystem updates | https://solana.com/news |

## Key Technical Facts (Quick Reference)

| Property | Value |
|----------|-------|
| Native token | SOL |
| Smallest unit | 1 lamport = 0.000000001 SOL (1 SOL = 1,000,000,000 lamports) |
| Signature scheme | Ed25519 |
| Address format | Base58-encoded 32-byte public key, or Program Derived Address (off-curve, no private key) |
| Base transaction fee | 5,000 lamports per signature (50% burned, 50% to validator) |
| Max transaction size | 1,232 bytes |
| Max compute units per transaction | 1,400,000 CU |
| Consensus building block | Proof of History (PoH): a verifiable sequence proving elapsed time between events |

## About This Guide

This guide is maintained by [Nemorix Group](https://github.com/nemorixgroup) as the technical foundation for `solana_flutter_sdk`. All technical claims are sourced from official Solana documentation and repositories; sources are cited per module. Corrections and clarifications are welcome via issues or pull requests.

---

Last updated: August 2026  
Maintained by [Nemorix Group](https://github.com/nemorixgroup)
