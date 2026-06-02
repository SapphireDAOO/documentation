# Dashboard Metrics — Technical Spec

Aggregated metrics maintained in the subgraph for the dashboard. Historical/past-window views are served by the subgraph's native **Timeseries and Aggregations** — daily (and, where useful, hourly) buckets are rolled up automatically by graph-node, so windowed and percentage-change views need no third-party block-by-timestamp lookups or time-travel queries.

## Schemas

The bulk of dashboard state is consolidated into a single `metricData` entity. Recent transactions, new-user count, and gas paid remain in their own entities.

### Consolidated metric entity

```ts
const metricData = {
  tokenData: [
    {
      name: "",
      volumeBalance: "",
      escrowBalance: "",
    },
  ],
  paidInvoices: "",
  lastPaidInvoiceTimestamp: "",
  transactionCount: {
    simple: "",
    advanced: "",
    lastUpdateTimestamp: "",
  },
};
```

Volume and escrow balance are stored **per token** (raw token amounts). The chosen [USD Conversion](#usd-conversion) strategy determines how these are rendered to USD on the client.

### Recent transactions

```ts
const recentTransaction = [
  {
    transactionHash: "0x", // used for frontend link
    timestamp: "",
    amount: "", // + for paid, - for release
  },
];
```

### User metrics

```ts
let newUsers: number;
```

### Gas paid

```ts
const gasPaid = {
  amount: "",
  transactionCount: "",
  lastTimeStamp: "",
};
```

All numeric fields are stored as strings (bignumber-safe). Timestamps are unix seconds (string-encoded).

## USD Conversion

USD values shown on the dashboard (Total Volume, escrow display, fees) are computed at read time using the **current market value** of each token.

- **Storage:** raw token amounts only — `metricData.volume.{usdc,eth}` and `metricData.escrowBalance.{usdc,eth}`. No USD value is persisted on-chain or in the subgraph.
- **Read path:** fetch the current market price for each held token from a third-party price service, multiply by the stored token amount, and sum across tokens to produce the displayed USD figure.
- **Caching:** cache the price globally and reuse it across every metric in a render pass. Refresh on whatever cadence the price source justifies.
- **Frontend rule:** the conversion happens once per render against the cached price; the frontend does not issue a price call per metric.

Total Volume and Total Escrow Balance both follow this same read-time market-conversion flow.

## First Section Metrics

### Total Volume

Per-token cumulative volume stored under `metricData.volume.{usdc,eth}`. The dashboard shows a 30-day window plus a percentage change vs. the prior 30-day window.

- **Storage:** `metricData.volume[token] += amountPaid` (raw token amount)
- **Display:** the per-token totals are converted to USD per the chosen [USD Conversion](#usd-conversion) strategy, then summed.
- **Source events:** invoice `Paid`
- **Update on:** every successful payment

#### Windowed Volume & Percentage Change

The stored `volume[token]` is monotonically increasing. Windowed views and percentage change are computed on the client; the subgraph itself does not persist percentages. The historical anchors come from the subgraph's **Timeseries and Aggregations** — no third-party block-by-timestamp resolution and no time-travel queries.

Notation (computed per token, then summed in USD on the client):

- `V_now` — `volume[token]` at the current block.
- `V_30` — `volume[token]` as of ~30 days ago.
- `V_60` — `volume[token]` as of ~60 days ago.

Derived values:

- **30-day volume (displayed):** `V_now - V_30`
- **Prior 30-day volume:** `V_30 - V_60`
- **% change:** `((V_now - V_30) - (V_30 - V_60)) / (V_30 - V_60) * 100`

Fetching historical anchors via Timeseries and Aggregations:

The subgraph records each payment in a timeseries and rolls it up into a daily aggregation. graph-node closes each day's bucket on its own, so the anchors are reconstructed from `V_now` (the live cumulative) minus the aggregated daily deltas.

```graphql
# One immutable record per payment
type PaymentVolume @entity(timeseries: true) {
  id: Int8!
  timestamp: Timestamp!
  token: String!
  amount: BigDecimal!
}

# Daily volume per token, rolled up automatically by graph-node
type VolumeStats @aggregation(intervals: ["day"], source: "PaymentVolume") {
  id: Int8!
  timestamp: Timestamp!
  token: String!                                  # grouping dimension
  dailyVolume: BigDecimal! @aggregate(fn: "sum", arg: "amount")
}
```

The `Paid` handler writes one `PaymentVolume` point per payment (it still increments `metricData.volume[token]` for `V_now`).

1. `V_now` is read directly from `metricData.volume[token]`.
2. Query the daily buckets for the last ~60 days:

   ```graphql
   {
     volumeStats(
       interval: day
       where: { token: "usdc", timestamp_gte: $sixtyDaysAgo }
       orderBy: timestamp
       orderDirection: desc
     ) {
       timestamp
       dailyVolume
     }
   }
   ```

3. Reconstruct the anchors by subtracting the aggregated deltas from `V_now`:
   - `V_30 = V_now − Σ dailyVolume` over the most recent 30 buckets.
   - `V_60 = V_now − Σ dailyVolume` over the most recent 60 buckets.
4. Feed `V_now`, `V_30`, `V_60` into the formulas above.

Closed daily buckets are immutable, so they can be cached indefinitely; only the current (open) bucket needs refreshing.

Guard against `V_30 - V_60 == 0` (no activity in the prior window) before computing the percentage on the client.

### Total Escrow Balance

Live escrow balance tracked **per token** in `metricData.escrowBalance` (e.g. `usdc`, `eth`).

- **Formula:**
  - On payment: `escrowBalance[token] += amount` (raw token amount)
  - On settlement: `escrowBalance[token] -= amount` (applies to `Refund`, `Disputed`, `Release`)
- **Source events:** invoice `Paid`, `Refund`, `Disputed`, `Release`
- **Amount source:** the amount is read from the invoice-level event payload, not from a contract balance call.
- **Increase/decrease rate:** computed using the same windowed pattern as [Windowed Volume & Percentage Change](#windowed-volume--percentage-change).
- **Display:** depends on the [USD Conversion](#usd-conversion) decision — either read a stored USD field or convert the per-token balances on the read path.

### Total Fees Paid

Cumulative protocol fees collected.

- **Formula:** `fee += (feePercentage * amountPaidUsd) / 100`
- **Source events:** invoice `Paid` (fee is computed against the USD-denominated amount).
- **Note:** `metricData` does not currently hold a dedicated `totalFeesPaid` field. Either add one when implementing, or derive fees as `feePercentage * displayedVolumeUsd` on the read path.

### Total Invoices Paid

Count of currently-paid, unsettled invoices, stored in `metricData.paidInvoice`.

- **Formula:**
  - On `Paid`: `paidInvoice += 1`
  - On `Refund`, `Disputed`, `Release`: `paidInvoice -= 1`
- **Source events:** invoice `Paid`, `Refund`, `Disputed`, `Release`
- **Timestamp:** update `metricData.lastPaidInvoiceTimestamp` on every mutation.

## Invoice Activity

Daily transaction count split between the **simple** and **advanced** payment processor contracts, stored in `metricData.transactionCount`.

- **Storage:** the subgraph keeps a single counter pair (`simple`, `advanced`) along with `lastUpdateTimestamp`.
- **Daily reset (subgraph):** on every update, if `now - lastUpdateTimestamp > 24h`, reset the relevant counter to 1 (the current event) instead of incrementing.
- **Daily reset (frontend):** mirror the same 24h-cutoff check — if the cached counter is older than 24 hours with no new event, display zero.
- **Historical days:** back the chart with a `TxCount` timeseries (one record per transaction, `processor` as a dimension) and a daily `count` aggregation. The ~7-day chart is then a single ranged query over daily buckets — no block anchors.
- **Caching:** closed daily buckets are immutable, so a past day can be cached indefinitely; only the current (open) bucket needs refreshing.

## Recent Transactions

A short, ordered list of the most recent transactions across the two processor subgraphs.

- **Schema:** array of `{ transactionHash, timestamp, amount }`.
- **Sign convention:** `amount` is positive for `Paid`, negative for `Release`.
- **Source:** the simple and advanced subgraphs; merge by timestamp and keep the most recent 5.
- **`transactionHash`:** consumed by the frontend to build the block-explorer link.

## User Metrics

Stored as a global `newUsers` counter; computed views are derived using the same daily-aggregation windows as Total Volume.

- **New Creators / New Payers:** if a creator or payer address is not already present in the subgraph record, increment the counter and emit a `NewUser` timeseries point (with a `role` dimension). The dashboard surfaces the count _added in the last 7 days_ (raw count, not percentage) from the daily `count` aggregation and compares it against the prior 7-day window — no block-by-timestamp anchors.
- **Active Users:** count of unique users observed in the last 24 hours. The counter resets every 24 hours. Day-over-day percentage growth is computed from the same daily-aggregation windows as [Total Volume](#windowed-volume--percentage-change).

## Gas Tracker

Tracked in `gasPaid` (`amount`, `transactionCount`, `lastTimeStamp`).

- **Total Gas Paid:** increment `amount` whenever a platform wallet sends a transaction; bump `transactionCount` for each.
- **Avg Gas Per Transaction:** `amount / transactionCount`, computed on the client.
- **Current Gas Price:** fetched from Etherscan.

## Wallet Balance

- **Fee Receiver:** query the subgraph for fees received per token (incremented on each fee-paid event) and convert per the [USD Conversion](#usd-conversion) decision made for Total Volume.
- **Gas Reserve:** RPC `eth_getBalance` against the platform wallet.

## Real-time Updates

The subgraph-derived flow above is poll-based and reflects committed state only. To surface live activity without waiting for the next poll, the dashboard subscribes to a **websocket** stream and applies incoming events optimistically on top of the last subgraph snapshot.

- The subgraph remains the source of truth for cumulative values and historical-window queries.
- Websocket deltas update only the live displayed values between polls.
- On reconnect or stale-session recovery, drop optimistic state and reseed from the subgraph.

## Notes for Implementation

- Update related metrics inside a single event-handler transaction so the entity stays internally consistent.
- Settlement events (`Refund`, `Disputed`, `Release`) must reverse `escrowBalance[token]` and `paidInvoice`. They do **not** reverse `volume[token]` — that is cumulative.
- For windowed/historical reads, query the daily aggregation buckets and sum the relevant windows on the client; closed daily buckets are immutable and safely cacheable, so only the current open bucket needs refreshing.
- The frontend should never recompute USD values from raw token amounts directly; it consumes whichever USD representation results from the chosen conversion strategy.
