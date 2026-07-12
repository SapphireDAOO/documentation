# AdvancedPaymentProcessor.sol

## Intermediated Payment Processor

The `AdvancedPaymentProcessor` contract supports creating, managing, and settling payments between sellers and buyers via the intermediated platform, with the following features:

* Meta invoice and sub-invoice
* Dispute resolution
* Invoice Cancellation
* Refunds

Contract Address: [0x3d07827e8a6ba46f37d129df8d99f4ee8aa5685f](https://sepolia.etherscan.io/address/0x3d07827e8a6ba46f37d129df8d99f4ee8aa5685f)

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/src/AdvancedPaymentProcessor.sol)

### State Variables

#### ppStorage

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public immutable ppStorage
```

#### oracle

OracleManager used to convert USD-denominated invoice prices into payment-token amounts. Not immutable — updatable via `setOracle`.

```solidity
IOracleManager public oracle
```

The invoice status codes and fee/decimal constants below are plain file-level constants imported from `constants/Advanced.sol`, not `public` members of the contract itself — there is no on-chain getter like `AdvancedPaymentProcessor.CREATED()`. 

#### CREATED

Invoice has been created but no payment has been made yet.

```solidity
uint8 constant CREATED = 1;
```

#### PAID

Invoice has been paid by the buyer.

```solidity
uint8 constant PAID = 2;
```

#### REFUNDED

Invoice has been refunded to the buyer.

```solidity
uint8 constant REFUNDED = 3;
```

#### CANCELED

Seller has canceled the invoice.

```solidity
uint8 constant CANCELED = 4;
```

#### DISPUTED

Buyer has raised a dispute.

```solidity
uint8 constant DISPUTED = 5;
```

#### DISPUTE\_RESOLVED

Dispute has been resolved in full favor of both parties.

```solidity
uint8 constant DISPUTE_RESOLVED = 6;
```

#### DISPUTE\_DISMISSED

Dispute has been dismissed without changes to payouts.

```solidity
uint8 constant DISPUTE_DISMISSED = 7;
```

#### DISPUTE\_SETTLED

Dispute has been settled with a split payout.

```solidity
uint8 constant DISPUTE_SETTLED = 8;
```

#### RELEASED

Payment has been released to the seller after acceptance or resolution.

```solidity
uint8 constant RELEASED = 9;
```

#### BASIS\_POINTS

Total basis points used for percentage calculations. 10\_000 = 100%.

```solidity
uint256 constant BASIS_POINTS = 10_000;
```

#### DEFAULT\_DECIMAL

Default number of decimals used for internal fixed-point arithmetic (e.g., 1e18 = 1.0). Also used as the ERC20 decimals fallback in `_getDecimals`.

```solidity
uint8 constant DEFAULT_DECIMAL = 18;
```

#### DEFAULT\_MINIMUM\_INVOICE\_PRICE

Minimum invoice price applied when none is explicitly set (1 USD in 8-decimal Chainlink format).

```solidity
uint256 constant DEFAULT_MINIMUM_INVOICE_PRICE = 1e8;
```

### Functions

#### constructor

Initializes the AdvancedPaymentProcessor contract with core configuration.

```solidity
constructor(address _paymentProcessorStorageAddress, address _oracle) ;
```

**Parameters**

|                Name               |    Type   |                          Description                          |
| :-------------------------------: | :-------: | :-----------------------------------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The address of the shared payment processor storage contract. |
|            `_oracle`              | `address` | The address of the OracleManager contract for price feeds.    |

#### createSingleInvoice

Creates a single invoice with the specified parameters and returns its unique hash.

Only callable by the intermediated platform.

```solidity
function createSingleInvoice(InvoiceCreationParam memory _param)
    external
    onlyMarketplace
    returns (uint216 invoiceId);
```

**Parameters**

|   Name   |          Type          |                   Description                  |
| :------: | :--------------------: | :--------------------------------------------: |
| `_param` | `InvoiceCreationParam` | The parameters required to create the invoice. |

**Returns**

|     Name    |    Type   |                 Description                 |
| :---------: | :-------: | :-----------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the newly created invoice. |

#### createMetaInvoice

Creates a meta-invoice composed of multiple sub-invoices for a buyer.

Only callable by the intermediated platform. Each sub-invoice is created using the provided parameters, and all are linked under a single meta-invoice key.

```solidity
function createMetaInvoice(InvoiceCreationParam[] memory _param)
    external
    onlyMarketplace
    returns (uint216 metaInvoiceId);
```

**Parameters**

|   Name   |           Type           |                       Description                       |
| :------: | :----------------------: | :-----------------------------------------------------: |
| `_param` | `InvoiceCreationParam[]` | An array of parameters used to create each sub-invoice. |

**Returns**

|       Name      |    Type   |                      Description                     |
| :-------------: | :-------: | :--------------------------------------------------: |
| `metaInvoiceId` | `uint216` | The keccak256 hash representing the meta-invoice ID. |

#### payInvoice

Pays a single invoice using native ETH or an approved ERC20 token.

Any caller other than the invoice's seller may pay — the payer becomes the invoice's `buyer` (there's no pre-existing buyer requirement; reverts with `BuyerCannotBeSeller` only if the caller is the seller). Use `address(0)` for native payments. Guarded by `nonReentrant`.

```solidity
function payInvoice(uint216 _invoiceId, address _paymentToken) external payable nonReentrant;
```

**Parameters**

|       Name      |    Type   |                          Description                          |
| :-------------: | :-------: | :-----------------------------------------------------------: |
|   `_invoiceId`  | `uint216` |               The ID of the invoice to be paid.               |
| `_paymentToken` | `address` | The token address used for payment (or zero address for ETH). |

#### payMetaInvoiceWithValue

Pays all sub-invoices in a meta-invoice using native ETH.

Caller must send exactly the oracle-converted total for the meta-invoice price. Any dust from per-sub-invoice integer rounding is refunded to the caller. Canceled sub-invoices are automatically excluded. Sub-invoices not in CREATED state are silently skipped. Guarded by `nonReentrant`.

```solidity
function payMetaInvoiceWithValue(uint216 _invoiceId) external payable nonReentrant;
```

**Parameters**

|      Name     |    Type   |             Description             |
| :-----------: | :-------: | :---------------------------------: |
| `_invoiceId`  | `uint216` | The meta-invoice ID to pay.         |

#### payMetaInvoice

Pays all sub-invoices in a meta invoice using native ETH or ERC20.

Canceled sub-invoices are automatically excluded. Sub-invoices not in CREATED state are silently skipped. Guarded by `nonReentrant`.

```solidity
function payMetaInvoice(uint216 _invoiceId, address _paymentToken) external nonReentrant;
```

**Parameters**

|       Name      |    Type   |                          Description                          |
| :-------------: | :-------: | :-----------------------------------------------------------: |
|   `_invoiceId`  | `uint216` |                The meta invoice ID to be paid.                |
| `_paymentToken` | `address` | The token address used for payment. |

#### createDispute

Creates a dispute for an invoice.

Callable only by the intermediated platform. Only valid for invoices in the PAID state (reverts `InvalidInvoiceState` otherwise). Transitions the invoice to DISPUTED, blocking `release` until the dispute is resolved, dismissed, or settled. There is no automated release queue/heap in this contract — release only ever happens via an explicit intermediated-platform call.

```solidity
function createDispute(uint216 _invoiceId) external onlyMarketplace;
```

**Parameters**

|     Name     |    Type   |            Description            |
| :----------: | :-------: | :-------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to dispute. |

#### handleDispute

handle a dispute on a given invoice.

Callable only by the intermediated platform. Must be called after a dispute is created. The resolution can be DISPUTE\_DISMISSED, or DISPUTE\_SETTLED. If settled, the seller and buyer receive a split of the funds based on sellerShare.

```solidity
function handleDispute(uint216 _invoiceId, uint8 _resolution, uint256 _sellerShare) external onlyMarketplace;
```

**Parameters**

|      Name      |    Type   |                                   Description                                   |
| :------------: | :-------: | :-----------------------------------------------------------------------------: |
|  `_invoiceId`  | `uint216` |                              The ID of the invoice.                             |
|  `_resolution` |  `uint8`  |     The resolution state (must be one of the defined DISPUTE\_\* constants).    |
| `_sellerShare` | `uint256` | The portion of the invoice price (in basis points) to be awarded to the seller. |

#### release

Releases escrowed funds to the seller after the release window has passed.

Callable only by the intermediated platform. Valid for invoices in the PAID, DISPUTE\_RESOLVED, or DISPUTE\_DISMISSED state once `releaseAt` has been reached (reverts `InvalidInvoiceState` otherwise). Platform fees are deducted before the net amount is transferred to the seller. The invoice transitions to RELEASED and its balance is zeroed. There is no heap — this is always a direct, manually-triggered release. If the fee transfer itself fails, it does not revert the release; a `TransferFailed` event is emitted instead.

```solidity
function release(uint216 _invoiceId) external onlyMarketplace;
```

**Parameters**

|     Name     |    Type   |       Description      |
| :----------: | :-------: | :--------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice. |

#### refund

Issues a partial or full refund for a paid invoice. Callable only by the intermediated platform; invoice must be in the PAID state. `_refundShare` must be between 1 and 10,000 basis points. A full refund (10,000 BPS) transitions the invoice to REFUNDED; a partial refund reduces the escrow balance but leaves the invoice in PAID state so it can still be released later.

```solidity
function refund(uint216 _invoiceId, uint256 _refundShare) external onlyMarketplace;
```

**Parameters**

|      Name      |    Type   |                                    Description                                    |
| :------------: | :-------: | :-------------------------------------------------------------------------------: |
|  `_invoiceId`  | `uint216` |                       The identifier of the order to refund.                      |
| `_refundShare` | `uint256` | The portion of the invoice price to refund, specified in basis points (1% = 100). |

#### cancelInvoice

Cancels a single invoice before payment.

Callable only by the intermediated platform. If the invoice belongs to a meta-invoice, the meta-invoice total price is reduced accordingly. Only valid for invoices in the CREATED state.

```solidity
function cancelInvoice(uint216 _invoiceId) public onlyMarketplace;
```

**Parameters**

|     Name     |    Type   |            Description           |
| :----------: | :-------: | :------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to cancel. |

#### resolveDispute

Finalizes a dispute and marks the invoice as resolved.

Callable only by the intermediated platform after a dispute has been raised by the buyer. This function is used when both parties (buyer and seller) have come to an agreement without requiring arbitration, or when the dispute period has expired with no further action. Transitions the invoice state from DISPUTED to DISPUTE\_RESOLVED.

```solidity
function resolveDispute(uint216 _invoiceId) external onlyMarketplace;
```

**Parameters**

|     Name     |    Type   |                   Description                  |
| :----------: | :-------: | :--------------------------------------------: |
| `_invoiceId` | `uint216` | The unique identifier of the disputed invoice. |

#### setInvoiceReleaseTime

Sets a custom release time for a given invoice by adding a hold period to the current timestamp.

Callable only by the owner. Valid for invoices in the PAID, DISPUTE\_RESOLVED, or DISPUTE\_DISMISSED state.

```solidity
function setInvoiceReleaseTime(uint216 _invoiceId, uint256 _holdPeriod) external onlyOwner;
```

**Parameters**

|      Name     |    Type   |                              Description                             |
| :-----------: | :-------: | :------------------------------------------------------------------: |
|  `_invoiceId` | `uint216` |                   The ID of the invoice to update.                   |
| `_holdPeriod` | `uint256` | Additional hold period (in seconds) to add to the current timestamp. |

#### setOracle

Updates the OracleManager contract used for token price conversions.

Only callable by the owner. Reverts with `InvalidOracle` if `_oracle` is the zero address.

```solidity
function setOracle(address _oracle) external onlyOwner;
```

**Parameters**

|   Name    |   Type    |                       Description                      |
| :--------: | :-------: | :--------------------------------------------------------: |
| `_oracle` | `address` | The address of the new OracleManager contract. |

#### setMinimumPrice

Sets the minimum USD price an invoice must have to be created.

```solidity
function setMinimumPrice(uint256 _newMinimumPrice) external onlyOwner;
```

**Parameters**

|         Name         |    Type   |                                      Description                                     |
| :------------------: | :-------: | :----------------------------------------------------------------------------------: |
| `_newMinimumPrice`   | `uint256` | The new minimum price threshold (8 decimals, same unit as invoice prices). |

#### getMinimumPrice

Returns the minimum USD price an invoice must meet to be created.

```solidity
function getMinimumPrice() external view returns (uint256 minimumPrice);
```

**Returns**

|       Name      |    Type   |                         Description                        |
| :-------------: | :-------: | :--------------------------------------------------------: |
| `minimumPrice`  | `uint256` | The current minimum price threshold (8 decimals). |

#### getTokenValueFromUsd

Converts a USD-denominated price to the equivalent amount in the specified payment token.

```solidity
function getTokenValueFromUsd(address _paymentToken, uint256 _usdAmount) public view returns (uint256 tokenValue);
```

**Parameters**

|       Name      |    Type   |                                Description                               |
| :-------------: | :-------: | :----------------------------------------------------------------------: |
| `_paymentToken` | `address` |  The address of the payment token (use address(0) for the native token). |
|   `_usdAmount`  | `uint256` | The USD amount to convert, expressed in 8 decimals (e.g., 100e8 = $100). |

**Returns**

|     Name     |    Type   |             Description             |
| :----------: | :-------: | :---------------------------------: |
| `tokenValue` | `uint256` | The converted payment token amount. |

#### getInvoice

Retrieves the invoice data for a specific invoice ID.

```solidity
function getInvoice(uint216 _invoiceId) external view returns (Invoice memory i);
```

**Parameters**

|     Name     |    Type   |       Description      |
| :----------: | :-------: | :--------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice. |

**Returns**

|  Name   |    Type   |    Description    |
| :-----: | :-------: | :---------------: |
|   `i`   | `Invoice` | The invoice data. |

#### getMetaInvoice

Retrieves the meta-invoice data for a specific meta-invoice ID.

```solidity
function getMetaInvoice(uint216 _metaInvoiceId) external view returns (MetaInvoice memory m);
```

**Parameters**

|       Name       |    Type   |         Description         |
| :--------------: | :-------: | :-------------------------: |
| `_metaInvoiceId` | `uint216` | The ID of the meta-invoice. |

**Returns**

|   Name  |      Type     |       Description      |
| :-----: | :-----------: | :--------------------: |
|   `m`   | `MetaInvoice` | The meta-invoice data. |

#### totalUniqueInvoiceCreated

Returns the total number of unique invoices created.

```solidity
function totalUniqueInvoiceCreated() external view returns (uint216 totalInvoices);
```

**Returns**

|       Name      |    Type   |                  Description                 |
| :-------------: | :-------: | :------------------------------------------: |
| `totalInvoices` | `uint216` | The total number of unique invoices created. |

#### totalMetaInvoiceCreated

Returns the total number of meta-invoices created.

```solidity
function totalMetaInvoiceCreated() external view returns (uint216 totalMetaInvoices);
```

**Returns**

|         Name        |    Type   |                 Description                |
| :-----------------: | :-------: | :----------------------------------------: |
| `totalMetaInvoices` | `uint216` | The total number of meta-invoices created. |

#### getNextInvoiceNonce

Returns the nonce that will be assigned to the next invoice.

```solidity
function getNextInvoiceNonce() external view returns (uint216 nextInvoiceNonce);
```

**Returns**

|        Name        |    Type   |          Description          |
| :----------------: | :-------: | :---------------------------: |
| `nextInvoiceNonce` | `uint216` | The next invoice nonce value. |

#### getNextMetaInvoiceNonce

Returns the nonce that will be assigned to the next meta-invoice.

```solidity
function getNextMetaInvoiceNonce() external view returns (uint216 nextMetaInvoiceNonce);
```

**Returns**

|           Name          |    Type   |             Description            |
| :---------------------: | :-------: | :--------------------------------: |
| `nextMetaInvoiceNonce`  | `uint216` | The next meta-invoice nonce value. |

#### \_getDecimals

Returns the decimal precision of an ERC20 token by calling its `decimals()` function. Declared `public`, so it's part of the contract's ABI despite the underscore-prefixed name. Falls back to `DEFAULT_DECIMAL` (18) if the call fails or the token doesn't implement `decimals()`.

```solidity
function _getDecimals(address _token) public view returns (uint8 tokenDecimals);
```

**Parameters**

|   Name   |   Type    |                Description                |
| :-------: | :-------: | :-------------------------------------------: |
| `_token` | `address` | The address of the ERC20 token. |

**Returns**

|        Name       |   Type  |               Description              |
| :------------------: | :-----: | :----------------------------------------: |
| `tokenDecimals` | `uint8` | The number of decimals the token uses. |

### Structs

#### Invoice

Represents a single invoice created by a buyer to pay a seller, with escrow and payment tracking.

```solidity
struct Invoice {
    uint216 invoiceNonce;
    uint40 paidAt;
    uint40 createdAt;
    uint40 releaseAt;
    uint40 expiresAt;
    uint8 state;
    uint8 withdrawalRetries;
    uint32 escrowHoldPeriod;
    uint216 metaInvoiceId;
    address buyer;
    address seller;
    address escrow;
    address paymentToken;
    uint256 amountPaid;
    uint256 price;
    uint256 balance;
}
```

|         Field         |    Type   |                                                                Description                                                              |
| :----------------------: | :-------: | :-------------------------------------------------------------------------------------------------------------------------------------------: |
|     `invoiceNonce`     | `uint216` |                                  A unique identifier assigned to this invoice, typically sequentially.                                 |
|         `paidAt`       |  `uint40` |                                            Timestamp when the payment was made.                                          |
|       `createdAt`      |  `uint40` |                                            Timestamp when the invoice was created.                                       |
|       `releaseAt`      |  `uint40` |                                The timestamp when funds in escrow can be released to the seller.                         |
|       `expiresAt`      |  `uint40` |                                     The timestamp after which the invoice is no longer payable.                          |
|         `state`        |  `uint8`  |                                                 Current state of the invoice.                                            |
|  `withdrawalRetries`   |  `uint8`  | Reserved retry counter retained for storage-layout compatibility; unused now that releases are manual. Packed with `state`. |
|   `escrowHoldPeriod`   | `uint32`  | Custom hold duration (in seconds) between payment and release, set at invoice creation. When non-zero, overrides the storage default. |
|     `metaInvoiceId`    | `uint216` |              Identifier linking the invoice to a meta invoice. 0 if not part of any meta invoice.                        |
|         `buyer`        | `address` |                                              Address of the buyer.                                                       |
|        `seller`        | `address` |                                              Address of the seller.                                                      |
|        `escrow`        | `address` |                              Address of the escrow contract holding the funds.                                           |
|      `paymentToken`    | `address` |                       Token used for payment. Address zero for native currency.                                         |
|      `amountPaid`      | `uint256` |            Total amount paid by the buyer for this invoice, in the payment token (native token if `paymentToken == address(0)`).       |
|         `price`        | `uint256` |                                    Invoice amount expressed in USD (8 decimals).                                         |
|        `balance`       | `uint256` |             Current balance of the escrow associated with the order, accounting for total amount paid minus refunds or releases.        |

#### MetaInvoice

Represents a collection of sub-invoices grouped into a single meta-invoice for batch payment and tracking.

```solidity
struct MetaInvoice {
    uint256 price;
    uint216[] subInvoiceIds;
}
```

|      Field       |    Type     |                        Description                       |
| :-----------------: | :---------: | :----------------------------------------------------------: |
|      `price`      |  `uint256`  |     Total price of all sub-invoices under this meta invoice. |
| `subInvoiceIds`   | `uint216[]` |    List of sub-invoice IDs grouped under this meta invoice.   |

#### InvoiceCreationParam

Parameters used to create a new invoice or sub-invoice.

```solidity
struct InvoiceCreationParam {
    string invoiceId;
    address seller;
    uint256 price;
    uint32 escrowHoldPeriod;
}
```

|       Field        |    Type   |                                                    Description                                                   |
| :-------------------: | :-------: | :------------------------------------------------------------------------------------------------------------------: |
|     `invoiceId`     |  `string` |             A unique string identifier for the invoice, provided by the caller and hashed for use in the contract.   |
|       `seller`      | `address` |                                          Address of the seller.                                          |
|       `price`       | `uint256` |                     Price or amount to be paid for the invoice in USD (8 decimals).                     |
| `escrowHoldPeriod`  |  `uint32` |            Duration (in seconds) that the escrow will lock the payment before it's releasable.          |

### Events

#### InvoiceCreated

Emitted when a new invoice is created.

```solidity
event InvoiceCreated(uint216 indexed invoiceId, Invoice invoice);
```

| Name        | Type      | Description                        |
| :----------: | :-------: | :---------------------------------: |
| `invoiceId` | `uint216` | The ID of the newly created invoice. |
| `invoice`   | `Invoice` | The invoice data.                  |

#### MetaInvoiceCreated

Emitted when a meta-invoice is successfully created.

```solidity
event MetaInvoiceCreated(uint216 indexed metaInvoiceId, uint256 indexed totalPrice);
```

| Name            | Type      | Description                                                                        |
| :--------------: | :-------: | :---------------------------------------------------------------------------------: |
| `metaInvoiceId` | `uint216` | The unique identifier of the newly created meta-invoice.                           |
| `totalPrice`    | `uint256` | The aggregated total price (in USD, 8 decimals) of all sub-invoices. |

#### InvoicePaid

Emitted when an invoice is successfully paid and an escrow contract is created.

```solidity
event InvoicePaid(uint216 indexed invoiceId, address paymentToken, address escrowAddress, uint256 amount, uint40 releaseAt);
```

| Name            | Type      | Description                                                              |
| :--------------: | :-------: | :-----------------------------------------------------------------------: |
| `invoiceId`     | `uint216` | The unique identifier of the paid invoice.                               |
| `paymentToken`  | `address` | The address of the token used for payment (address(0) for native ETH).   |
| `escrowAddress` | `address` | The address of the escrow contract created to hold the payment.          |
| `amount`        | `uint256` | The amount paid, denominated in the token's smallest unit.               |
| `releaseAt`     | `uint40`  | The UNIX timestamp when the escrowed funds become releasable.            |

#### PaymentReleased

Emitted when escrowed funds for an invoice are released.

```solidity
event PaymentReleased(uint216 indexed invoiceId, address receiver, address currency, uint256 sellerAmount, uint256 fee);
```

| Name           | Type      | Description                                                        |
| :-------------: | :-------: | :-----------------------------------------------------------------: |
| `invoiceId`    | `uint216` | The unique identifier of the invoice.                              |
| `receiver`     | `address` | The address that receives the released funds (typically the seller). |
| `currency`     | `address` | The address of the token used for payment (address(0) for ETH).   |
| `sellerAmount` | `uint256` | The net amount transferred to the receiver, after fees.           |
| `fee`          | `uint256` | The platform fee deducted and sent to the fee receiver.           |

#### Refunded

Emitted when a refund is issued for a specific order.

```solidity
event Refunded(uint216 indexed invoiceId, uint256 indexed amount);
```

| Name        | Type      | Description                                    |
| :----------: | :-------: | :---------------------------------------------: |
| `invoiceId` | `uint216` | The unique identifier of the refunded order.   |
| `amount`    | `uint256` | The amount refunded to the buyer.              |

#### InvoiceCanceled

Emitted when an invoice is canceled before any payment has been made.

```solidity
event InvoiceCanceled(uint216 indexed invoiceId);
```

| Name        | Type      | Description                     |
| :----------: | :-------: | :------------------------------: |
| `invoiceId` | `uint216` | The ID of the canceled invoice. |

#### DisputeCreated

Emitted when a dispute is raised for an invoice by the buyer.

```solidity
event DisputeCreated(uint216 indexed invoiceId);
```

| Name        | Type      | Description                        |
| :----------: | :-------: | :---------------------------------: |
| `invoiceId` | `uint216` | The ID of the disputed invoice.    |

#### DisputeDismissed

Emitted when a dispute is dismissed and no party receives a refund or payout.

```solidity
event DisputeDismissed(uint216 indexed invoiceId);
```

| Name        | Type      | Description                                          |
| :----------: | :-------: | :---------------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the invoice involved in the dispute.       |

#### DisputeResolved

Emitted when a dispute is resolved in the seller's favor via `resolveDispute`.

```solidity
event DisputeResolved(uint216 indexed invoiceId);
```

| Name        | Type      | Description                                    |
| :----------: | :-------: | :---------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the invoice involved in the dispute. |

#### DisputeSettled

Emitted when a dispute is settled and the funds are split between buyer and seller.

```solidity
event DisputeSettled(uint216 indexed invoiceId, uint256 sellerAmount, uint256 buyerAmount, uint256 fee);
```

| Name           | Type      | Description                               |
| :-------------: | :-------: | :----------------------------------------: |
| `invoiceId`    | `uint216` | The ID of the invoice that was disputed.  |
| `sellerAmount` | `uint256` | The net amount transferred to the seller, after fees. |
| `buyerAmount`  | `uint256` | The amount refunded to the buyer.         |
| `fee`          | `uint256` | The platform fee deducted from the seller's share and sent to the fee receiver. |

#### UpdateReleaseTime

Emitted when the escrow release time is updated for a given invoice.

```solidity
event UpdateReleaseTime(uint216 indexed invoiceId, uint256 newHoldPeriod);
```

| Name            | Type      | Description                                                        |
| :--------------: | :-------: | :-----------------------------------------------------------------: |
| `invoiceId`     | `uint216` | The unique identifier of the invoice whose release time was modified. |
| `newHoldPeriod` | `uint256` | The updated escrow hold duration in seconds.                       |

#### LockedPaymentRecovered

Declared in `IAdvancedPaymentProcessor` but **never emitted** by this contract — there is no `releaseLocked`-equivalent function here (unlike `SimplePaymentProcessor`), so this event is currently unreachable dead ABI surface.

```solidity
event LockedPaymentRecovered(uint216 indexed invoiceId, address indexed recipient, uint256 amount);
```

| Name        | Type      | Description                                       |
| :----------: | :-------: | :------------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the locked invoice that was recovered.  |
| `recipient` | `address` | The address that received the recovered funds.    |
| `amount`    | `uint256` | The amount of funds recovered.                    |

#### TransferFailed

Emitted when a best-effort fee or payout transfer fails during release or dispute settlement. Does not revert the calling transaction — funds remain in escrow for later manual recovery.

```solidity
event TransferFailed(uint216 indexed invoiceId, address indexed recipient, uint256 amount);
```

| Name        | Type      | Description                                  |
| :----------: | :-------: | :-------------------------------------------: |
| `invoiceId` | `uint216` | The invoice whose fee transfer failed. |
| `recipient` | `address` | The intended recipient of the failed transfer. |
| `amount`    | `uint256` | The amount that could not be transferred. |

#### OracleUpdated

Emitted when the OracleManager contract is updated via `setOracle`.

```solidity
event OracleUpdated(address indexed previousOracle, address indexed newOracle);
```

| Name        | Type      | Description                                  |
| :----------: | :-------: | :-------------------------------------------: |
| `previousOracle` | `address` | The previously configured OracleManager address. |
| `newOracle`      | `address` | The newly configured OracleManager address. |

### Errors

| Error | Description |
| :----: | :----------: |
| `UnsupportedToken()` | Thrown when a payment is attempted with a token not supported by the processor. |
| `InvoiceExpired()` | Thrown when a payment is attempted on an invoice that has passed its expiry timestamp. |
| `StalePrice()` | Thrown when the Chainlink round is incomplete (`answeredInRound < roundId`). |
| `SequencerDown()` | Thrown when the L2 sequencer is down or still within the post-restart grace period. |
| `EmptyMetaInvoice()` | Thrown when a meta-invoice is created with an empty sub-invoice list. |
| `StalePriceFeed()` | Thrown when the Chainlink price feed is stale and cannot be trusted. |
| `InvalidPrice()` | Thrown when the Chainlink price feed returns a zero or negative answer. |
| `InsufficientBalance()` | Thrown when an account attempts to spend more than its available balance. |
| `PriceIsTooLow()` | Thrown when the provided price does not meet the required minimum threshold. |
| `NotAuthorized()` | Thrown when the caller lacks the required role or permission. |
| `InvoiceAlreadyExists()` | Thrown when trying to create an invoice that already exists. |
| `PriceCannotBeZero()` | Thrown when an attempt is made to create an invoice with a price of zero. |
| `BuyerCannotBeSeller()` | Thrown if the buyer and seller are the same address. |
| `InvalidInvoiceState()` | Thrown when the invoice is in a state that does not allow the attempted action. |
| `InvoiceDoesNotExist()` | Thrown when the invoice does not exist. |
| `InvalidNativePayment()` | Thrown when an invalid amount of native currency is sent with a payment. |
| `InvalidMetaInvoicePaymentAmount(uint256 sent, uint256 expected)` | Thrown when a meta-invoice native payment does not match the expected total. |
| `MetaInvoiceAlreadyExists()` | Thrown when a computed meta-invoice ID is already assigned in storage. |
| `InvalidDisputeResolution()` | Thrown when the dispute resolution type is invalid. |
| `InvalidSellersPayoutShare()` | Thrown when the seller's payout share exceeds the allowed limit (10000 BPS). |
| `InvalidSeller()` | Thrown when an invoice is created with the zero address as the seller. |
| `EscrowWithdrawFailed()` | Thrown when the escrow contract fails to execute a withdrawal. |
| `InvalidOracle()` | Thrown when attempting to set the oracle address to the zero address. |
