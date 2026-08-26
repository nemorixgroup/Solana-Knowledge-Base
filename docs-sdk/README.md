# SDK Technical Decisions

## Why This Documentation Exists

`solana_flutter_sdk` is built entirely from scratch, verified exclusively against official Solana documentation and official repositories, with no dependency on third-party Solana Dart packages for its core. Every non-obvious implementation decision, library choice, encoding rule, and deviation from a "natural" approach is documented here, phase by phase, so contributors and auditors can trace *why* the code looks the way it does back to an official source.

## Structure

```
docs-sdk/
  README.md
  phase-1/
  phase-2/
  phase-3/
  phase-4/
  phase-5/
  phase-6/
  phase-7/
  phase-8/
  phase-9/
  phase-10/
```

## Phase 1 - Cryptographic Fundamentals (Ed25519)

| Feature | Description | Status |
|---------|--------------|--------|
| [Keypair generation](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/phase1/keypair-generation/README.md) | Ed25519 keypair generation | ✅ Done |
| Signing & verification | Message and transaction signing/verification | 🔄 Current |

## Phase 2 - Addresses & Encoding

| Feature | Description | Status |
|---------|--------------|--------|
| Base58 codec | Encode/decode of 32-byte public keys | ⏳ Pending |
| PDA derivation | `find_program_address` / `create_program_address`, off-curve check | ⏳ Pending |
| ATA derivation | Associated Token Account address derivation | ⏳ Pending |

## Phase 3 - RPC Connection Layer

| Feature | Description | Status |
|---------|--------------|--------|
| HTTP JSON-RPC client | Payments-relevant method subset | ⏳ Pending |
| WebSocket client | `accountSubscribe`, `signatureSubscribe`, `logsSubscribe` | ⏳ Pending |

## Phase 4 - Transaction Construction & Signing

| Feature | Description | Status |
|---------|--------------|--------|
| Legacy message format | | ⏳ Pending |
| Versioned (v0) message format | Including Address Lookup Table usage | ⏳ Pending |
| Compute Budget instructions | `SetComputeUnitLimit`, `SetComputeUnitPrice` | ⏳ Pending |

## Phase 5 - System Program & Core Token Operations

| Feature | Description | Status |
|---------|--------------|--------|
| System Program | `Transfer`, `CreateAccount` | ⏳ Pending |
| Token Program / Token-2022 | `Transfer`, `TransferChecked`, ATA creation | ⏳ Pending |

## Phase 6 - Solana Pay: Transfer Request

| Feature | Description | Status |
|---------|--------------|--------|
| URL scheme encode/decode | `recipient`, `amount`, `spl-token`, `reference`, `label`, `message`, `memo` | ⏳ Pending |
| QR code generation | | ⏳ Pending |

## Phase 7 - Solana Pay: Transaction Request

| Feature | Description | Status |
|---------|--------------|--------|
| GET/POST flow | Label/icon retrieval, transaction request/response | ⏳ Pending |
| Validation rules | feePayer/blockhash/signature validation per spec | ⏳ Pending |

## Phase 8 - Subscriptions & Allowances: Delegations

| Feature | Description | Status |
|---------|--------------|--------|
| Fixed delegation | `create_fixed_delegation`, `transfer_fixed` | ⏳ Pending |
| Recurring delegation | `create_recurring_delegation`, `transfer_recurring` | ⏳ Pending |

## Phase 9 - Subscriptions & Allowances: Plans

| Feature | Description | Status |
|---------|--------------|--------|
| Plan management | `create_plan`, `update_plan`, `delete_plan` | ⏳ Pending |
| Subscription lifecycle | `subscribe`, `cancel_subscription`, `resume_subscription` | ⏳ Pending |
| Event parsing | The 7 self-CPI events | ⏳ Pending |

## Phase 10 - Error Handling, Test Suite & v1.0.0 Close-out

| Feature | Description | Status |
|---------|--------------|--------|
| Error handling consolidation | | ⏳ Pending |
| Full test suite | | ⏳ Pending |

## Related

- [Solana Knowledge Base](https://github.com/nemorixgroup/solana-knowledge-base) - blockchain-level background for this SDK
- [solana_flutter_sdk](https://github.com/nemorixgroup/solana-flutter-sdk) - roadmap and installation
