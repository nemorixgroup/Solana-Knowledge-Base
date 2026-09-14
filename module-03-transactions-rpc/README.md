# Module 3: Transactions & RPC
 
Legacy and versioned (v0) transaction formats, Address Lookup Tables, the Compute Budget Program, and the payments-relevant subset of the JSON-RPC HTTP and WebSocket API. Sourced exclusively from [official Solana documentation](https://solana.com/docs) and official GitHub repositories (`anza-xyz`, `solana-program`).
 
## 1. Legacy Transaction Message Format
 
A Solana transaction is, at its core, a package that bundles everything needed to ask the network to do something: which accounts are affected, what instructions to run on them, and the signatures authorizing the operation. The "legacy" format is that package's original shape, the one that existed before versioned transactions (next section) were introduced. Every wallet transfer, token swap, or program call starts as one of these before being sent to a validator.
 
**Why it matters**  
Everything the SDK builds later (Phase 4: transaction construction, Solana Pay, Subscriptions) really comes down to assembling one of these packages correctly, signing it, and sending it. Its 1,232-byte size limit is also a real engineering constraint: no matter how many accounts or instructions a transaction needs to bundle, everything has to fit inside it, which is exactly the limitation versioned transactions and Address Lookup Tables exist to work around.
 
**How it is built**  
The message header is exactly 3 bytes: `num_required_signatures`, `num_readonly_signed_accounts`, `num_readonly_unsigned_accounts` (1 byte each). Account key ordering: Signer+Writable, then Signer+Read-only, then Non-signer+Writable, then Non-signer+Read-only. The rest of the message: `account_keys` (compact array of 32-byte public keys), `recent_blockhash` (32-byte hash), `instructions` (compact array of `CompiledInstruction`, each with `program_id_index`, `accounts`, and `data`). Each signer provides a 64-byte Ed25519 signature.
 
Size limit: 1,232 bytes, derived exactly as follows in source: `1280 (IPv6 minimum MTU) - 40 (IPv6 header) - 8 (fragment header) = 1232`.
 
Flagged discrepancy: the docs page states "max 64 accounts per transaction (128 when `increase_tx_account_lock_limit` activates, currently inactive)." Current source code, however, hardcodes `MAX_TX_ACCOUNT_LOCKS = 128` unconditionally. The feature gate that raised the limit to 128 dates to 2022 and is almost certainly active on mainnet by now, but no official page could be found confirming that specific gate's current activation status. Recommendation: use 128 (backed by current source) with this discrepancy noted.
 
## 2. Versioned Transactions (v0)
 
A versioned transaction is the evolution of the legacy format: basically the same kind of package, but with an extra byte at the start declaring "which format version you're reading," which lets Solana add new functionality without breaking what already existed. "v0" is the first, and so far the only, version beyond legacy in production.
 
**The problem it solves**  
v0 exists almost entirely to enable Address Lookup Tables (next section): a way to reference far more accounts than the 1,232-byte limit would allow with the legacy format. Importantly, on-chain programs need no changes at all to support v0; the upgrade is entirely client-side.
 
**How it works under the hood**  
A client distinguishes legacy from v0 via the `maxSupportedTransactionVersion` RPC parameter: omitting it means only legacy transactions are returned; setting it to `0` allows v0.
 
Version-byte mechanics: `MESSAGE_VERSION_PREFIX = 0x80`. If the first bit of the first byte is set, the remaining 7 bits indicate the message version (starting at 0). If not set, all bytes are read as the legacy format. Since a legacy message's first byte (`num_required_signatures`) always stays below `0x80`, there is never ambiguity.
 
v0 message structure: same header, plus `account_keys` (static keys only), `recent_blockhash`, `instructions`, and one new field: `address_table_lookups`.
 
Side note, outside the original scope: current source already defines a "v1" format (related to SIMD-0385) carrying `priority_fee`, `compute_unit_limit`, `loaded_accounts_data_size_limit`, and `heap_size` directly in the message, supporting transactions up to 4,096 bytes. Noted here only as a heads-up; its mainnet activation status was not verified.
 
## 3. Address Lookup Tables (ALT)
 
An Address Lookup Table is a special account, stored on-chain, holding a list of up to 256 addresses. Instead of writing each full address (32 bytes) inside a transaction, the message can simply say "use address number 5 from this table," taking just 1 byte per reference instead of 32.
 
**What they are for**  
This is what makes complex transactions practical, the kind that touch many tokens or several program accounts at once, common in Solana Pay and in Subscriptions & Allowances: without ALTs, that kind of transaction simply would not fit within the 1,232-byte limit.
 
**The specifics**  
Program: `AddressLookupTableProgram`, ID `AddressLookupTab1e1111111111111111111111111`. Account structure: maximum 256 addresses per table, 56-byte metadata (`deactivation_slot`, `last_extended_slot`, `last_extended_slot_start_index`, `authority`).
 
Lifecycle: `CreateLookupTable` (PDA derived from `[authority, recent_slot]`), `FreezeLookupTable` (permanently freezes it; cannot be done on an empty table), `ExtendLookupTable` (adds addresses), `DeactivateLookupTable` (makes it unusable, eligible for closure after a short period), `CloseLookupTable` (deallocates the account, only after the cooldown).
 
Deactivation cooldown: 513 slots (`MAX_ENTRIES = 512` plus 1), because a table cannot close until its `deactivation_slot` is no longer present in the `SlotHashes` sysvar (which retains the most recent 512 slots).
 
A v0 message references table entries via `MessageAddressTableLookup { account_key, writable_indexes, readonly_indexes }`. Total addressable accounts per v0 message: static plus dynamic keys must stay at or under 256 (1-byte indexes), distinct from the 128-account transaction-wide lock limit in Section 1, which is the real ceiling on how many accounts a single transaction can actually touch at runtime.
 
## 4. Compute Budget Program
 
Every Solana transaction consumes "compute units" (CU), a measure of how much processing work it asks the validator to do. The Compute Budget Program is a native program that lets a transaction explicitly declare how many CU it will need and how much extra it is willing to pay for priority.
 
**Why it is needed**  
Without this, the network would not know how many resources to reserve for a transaction ahead of time, and there would be no way to pay a priority fee to get a transaction processed sooner during congestion, something that matters for a payments SDK where confirmation speed counts.
 
**Exact specification**  
Program ID: `ComputeBudget111111111111111111111111111111`. Instructions (exact discriminator, confirmed by the crate's own unit test): `SetComputeUnitLimit(units: u32)` (discriminator 2, 5-byte data), `SetComputeUnitPrice(micro_lamports: u64)` (discriminator 3, 9-byte data).
 
Default/max values: 200,000 CU per instruction (default), 3,000 CU for builtin instructions, 1,400,000 CU max per transaction, price denominator of 1,000,000 micro-lamports per lamport. Priority fee formula: `ceil(compute_unit_price * compute_unit_limit / 1,000,000)` lamports, 100% to the validator. The fee is calculated on the requested CU limit, not actual usage.
 
## 5. JSON-RPC HTTP API (Payments-Relevant Subset)

RPC (Remote Procedure Call) is how any application, including the SDK, talks to a Solana node: sending it transactions, checking balances, or asking whether a transaction has confirmed. It is a standard JSON-RPC API over HTTP.
 
**Its role in a payment flow**  
A typical payment flow uses several of these methods in sequence: request a recent blockhash, send the signed transaction, then repeatedly check its status until it confirms or expires. Understanding commitment levels and the polling pattern well is what separates a reliable payments SDK from one that fails silently.
 
**Relevant methods**  
Commitment levels: processed (most recent processed block, can still roll back), confirmed (voted on by supermajority, more than 2/3 of active stake), finalized (recognized as finalized, default when omitted).
 
| Method | Returns |
|---|---|
| `getLatestBlockhash` | `{ blockhash, lastValidBlockHeight }` |
| `sendTransaction` | The signature, base58; returns immediately, without waiting for confirmation |
| `simulateTransaction` | `err`, `logs`, `unitsConsumed`, `returnData` |
| `getSignatureStatuses` | `{ slot, confirmations, err, confirmationStatus }` |
| `getTransaction` | Full transaction or `null` |
| `getAccountInfo` | `{ lamports, owner, data, executable, rentEpoch }` |
| `getBalance` | Balance in lamports |
 
Confirmation pattern: poll `getSignatureStatuses`, and to handle expiry, poll `getBlockHeight` until it exceeds `lastValidBlockHeight`. Blockhash validity: roughly 150 slots (about one minute); two official pages give slightly different figures (150 vs 151), so this is stated as approximate.
 
## 6. WebSocket Subscriptions
 
Beyond regular HTTP (ask and answer), Solana offers a WebSocket channel where a client subscribes to events and the node notifies it automatically whenever something changes, without having to keep asking.
 
**When to use it**  
For a payments SDK, this is more efficient than constant polling: instead of asking every second whether a payment has arrived, a client subscribes once and the node notifies it the moment it happens, saving RPC calls and reducing perceived latency.
 
**Available subscriptions**  
 
| Subscription | Purpose |
|---|---|
| `accountSubscribe` | Notifies on changes to an account's lamports or data |
| `signatureSubscribe` | Notifies on a transaction's status by its signature; auto-unsubscribes once the requested commitment is reached |
| `logsSubscribe` | Notifies on log messages matching a filter |
 
Parameters: `accountSubscribe` requires `pubkey`; `signatureSubscribe` requires `signature`; `logsSubscribe` requires a filter (`"all"`, `"allWithVotes"`, or `{ mentions: [pubkey] }`). All three accept `commitment` (default finalized).
 
## Summary
 
Legacy vs. v0 (with the 0x80 version byte and Address Lookup Tables as the key difference), the Compute Budget Program for priority fees, and the payments-relevant RPC HTTP/WebSocket subset form the foundation for SDK Phase 3 (RPC Connection Layer) and Phase 4 (Transaction Construction & Signing).
 
## Sources
 
- [Solana Docs: Transaction Structure](https://solana.com/docs/core/transactions/transaction-structure)
- [Solana Docs: Transactions](https://solana.com/docs/core/transactions)
- [Solana Docs: Transaction Versions](https://solana.com/docs/core/transactions/versions)
- [Solana Docs: Address Lookup Tables](https://solana.com/docs/advanced/lookup-tables)
- [Solana Docs: Compute Budget](https://solana.com/docs/core/fees/compute-budget)
- [Solana Docs: Transaction Retry](https://solana.com/docs/core/transactions/retry)
- [Solana Docs: Transaction Confirmation Guide](https://solana.com/developers/guides/advanced/confirmation)
- [Solana Docs: RPC Overview / Commitment](https://solana.com/docs/rpc)
- [Solana Docs: getLatestBlockhash](https://solana.com/docs/rpc/http/getlatestblockhash)
- [Solana Docs: sendTransaction](https://solana.com/docs/rpc/http/sendtransaction)
- [Solana Docs: simulateTransaction](https://solana.com/docs/rpc/http/simulatetransaction)
- [Solana Docs: getSignatureStatuses](https://solana.com/docs/rpc/http/getsignaturestatuses)
- [Solana Docs: getTransaction](https://solana.com/docs/rpc/http/gettransaction)
- [Solana Docs: getAccountInfo](https://solana.com/docs/rpc/http/getaccountinfo)
- [Solana Docs: getBalance](https://solana.com/docs/rpc/http/getbalance)
- [Solana Docs: accountSubscribe](https://solana.com/docs/rpc/websocket/accountsubscribe)
- [Solana Docs: signatureSubscribe](https://solana.com/docs/rpc/websocket/signaturesubscribe)
- [Solana Docs: logsSubscribe](https://solana.com/docs/rpc/websocket/logssubscribe)
- [anza-xyz/solana-sdk - message/src/lib.rs](https://github.com/anza-xyz/solana-sdk/blob/master/message/src/lib.rs)
- [anza-xyz/solana-sdk - message/src/compiled_instruction.rs](https://github.com/anza-xyz/solana-sdk/blob/master/message/src/compiled_instruction.rs)
- [anza-xyz/solana-sdk - packet/src/lib.rs](https://github.com/anza-xyz/solana-sdk/blob/master/packet/src/lib.rs)
- [anza-xyz/solana-sdk - transaction/src/sanitized.rs](https://github.com/anza-xyz/solana-sdk/blob/master/transaction/src/sanitized.rs)
- [anza-xyz/solana-sdk - message/src/versions/mod.rs](https://github.com/anza-xyz/solana-sdk/blob/master/message/src/versions/mod.rs)
- [anza-xyz/solana-sdk - message/src/versions/v0/mod.rs](https://github.com/anza-xyz/solana-sdk/blob/master/message/src/versions/v0/mod.rs)
- [anza-xyz/solana-sdk - sdk-ids/src/lib.rs](https://github.com/anza-xyz/solana-sdk/blob/master/sdk-ids/src/lib.rs)
- [anza-xyz/solana-sdk - compute-budget-interface/src/lib.rs](https://github.com/anza-xyz/solana-sdk/blob/master/compute-budget-interface/src/lib.rs)
- [anza-xyz/solana-sdk - slot-hashes/src/lib.rs](https://github.com/anza-xyz/solana-sdk/blob/master/slot-hashes/src/lib.rs)
- [solana-program/address-lookup-table - program/src/lib.rs](https://github.com/solana-program/address-lookup-table/blob/main/program/src/lib.rs)
- [solana-program/address-lookup-table - program/src/state.rs](https://github.com/solana-program/address-lookup-table/blob/main/program/src/state.rs)
- [solana-program/address-lookup-table - program/src/instruction.rs](https://github.com/solana-program/address-lookup-table/blob/main/program/src/instruction.rs)
- [solana-program/address-lookup-table - program/src/processor.rs](https://github.com/solana-program/address-lookup-table/blob/main/program/src/processor.rs)
## Related
 
- [solana_flutter_sdk](https://github.com/nemorixgroup/solana-flutter-sdk) - the SDK this knowledge base supports
- [docs-sdk](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/docs-sdk/README.md) - SDK implementation decisions
- [Module 1: Architecture & Core Concepts](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/module-01-architecture-core-concepts/README.md)
- [Module 2: Cryptography & Addresses](https://github.com/nemorixgroup/Solana-Knowledge-Base/blob/main/module-02-cryptography-addresses/README.md)
