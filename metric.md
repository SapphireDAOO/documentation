# Dashboard Metrics — Technical Spec

Aggregated metrics maintained in the subgraph for the dashboard. Historical/past-window views are served via block-by-timestamp lookups against time-travel queries on a small set of global entities.

## Schemas

The bulk of dashboard state is consolidated into a single `metricData` entity. Recent transactions, new-user count, and gas paid remain in their own entities.

### Consolidated metric entity

```ts
const metricData = {
  volume: {
    usdc: "",
    eth: "",
  },
  escrowBalance: {
    usdc: "",
    eth: "",
  },
  paidInvoice: "",
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

The stored `volume[token]` is monotonically increasing. Windowed views and percentage change are computed on the client from subgraph time-travel queries — the subgraph itself does not persist percentages.

Notation (computed per token, then summed in USD on the client):

- `V_now` — `volume[token]` at the current block.
- `V_30` — `volume[token]` at the block ~30 days ago.
- `V_60` — `volume[token]` at the block ~60 days ago.

Derived values:

- **30-day volume (displayed):** `V_now - V_30`
- **Prior 30-day volume:** `V_30 - V_60`
- **% change:** `((V_now - V_30) - (V_30 - V_60)) / (V_30 - V_60) * 100`

Fetching historical values:

1. Resolve "block at timestamp `T`" via a third-party service — e.g. Etherscan's block-by-timestamp endpoint. The subgraph cannot answer this on its own.
2. **Cache** the resolved block number with a TTL that runs to end-of-day. Within a single day every client converges on the same `V_30` / `V_60` anchors.
3. Re-query `metricData` at the cached block using the subgraph's `block: { number: ... }` argument.
4. Apply the same procedure for `V_30` and `V_60`.

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
- **Historical days:** use the block-by-timestamp lookup (same pattern as Total Volume) to fetch counts for prior days. The chart typically needs ~7 block anchors.
- **Caching:** persist resolved block numbers via CDN, **per window per day** — once a day's anchors are resolved, every client reuses them.

## Recent Transactions

A short, ordered list of the most recent transactions across the two processor subgraphs.

- **Schema:** array of `{ transactionHash, timestamp, amount }`.
- **Sign convention:** `amount` is positive for `Paid`, negative for `Release`.
- **Source:** the simple and advanced subgraphs; merge by timestamp and keep the most recent 5.
- **`transactionHash`:** consumed by the frontend to build the block-explorer link.

## User Metrics

Stored as a global `newUsers` counter; computed views are derived using the same windowed pattern as Total Volume.

- **New Creators / New Payers:** if a creator or payer address is not already present in the subgraph record, increment the counter. The dashboard surfaces the count _added in the last 7 days_ (raw count, not percentage) and compares it against the prior 7-day count using the [block-by-timestamp lookup](#windowed-volume--percentage-change).
- **Active Users:** count of unique users observed in the last 24 hours. The counter resets every 24 hours. Day-over-day percentage growth is computed using the same block-by-timestamp lookup pattern as Total Volume.

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
- For windowed/historical reads, share the block-by-timestamp cache across metrics — every metric that needs a 30-day-ago anchor reads from the same cached value.
- The frontend should never recompute USD values from raw token amounts directly; it consumes whichever USD representation results from the chosen conversion strategy.
