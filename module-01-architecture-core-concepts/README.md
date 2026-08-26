# Module 1: Architecture & Core Concepts

Foundational concepts behind Solana's design, referenced throughout every later module and every phase of `solana_flutter_sdk`. Sourced exclusively from [official Solana documentation](https://solana.com/docs).

## 1. The Account Model

Everything on Solana is an account: a wallet, a program, a token mint, a subscription record. There is no separate "smart contract" type; a program is simply an account flagged as executable.

Every account has exactly five fields:

| Field | Description |
|---|---|
| `lamports` | SOL balance, in the smallest unit (1 SOL = 1,000,000,000 lamports) |
| `data` | Variable-length byte array holding the account's state |
| `owner` | The program with exclusive authority to modify `data` or debit `lamports` |
| `executable` | Whether this account holds executable program code |
| `rent_epoch` | Tracks the last rent collection (largely vestigial now that most accounts pay a one-time rent-exemption) |

Official ownership rule: *"Only the account's owner program can modify its data or debit lamports. Any program can credit lamports to any writable account."*

## 2. Addresses and Program Derived Addresses (PDA)

**Regular addresses**: a 32-byte Ed25519 public key with a matching private key. This is what `SolanaKeypair.generate()` (shipped in `0.0.2-dev`) produces.

**Program Derived Addresses (PDA)**: a deterministic address with no associated private key. Derived from a `program_id` plus up to 16 seeds (each up to 32 bytes) via `find_program_address`, which appends a "bump seed" (tried from 255 down to 1) until it lands off the Ed25519 curve, guaranteeing no valid private key can exist for it. Only the program whose ID was used to derive a PDA can authorize actions on it, via `invoke_signed` in a CPI, without a cryptographic signature.

Compute cost: `create_program_address` costs 1,500 compute units; `find_program_address` costs that plus 1,500 per failed bump attempt. Maximum 16 PDA signers per CPI.

PDAs are the mechanism behind Associated Token Accounts (Module 2 / Phase 2) and, especially, Subscriptions & Allowances (Phases 8 and 9): `SubscriptionAuthority`, fixed and recurring delegations, and subscription plans are all PDA accounts controlled by the Subscriptions program, not by the user.

Source: [PDA Derivation](https://solana.com/docs/core/pda/pda-derivation)

## 3. Rent Exemption

Every account must maintain a minimum SOL balance, proportional to its size in bytes, to avoid removal by the rent collector. This is paid once at account creation, not recurrently.

```
minimum_rent_exempt_balance = (account_size + 128) x 3,480 lamports/byte-year x 2 years
```

| Parameter | Official limit |
|---|---|
| Max account data size | 10 MiB |
| Data growth per instruction | 10 KiB |
| Data growth per transaction | 20 MiB |
| Base account overhead | 64 bytes |

The `getMinimumBalanceForRentExemption` RPC method (Module 3 / Phase 3) computes this value directly.

## 4. Cross-Program Invocation (CPI)

A CPI occurs when a program invokes another program's instruction during its own execution, the core composability mechanism behind Solana: a single transaction can atomically combine, for example, a System Program instruction with a Token Program instruction.

- **`invoke`**: used when all required signers already authorized the full transaction.
- **`invoke_signed`**: used when a program must authorize on behalf of a PDA it controls, passing the original seeds as proof. `invoke` is functionally `invoke_signed` with an empty seeds array.

## 5. Fees

- **Base fee**: 5,000 lamports per signature, split 50% burned / 50% to the validator. Charged on every transaction, no exceptions.
- **Priority fee (optional)**: `ceil(compute_unit_price x compute_unit_limit / 1,000,000)` lamports, paid entirely to the validator, added via dedicated Compute Budget Program instructions (`SetComputeUnitLimit`, `SetComputeUnitPrice`) without changing the base fee.

| Compute unit parameter | Official value |
|---|---|
| Default per instruction | 200,000 CU |
| Default built-in instruction | 3,000 CU |
| Max per transaction | 1,400,000 CU |
| Price denominator | 1,000,000 micro-lamports per lamport |

## Summary

Accounts, PDAs, rent exemption, CPI, and fees are the shared vocabulary used across every remaining module and SDK phase. `SolanaKeypair.generate()` (`0.0.2-dev`) produced your first regular address; starting with Phase 2, PDA derivation becomes central to nearly everything the SDK builds.

## Sources

- [Solana Docs: Accounts](https://solana.com/docs/core/accounts)
- [Solana Docs: Fees on Solana](https://solana.com/docs/core/fees)
- [Solana Docs: Program Derived Address](https://solana.com/docs/core/pda)
- [Solana Docs: Cross Program Invocation](https://solana.com/docs/core/cpi)

## Related

- [solana_flutter_sdk](https://github.com/nemorixgroup/solana-flutter-sdk) - the SDK this knowledge base supports
- [docs-sdk](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/README.md../docs-sdk/phase-1) - SDK/Implementation decisions 
