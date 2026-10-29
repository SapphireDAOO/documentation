# Payment Processor Subgraph

### Table of Contents

1. Overview
2. Data Sources
3. Entity Reference
4. Invoice State Machines
5. Example Queries

---

### 1. Overview

This subgraph indexes **seven** Sapphire DAO smart contracts deployed on the Base Sepolia testnet:

- **SimplePaymentProcessor**: A native-token (ETH) escrow contract. A seller creates an invoice, the buyer pays in ETH, and the seller accepts (releasing funds after a hold period) or rejects (triggering a refund). The escrow hold period is fixed on the contract at deployment rather than chosen per invoice, so there is no per-invoice hold-period update event to index. The contract's own burn-on-exhaustion path (`PaymentBurned`, after every automated withdrawal retry fails) is not currently indexed either; neither is a `BURNED` or `LOCKED` state tracked here.
- **AdvancedPaymentProcessor**: The subgraph's name for the deployed `IntermediatedPaymentProcessor` contract (kept from an earlier rename so entity names stay stable). A multi-token escrow contract with dispute resolution, partial refunds, meta-invoices (batch invoices), and USD-price-pegged payments via an `OracleManager` contract wrapping Chainlink price feeds. A dispute that lapses without a decision transitions to a dedicated `DISPUTE_EXPIRED` state rather than being silently releasable. `release` no longer waits for a stored release timestamp; the recorded `releaseAt` is kept for reference only.
- **PaymentProcessorStorage**: Shared configuration contract, now holding only what's actually mutable after deployment: `owner`, `feeReceiver`, `intermediatedPlatformsOperator`, the WETH address (`weth`), `feeSigner`, and `emergencyPauser`; plus the authorized-address allowlist and the pause/emergency-pause system both processors check. The fee rate and gas threshold are compile-time constants on the contracts now, not stored configuration, so this entity no longer tracks either.
- **Notes**: An encrypted note store attached to invoices, plus each user's published messaging public key and the note-encryption version it was registered under. Notes are stored off-chain but their on-chain references and per-user "opened" state are indexed here.
- **MultiSig**: Multisig governance contract gating privileged admin calls into the processors and storage contract. Its signer set, threshold (which now auto-syncs to a majority of the signer set on every add or remove), proposed/approved/executed/canceled transactions, and approvals are indexed here.
- **OracleManager**: Emits `PriceFeedSet` when a token gains a Chainlink price feed; the handler registers/refreshes the token's `PaymentToken` metadata.
- **Sweeper**: Emits `Swept` when a stealth fee receiver's accrued tokens are moved to a destination. Indexed alongside both processors' fee-crediting logic to track, per fee receiver and token, how much has accrued versus been swept out.

Platform fees paid to a per-invoice stealth fee receiver (see [Fee Receiver Privacy](fee-receiver-privacy.md)) are tracked across both processors and the Sweeper through a shared `FeeReceiver`/`FeeReceiverTokenBalance` pair: credited at release or dispute settlement, debited on each `Swept` event. A native fee is credited in WETH, since both processors wrap it before paying it out.

The subgraph also maintains a set of **dashboard-metrics** timeseries/aggregation entities (volume, escrow balance, fees, invoice activity, user growth, gas spend) derived from the same events; see [Entity Reference](#3-entity-reference).

---

### 2. Data Sources

Addresses and start blocks below reflect `config/base-sepolia.json` as applied to `subgraph.template.yaml`; confirm against those files before relying on them, since a redeploy can change both.

#### SimplePaymentProcessor

- **Address:** `0xbCd3f9AB4e0b09044053102A353C0B710106e08a`
- **Start block:** `51833746`
- **Handler file:** `src/simple-payment-processor.ts`

| Event | Handler | Description |
| :--: | :--: | :--: |
| `InvoiceCreated(invoiceId, seller, invoiceNonce, price, expiresAt, feeRate)` | `handleInvoiceCreated` | Creates the `SimplePaymentProcessor` entity, tracks the seller as a `CREATOR` user, writes an `InvoiceEvent` |
| `InvoicePaid(invoiceId, buyer, amountPaid, sellerActionDeadline)` | `handleInvoicePaid` | Records buyer, amount, and the seller's decision deadline; tracks the buyer as a `PAYER` user; pushes `PaymentVolume` and a positive `EscrowBalance` delta (native ETH) |
| `InvoiceAccepted(invoiceId, feeReceiver, releaseAt)` | `handleInvoiceAccepted` | Sets `state = ACCEPTED`, records the invoice's `feeReceiver` and `releaseAt` |
| `InvoiceCanceled(invoiceId)` | `handleInvoiceCanceled` | Marks the invoice `CANCELED` |
| `InvoiceRejected(invoiceId, amount)` | `handleInvoiceRejected` | Marks the invoice `REJECTED`; pushes a negative `EscrowBalance` delta if the invoice was escrowed |
| `InvoiceRefunded(invoiceId, amount)` | `handleInvoiceRefunded` | Marks the invoice `REFUNDED`; pushes a negative `EscrowBalance` delta if the invoice was escrowed |
| `InvoiceReleased(invoiceId, sellerAmount, fee)` | `handleInvoiceReleased` | Marks the invoice `RELEASED`, records `fee`; pushes a negative `EscrowBalance` delta for the full escrowed amount and a `FeePaid` point; credits the fee to the resolved `FeeReceiver` (the invoice's own, falling back to the global one) unless `TransferFailed` already fired for that payout in the same transaction |
| `TransferFailed(invoiceId, recipient, amount)` | `handleTransferFailed` | Records a failed transfer event; if `recipient` is the resolved fee receiver, remembers the transaction so the release handler above skips crediting it |
| `WithdrawalRetried(invoiceId, recipient, amount, attempt)` | `handleWithdrawalRetried` | Records an automated-retry event |

Every handler above also writes an `InvoiceEvent` row and increments the `InvoiceActivity` (`SIMPLE`) timeseries. None of them touch `GasPaid`; that singleton is only updated from `AdvancedPaymentProcessor` handlers.

#### AdvancedPaymentProcessor

- **Address:** `0x72AD25c3AED936120842B4b2dBcD28744bb412Ed`
- **Start block:** `51833751`
- **Handler file:** `src/advanced-payment-processor.ts`

| Event | Handler | Description |
| :--: | :--: | :--: |
| `InvoiceCreated(invoiceId, seller, metaInvoiceId, orderId, invoiceNonce, price, expiresAt, holdPeriod, feeRate)` | `handleAdvancedPaymentProcessorCreated` | Creates the `AdvancedPaymentProcessor` entity, records `orderId`, links `metaInvoice` if `metaInvoiceId` is non-zero, tracks the seller as `CREATOR` |
| `PaymentTokensRegistered(invoiceId, paymentTokens)` | `handlePaymentTokensRegistered` | Sets `expectedPaymentTokens` to the registered tokens. Writes no `InvoiceEvent` and does not touch `GasPaid` |
| `InvoicePaid(invoiceId, payer, paymentToken, escrowAddress, amount, releaseAt, feeReceiver)` | `handleInvoicePaid` | Records payer, token, escrow address, and `feeReceiver`; tracks the payer as `PAYER`; pushes `PaymentVolume` and a positive `EscrowBalance` delta. Does not touch `GasPaid` |
| `InvoiceCanceled(invoiceId)` | `handleInvoiceCanceled` | Marks the invoice `CANCELED`. Does not adjust the parent `MetaInvoice`'s indexed `price`, even though the contract reduces its own on-chain total |
| `DisputeCreated(invoiceId)` | `handleDisputeCreated` | Marks the invoice `DISPUTED` |
| `DisputeDismissed(invoiceId)` | `handleDisputeDismissed` | Marks the invoice `DISPUTE_DISMISSED` |
| `DisputeResolved(invoiceId)` | `handleDisputeResolved` | Marks the invoice `DISPUTE_RESOLVED` |
| `DisputeExpired(invoiceId)` | `handleDisputeExpired` | Marks the invoice `DISPUTE_EXPIRED` |
| `DisputeSettled(invoiceId, sellerAmount, buyerAmount, fee)` | `handleDisputeSettled` | Marks the invoice `DISPUTE_SETTLED`, zeroes `balance`, records the split and `fee`; pushes a negative `EscrowBalance` delta for the prior balance and a `FeePaid` point; credits the fee to the resolved `FeeReceiver` unless `TransferFailed` already fired in this transaction |
| `EscrowCreated(invoiceId, escrow)` | `handleEscrowCreated` | Records the escrow address on the invoice |
| `LockedPaymentRecovered(invoiceId, recipient, amount)` | `handleLockedPaymentRecovered` | Declared in the interface but never actually emitted by this contract (no locked-fund path here) |
| `MetaInvoiceCreated(metaInvoiceId, totalPrice)` | `handleMetaInvoiceCreated` | Creates a `MetaInvoice` entity; tracks the transaction sender (`tx.from`) as `PAYER` |
| `OracleUpdated(previousOracle, newOracle)` | `handleOracleUpdated` | Recorded as a processor event (not tied to a specific invoice); bumps `GasPaid` |
| `PaymentReleased(invoiceId, receiver, currency, sellerAmount, fee)` | `handlePaymentReleased` | Marks the invoice `RELEASED`, zeroes `balance`; pushes a negative `EscrowBalance` delta and a `FeePaid` point; credits the fee the same way as `DisputeSettled` above |
| `Refunded(invoiceId, amount)` | `handleRefunded` | Reduces `balance` by `amount` and accumulates `amountRefunded`; marks the invoice `REFUNDED` only once `balance` reaches zero; pushes a negative `EscrowBalance` delta for the refunded amount |
| `TransferFailed(invoiceId, recipient, amount)` | `handleTransferFailed` | This contract only emits `TransferFailed` for a failed fee payout, so the handler always records the transaction as the invoice's `feeTransferFailedTx` |
| `UpdateReleaseTime(invoiceId, newHoldPeriod)` | `handleUpdateReleaseTime` | Sets `releaseAt` to the handling block's own timestamp plus `newHoldPeriod`, not the contract's computed value |

Every handler above except `InvoicePaid` and `PaymentTokensRegistered` bumps the `GasPaid` singleton; every handler that touches an invoice (all but `OracleUpdated` and `PaymentTokensRegistered`) also writes an `InvoiceEvent` and increments `InvoiceActivity` (`ADVANCED`).

#### PaymentProcessorStorage

- **Address:** `0x58422C5A8959502a2304b2AfF97dAebF30262CA6`
- **Start block:** `51833751`
- **Handler file:** `src/payment-processor-storage.ts`

| Event | Handler | Description |
| :--: | :--: | :--: |
| `AuthorizationUpdated(account, authorized)` | `handleAuthorizationUpdated` | Upserts an `AuthorizedAddress` entity |
| `ConfigurationInitialized(config)` | `handleConfigurationInitialized` | Seeds the `StorageConfiguration` singleton's `owner`, `feeReceiver`, `intermediatedPlatformsOperator`, and `weth` from the deployment-time config struct |
| `IntermediatedPlatformsOperatorUpdated(intermediatedPlatformsOperator)` | `handleIntermediatedPlatformsOperatorUpdated` | Updates the singleton's `intermediatedPlatformsOperator` |
| `FeeSignerUpdated(feeSigner)` | `handleFeeSignerUpdated` | Updates the singleton's `feeSigner` |
| `EmergencyPauserUpdated(emergencyPauser)` | `handleEmergencyPauserUpdated` | Updates the singleton's `emergencyPauser` |
| `OwnershipTransferred(previousOwner, newOwner)` | `handleOwnershipTransferred` | Updates the singleton's `owner` |
| `Paused(account)` | `handlePaused` | Sets `PauseState.ownerPaused = true`; appends a `PauseAction` row |
| `Unpaused(account)` | `handleUnpaused` | Clears `ownerPaused` and every emergency-pause field on `PauseState`; appends a `PauseAction` row |
| `EmergencyPaused(account, expiry)` | `handleEmergencyPaused` | Records the pending emergency pause's `expiry`, pauser, and start time on `PauseState`; appends a `PauseAction` row |
| `EmergencyPauseApproved(account)` | `handleEmergencyPauseApproved` | Promotes a pending emergency pause into a standing owner pause (`ownerPaused = true`, emergency fields cleared); appends a `PauseAction` row |

`ConfigurationInitialized`, `IntermediatedPlatformsOperatorUpdated`, `FeeSignerUpdated`, `EmergencyPauserUpdated`, and `OwnershipTransferred` all stamp `StorageConfiguration.updatedAt`/`updatedAtBlock`. Every pause-related event appends to the `PauseAction` log and updates `PauseState.lastAction`/`lastActionBy`/`lastActionAt`/`lastActionTx`.

#### Notes

- **Address:** `0x934fa7109ff129Cb70997B6be330401fa5946671`
- **Start block:** `51833751`
- **Handler file:** `src/notes.ts`

| Event | Handler | Description |
| :--: | :--: | :--: |
| `NoteCreated(invoiceId, noteId, author, share, encryptedContent)` | `handleNoteCreated` | Creates a `Note` entity, id `{invoiceId}-{noteId}` |
| `NoteStateChanged(invoiceId, noteId, user, opened)` | `handleNoteStateChanged` | Upserts a `NoteOpenState` entity, id `{invoiceId}-{noteId}-{userAddress}` |
| `PublicKeySet(account, publicKey, version)` | `handlePublicKeySet` | Upserts the `User` entity for `account`, setting `publicKey` and `publicKeyVersion`. Does not count as a new-user or active-user action; that is only recorded against an account's first invoice activity |

#### MultiSig

- **Address:** `0xf2EDEf709357e6587DC9600B58A34DA3d54369d1`
- **Start block:** `51833746`
- **Handler file:** `src/multi-sig.ts`

| Event | Handler | Description |
| :--: | :--: | :--: |
| `SignerAdded(signer)` | `handleSignerAdded` | Activates the `MultiSigSigner` (creates it if new), refreshes `MultiSigWallet` from live contract state |
| `SignerRemoved(signer)` | `handleSignerRemoved` | Deactivates the `MultiSigSigner` |
| `ThresholdUpdated(oldThreshold, newThreshold)` | `handleThresholdUpdated` | Updates `MultiSigWallet.threshold`. Fires both for a manual `updateThreshold` call and for the contract's own auto-resync after `addSigner`/`removeSigner` |
| `TransactionProposed(txHash, target, value, data, nonce, proposer)` | `handleTransactionProposed` | Creates a `MultiSigTransaction` (status `PROPOSED`), refreshed from a live `getTransaction` call |
| `ApprovalAdded(txHash, approver, approvalCount)` | `handleApprovalAdded` | Upserts a `MultiSigApproval`, updates `approvalCount` |
| `TransactionApproved(txHash)` | `handleTransactionApproved` | Sets status `APPROVED` |
| `TransactionExecuted(txHash, executor)` | `handleTransactionExecuted` | Sets status `EXECUTED`, records `executor`/`executedAt` |
| `TransactionCanceled(txHash)` | `handleTransactionCanceled` | Sets status `CANCELED` |

#### OracleManager

- **Address:** `0xE201Ff9D0A3d525F98161407e079f680d89cF2BC`
- **Start block:** `51833751`
- **Handler file:** `src/oracle-manager.ts`

| Event | Handler | Description |
| :--: | :--: | :--: |
| `PriceFeedSet(token, aggregator, heartbeat)` | `handlePriceFeedSet` | Registers a `PaymentToken` if it doesn't already exist (no-op otherwise) |

#### Sweeper

- **Address:** `0xCa65Bbd11988Efe078fe98D127cab5145D5050BB`
- **Start block:** `51833751`
- **Handler file:** `src/sweeper.ts`

| Event | Handler | Description |
| :--: | :--: | :--: |
| `Swept(from, token, destination, amount)` | `handleSwept` | Debits the `FeeReceiver`/`FeeReceiverTokenBalance` for `from` in `token` by `amount` and increments that receiver's `sweepCount`; creates a `FeeSweep` row |

---

### 3. Entity Reference

#### `InvoiceEvent`

An immutable append-only log row, one per processor event that is tied to an invoice. The `id` is `{txHash}-{logIndex}`. Not written for `PaymentTokensRegistered` on the advanced processor, and not written at all by the `OracleManager` or `Sweeper` data sources.

| Field | Type | Description |
| :--: | :--: | :--: |
| `id` | `ID!` | `{txHash}-{logIndex}` |
| `eventType` | `PaymentProcessorEventType!` | e.g. `INVOICE_CREATED`, `INVOICE_PAID`, `DISPUTE_EXPIRED`, etc. |
| `txHash` | `Bytes!` | Transaction hash |
| `timestamp` | `BigInt!` | Block timestamp |
| `simpleInvoice` | `SimplePaymentProcessor` | Set for simple-processor events |
| `advancedInvoice` | `AdvancedPaymentProcessor` | Set for advanced-processor events |

#### `SimplePaymentProcessor`

One invoice on the SimplePaymentProcessor contract. The `id` is the on-chain `invoiceId` (numeric string).

| Field | Type | Description |
| :--: | :--: | :--: |
| `id` | `ID!` | On-chain invoice ID |
| `invoiceNonce` | `BigInt!` | Internal invoice nonce |
| `state` | `SimplePaymentProcessorState!` | See [State Machine](#41-simple-payment-processor-states) |
| `seller` | `User!` | Invoice creator |
| `buyer` | `User` | Set once paid |
| `price` | `BigInt!` | Invoice price in wei |
| `amountPaid` | `BigInt` | Amount paid by the buyer |
| `expiresAt` | `BigInt` | Payment-validity deadline |
| `sellerActionDeadline` | `BigInt` | Deadline for the seller to accept or reject |
| `releaseAt` | `BigInt` | Timestamp set once accepted |
| `fee` | `BigInt` | Protocol fee deducted on release |
| `feeReceiver` | `Bytes` | Fee receiver recorded at acceptance |
| `feeTransferFailedTx` | `Bytes` | Transaction hash in which the best-effort fee payout failed; that fee was not credited to `feeReceiver` |
| `contract` | `Bytes!` | SimplePaymentProcessor address |
| `events` | `[InvoiceEvent!]!` | Derived from `InvoiceEvent.simpleInvoice` |
| `lastActionTime` | `BigInt` | Timestamp of the most recent state change |
| `buyerNote` / `sellerNote` | `String` | Optional notes |

#### `AdvancedPaymentProcessor`

One invoice on the AdvancedPaymentProcessor (`IntermediatedPaymentProcessor`) contract; supports multi-token payments and dispute resolution. The `id` is the on-chain `invoiceId`.

| Field | Type | Description |
| :--: | :--: | :--: |
| `id` | `ID!` | On-chain invoice ID |
| `orderId` | `String!` | Caller-supplied order identifier the invoice ID was derived from |
| `invoiceNonce` | `BigInt!` | Internal invoice nonce |
| `state` | `AdvancedPaymentProcessorState!` | See [State Machine](#42-intermediated-payment-processor-states); no `LOCKED` state exists here |
| `seller` | `User!` | Invoice seller |
| `buyer` | `User` | Set once paid |
| `escrow` | `Bytes` | Escrow contract address |
| `balance` | `BigInt` | Current escrow balance |
| `paymentToken` | `PaymentToken` | Payment token the buyer paid with (native ETH if unset/zero address) |
| `expectedPaymentTokens` | `[PaymentToken!]!` | Tokens the invoice may be paid in, registered at creation; the buyer's chosen token becomes `paymentToken` |
| `releaseAt` | `BigInt` | Release timestamp recorded at payment; not enforced by `release` |
| `amountReleased` / `amountRefunded` | `BigInt` | Cumulative released/refunded amounts |
| `sellerAmountReceivedAfterDispute` / `buyerAmountReceivedAfterDispute` | `BigInt` | Dispute-settlement split |
| `amountPaid` | `BigInt` | Total paid by buyer |
| `contract` | `Bytes!` | AdvancedPaymentProcessor address |
| `events` | `[InvoiceEvent!]!` | Derived from `InvoiceEvent.advancedInvoice` |
| `fee` | `BigInt` | Protocol fee |
| `feeReceiver` | `Bytes` | Fee receiver recorded at payment |
| `feeTransferFailedTx` | `Bytes` | Transaction hash in which the best-effort fee payout failed |
| `lastActionTime` | `BigInt` | Timestamp of the most recent state change |
| `metaInvoice` | `MetaInvoice` | Set if part of a meta-invoice |
| `buyerNote` / `sellerNote` | `String` | Optional notes |

#### `MetaInvoice`

Groups one or more `AdvancedPaymentProcessor` invoices into a single payable unit. The `id` is the `metaInvoiceId`.

| Field | Type | Description |
| :--: | :--: | :--: |
| `id` | `ID!` | On-chain meta-invoice ID |
| `invoiceNonce` | `BigInt!` | Same value as `id` |
| `buyer` | `User!` | Address that initiated the meta-invoice payment |
| `price` | `BigInt!` | Combined price across sub-invoices at creation; not reduced when a sub-invoice is later canceled |
| `contract` | `Bytes!` | AdvancedPaymentProcessor address |
| `invoices` | `[AdvancedPaymentProcessor!]!` | Derived: child invoices linked via `metaInvoice` |

#### `User`

A unique wallet address seen by either processor or by `Notes`. The `id` is the hex address.

| Field | Type | Description |
| :--: | :--: | :--: |
| `id` | `ID!` | Wallet address (hex string) |
| `publicKey` | `Bytes` | Messaging public key published via `Notes.setPublicKey` |
| `publicKeyVersion` | `Int` | Note-encryption version the key was registered under |
| `countedAsNewUser` | `Boolean!` | Flips true the first time this address takes an invoice action; decouples "entity exists" (which `setPublicKey` alone can trigger) from the `NewUser` count |
| `lastActiveDay` | `BigInt` | UTC day index of last observed invoice activity; used to emit at most one `ActiveUser` point per day |
| `ownedSimpleInvoices` | `[SimplePaymentProcessor!]!` | Seller-side simple invoices |
| `paidSimpleInvoices` | `[SimplePaymentProcessor!]!` | Buyer-side simple invoices |
| `issuedAdvancedInvoices` | `[AdvancedPaymentProcessor!]!` | Seller-side advanced invoices |
| `receivedAdvancedInvoices` | `[AdvancedPaymentProcessor!]!` | Buyer-side advanced invoices |
| `metaInvoices` | `[MetaInvoice!]!` | Meta-invoices paid by this user |

#### `PaymentToken`

Metadata for a token registered either via a payment or via `OracleManager.PriceFeedSet`. The `id` is the token's contract address (zero address = native ETH).

| Field | Type | Description |
| :--: | :--: | :--: |
| `id` | `ID!` | Token contract address, or zero address for ETH |
| `name` | `String` | Token name |
| `decimal` | `Int` | Token decimals |

#### `Note` / `NoteOpenState`

`Note` (`id`: `{invoiceId}-{noteId}`): `invoiceId`, `noteId` (`BigInt!`), `author` (`Bytes!`), `share` (`Boolean!`), `encryptedContent` (`Bytes!`), `createdAtBlock`/`createdAtTx`.

`NoteOpenState` (`id`: `{invoiceId}-{noteId}-{userAddress}`): `invoiceId`, `noteId`, `user` (`Bytes!`), `opened` (`Boolean!`), `updatedAtBlock`/`updatedAtTx`.

#### `MultiSigWallet` / `MultiSigSigner` / `MultiSigTransaction` / `MultiSigApproval`

See [core-contracts/multisig.sol.md](core-contracts/multisig.sol.md) for the full field reference; summary:

- `MultiSigWallet` (`id`: contract address): `threshold`, `signerCount`, `transactionCount`, derived `signers`/`transactions`.
- `MultiSigSigner` (`id`: `{wallet}-{signer}`): `wallet`, `address`, `active`, `addedAt`, `removedAt`.
- `MultiSigTransaction` (`id`: `txHash`): `wallet`, `target`, `value`, `data`, `nonce`, `proposer`, `status` (`MultisigStatus`: `PROPOSED`/`APPROVED`/`CANCELED`/`EXECUTED`), `approvalCount`, `proposedAt`, `executedAt`, `executor`.
- `MultiSigApproval` (`id`: `{txHash}-{approver}`): `transaction`, `signer`, `approver`, `approvalCount`, `approvedAt`.

#### `StorageConfiguration` / `AuthorizedAddress`

`StorageConfiguration` (singleton, `id: "global"`): mirrors `PaymentProcessorStorage`'s current mutable config: `owner`, `feeReceiver`, `intermediatedPlatformsOperator`, `weth` (the wrapped native token both processors pay platform fees in), `feeSigner`, `emergencyPauser`, `updatedAt`, `updatedAtBlock`. No longer tracks `feeRate`, `gasThreshold`, or `paymentValidityDuration`; those are compile-time constants on the contract now.

`AuthorizedAddress` (`id`: account address): `account` (`Bytes!`), `authorized` (`Boolean!`), `updatedAt`/`updatedAtBlock`.

#### `PauseAction` / `PauseState`

`PauseAction` (immutable, `id`: `{txHash}-{logIndex}`): an append-only log of every pause-related event. `type` (`PauseActionType`: `PAUSED`/`UNPAUSED`/`EMERGENCY_PAUSED`/`EMERGENCY_PAUSE_APPROVED`), `account` (`Bytes!`, who triggered it), `expiry` (`BigInt`, set only for `EMERGENCY_PAUSED`), `timestamp`, `block`, `txHash`.

`PauseState` (singleton, `id: "global"`): the current pause status. `ownerPaused` (`Boolean!`), `emergencyPauseExpiry`, `emergencyPausedBy`, `emergencyPausedAt` (all cleared on `Unpaused` or `EmergencyPauseApproved`), plus `lastAction`/`lastActionBy`/`lastActionAt`/`lastActionTx` mirroring the most recent `PauseAction`.

#### `FeeReceiver` / `FeeReceiverTokenBalance` / `FeeSweep`

Track what a per-invoice stealth fee receiver (see [Fee Receiver Privacy](fee-receiver-privacy.md)) has accrued from platform fees and had swept out, across both processors and the `Sweeper`.

`FeeReceiver` (`id`: receiver address): `address` (`Bytes!`), `sweepCount` (`BigInt!`, number of `Swept` transfers out of this receiver across all tokens), `firstCreditedAt` (`BigInt`, set on the first fee credit), `updatedAt`/`updatedAtBlock`, derived `balances`/`sweeps`.

`FeeReceiverTokenBalance` (`id`: `{feeReceiver}-{token}`): `feeReceiver`, `token`, `accrued` (`BigInt!`, cumulative fees credited in this token), `swept` (`BigInt!`, cumulative amount swept out), `balance` (`BigInt!`, `accrued - swept`), `updatedAt`/`updatedAtBlock`.

`FeeSweep` (immutable, `id`: `{txHash}-{logIndex}`): one `Swept` transfer. `feeReceiver`, `token`, `destination` (`Bytes!`), `amount` (`BigInt!`), `timestamp`, `block`, `txHash`.

#### Dashboard-metrics entities

Full detail lives in [`dashboard-metrics-technical-spec.md`](../dashboard-metrics-technical-spec.md) at the repo root; summary:

| Entity | Purpose |
| :--: | :--: |
| `PaymentVolume` / `VolumeStats` | Per-payment timeseries and daily-aggregated volume + cumulative paid-invoice count, per token |
| `EscrowBalance` / `EscrowStat` | Signed escrow deltas (`balance`) and gross inflow (`amountPaid`), rolled up hourly/daily into `totalBalance`/`totalAmountPaid` |
| `FeePaid` / `FeePaidStats` | Per-token protocol fee collection, daily-aggregated |
| `InvoiceActivity` / `InvoiceActivityStats` | Daily transaction-count split by `SIMPLE`/`ADVANCED` processor |
| `NewUser` / `NewUserStats`, `ActiveUser` / `ActiveUserStats` | New/active user counts by `CREATOR`/`PAYER` role, daily-aggregated |
| `GasPaid` | Mutable singleton (`id: "global"`) tracking cumulative platform-wallet gas spend and transaction count, updated only from `AdvancedPaymentProcessor` events |

---

### 4. Invoice State Machines

All timestamps are Unix seconds stored as `BigInt`.

#### 4.1 Simple Payment Processor States

| State | Meaning |
| :--: | :--: |
| `CREATED` | Invoice created by seller, awaiting payment |
| `PAID` | Buyer paid; seller must accept or reject within the decision window |
| `ACCEPTED` | Seller accepted; funds enter hold period before release |
| `REJECTED` | Seller rejected the payment; buyer refunded |
| `CANCELED` | Invoice canceled before payment |
| `REFUNDED` | Buyer refunded (rejection, or no seller action in time) |
| `RELEASED` | Hold period elapsed; funds released to seller |

There is no `LOCKED` or `BURNED` state in this subgraph's enum. The contract itself still burns an invoice's escrowed funds after every automated withdrawal retry fails, but that transition is not currently indexed here.

#### 4.2 Intermediated Payment Processor States

| State | Meaning |
| :--: | :--: |
| `CREATED` | Invoice created, awaiting payment |
| `PAID` | Buyer paid; funds held in escrow |
| `CANCELED` | Invoice canceled before payment |
| `DISPUTED` | Buyer raised a dispute |
| `DISPUTE_DISMISSED` | Dispute dismissed; invoice becomes releasable again |
| `DISPUTE_RESOLVED` | Dispute resolved in the seller's favor; becomes releasable |
| `DISPUTE_EXPIRED` | Dispute lapsed without a decision; becomes releasable, no funds move |
| `DISPUTE_SETTLED` | Funds split between buyer and seller; terminal |
| `REFUNDED` | Only reached once a refund's cumulative share brings the escrow balance to zero; a partial refund stays `PAID` |
| `RELEASED` | Funds released to seller |

There is no `LOCKED` state on this contract; releases and refunds are always triggered manually by the intermediated platform, with no automated retry path. `release` no longer requires `releaseAt` to have passed; the operator decides when to call it.

---

### 5. Example Queries

Confirm the current deployment's query endpoint in `payment-processor-subgraph/subgraph.yaml` / your Graph Studio dashboard before using these; the URL is environment-specific and not reproduced here.

#### Fetch recent simple invoices with their event log

```graphql
{
  simplePaymentProcessors(first: 10, orderBy: lastActionTime, orderDirection: desc) {
    id
    state
    price
    amountPaid
    fee
    seller {
      id
    }
    buyer {
      id
    }
    lastActionTime
    events(orderBy: timestamp, orderDirection: desc) {
      eventType
      txHash
      timestamp
    }
  }
}
```

#### Fetch all invoices for a specific user

```graphql
{
  user(id: "0xabc123...") {
    ownedSimpleInvoices {
      id
      state
      price
      buyer {
        id
      }
    }
    issuedAdvancedInvoices {
      id
      state
      price
      paymentToken {
        name
        decimal
      }
      buyer {
        id
      }
    }
  }
}
```

#### Fetch active disputes

```graphql
{
  advancedPaymentProcessors(
    where: { state: DISPUTED }
    orderBy: lastActionTime
    orderDirection: desc
  ) {
    id
    orderId
    invoiceNonce
    seller {
      id
    }
    buyer {
      id
    }
    balance
    paymentToken {
      id
      name
    }
  }
}
```

#### Fetch a meta-invoice and its child invoices

```graphql
{
  metaInvoice(id: "42") {
    id
    price
    contract
    buyer {
      id
    }
    invoices {
      id
      state
      price
    }
  }
}
```

#### Fetch notes for an invoice

```graphql
{
  notes(where: { invoiceId: 7 }, orderBy: noteId) {
    id
    noteId
    author
    share
    encryptedContent
    createdAtBlock
    createdAtTx
  }
}
```

#### Fetch whitelisted payment tokens

```graphql
{
  paymentTokens {
    id
    name
    decimal
  }
}
```

#### Fetch a fee receiver's balances and recent sweeps

```graphql
{
  feeReceiver(id: "0xabc123...") {
    address
    sweepCount
    firstCreditedAt
    balances {
      token {
        id
        name
      }
      accrued
      swept
      balance
    }
    sweeps(orderBy: timestamp, orderDirection: desc, first: 5) {
      token {
        name
      }
      destination
      amount
      timestamp
    }
  }
}
```

#### Fetch a multisig wallet with active signers and pending transactions

```graphql
{
  multiSigWallet(id: "0xf2EDEf709357e6587DC9600B58A34DA3d54369d1") {
    threshold
    signerCount
    signers(where: { active: true }) {
      address
      addedAt
    }
    transactions(where: { status_in: [PROPOSED, APPROVED] }, orderBy: proposedAt) {
      id
      target
      status
      approvalCount
    }
  }
}
```

#### Time-travel: volume 30 days ago

Not applicable: historical windows are served by the subgraph's native **Timeseries and Aggregations** (`VolumeStats`, `EscrowStat`, etc.), not block-by-timestamp lookups. See [`dashboard-metrics-technical-spec.md`](../dashboard-metrics-technical-spec.md#windowed-volume--percentage-change) for the exact query pattern.
