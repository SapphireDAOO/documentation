# Payment Processor Subgraph

### Table of Contents

1. Overview
2. Data Sources
3. Entity Reference
4. Invoice State Machines
5. Example Queries

---

### 1. Overview

This subgraph indexes **six** Sapphire DAO smart contracts deployed on the Base Sepolia testnet:

- **SimplePaymentProcessor** — A native-token (ETH) escrow contract. A seller creates an invoice, the buyer pays in ETH, and the seller accepts (releasing funds after a hold period) or rejects (triggering a refund). Backed by an on-chain min-heap and Chainlink Automation for automated release/refund/retry, with a `LOCKED` fallback state if all automated withdrawal attempts fail.
- **AdvancedPaymentProcessor** — A multi-token escrow contract with dispute resolution, partial refunds, meta-invoices (batch invoices), and USD-price-pegged payments via an `OracleManager` contract wrapping Chainlink price feeds. Releases and refunds here are always triggered manually by the intermediated platform — there is no automated retry/locked-fund path.
- **PaymentProcessorStorage** — Shared configuration contract (fee rate, fee receiver, default hold period, intermediated platform address, gas threshold, payment validity duration) and the authorized-address allowlist used by both processors.
- **Notes** — An encrypted note store attached to invoices. Notes are stored off-chain but their on-chain references and per-user "opened" state are indexed here.
- **MultiSig** — Multisig governance contract gating privileged admin calls into the processors and storage contract. Its signer set, threshold, proposed/approved/executed/canceled transactions, and approvals are indexed here.
- **OracleManager** — Emits `PriceFeedSet` when a token gains a Chainlink price feed; the handler registers/refreshes the token's `PaymentToken` metadata.

The subgraph also maintains a set of **dashboard-metrics** timeseries/aggregation entities (volume, escrow balance, fees, invoice activity, user growth, gas spend) derived from the same events — see [Entity Reference](#3-entity-reference).

---

### 2. Data Sources

Addresses and start blocks below reflect the current `subgraph.yaml` on the `base-sepolia` network; confirm against that file before relying on them; a redeploy can change both.

#### SimplePaymentProcessor

- **Address:** `0xC785B7f52F591BF0ce80beE45B09e1cf0A972957`
- **Start block:** `42588324`
- **Handler file:** `src/simple-payment-processor.ts`

| Event | Handler | Description |
| :--- | :--- | :--- |
| `InvoiceCreated(invoiceId, invoice)` | `handleInvoiceCreated` | Creates the `SimplePaymentProcessor` entity, tracks the seller as a `CREATOR` user, writes an `InvoiceEvent` |
| `InvoicePaid(invoiceId, buyer, amountPaid, expiresAt)` | `handleInvoicePaid` | Records buyer and amount; tracks the buyer as a `PAYER` user; pushes `PaymentVolume` and a positive `EscrowBalance` delta |
| `InvoiceAccepted(invoiceId)` | `handleInvoiceAccepted` | Sets `state = ACCEPTED` and, if not already set, `releaseAt` from the default hold period |
| `InvoiceCanceled(invoiceId)` | `handleInvoiceCanceled` | Marks the invoice `CANCELED` |
| `InvoiceRejected(invoiceId, amount)` | `handleInvoiceRejected` | Marks the invoice `REJECTED`; pushes a negative `EscrowBalance` delta |
| `InvoiceRefunded(invoiceId, amount)` | `handleInvoiceRefunded` | Marks the invoice `REFUNDED`; pushes a negative `EscrowBalance` delta |
| `InvoiceReleased(invoiceId, sellerAmount, fee)` | `handleInvoiceReleased` | Marks the invoice `RELEASED`, records `fee`; pushes a negative `EscrowBalance` delta and a `FeePaid` point |
| `LockedPaymentRecovered(invoiceId, recipient, amount)` | `handleLockedPaymentRecovered` | Records the recovery event (does not itself adjust escrow-balance tracking) |
| `TransferFailed(invoiceId, recipient, amount)` | `handleTransferFailed` | Records a failed transfer event |
| `UpdateHoldPeriod(invoiceId, releaseDueTimestamp)` | `handleHoldPeriod` | Updates `releaseAt` |
| `WithdrawalRetried(invoiceId, recipient, amount, attempt)` | `handleWithdrawalRetried` | Records an automated-retry event |

Every handler above also writes an `InvoiceEvent` row and increments the `InvoiceActivity` (`SIMPLE`) timeseries.

#### AdvancedPaymentProcessor

- **Address:** `0x0EecA9DE862fDFF9147aa1c55f186BB3881478E7`
- **Start block:** `42588324`
- **Handler file:** `src/advanced-payment-processor.ts`

| Event | Handler | Description |
| :--- | :--- | :--- |
| `InvoiceCreated(invoiceId, invoice)` | `handleAdvancedPaymentProcessorCreated` | Creates the `AdvancedPaymentProcessor` entity, links `metaInvoice` if any, tracks the seller as `CREATOR` |
| `InvoicePaid(invoiceId, paymentToken, escrowAddress, amount, releaseAt)` | `handleInvoicePaid` | Records payment/escrow details; tracks the buyer as `PAYER`; pushes `PaymentVolume` and a positive `EscrowBalance` delta |
| `InvoiceCanceled(invoiceId)` | `handleInvoiceCanceled` | Marks the invoice `CANCELED`; reduces the parent meta-invoice's price if applicable |
| `DisputeCreated(invoiceId)` | `handleDisputeCreated` | Marks the invoice `DISPUTED` |
| `DisputeDismissed(invoiceId)` | `handleDisputeDismissed` | Marks the invoice `DISPUTE_DISMISSED` |
| `DisputeResolved(invoiceId)` | `handleDisputeResolved` | Marks the invoice `DISPUTE_RESOLVED` |
| `DisputeSettled(invoiceId, sellerAmount, buyerAmount, fee)` | `handleDisputeSettled` | Marks the invoice `DISPUTE_SETTLED`, zeroes balance; pushes a negative `EscrowBalance` delta and a `FeePaid` point |
| `EscrowCreated(invoiceId, escrow)` | `handleEscrowCreated` | Records the escrow address on the invoice |
| `LockedPaymentRecovered(invoiceId, recipient, amount)` | `handleLockedPaymentRecovered` | Declared in the interface but never actually emitted by this contract (no locked-fund path here) |
| `MetaInvoiceCreated(metaInvoiceId, totalPrice)` | `handleMetaInvoiceCreated` | Creates a `MetaInvoice` entity; tracks the caller as `PAYER` |
| `OracleUpdated(previousOracle, newOracle)` | `handleOracleUpdated` | Recorded as a processor event; not tied to a specific invoice |
| `PaymentReleased(invoiceId, receiver, currency, sellerAmount, fee)` | `handlePaymentReleased` | Marks the invoice `RELEASED`, zeroes balance; pushes a negative `EscrowBalance` delta and a `FeePaid` point |
| `Refunded(invoiceId, amount)` | `handleRefunded` | Reduces escrow balance by `amount`; marks the invoice `REFUNDED` only on a full refund |
| `TransferFailed(invoiceId, recipient, amount)` | `handleTransferFailed` | Records a failed best-effort transfer (release/dispute fee payout) |
| `UpdateReleaseTime(invoiceId, newHoldPeriod)` | `handleUpdateReleaseTime` | Updates `releaseAt` |

Every handler above except `InvoicePaid` also bumps the `GasPaid` singleton; every handler also writes an `InvoiceEvent` and increments `InvoiceActivity` (`ADVANCED`).

#### PaymentProcessorStorage

- **Address:** `0x74b1301b8a1DBdF0318bC81dD8c1b1375d0BF9AF`
- **Start block:** `42588324`
- **Handler file:** `src/payment-processor-storage.ts`

| Event | Handler | Description |
| :--- | :--- | :--- |
| `AuthorizationUpdated(account, authorized)` | `handleAuthorizationUpdated` | Upserts an `AuthorizedAddress` entity |
| `ConfigurationInitialized(config)` | `handleConfigurationInitialized` | Seeds the `StorageConfiguration` singleton from the full config struct |
| `DefaultHoldPeriodUpdated(defaultHoldPeriod)` | `handleDefaultHoldPeriodUpdated` | Updates the singleton's `defaultHoldPeriod` |
| `FeeRateUpdated(feeRate)` | `handleFeeRateUpdated` | Updates the singleton's `feeRate` |
| `FeeReceiverUpdated(feeReceiver)` | `handleFeeReceiverUpdated` | Updates the singleton's `feeReceiver` |
| `GasThresholdUpdated(gasThreshold)` | `handleGasThresholdUpdated` | Updates the singleton's `gasThreshold` |
| `MarketplaceUpdated(marketplace)` | `handleMarketplaceUpdated` | Updates the singleton's `marketplace` |
| `OwnershipTransferred(previousOwner, newOwner)` | `handleOwnershipTransferred` | Updates the singleton's `owner` |
| `PaymentValidityDurationUpdated(validityDuration)` | `handlePaymentValidityDurationUpdated` | Updates the singleton's `paymentValidityDuration` |

#### Notes

- **Address:** `0x38844FD5258943F0Af0db706CeC75a9233140087`
- **Start block:** `42588324`
- **Handler file:** `src/notes.ts`

| Event | Handler | Description |
| :--- | :--- | :--- |
| `NoteCreated(invoiceId, noteId, author, share, encryptedContent)` | `handleNoteCreated` | Creates a `Note` entity, id `{invoiceId}-{noteId}` |
| `NoteStateChanged(invoiceId, noteId, user, opened)` | `handleNoteStateChanged` | Upserts a `NoteOpenState` entity, id `{invoiceId}-{noteId}-{userAddress}` |

#### MultiSig

- **Address:** `0xA42498b1a91cB61B5303Ec0432f27b87B8255B4e`
- **Start block:** `42588324`
- **Handler file:** `src/multi-sig.ts`

| Event | Handler | Description |
| :--- | :--- | :--- |
| `SignerAdded(signer)` | `handleSignerAdded` | Activates the `MultiSigSigner` (creates it if new), refreshes `MultiSigWallet` from live contract state |
| `SignerRemoved(signer)` | `handleSignerRemoved` | Deactivates the `MultiSigSigner` |
| `ThresholdUpdated(oldThreshold, newThreshold)` | `handleThresholdUpdated` | Updates `MultiSigWallet.threshold` |
| `TransactionProposed(txHash, target, value, data, nonce, proposer)` | `handleTransactionProposed` | Creates a `MultiSigTransaction` (status `PROPOSED`), refreshed from a live `getTransaction` call |
| `ApprovalAdded(txHash, approver, approvalCount)` | `handleApprovalAdded` | Upserts a `MultiSigApproval`, updates `approvalCount` |
| `TransactionApproved(txHash)` | `handleTransactionApproved` | Sets status `APPROVED` |
| `TransactionExecuted(txHash, executor)` | `handleTransactionExecuted` | Sets status `EXECUTED`, records `executor`/`executedAt` |
| `TransactionCanceled(txHash)` | `handleTransactionCanceled` | Sets status `CANCELED` |

#### OracleManager

- **Address:** `0x95423c49f0550e5Aaba0f53B434B05E65B0B1254`
- **Start block:** `42588324`
- **Handler file:** `src/oracle-manager.ts`

| Event | Handler | Description |
| :--- | :--- | :--- |
| `PriceFeedSet(token, aggregator, heartbeat)` | `handlePriceFeedSet` | Registers a `PaymentToken` if it doesn't already exist (no-op otherwise) |

---

### 3. Entity Reference

#### `InvoiceEvent`

An immutable append-only log row, one per processor event. The `id` is `{txHash}-{logIndex}`.

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `ID!` | `{txHash}-{logIndex}` |
| `eventType` | `PaymentProcessorEventType!` | e.g. `INVOICE_CREATED`, `INVOICE_PAID`, `DISPUTE_SETTLED`, etc. |
| `txHash` | `Bytes!` | Transaction hash |
| `timestamp` | `BigInt!` | Block timestamp |
| `simpleInvoice` | `SimplePaymentProcessor` | Set for simple-processor events |
| `advancedInvoice` | `AdvancedPaymentProcessor` | Set for advanced-processor events |

#### `SimplePaymentProcessor`

One invoice on the SimplePaymentProcessor contract. The `id` is the on-chain `invoiceId` (numeric string).

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `ID!` | On-chain invoice ID |
| `invoiceNonce` | `BigInt!` | Internal invoice nonce |
| `state` | `SimplePaymentProcessorState!` | See [State Machine](#41-simple-payment-processor-states) |
| `seller` | `User!` | Invoice creator |
| `buyer` | `User` | Set once paid |
| `price` | `BigInt!` | Invoice price in wei |
| `amountPaid` | `BigInt` | Amount paid by the buyer |
| `invalidateAt` / `expiresAt` / `releaseAt` | `BigInt` | Validity/decision/release timestamps |
| `fee` | `BigInt` | Protocol fee deducted on release |
| `contract` | `Bytes!` | SimplePaymentProcessor address |
| `events` | `[InvoiceEvent!]!` | Derived from `InvoiceEvent.simpleInvoice` |
| `lastActionTime` | `BigInt` | Timestamp of the most recent state change |
| `buyerNote` / `sellerNote` | `String` | Optional notes |

#### `AdvancedPaymentProcessor`

One invoice on the AdvancedPaymentProcessor contract; supports multi-token payments and dispute resolution. The `id` is the on-chain `invoiceId`.

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `ID!` | On-chain invoice ID |
| `invoiceNonce` | `BigInt!` | Internal invoice nonce |
| `state` | `AdvancedPaymentProcessorState!` | See [State Machine](#42-intermediated-payment-processor-states) — no `LOCKED` state exists here |
| `seller` | `User!` | Invoice seller |
| `buyer` | `User` | Set once paid |
| `escrow` | `Bytes` | Escrow contract address |
| `balance` | `BigInt` | Current escrow balance |
| `paymentToken` | `PaymentToken` | Payment token (native ETH if unset/zero address) |
| `releaseAt` | `BigInt` | Release timestamp |
| `amountReleased` / `amountRefunded` | `BigInt` | Cumulative released/refunded amounts |
| `sellerAmountReceivedAfterDispute` / `buyerAmountReceivedAfterDispute` | `BigInt` | Dispute-settlement split |
| `amountPaid` | `BigInt` | Total paid by buyer |
| `contract` | `Bytes!` | AdvancedPaymentProcessor address |
| `events` | `[InvoiceEvent!]!` | Derived from `InvoiceEvent.advancedInvoice` |
| `fee` | `BigInt` | Protocol fee |
| `lastActionTime` | `BigInt` | Timestamp of the most recent state change |
| `metaInvoice` | `MetaInvoice` | Set if part of a meta-invoice |
| `buyerNote` / `sellerNote` | `String` | Optional notes |

#### `MetaInvoice`

Groups one or more `AdvancedPaymentProcessor` invoices into a single payable unit. The `id` is the `metaInvoiceId`.

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `ID!` | On-chain meta-invoice ID |
| `invoiceNonce` | `BigInt!` | Same value as `id` |
| `buyer` | `User!` | Address that initiated the meta-invoice payment |
| `price` | `BigInt!` | Combined price across sub-invoices (reduced as sub-invoices are canceled) |
| `contract` | `Bytes!` | AdvancedPaymentProcessor address |
| `invoices` | `[AdvancedPaymentProcessor!]!` | Derived: child invoices linked via `metaInvoice` |

#### `User`

A unique wallet address seen by either processor. The `id` is the hex address.

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `ID!` | Wallet address (hex string) |
| `lastActiveDay` | `BigInt` | UTC day index of last observed activity — used to emit at most one `ActiveUser` point per day |
| `ownedSimpleInvoices` | `[SimplePaymentProcessor!]!` | Seller-side simple invoices |
| `paidSimpleInvoices` | `[SimplePaymentProcessor!]!` | Buyer-side simple invoices |
| `issuedAdvancedInvoices` | `[AdvancedPaymentProcessor!]!` | Seller-side advanced invoices |
| `receivedAdvancedInvoices` | `[AdvancedPaymentProcessor!]!` | Buyer-side advanced invoices |
| `metaInvoices` | `[MetaInvoice!]!` | Meta-invoices paid by this user |

#### `PaymentToken`

Metadata for a token registered either via a payment or via `OracleManager.PriceFeedSet`. The `id` is the token's contract address (zero address = native ETH).

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `ID!` | Token contract address, or zero address for ETH |
| `name` | `String` | Token name |
| `decimal` | `Int` | Token decimals |

#### `Note` / `NoteOpenState`

`Note` (`id`: `{invoiceId}-{noteId}`): `invoiceId`, `noteId` (`BigInt!`), `author` (`Bytes!`), `share` (`Boolean!`), `encryptedContent` (`Bytes!`), `createdAtBlock`/`createdAtTx`.

`NoteOpenState` (`id`: `{invoiceId}-{noteId}-{userAddress}`): `invoiceId`, `noteId`, `user` (`Bytes!`), `opened` (`Boolean!`), `updatedAtBlock`/`updatedAtTx`.

#### `MultiSigWallet` / `MultiSigSigner` / `MultiSigTransaction` / `MultiSigApproval`

See [core-contracts/multisig.sol.md](core-contracts/multisig.sol.md) for the full field reference — summary:

- `MultiSigWallet` (`id`: contract address): `threshold`, `signerCount`, `transactionCount`, derived `signers`/`transactions`.
- `MultiSigSigner` (`id`: `{wallet}-{signer}`): `wallet`, `address`, `active`, `addedAt`, `removedAt`.
- `MultiSigTransaction` (`id`: `txHash`): `wallet`, `target`, `value`, `data`, `nonce`, `proposer`, `status` (`MultisigStatus`: `PROPOSED`/`APPROVED`/`CANCELED`/`EXECUTED`), `approvalCount`, `proposedAt`, `executedAt`, `executor`.
- `MultiSigApproval` (`id`: `{txHash}-{approver}`): `transaction`, `signer`, `approver`, `approvalCount`, `approvedAt`.

#### `StorageConfiguration` / `AuthorizedAddress`

`StorageConfiguration` (singleton, `id: "global"`): mirrors `PaymentProcessorStorage`'s current config — `owner`, `feeRate`, `feeReceiver`, `defaultHoldPeriod`, `marketplace`, `gasThreshold`, `paymentValidityDuration`, `updatedAt`, `updatedAtBlock`.

`AuthorizedAddress` (`id`: account address): `account` (`Bytes!`), `authorized` (`Boolean!`), `updatedAt`/`updatedAtBlock`.

#### Dashboard-metrics entities

Full detail lives in [`metric.md`](../metric.md) at the repo root; summary:

| Entity | Purpose |
| :--- | :--- |
| `PaymentVolume` / `VolumeStats` | Per-payment timeseries and daily-aggregated volume + cumulative paid-invoice count, per token |
| `EscrowBalance` / `EscrowStat` | Signed escrow deltas (`balance`) and gross inflow (`amountPaid`), rolled up hourly/daily into `totalBalance`/`totalAmountPaid` |
| `FeePaid` / `FeePaidStats` | Per-token protocol fee collection, daily-aggregated |
| `InvoiceActivity` / `InvoiceActivityStats` | Daily transaction-count split by `SIMPLE`/`ADVANCED` processor |
| `NewUser` / `NewUserStats`, `ActiveUser` / `ActiveUserStats` | New/active user counts by `CREATOR`/`PAYER` role, daily-aggregated |
| `GasPaid` | Mutable singleton (`id: "global"`) tracking cumulative platform-wallet gas spend and transaction count |

---

### 4. Invoice State Machines

All timestamps are Unix seconds stored as `BigInt`.

#### 4.1 Simple Payment Processor States

| State | Meaning |
| :--- | :--- |
| `CREATED` | Invoice created by seller, awaiting payment |
| `PAID` | Buyer paid; seller must accept or reject within the decision window |
| `ACCEPTED` | Seller accepted; funds enter hold period before release |
| `REJECTED` | Seller rejected the payment; buyer refunded |
| `CANCELED` | Invoice canceled before payment |
| `REFUNDED` | Buyer refunded (rejection, or no seller action in time) |
| `RELEASED` | Hold period elapsed; funds released to seller |
| `LOCKED` | All automated withdrawal retries exhausted; recoverable only via `releaseLocked` |

#### 4.2 Intermediated Payment Processor States

| State | Meaning |
| :--- | :--- |
| `CREATED` | Invoice created, awaiting payment |
| `PAID` | Buyer paid; funds held in escrow |
| `CANCELED` | Invoice canceled before payment |
| `DISPUTED` | Buyer raised a dispute |
| `DISPUTE_DISMISSED` | Dispute dismissed; invoice becomes releasable again |
| `DISPUTE_RESOLVED` | Dispute resolved in the seller's favor; becomes releasable |
| `DISPUTE_SETTLED` | Funds split between buyer and seller; terminal |
| `REFUNDED` | Only reached on a **full** refund (`_refundShare == 10000` bps) — partial refunds stay `PAID` |
| `RELEASED` | Funds released to seller |

There is no `LOCKED` state on this contract — releases and refunds are always triggered manually by the intermediated platform, with no automated retry path.

---

### 5. Example Queries

Confirm the current deployment's query endpoint in `payment-processor-subgraph/subgraph.yaml` / your Graph Studio dashboard before using these — the URL is environment-specific and not reproduced here.

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

#### Fetch a multisig wallet with active signers and pending transactions

```graphql
{
  multiSigWallet(id: "0xA42498b1a91cB61B5303Ec0432f27b87B8255B4e") {
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

Not applicable — historical windows are served by the subgraph's native **Timeseries and Aggregations** (`VolumeStats`, `EscrowStat`, etc.), not block-by-timestamp lookups. See [`metric.md`](../metric.md#windowed-volume--percentage-change) for the exact query pattern.
