# API communication

## Sapphire Contract API – REST Endpoints

This API provides HTTP endpoints for interacting with the Sapphire DAO's `AdvancedPaymentProcessor` smart contract on the Ethereum Sepolia testnet at address `0x90F3F9816a637A8f30576Deecd6B09D825EB94C2`. It supports invoice creation, cancellation, refunding, dispute creation, dispute resolution, and fund release for secure, decentralized transactions in a marketplace.

**Base URL**: [https://sapphiredaotesting.com/](https://sapphiredaotesting.com/)

### Endpoints

* POST `/create`
* POST `/release`
* POST `/createDispute`
* POST `/handleDispute`
* POST `/cancel`
* POST `/refund`

***

#### Endpoint: `/create`

* **Method**: POST
* **Description**: Creates one or more on-chain invoices using the `AdvancedPaymentProcessor` contract's `createSingleInvoice` (for one invoice) or `createMetaInvoice` (for multiple invoices) functions. Invoices are stored in the contract's `invoice` or `metaInvoice` mappings with unique IDs.

**Request Body**

```json
[
  {
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "seller": "0x0f447989b14A3f0bbf08808020Ec1a6DE0b8cbC4",
    "price": 100,
    "escrowHoldPeriod": 604800,
    "currency": "USD"
  }
]
```

**Field Details**

|        Field       |   Type  | Required |                                                               Description                                                              |
| :----------------: | :-----: | :------: | :------------------------------------------------------------------------------------------------------------------------------------: |
|      `orderId`     |  string |     ✅    |                               Unique client-side identifier for the invoice (e.g., a UUID or any string).                              |
|      `seller`      |  string |     ✅    |                                          Ethereum address of the seller (e.g., `0xabc123...`).                                         |
|       `price`      | integer |     ✅    |                                                Invoice price in USD (e.g., 100 = $1.00).                                               |
| `escrowHoldPeriod` | integer |     ✅    |                              Duration in seconds for holding funds in escrow (e.g., `"604800"` = 7 days).                              |
|     `currency`     |  string |     ✅    | Type of fiat currency used for pricing the invoice. Currently supports **USD** as the base unit for price calculations and conversions |

**Response**

**Success (200)**:

```json
{
  "url": "https://sapphire-dao-website-six.vercel.app/checkout/<token>",
  "metaInvoiceId": "59808737901387817475691215581034097896123425895641016234844280889",
  "orders": {
    "550e8400-e29b-41d4-a716-446655440000": {
      "seller": "0x329C3E1bEa46Abc22F307eE30Cbb522B82Fe7082",
      "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
    },
    "6ba7b810-9dad-11d1-80b4-00c04fd430c8": {
      "seller": "0x60D7dD3b4248D53Abba8DA999B22023656A2E4B3",
      "orderId": "59808737901387817475691215581034097896123425895641016234844280890"
    }
  }
}
```

**Notes**:

* When there is only a single invoice, the `metaInvoiceId` field returns an empty string
* The `price` is converted to token amounts using Chainlink price feeds via the contract’s `getTokenValueFromUsd` function.
* A single invoice triggers `createSingleInvoice`, emitting an `InvoiceCreated` event. Multiple invoices trigger `createMetaInvoice`, emitting a `MetaInvoiceCreated` event.
* Only the `marketplace` address (retrieved via `PaymentProcessorStorage.GetMarketplaceAddress`) can call these functions.
* The `orderId` in the request (client-provided `requestId`) is hashed to a `uint216` using `utils.OrderIDToUint216` for on-chain storage, resulting in a numeric string (e.g., `"59808737901387817475691215581034097896123425895641016234844280889"`).
* The response uses `requestId` for the client-provided ID and `orderId` for the generated on-chain ID.

**Error Responses**:

**Error (400)**:

```json
{
  "error": "invalid request body",
  "reason": "<decoding error message>"
}
```

* Returned for malformed JSON or incorrect field types.

**Error (400)**:

```json
{
  "error": "no invoice parameters provided",
  "reason": "invoice array is empty"
}
```

* Returned if the invoice array is empty.

**Error (400)**:

```json
{
  "error": "<validation error>",
  "reason": "<specific validation error, e.g., missing required field>"
}
```

* Returned if invoice parameters fail validation (e.g., invalid seller address, missing `escrowHoldPeriod`).

**Error (500)**:

```json
{
  "error": "error fetching marketplace address",
  "reason": "<blockchain error message>"
}
```

* Returned if fetching the marketplace address fails.

**Error (500)**:

```json
{
  "error": "error creating invoice",
  "reason": "<blockchain error message, e.g., The price cannot be zero>"
}
```

* Returned if the blockchain transaction fails (e.g., `InvoiceAlreadyExists`, `PriceCannotBeZero`, `NotAuthorized`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/https://sapphiredaotesting.com/create \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '[
  {
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "seller": "0x0f447989b14A3f0bbf08808020Ec1a6DE0b8cbC4",
    "price": "8680000000",
    "escrowHoldPeriod": "604800",
    "currency": "USD"
  }
]'
```

***

#### Endpoint: `/release`

* **Method**: POST
* **Description**: Releases escrow funds for a specific invoice using the `AdvancedPaymentProcessor` contract’s `release` function, distributing funds to the seller and platform.

**Request Body**

```json
{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
}
```

**Field Details**

|   Field   |  Type  | Required |                   Description                   |
| :-------: | :----: | :------: | :---------------------------------------------: |
| `orderId` | string |     ✅    | On-chain order ID (e.g., `598087379013878...`). |

**Response**:

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.etherscan.io/tx/0x123456..."
}
```

**Error (400)**:

```json
{
  "error": "invalid request body",
  "reason": "<decoding error message>"
}
```

* Returned for malformed JSON or incorrect field types.

**Error (500)**:

```json
{
  "error": "Error sending transaction",
  "reason": "<blockchain error message, e.g., The invoice is not in a valid state for this action>"
}
```

* Returned if the blockchain transaction fails (e.g., `InvalidInvoiceState`, `NotAuthorized`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/release \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
}'
```

***

#### Endpoint: `/createDispute`

* **Method**: POST
* **Description**: Initiates a dispute for a specific invoice using the `AdvancedPaymentProcessor` contract’s `createDispute` function, setting the invoice state to `DISPUTED`.

**Request Body**

```json
{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
}
```

**Field Details**

|   Field   |  Type  | Required |                   Description                   |
| :-------: | :----: | :------: | :---------------------------------------------: |
| `orderId` | string |     ✅    | On-chain order ID (e.g., `598087379013878...`). |

**Response**:

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.etherscan.io/tx/0x123456..."
}
```

**Error (400)**:

```json
{
  "error": "invalid request body",
  "reason": "<decoding error message>"
}
```

* Returned for malformed JSON or incorrect field types.

**Error (500)**:

```json
{
  "error": "error fetching marketplace address",
  "reason": "<blockchain error message>"
}
```

* Returned if fetching the marketplace address fails.

**Error (500)**:

```json
{
  "error": "Error sending transaction",
  "reason": "<blockchain error message, e.g., The invoice is not in a valid state for this action>"
}
```

* Returned if the blockchain transaction fails (e.g., `InvalidInvoiceState`, `NotAuthorized`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/createDispute \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
}'
```

***

#### Endpoint: `/handleDispute`

* **Method**: POST
* **Description**: Resolves a dispute for a specific invoice using the `AdvancedPaymentProcessor` contract’s `resolveDispute` or `handleDispute` functions, updating the invoice state and distributing funds if applicable.

**Request Body**

```json
{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889",
  "resolution": 2,
  "sellerShare": "9000"
}
```

**Field Details**

|     Field     |   Type  |                  Required                  |                                Description                               |
| :-----------: | :-----: | :----------------------------------------: | :----------------------------------------------------------------------: |
|   `orderId`   |  string |                      ✅                     |              On-chain order ID (e.g., `598087379013878...`).             |
|  `resolution` | integer |                      ✅                     |   Enum value specifying the action type (see MarketplaceAction below).   |
| `sellerShare` |  string | ❌ Only if `resolution = 2` (SettleDispute) | Seller’s share in basis points (e.g., `"10000"` = 100%, `"9000"` = 90%). |

**MarketplaceAction Enum (`resolution`)**

| Value |      Name      |                                               Description                                              |             Contract Function            |
| :---: | :------------: | :----------------------------------------------------------------------------------------------------: | :--------------------------------------: |
|  `1`  | ResolveDispute | Both buyer and seller agree to dismiss the dispute, with escrow allocation unchanged (fully to seller) |             `resolveDispute`             |
|  `2`  |  SettleDispute |      Dispute resolved by an arbitrator, with `sellerShare` to the seller and the rest to the buyer     |  `handleDispute` with `DISPUTE_SETTLED`  |
|  `3`  | DismissDispute |         Arbitrator dismisses the dispute, leaving escrow allocation unchanged (fully to seller)        | `handleDispute` with `DISPUTE_DISMISSED` |

**Notes**:

* The contract validates that the invoice is in the `DISPUTED` state and that `sellerShare` does not exceed 10,000 basis points.
* For `SettleDispute`, funds are distributed with platform fees applied, emitting `DisputeSettled`.
* For `DismissDispute`, emits `DisputeDismissed` without fund distribution.
* For `ResolveDispute`, emits `DisputeResolved` and sets the state to `DISPUTE_RESOLVED`.

**Response**:

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.etherscan.io/tx/0x123456..."
}
```

**Error (400)**:

```json
{
  "error": "invalid request body",
  "reason": "<decoding error message>"
}
```

* Returned for malformed JSON or incorrect field types.

**Error (500)**:

```json
{
  "error": "Error sending transaction",
  "reason": "<blockchain error message, e.g., The provided dispute resolution is invalid>"
}
```

* Returned if the blockchain transaction fails (e.g., `InvalidDisputeResolution`, `InvalidInvoiceState`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/handleDispute \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889",
  "resolution": 2,
  "sellerShare": "9000"
}'
```

***

#### Endpoint: `/cancel`

* **Method**: POST
* **Description**: Cancels a specific invoice using the `AdvancedPaymentProcessor` contract’s `cancelInvoice` function, setting the invoice state to `CANCELED`. Can only be called before payment.

**Request Body**

```json
{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
}
```

**Field Details**

|   Field   |  Type  | Required |                   Description                   |
| :-------: | :----: | :------: | :---------------------------------------------: |
| `orderId` | string |     ✅    | On-chain order ID (e.g., `598087379013878...`). |

**Response**:

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.etherscan.io/tx/0x123456..."
}
```

**Error (400)**:

```json
{
  "error": "invalid request body",
  "reason": "<decoding error message>"
}
```

* Returned for malformed JSON or incorrect field types.

**Error (500)**:

```json
{
  "error": "Error sending transaction",
  "reason": "<blockchain error message, e.g., The invoice is not in a valid state for this action>"
}
```

* Returned if the blockchain transaction fails (e.g., `InvalidInvoiceState`, `NotAuthorized`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/cancel \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889"
}'
```

***

#### Endpoint: `/refund`

* **Method**: POST
* **Description**: Issues a refund for a specific invoice using the `AdvancedPaymentProcessor` contract’s `refund` function, withdrawing funds from escrow to the buyer.

**Request Body**

```json
{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889",
  "refundShare": "5000"
}
```

**Field Details**

|     Field     |  Type  | Required |                               Description                              |
| :-----------: | :----: | :------: | :--------------------------------------------------------------------: |
|   `orderId`   | string |     ✅    |             On-chain order ID (e.g., `598087379013878...`).            |
| `refundShare` | string |     ✅    | Refund share in basis points (e.g., `"10000"` = 100%, `"5000"` = 50%). |

**Response**:

**Success (200)**:

```json
{
  "status": "success",
  "transactionUrl": "https://sepolia.etherscan.io/tx/0x123456..."
}
```

**Error (400)**:

```json
{
  "error": "invalid request body",
  "reason": "<decoding error message>"
}
```

* Returned for malformed JSON or incorrect field types.

**Error (400)**:

```json
{
  "error": "invalid request body",
  "reason": "share can not be zero"
}
```

* Returned if `refundShare` is zero.

**Error (500)**:

```json
{
  "error": "Error sending transaction",
  "reason": "<blockchain error message, e.g., The account balance is insufficient to perform this action>"
}
```

* Returned if the blockchain transaction fails (e.g., `InsufficientBalance`, `InvalidInvoiceState`).

**Example**:

```bash
curl -X POST https://sapphiredaotesting.com/refund \
-H "Content-Type: application/json" \
-H "X-API-KEY: YOUR_API_KEY_HERE" \
-d '{
  "orderId": "59808737901387817475691215581034097896123425895641016234844280889",
  "refundShare": "5000"
}'
```

***

### Notes

* All endpoints require an `X-API-KEY` header for authentication, enforced by `AccessControlMiddleWare`.
* The `AdvancedPaymentProcessor` contract is deployed on the Ethereum Sepolia testnet and uses Solady libraries (`SafeTransferLib`, `SafeCastLib`, `FixedPointMathLib`) for secure token transfers, type casting, and fixed-point arithmetic.
* Invoice states are: `INITIATED` (1), `PAID` (2), `REFUNDED` (3), `CANCELED` (4), `DISPUTED` (5), `DISPUTE_RESOLVED` (6), `DISPUTE_DISMISSED` (7), `DISPUTE_SETTLED` (8), `RELEASED` (9).
* The contract uses Chainlink price feeds (`AggregatorV3Interface`) for USD-to-token conversions, supporting ETH and ERC20 tokens.
* The `marketplace` address, retrieved via `PaymentProcessorStorage.GetMarketplaceAddress`, controls privileged operations (`createSingleInvoice`, `createMetaInvoice`, `createDispute`).
* Transaction hashes are linked to `https://sepolia.etherscan.io/tx/`.
* The `IAdvancedPaymentProcessorInvoiceCreationParam` struct in Go ensures type safety for `orderId` (string), `seller` (Ethereum address), `price` (big.Int), and `escrowHoldPeriod` (big.Int).
* Blockchain errors are mapped to human-readable messages via `utils.RevertErrorDescriptions`, including:
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
