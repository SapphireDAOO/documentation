# Dashboard Metrics — Technical Spec

Aggregated metrics maintained in the subgraph for the dashboard. Historical/past-window views are served by the subgraph's native **Timeseries and Aggregations** — daily (and, where useful, hourly) buckets are rolled up automatically by graph-node, so windowed and percentage-change views need no third-party block-by-timestamp lookups or time-travel queries.

## Schemas

There is no consolidated `metricData` entity. Each metric is backed by its own immutable **timeseries** entity plus an **aggregation** that graph-node rolls up per interval. The only mutable singleton is `GasPaid`. Per-token metrics reference the `PaymentToken` entity (its `id` is the token address; the native token ETH uses the zero address).

### Volume — `PaymentVolume` / `VolumeStats`

```graphql
# One immutable record per payment
type PaymentVolume @entity(timeseries: true) {
  id: Int8!
  timestamp: Timestamp!
  token: PaymentToken!
  amount: BigInt!
}

# Daily volume per token, rolled up automatically by graph-node
type VolumeStats @aggregation(intervals: ["day"], source: "PaymentVolume") {
  id: Int8!
  timestamp: Timestamp!
  token: PaymentToken! # grouping dimension
  dailyVolume: BigInt! @aggregate(fn: "sum", arg: "amount")
  invoicePaid: BigInt! @aggregate(fn: "count", cumulative: true)
}
```

### Escrow — `EscrowBalance` / `EscrowStat`

```graphql
# Signed escrow deltas: + on payment, − on refund / release / dispute settlement
type EscrowBalance @entity(timeseries: true) {
  id: Int8!
  timestamp: Timestamp!
  token: PaymentToken!
  balance: BigInt!
}

type EscrowStat
  @aggregation(intervals: ["hour", "day"], source: "EscrowBalance") {
  id: Int8!
  timestamp: Timestamp!
  token: PaymentToken!
  total: BigInt! @aggregate(fn: "sum", arg: "balance")
}
```

### Fees — `FeePaid` / `FeePaidStats`

```graphql
type FeePaid @entity(timeseries: true) {
  id: Int8!
  timestamp: Timestamp!
  token: PaymentToken!
  amount: BigInt!
}

type FeePaidStats @aggregation(intervals: ["day"], source: "FeePaid") {
  id: Int8!
  timestamp: Timestamp!
  token: PaymentToken!
  totalFeePaid: BigInt! @aggregate(fn: "sum", arg: "amount")
}
```

### Invoice activity — `InvoiceActivity` / `InvoiceActivityStats`

```graphql
type InvoiceActivity @entity(timeseries: true) {
  id: Int8!
  timestamp: Timestamp!
  invoiceType: InvoiceType! # SIMPLE | ADVANCED, grouping dimension
}

type InvoiceActivityStats
  @aggregation(intervals: ["day"], source: "InvoiceActivity") {
  id: Int8!
  timestamp: Timestamp!
  invoiceType: InvoiceType!
  totalActivity: BigInt! @aggregate(fn: "count", cumulative: true)
}
```

### User metrics — `NewUser` / `ActiveUser`

```graphql
enum UserRole {
  CREATOR # seller
  PAYER # buyer
}

type NewUser @entity(timeseries: true) {
  id: Int8!
  timestamp: Timestamp!
  user: Bytes!
  role: UserRole!
}

type NewUserStats @aggregation(intervals: ["day"], source: "NewUser") {
  id: Int8!
  timestamp: Timestamp!
  role: UserRole! # grouping dimension
  newUsers: BigInt! @aggregate(fn: "count")
  totalUsers: BigInt! @aggregate(fn: "count", cumulative: true)
}

type ActiveUser @entity(timeseries: true) {
  id: Int8!
  timestamp: Timestamp!
  user: Bytes!
  role: UserRole!
}

type ActiveUserStats @aggregation(intervals: ["day"], source: "ActiveUser") {
  id: Int8!
  timestamp: Timestamp!
  activeUsers: BigInt! @aggregate(fn: "count")
}
```

### Gas — `GasPaid`

```graphql
type GasPaid @entity(immutable: false) {
  id: ID! # "global"
  amount: BigInt!
  transactionCount: BigInt!
  lastTimeStamp: BigInt!
}
```

All numeric fields are stored as `BigInt` (raw integer units; bignumber-safe). Timestamps are unix seconds.

## USD Conversion

USD values shown on the dashboard (Total Volume, escrow display, fees) are computed at read time using the **current market value** of each token.

- **Storage:** raw token amounts only — `VolumeStats.dailyVolume`, `EscrowStat.total`, and `FeePaidStats.totalFeePaid`, each keyed by `PaymentToken`. No USD value is persisted on-chain or in the subgraph.
- **Read path:** fetch the current market price for each held token from a third-party price service, multiply by the stored token amount, and sum across tokens to produce the displayed USD figure.
- **Caching:** cache the price globally and reuse it across every metric in a render pass. Refresh on whatever cadence the price source justifies.
- **Frontend rule:** the conversion happens once per render against the cached price; the frontend does not issue a price call per metric.

Total Volume and Total Escrow Balance both follow this same read-time market-conversion flow.

## First Section Metrics

### Total Volume

Per-token volume recorded in `PaymentVolume` and rolled up daily into `VolumeStats.dailyVolume`. The dashboard shows a 30-day window plus a percentage change vs. the prior 30-day window.

- **Storage:** one `PaymentVolume` point per payment carrying `token` and `amount` (raw token amount). Cumulative volume is **not** persisted as a single field; window totals are summed from the daily buckets.
- **Aggregation:** daily volume rolled up with `(fn: "sum", arg: "amount")` per token.
- **Display:** the per-token window totals are converted to USD per the chosen [USD Conversion](#usd-conversion) strategy, then summed.
- **Source events:** invoice `Paid` (both simple and advanced processors).
- **Update on:** every successful payment.

#### Windowed Volume & Percentage Change

Windowed views and percentage change are computed on the client; the subgraph itself does not persist percentages. The window totals come from the subgraph's **Timeseries and Aggregations** — no third-party block-by-timestamp resolution and no time-travel queries.

Notation (computed per token, then summed in USD on the client):

- `W_curr` — Σ `dailyVolume` over the current 30-day window.
- `W_prior` — Σ `dailyVolume` over the prior 30-day window (≈60→30 days ago).

Derived values:

- **30-day volume (displayed):** `W_curr`
- **% change:** `(W_curr - W_prior) / W_prior * 100`

Window boundaries (`$thirtyDaysAgo` / `$sixtyDaysAgo`) are derived by day-aligning the current timestamp and stepping back in whole days, so each bound lands on a daily-bucket edge:

- `dayMark = now - (now % 86400)`
- `$thirtyDaysAgo = dayMark - 30 days`
- `$sixtyDaysAgo = dayMark - 60 days`

Fetching window totals via Timeseries and Aggregations:

The subgraph records each payment in `PaymentVolume` and rolls it up into the daily `VolumeStats` aggregation. graph-node closes each day's bucket on its own, so each window's volume is obtained by summing the daily buckets that fall inside it (bounded by `timestamp_gte`/`timestamp_lte`).

1. Sum the daily buckets for the **prior** 30-day window (≈60→30 days ago):

   ```graphql
   {
     volumeStats(
       interval: day
       where: {
         token: "0x…" # PaymentToken id (token address)
         timestamp_gte: $sixtyDaysAgo
         timestamp_lte: $thirtyDaysAgo
       }
       orderBy: timestamp
       orderDirection: desc
     ) {
       timestamp
       dailyVolume
     }
   }
   ```

   `Σ dailyVolume` over this window is `W_prior`.

2. Run the same query bounded to the **current** 30-day window (`timestamp_gte: $thirtyDaysAgo`). Its `Σ dailyVolume` is the displayed 30-day volume, `W_curr`.
3. Feed the two window sums into the % change formula: `(W_curr − W_prior) / W_prior * 100`.

Closed daily buckets are immutable, so they can be cached indefinitely; only the current (open) bucket needs refreshing.

Guard against `W_prior == 0` (no activity in the prior window) before computing the percentage on the client.

### Total Escrow Balance

Live escrow balance tracked **per token** as signed deltas in `EscrowBalance`, rolled up into `EscrowStat.total`.

- **Formula:**
  - On payment: push `+amount` (raw token amount).
  - On settlement: push the **negated** amount (`Refund`, `Disputed`-settled, `Release`) so the daily/hourly `sum` nets out to the live balance.
- **Source events:** invoice `Paid`, `Refund`, `Disputed`-settled, `Release`.
- **Amount source:** the amount is read from the invoice-level event payload, not from a contract balance call.
- **Aggregation:** `EscrowStat.total` via `(fn: "sum", arg: "balance")` over its own signed-point timeseries — separate from `PaymentVolume`, which stays positive-only because volume is cumulative. `EscrowStat` is rolled up at both `hour` and `day` intervals; live escrow is the running sum across buckets.
- **Increase/decrease rate:** tracked day-by-day (daily granularity), computed using the same windowed pattern as [Windowed Volume & Percentage Change](#windowed-volume--percentage-change).
- **Display:** depends on the [USD Conversion](#usd-conversion) decision — convert the per-token balances on the read path.

### Total Fees Paid

Per-token protocol fees collected, recorded in `FeePaid` and rolled up into `FeePaidStats.totalFeePaid`. The dashboard shows a 30-day window plus a percentage change vs. the prior 30-day window (same windowed pattern as [Total Volume](#windowed-volume--percentage-change)).

- **Storage / Aggregation:** the fee amount is summed **per token** via `(fn: "sum", arg: "amount")` (raw token amount — not a percentage computed in the subgraph).
- **Source events:** the **advanced** processor realizes the fee at payment (`InvoicePaid`); the **simple** processor realizes it at `InvoiceAccepted`. Zero-fee events are not recorded.
- **Display:** the per-token fee totals are converted to USD per the chosen [USD Conversion](#usd-conversion) strategy, then summed.

### Total Invoices Paid

Cumulative count of invoices paid, surfaced from `VolumeStats.invoicePaid`. The dashboard surfaces this over a **7-day** window (with the prior-7-day comparison following the same windowed pattern as [Total Volume](#windowed-volume--percentage-change)).

- **Formula:** one `PaymentVolume` point per payment; the count is **cumulative** — settlement events do **not** decrement it.
- **Aggregation:** `VolumeStats.invoicePaid` via `(fn: "count", cumulative: true)` over the volume timeseries.
- **Source events:** invoice `Paid`.

## Invoice Activity

Daily transaction count split between the **simple** and **advanced** payment processor contracts, driven by the `InvoiceActivity` timeseries.

- **Timeseries source:** one `InvoiceActivity` record per invoice event, with `invoiceType` (`SIMPLE` / `ADVANCED`, the `InvoiceType` enum) as the grouping dimension, aggregated into `InvoiceActivityStats.totalActivity` with `(fn: "count", cumulative: true)`. The daily `count` bucket yields each day's activity per processor.
- **Historical days:** the ~7-day chart is a single ranged query over the daily `InvoiceActivityStats` buckets — no block anchors.
- **Caching:** closed daily buckets are immutable, so a past day can be cached indefinitely; only the current (open) bucket needs refreshing.

## Recent Transactions

A short, ordered list of the most recent transactions across the two processors.

- **Schema:** served from the `InvoiceEvent` entity (`eventType`, `txHash`, `timestamp`, plus the `simpleInvoice` / `advancedInvoice` link); the displayed amount is read from the linked invoice.
- **Source:** query `InvoiceEvent` restricted to `INVOICE_PAID`, `INVOICE_REFUNDED`/`REFUNDED`, `INVOICE_RELEASED`/`PAYMENT_RELEASED`, and `DISPUTE_SETTLED` event types; order by `timestamp` desc and keep the most recent 5.
- **`txHash`:** consumed by the frontend to build the block-explorer link.

## User Metrics

Backed by **user timeseries entities** (`NewUser`, `ActiveUser`) carrying the user (`Bytes`) and their `role` (the `UserRole` enum: `CREATOR` = seller, `PAYER` = buyer); computed views are derived using the same daily-aggregation windows as Total Volume.

- **New Creators / New Payers:** the first time a creator or payer address is seen in the subgraph, a `NewUser` point is emitted (with `role` as a dimension). `NewUserStats` aggregates it with `newUsers: (fn: "count")` per day and `totalUsers: (fn: "count", cumulative: true)`. The dashboard surfaces the count _added in the last 7 days_ (raw count, not percentage) by summing the daily `newUsers` buckets and compares it against the prior 7-day window — no block-by-timestamp anchors.
- **Active Users:** count of unique users observed per day. The mapping emits at most one `ActiveUser` point per user per UTC day (deduped via `User.lastActiveDay`), so `ActiveUserStats.activeUsers` (`fn: "count"`) over a day equals the number of unique users active that day. Day-over-day percentage growth is computed from the same daily-aggregation windows as [Total Volume](#windowed-volume--percentage-change).

## Gas Tracker

Tracked in the `GasPaid` singleton (`id: "global"`, fields `amount`, `transactionCount`, `lastTimeStamp`). It is a mutable cumulative entity, not an aggregation.

- **Total Gas Paid:** `amount` accumulates whenever a platform wallet sends a transaction; `transactionCount` bumps for each. Gas cost is approximated as `gasPrice * gasLimit` (receipts/`gasUsed` are not available in handlers).
- **Avg Gas Per Transaction:** `amount / transactionCount`, computed on the client.
- **Current Gas Price:** fetched from Etherscan.

## Wallet Balance

- **Fee Receiver:** query `FeePaidStats.totalFeePaid` per token and convert per the [USD Conversion](#usd-conversion) decision made for Total Volume.
- **Gas Reserve:** RPC `eth_getBalance` against the platform wallet.

## Real-time Updates

The subgraph-derived flow above is poll-based and reflects committed state only. To surface live activity without waiting for the next poll, the dashboard subscribes to a **websocket** stream and applies incoming events optimistically on top of the last subgraph snapshot.

- The subgraph remains the source of truth for cumulative values and historical-window queries.
- Websocket deltas update only the live displayed values between polls.
- On reconnect or stale-session recovery, drop optimistic state and reseed from the subgraph.

## Notes for Implementation

- Update related metrics inside a single event-handler transaction so the aggregations stay internally consistent.
- Settlement events (`Refund`, `Disputed`-settled, `Release`) must reverse escrow by pushing a negated `EscrowBalance` point. They do **not** push to `PaymentVolume` or affect `VolumeStats.invoicePaid` — both are cumulative.
- For windowed/historical reads, query the daily aggregation buckets and sum the relevant windows on the client; closed daily buckets are immutable and safely cacheable, so only the current open bucket needs refreshing.
- The frontend should never recompute USD values from raw token amounts directly; it consumes whichever USD representation results from the chosen conversion strategy.
