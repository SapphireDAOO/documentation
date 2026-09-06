# API communication

## Sapphire Contract API – REST Endpoints

This API provides HTTP endpoints for interacting with the Sapphire DAO's [IntermediatedPaymentProcessor](core-contracts/intermediatedpaymentprocessor.sol.md) and [SimplePaymentProcessor](core-contracts/simplepaymentprocessor.sol.md) smart contracts on **Base Sepolia**. It supports invoice creation, cancellation, refunding, dispute creation, dispute resolution, and fund release.

Contract addresses and endpoints are not compiled in; they come from a `config.yaml` holding one section per network (`local`, `testnet`, `mainnet`), selected by the `NETWORK` environment variable at startup.

**Base URL**: `https://sapphiredaotesting.com/`

### Endpoints

| Method | Path                                            | Description                          |
| ------ | ------------------------------------------------ | ------------------------------------- |
| GET    | `/`                                              | Health check                          |
| POST   | `/v1/invoices`                                   | Create one or more invoices           |
| GET    | `/v1/invoices/{invoiceId}`                       | Read invoice data from the subgraph   |
| POST   | `/v1/invoices/{invoiceId}/release`               | Release escrowed funds                |
| POST   | `/v1/invoices/{invoiceId}/cancel`                | Cancel an unpaid invoice              |
| POST   | `/v1/invoices/{invoiceId}/refund`                | Refund a paid invoice                 |
| POST   | `/v1/invoices/{invoiceId}/disputes`              | Open a dispute                        |
| POST   | `/v1/invoices/{invoiceId}/disputes/resolution`   | Resolve a dispute                     |
| GET    | `/v1/settlements/status`                         | Simple processor settlement window    |
| GET    | `/v1/exchangeRate`                               | How much of a token one USD buys      |

The invoice id is a path segment on every invoice operation, so the request body carries only what is specific to that operation. `release`, `cancel` and `disputes` take no body at all.

Every endpoint except `GET /` requires an `X-API-KEY` header, enforced by `AccessControlMiddleWare`.

***

#### Endpoint: `/`

* **Method**: GET
* **Description**: Health check. Returns `200` with the current time. Any other unrouted path returns `404`.

```json
{ "status": "ok", "time": "2026-09-05T10:48:12Z" }
```

***

#### Endpoint: `/v1/invoices`

* **Method**: POST
* **Description**: Creates one or more on-chain invoices using [createSingleInvoice](core-contracts/intermediatedpaymentprocessor.sol.md#createsingleinvoice) (for one invoice) or [createMetaInvoice](core-contracts/intermediatedpaymentprocessor.sol.md#createmetainvoice) (for multiple).

**Request Body**

```json
[
  {
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "seller": "0x0f447989b14A3f0bbf08808020Ec1a6DE0b8cbC4",
    "price": 8680000000,
    "escrowHoldPeriod": 604800,
    "currency": "USD",
    "paymentTokens": ["ETH", "USDC"]
  }
]
```

**Field Details**

| Field              | Type     | Required | Description                                                                            |
| ------------------ | -------- | -------- | --------------------------------------------------------------------------------------- |
| `orderId`          | string   | ✅        | Unique client-side identifier for the invoice (e.g., a UUID or any string).             |
| `seller`           | string   | ✅        | Ethereum address of the seller. Must not be the zero address.                           |
| `price`            | number   | ✅        | Invoice price in cents; scaled on the server using the `currency` precision.            |
| `escrowHoldPeriod` | number   | ✅        | Duration in seconds for holding funds in escrow (e.g., `604800` = 7 days).              |
| `currency`         | string   | ✅        | Pricing currency; sets the decimal precision applied to `price` (`USD` = 8 decimals).   |
| `paymentTokens`    | string\[] | ✅        | Token **symbols** the buyer may pay with, e.g. `["ETH", "USDC"]`. At least one.          |

**About `paymentTokens`**: callers name tokens by **symbol**, not address. Each symbol is resolved to the address deployed on the selected network using the config's `tokens` table, so the same request body works against local, testnet and mainnet. Matching is case-insensitive (`usdc` resolves `USDC`). `ETH` maps to the zero address, which is how the contracts denote the native token.

`createSingleInvoice`/`createMetaInvoice` take an `address[]` and revert with `NoPaymentTokens` on an empty list, so an empty or missing `paymentTokens` list is rejected with `400`. An unknown symbol is rejected with the list of configured ones:

```json
{ "error": "invoice 0: unknown payment token \"DOGE\" (known: ETH, USDC, wBTC)", "reason": "" }
```

**Response**

**Success (200)** (the same shape for one invoice or many):

```json
{
  "url": "https://sapphire-dao-website-six.vercel.app/checkout/?id=kWMRqBU-7H64tqp04CM5IfzKRMP1DbH2Ytg5",
  "orders": {
    "550e8400-e29b-41d4-a716-446655440000": {
      "seller": "0x329C3E1bEa46Abc22F307eE30Cbb522B82Fe7082",
      "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
    }
  }
}
```

**Notes**:

* The `url` is the network's checkout URL followed by the invoice id encoded as unpadded URL-safe base64 of its big-endian bytes, the same encoding the website uses, so it can decode the id straight from the link. For a single invoice that is the invoice's own id. For several it is the **meta-invoice** id, prefixed with `mt-` so the two kinds of identifier cannot be confused:

  ```
  single: .../checkout/?id=dJNOQid37cGYw04ceDk1kxjAME8ElIMK9yLm
  meta:   .../checkout/?id=mt-dJNOQid37cGYw04ceDk1kxjAME8ElIMK9yLm
  ```
* `price` is converted to token amounts using Chainlink price feeds via [getTokenValueFromUsd](core-contracts/intermediatedpaymentprocessor.sol.md#gettokenvaluefromusd).
* A single invoice triggers `createSingleInvoice`, emitting `InvoiceCreated`. Multiple invoices trigger `createMetaInvoice`, emitting `MetaInvoiceCreated`.
* Only the intermediated platform operator (retrieved via [getIntermediatedPlatformsOperator](core-contracts/paymentprocessorstorage.sol.md#getintermediatedplatformsoperator)) can call these functions.
* The client-provided `orderId` is hashed to a `uint216` for on-chain storage, producing the numeric `orderId` in the response. That numeric id is the `{invoiceId}` used by every other endpoint.

**Error Responses**:

**Error (400)** (malformed JSON, or failed validation such as an invalid seller address, a non-positive price, or a missing `paymentTokens` entry):

```json
{
  "error": "invoice 0: at least one payment token is required",
  "reason": "<validation error>"
}
```

**Error (500)** (fetching the operator address failed, or the transaction reverted):

```json
{
  "error": "error creating invoice",
  "reason": "An invoice with this identifier already exists."
}
```

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/v1/invoices \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '[
  {
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "seller": "0x0f447989b14A3f0bbf08808020Ec1a6DE0b8cbC4",
    "price": 8680000000,
    "escrowHoldPeriod": 604800,
    "currency": "USD",
    "paymentTokens": ["ETH", "USDC"]
  }
]'
```

***

#### Endpoint: `/v1/invoices/{invoiceId}`

* **Method**: GET
* **Description**: Reads invoice data from the subgraph rather than from the chain directly.

**Success (200)**: the invoice record as stored in the subgraph.

**Error (500)**:

```json
{
  "error": "failed to fetch invoice data",
  "reason": "<subgraph error message>"
}
```

**Example**:

```bash
curl https://sapphiredaotesting.com/v1/invoices/59808737901387817475691215581034097896123425895641016234844280889 \
-H "X-API-KEY: YOUR_API_KEY_HERE"
```

***

#### Endpoint: `/v1/invoices/{invoiceId}/release`

* **Method**: POST
* **Description**: Releases escrow funds for an invoice using [release](core-contracts/intermediatedpaymentprocessor.sol.md#release), distributing funds to the seller and platform. **No request body.**

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.basescan.org/tx/0x123456..."
}
```

**Error (400)**: `invoiceId` in the path is not a base-10 integer.

**Error (500)**:

```json
{
  "error": "Error sending transaction",
  "reason": "The invoice is not in a valid state for this action."
}
```

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/v1/invoices/59808737901387817475691215581034097896123425895641016234844280889/release \
-H "X-API-KEY: YOUR_API_KEY_HERE"
```

***

#### Endpoint: `/v1/invoices/{invoiceId}/cancel`

* **Method**: POST
* **Description**: Cancels an invoice using [cancelInvoice](core-contracts/intermediatedpaymentprocessor.sol.md#cancelinvoice), setting its state to `CANCELED`. Can only be called before payment. **No request body.**

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.basescan.org/tx/0x123456..."
}
```

**Error (500)**: transaction reverted (e.g., `InvalidInvoiceState`, `NotAuthorized`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/v1/invoices/59808737901387817475691215581034097896123425895641016234844280889/cancel \
-H "X-API-KEY: YOUR_API_KEY_HERE"
```

***

#### Endpoint: `/v1/invoices/{invoiceId}/refund`

* **Method**: POST
* **Description**: Issues a refund for an invoice using [refund](core-contracts/intermediatedpaymentprocessor.sol.md#refund), withdrawing funds from escrow to the buyer.

**Request Body**

```json
{ "refundShare": "5000" }
```

| Field         | Type   | Required | Description                                                             |
| ------------- | ------ | -------- | ------------------------------------------------------------------------ |
| `refundShare` | string | ✅        | Refund share in basis points (e.g., `"10000"` = 100%, `"5000"` = 50%).  |

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.basescan.org/tx/0x123456..."
}
```

**Error (400)**:

```json
{ "error": "refundShare is required", "reason": "" }
```

* Also returned when `refundShare` is zero (`"share can not be zero"`).

**Error (500)**: transaction reverted (e.g., `InsufficientBalance`, `InvalidInvoiceState`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/v1/invoices/59808737901387817475691215581034097896123425895641016234844280889/refund \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '{ "refundShare": "5000" }'
```

***

#### Endpoint: `/v1/invoices/{invoiceId}/disputes`

* **Method**: POST
* **Description**: Opens a dispute for an invoice using [createDispute](core-contracts/intermediatedpaymentprocessor.sol.md#createdispute), setting the invoice state to `DISPUTED`. **No request body.**

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.basescan.org/tx/0x123456..."
}
```

**Error (500)**: fetching the operator address failed, or the transaction reverted.

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/v1/invoices/59808737901387817475691215581034097896123425895641016234844280889/disputes \
-H "X-API-KEY: YOUR_API_KEY_HERE"
```

***

#### Endpoint: `/v1/invoices/{invoiceId}/disputes/resolution`

* **Method**: POST
* **Description**: Resolves an open dispute using [resolveDispute](core-contracts/intermediatedpaymentprocessor.sol.md#resolvedispute) or [handleDispute](core-contracts/intermediatedpaymentprocessor.sol.md#handledispute), updating the invoice state and distributing funds if applicable.

**Request Body**

```json
{
  "resolution": 2,
  "sellerShare": "9000"
}
```

| Field         | Type    | Required                                    | Description                                                               |
| ------------- | ------- | -------------------------------------------- | --------------------------------------------------------------------------- |
| `resolution`  | integer | ✅                                            | Enum value specifying the action type (see IntermediatedPlatformsOperatorAction below). |
| `sellerShare` | string  | ❌ Only if `resolution = 2` (SettleDispute)   | Seller's share in basis points (e.g., `"10000"` = 100%, `"9000"` = 90%).   |

**IntermediatedPlatformsOperatorAction Enum (`resolution`)**

| Value | Name           | Description                                                                                             | Contract Function                        |
| ----- | -------------- | --------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `1`   | ResolveDispute | Both buyer and seller agree to dismiss the dispute, with escrow allocation unchanged (fully to seller)   | `resolveDispute`                          |
| `2`   | SettleDispute  | Dispute resolved by an arbitrator, with `sellerShare` to the seller and the rest to the buyer            | `handleDispute` with `DISPUTE_SETTLED`    |
| `3`   | DismissDispute | Arbitrator dismisses the dispute, leaving escrow allocation unchanged (fully to seller)                  | `handleDispute` with `DISPUTE_DISMISSED`  |

**Notes**:

* The contract validates that the invoice is in the `DISPUTED` state and that `sellerShare` does not exceed 10,000 basis points.
* For `SettleDispute`, funds are distributed with platform fees applied, emitting `DisputeSettled`.
* For `DismissDispute`, emits `DisputeDismissed` without fund distribution.
* For `ResolveDispute`, emits `DisputeResolved` and sets the state to `DISPUTE_RESOLVED`.

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.basescan.org/tx/0x123456..."
}
```

**Error (500)**: transaction reverted (e.g., `InvalidDisputeResolution`, `InvalidInvoiceState`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/v1/invoices/59808737901387817475691215581034097896123425895641016234844280889/disputes/resolution \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '{ "resolution": 2, "sellerShare": "9000" }'
```

***

#### Endpoint: `/v1/settlements/status`

* **Method**: GET
* **Description**: Reports whether the [SimplePaymentProcessor](core-contracts/simplepaymentprocessor.sol.md) settlement window is still open.

* **200** with an empty body: the window is open.
* **400**: the window has passed.

```json
{ "error": "settlement time passed", "reason": "settlement window has expired" }
```

***

#### Endpoint: `/v1/exchangeRate`

* **Method**: GET `/v1/exchangeRate?From=USD&to=wBTC&to=ETH`
* **Description**: Reports how much of each requested token one USD buys, inverting [OracleManager.getUsdPerToken](core-contracts/oraclemanager.sol.md#getusdpertoken). The oracle address comes from the network's configured `contracts.oracleManager`.

| Query  | Required | Description                                                                                        |
| ------ | -------- | ---------------------------------------------------------------------------------------------------- |
| `from` | ❌        | Must be `USD`, the only currency the oracle prices against. Defaults to `USD`. `From` also works.   |
| `to`   | ✅        | Token symbols from the network's tokens table. Repeated, comma-separated, or bracketed.             |

`to` accepts `to=wBTC&to=ETH`, `to=ETH,wBTC`, and `to=[ETH, wBTC]`.

**Success (200)** (the rate is *from* USD, so each value is how much of that token one USD buys):

```json
{ "from": "USD", "to": { "ETH": 0.000510204081632653, "wBTC": 0.00001111 } }
```

* Values are JSON numbers carrying the full precision of the token's own decimals, so an 18-decimal token keeps all 18.
* A token the oracle cannot price falls back to a rate of `1`, treating it as a dollar-pegged token. The response does not distinguish that from a quoted rate.
* The conversion truncates, as any fixed-point division must: at $90,000 wBTC resolves to `0.00001111`, not a repeating `0.0000111111…`, because the token has 8 decimals.

**Error (400)** (an unknown symbol, a `from` other than `USD`, or a missing `to`):

```json
{ "error": "unknown token BTC (known: ETH, USDC, wBTC)", "reason": "unknown token BTC" }
```

**Error (502)**: the oracle call failed for a reason other than a missing feed (a stale price, or the sequencer being down). **503**: the network has no `contracts.oracleManager` configured, so rates are disabled.

**Example**:

```bash
curl "https://sapphiredaotesting.com/v1/exchangeRate?From=USD&to=ETH&to=wBTC" \
-H "X-API-KEY: YOUR_API_KEY_HERE"
```

***

### Configuration

Contract addresses, RPC endpoints, checkout/subgraph URLs, and the signer key are not compiled into the API; they live in a `config.yaml` with one section per network (`local`, `testnet`, `mainnet`). `NETWORK` selects which section is active and is required; there is no silent default. Only the selected section is validated, so a network that is not deployed yet cannot break startup.

* **Tokens**: each network defines a `tokens` table mapping a symbol to its address and decimals. This is the one place a payment token is defined: `paymentTokens` in a create request names a symbol from this table, and callback payloads render an event's token address back to its symbol and decimals through the same table.
* **Signer key**: `signerKey` selects which key signs transactions, per network (e.g. a local-only key on `local` versus the production signing key on the deployed networks), so a local run cannot touch a deployed network's key.
* **Checkout/explorer/subgraph URLs**: each network configures its own `urls.checkout`, `urls.explorer` and `urls.subgraph`, which is why the same request body against `local`, `testnet` or `mainnet` produces checkout links and transaction links for the right network.
* **Oracle**: `contracts.oracleManager` is optional; without it, `/v1/exchangeRate` responds `503`.

### Notes

* All endpoints except `GET /` require an `X-API-KEY` header, enforced by `AccessControlMiddleWare`.
* Invoice states are: `INITIATED` (1), `PAID` (2), `REFUNDED` (3), `CANCELED` (4), `DISPUTED` (5), `DISPUTE_RESOLVED` (6), `DISPUTE_DISMISSED` (7), `DISPUTE_SETTLED` (8), `RELEASED` (9).
* The contracts use Chainlink price feeds (`AggregatorV3Interface`) for USD-to-token conversions, supporting the native token and ERC20 tokens.
* The intermediated platform operator, retrieved via [getIntermediatedPlatformsOperator](core-contracts/paymentprocessorstorage.sol.md#getintermediatedplatformsoperator), controls privileged operations (`createSingleInvoice`, `createMetaInvoice`, `createDispute`).
* Transaction links are built from the network's configured `urls.explorer` (`https://sepolia.basescan.org` on Base Sepolia).
* The client-provided `orderId` is hashed to a `uint216` for on-chain storage, producing a numeric string (e.g., `"59808737901387817475691215581034097896123425895641016234844280889"`). That value is the `{invoiceId}` path segment.
* Blockchain reverts are mapped to human-readable messages, including:
  * `The buyer and seller addresses cannot be the same.`
  * `The account balance is insufficient to perform this action.`
  * `The provided dispute resolution is invalid.`
  * `The invoice is not in a valid state for this action.`
  * `The native token payment is invalid for this invoice.`
  * `The specified payment token is not supported or invalid.`
  * `The seller's payout share is invalid.`
  * `An invoice with this identifier already exists.`
  * `The specified invoice does not exist.`
  * `A meta-invoice with this identifier already exists.`
  * `The caller is not authorized to perform this action.`
  * `The price cannot be zero.`
  * `The price specified is too low.`
