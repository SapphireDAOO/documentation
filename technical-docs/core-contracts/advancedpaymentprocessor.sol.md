# AdvancedPaymentProcessor.sol

## Advanced Payment Processor

The⁣`IAdvancedPaymentProcessor` is an interface designed to support creating, managing, and settling payments between sellers and buyers using the marketplace with the following features:

* Meta invoice and sub-invoice
* Dispute resolution
* Invoice Cancellation
* Refunds

Contract Address: [0x3d07827e8a6ba46f37d129df8d99f4ee8aa5685f](https://sepolia.etherscan.io/address/0x3d07827e8a6ba46f37d129df8d99f4ee8aa5685f)

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/AdvancedPaymentProcessor.sol)

### State Variables

#### ppStorage

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public ppStorage
```

#### CREATED

Invoice has been created but no payment has been made yet.

```solidity
uint8 public constant CREATED = 1
```

#### PAID

Invoice has been paid by the buyer.

```solidity
uint8 public constant PAID = CREATED + 1
```

#### REFUNDED

Invoice has been refunded to the buyer (e.g., after expiration or rejection).

```solidity
uint8 public constant REFUNDED = PAID + 1
```

#### CANCELED

Seller has canceled the invoice before acceptance.

```solidity
uint8 public constant CANCELED = REFUNDED + 1
```

#### DISPUTED

Buyer has raised a dispute after acceptance.

```solidity
uint8 public constant DISPUTED = CANCELED + 1
```

#### DISPUTE\_RESOLVED

Dispute has been resolved in full favor of both parties.

```solidity
uint8 public constant DISPUTE_RESOLVED = DISPUTED + 1
```

#### DISPUTE\_DISMISSED

Dispute has been dismissed without changes to payouts.

```solidity
uint8 public constant DISPUTE_DISMISSED = DISPUTE_RESOLVED + 1
```

#### DISPUTE\_SETTLED

Dispute has been settled with a split payout.

```solidity
uint8 public constant DISPUTE_SETTLED = DISPUTE_DISMISSED + 1
```

#### RELEASED

Payment has been released to the seller after acceptance or resolution.

```solidity
uint8 public constant RELEASED = DISPUTE_SETTLED + 1
```

#### BASIS\_POINTS

Total basis points used for percentage calculations. 10\_000 = 100%.

```solidity
uint256 public constant BASIS_POINTS = 10_000
```

#### DEFAULT\_DECIMAL

Default number of decimals used for internal fixed-point arithmetic (e.g., 1e18 = 1.0)

```solidity
uint8 public constant DEFAULT_DECIMAL = 18
```

#### STALE\_THRESHOLD

Maximum age allowed for Chainlink price data before it is considered stale.

```solidity
uint256 public constant STALE_THRESHOLD = 1 hours
```

### Functions

#### constructor

Initializes the AdvancedPaymentProcessor contract with core configuration.

```solidity
constructor(address _paymentProcessorStorageAddress) ;
```

**Parameters**

|                Name               |    Type   |                          Description                          |
| :-------------------------------: | :-------: | :-----------------------------------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The address of the shared payment processor storage contract. |

#### createSingleInvoice

Creates a single invoice with the specified parameters and returns its unique hash.

Only callable by the marketplace contract.

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

Only callable by the marketplace contract. Each sub-invoice is created using the provided parameters, and all are linked under a single meta-invoice key.

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

Caller must be the invoice buyer. Use `address(0)` for native payments.

```solidity
function payInvoice(uint216 _invoiceId, address _paymentToken) external payable;
```

**Parameters**

|       Name      |    Type   |                          Description                          |
| :-------------: | :-------: | :-----------------------------------------------------------: |
|   `_invoiceId`  | `uint216` |               The ID of the invoice to be paid.               |
| `_paymentToken` | `address` | The token address used for payment (or zero address for ETH). |

#### payMetaInvoiceWithValue

Pays all sub-invoices in a meta-invoice using native ETH.

Caller must send exactly the oracle-converted total for the meta-invoice price. Any dust from per-sub-invoice integer rounding is refunded to the caller. Canceled sub-invoices are automatically excluded. Sub-invoices not in CREATED state are silently skipped.

```solidity
function payMetaInvoiceWithValue(uint216 _invoiceId) external payable;
```

**Parameters**

|      Name     |    Type   |             Description             |
| :-----------: | :-------: | :---------------------------------: |
| `_invoiceId`  | `uint216` | The meta-invoice ID to pay.         |

#### payMetaInvoice

Pays all sub-invoices in a meta invoice using native ETH or ERC20.

Caller must be the buyer of all sub-invoices. Canceled sub-invoices are automatically excluded. Sub-invoices not in CREATED state are silently skipped.

```solidity
function payMetaInvoice(uint216 _invoiceId, address _paymentToken) external;
```

**Parameters**

|       Name      |    Type   |                          Description                          |
| :-------------: | :-------: | :-----------------------------------------------------------: |
|   `_invoiceId`  | `uint216` |                The meta invoice ID to be paid.                |
| `_paymentToken` | `address` | The token address used for payment. |

#### createDispute

Creates a dispute for an invoice.

Callable only by the marketplace. Only valid for invoices in the PAID state. Removes the invoice from the auto-release queue, canceling the pending release timer until the dispute is resolved or dismissed.

```solidity
function createDispute(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |            Description            |
| :----------: | :-------: | :-------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to dispute. |

#### handleDispute

handle a dispute on a given invoice.

Callable only by the marketplace. Must be called after a dispute is created. The resolution can be DISPUTE\_DISMISSED, or DISPUTE\_SETTLED. If settled, the seller and buyer receive a split of the funds based on sellerShare.

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

Callable only by the marketplace. Valid for invoices in the PAID, DISPUTE\_RESOLVED, or DISPUTE\_DISMISSED state once `releaseAt` has been reached. Platform fees are deducted before the net amount is transferred to the seller. The invoice transitions to RELEASED, its balance is zeroed, and it is removed from the auto-release heap.

```solidity
function release(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |       Description      |
| :----------: | :-------: | :--------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice. |

#### refund

Issues a refund for a given order.

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

Callable only by the marketplace. If the invoice belongs to a meta-invoice, the meta-invoice total price is reduced accordingly. Only valid for invoices in the CREATED state.

```solidity
function cancelInvoice(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |            Description           |
| :----------: | :-------: | :------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to cancel. |

#### resolveDispute

Finalizes a dispute and marks the invoice as resolved.

Callable only by the marketplace after a dispute has been raised by the buyer. This function is used when both parties (buyer and seller) have come to an agreement without requiring arbitration, or when the dispute period has expired with no further action. Transitions the invoice state from DISPUTED to DISPUTE\_RESOLVED.

```solidity
function resolveDispute(uint216 _invoiceId) external onlyMarketplace;
```

**Parameters**

|     Name     |    Type   |                   Description                  |
| :----------: | :-------: | :--------------------------------------------: |
| `_invoiceId` | `uint216` | The unique identifier of the disputed invoice. |

#### checkUpkeep

Checks if upkeep is needed.

```solidity
function checkUpkeep(bytes calldata) external view returns (bool upkeepNeeded, bytes memory performData);
```

**Parameters**

|   Name   |   Type  | Description |
| :------: | :-----: | :---------: |
| `<none>` | `bytes` |             |

**Returns**

|      Name      |   Type  |                          Description                         |
| :------------: | :-----: | :----------------------------------------------------------: |
| `upkeepNeeded` |  `bool` | Boolean indicating whether `performUpkeep` should be called. |
|  `performData` | `bytes` |     Data to pass to `performUpkeep` if upkeep is needed.     |

#### performUpkeep

Performs the actual upkeep work, such as executing a function or releasing funds.

```solidity
function performUpkeep(bytes calldata) external;
```

**Parameters**

|   Name   |   Type  | Description |
| :------: | :-----: | :---------: |
| `<none>` | `bytes` |             |

#### setPriceFeed

Sets the Chainlink price feed aggregator for a specific payment token.

```solidity
function setPriceFeed(address _token, address _aggregator) external;
```

**Parameters**

|      Name     |    Type   |                       Description                      |
| :-----------: | :-------: | :----------------------------------------------------: |
|    `_token`   | `address` |             The address of the ERC20 token.            |
| `_aggregator` | `address` | The address of the Chainlink aggregator for the token. |

#### setInvoiceReleaseTime

Sets a custom release time for a given invoice by adding a hold period to the current timestamp.

Callable only by the owner. Valid for invoices in the PAID, DISPUTE\_RESOLVED, or DISPUTE\_DISMISSED state (i.e., invoices currently tracked in the heap).

```solidity
function setInvoiceReleaseTime(uint216 _invoiceId, uint256 _holdPeriod) external;
```

**Parameters**

|      Name     |    Type   |                              Description                             |
| :-----------: | :-------: | :------------------------------------------------------------------: |
|  `_invoiceId` | `uint216` |                   The ID of the invoice to update.                   |
| `_holdPeriod` | `uint256` | Additional hold period (in seconds) to add to the current timestamp. |

#### setForwarderAddress

Updates the address of the forwarder contract used for relayed or automated calls.

```solidity
function setForwarderAddress(address _forwarderAddress) external;
```

**Parameters**

|         Name        |    Type   |                  Description                  |
| :-----------------: | :-------: | :-------------------------------------------: |
| `_forwarderAddress` | `address` | The new forwarder contract address to be set. |

#### setMinimumPrice

Sets the minimum USD price an invoice must have to be created.

```solidity
function setMinimumPrice(uint256 _newMinimumPrice) external;
```

**Parameters**

|         Name         |    Type   |                                      Description                                     |
| :------------------: | :-------: | :----------------------------------------------------------------------------------: |
| `_newMinimumPrice`   | `uint256` | The new minimum price threshold (8 decimals, same unit as invoice prices). |

#### getForwarder

Returns the address of the configured forwarder contract.

```solidity
function getForwarder() external view returns (address forwarderAddress);
```

**Returns**

|        Name        |    Type   |            Description            |
| :----------------: | :-------: | :-------------------------------: |
| `forwarderAddress` | `address` | The configured forwarder address. |

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
function getInvoice(uint216 _invoiceId) external view returns (Invoice memory invoiceData);
```

**Parameters**

|     Name     |    Type   |       Description      |
| :----------: | :-------: | :--------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice. |

**Returns**

|      Name     |    Type   |    Description    |
| :-----------: | :-------: | :---------------: |
| `invoiceData` | `Invoice` | The invoice data. |

#### getMetaInvoice

Retrieves the meta-invoice data for a specific meta-invoice ID.

```solidity
function getMetaInvoice(uint216 _metaInvoiceId) public view returns (MetaInvoice memory metaInvoiceData);
```

**Parameters**

|       Name       |    Type   |         Description         |
| :--------------: | :-------: | :-------------------------: |
| `_metaInvoiceId` | `uint216` | The ID of the meta-invoice. |

**Returns**

|        Name       |      Type     |       Description      |
| :---------------: | :-----------: | :--------------------: |
| `metaInvoiceData` | `MetaInvoice` | The meta-invoice data. |

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

#### getItems

Returns a list of all task IDs currently in the heap.

Retrieves the uint216 task identifiers extracted from the internal encoded heap structure.

```solidity
function getItems() external view returns (uint216[] memory items);
```

**Returns**

|   Name  |     Type    |     Description    |
| :-----: | :---------: | :----------------: |
| `items` | `uint216[]` | Array of task IDs. |

### Events

#### InvoiceCreated

Emitted when a new invoice is created.

```solidity
event InvoiceCreated(uint216 indexed invoiceId, Invoice invoice);
```

| Name        | Type      | Description                        |
| :---------- | :-------: | :--------------------------------- |
| `invoiceId` | `uint216` | The ID of the newly created invoice. |
| `invoice`   | `Invoice` | The invoice data.                  |

#### MetaInvoiceCreated

Emitted when a meta-invoice is successfully created.

```solidity
event MetaInvoiceCreated(uint216 indexed metaInvoiceId, uint256 indexed totalPrice);
```

| Name            | Type      | Description                                                                        |
| :-------------- | :-------: | :--------------------------------------------------------------------------------- |
| `metaInvoiceId` | `uint216` | The unique identifier of the newly created meta-invoice.                           |
| `totalPrice`    | `uint256` | The aggregated total price (in USD, 8 decimals) of all sub-invoices. |

#### InvoicePaid

Emitted when an invoice is successfully paid and an escrow contract is created.

```solidity
event InvoicePaid(uint216 indexed invoiceId, address paymentToken, address escrowAddress, uint256 amount, uint40 releaseAt);
```

| Name            | Type      | Description                                                              |
| :-------------- | :-------: | :----------------------------------------------------------------------- |
| `invoiceId`     | `uint216` | The unique identifier of the paid invoice.                               |
| `paymentToken`  | `address` | The address of the token used for payment (address(0) for native ETH).   |
| `escrowAddress` | `address` | The address of the escrow contract created to hold the payment.          |
| `amount`        | `uint256` | The amount paid, denominated in the token's smallest unit.               |
| `releaseAt`     | `uint40`  | The UNIX timestamp when the escrowed funds become releasable.            |

#### PaymentReleased

Emitted when escrowed funds for an invoice are released.

```solidity
event PaymentReleased(uint216 indexed invoiceId, address receiver, address currency, uint256 sellerAmount);
```

| Name           | Type      | Description                                                        |
| :------------- | :-------: | :----------------------------------------------------------------- |
| `invoiceId`    | `uint216` | The unique identifier of the invoice.                              |
| `receiver`     | `address` | The address that receives the released funds (typically the seller). |
| `currency`     | `address` | The address of the token used for payment (address(0) for ETH).   |
| `sellerAmount` | `uint256` | The amount transferred to the receiver.                            |

#### Refunded

Emitted when a refund is issued for a specific order.

```solidity
event Refunded(uint216 indexed invoiceId, uint256 indexed amount);
```

| Name        | Type      | Description                                    |
| :---------- | :-------: | :--------------------------------------------- |
| `invoiceId` | `uint216` | The unique identifier of the refunded order.   |
| `amount`    | `uint256` | The amount refunded to the buyer.              |

#### InvoiceCanceled

Emitted when an invoice is canceled before any payment has been made.

```solidity
event InvoiceCanceled(uint216 indexed invoiceId);
```

| Name        | Type      | Description                     |
| :---------- | :-------: | :------------------------------ |
| `invoiceId` | `uint216` | The ID of the canceled invoice. |

#### DisputeCreated

Emitted when a dispute is raised for an invoice by the buyer.

```solidity
event DisputeCreated(uint216 indexed invoiceId);
```

| Name        | Type      | Description                        |
| :---------- | :-------: | :--------------------------------- |
| `invoiceId` | `uint216` | The ID of the disputed invoice.    |

#### DisputeDismissed

Emitted when a dispute is dismissed and no party receives a refund or payout.

```solidity
event DisputeDismissed(uint216 indexed invoiceId);
```

| Name        | Type      | Description                                          |
| :---------- | :-------: | :--------------------------------------------------- |
| `invoiceId` | `uint216` | The ID of the invoice involved in the dispute.       |

#### DisputeResolved

Emitted when a dispute is resolved in the seller's favor via `resolveDispute`.

```solidity
event DisputeResolved(uint216 indexed invoiceId);
```

| Name        | Type      | Description                                    |
| :---------- | :-------: | :--------------------------------------------- |
| `invoiceId` | `uint216` | The ID of the invoice involved in the dispute. |

#### DisputeSettled

Emitted when a dispute is settled and the funds are split between buyer and seller.

```solidity
event DisputeSettled(uint216 indexed invoiceId, uint256 sellerAmount, uint256 buyerAmount);
```

| Name           | Type      | Description                               |
| :------------- | :-------: | :---------------------------------------- |
| `invoiceId`    | `uint216` | The ID of the invoice that was disputed.  |
| `sellerAmount` | `uint256` | The amount transferred to the seller.     |
| `buyerAmount`  | `uint256` | The amount refunded to the buyer.         |

#### UpdateReleaseTime

Emitted when the escrow release time is updated for a given invoice.

```solidity
event UpdateReleaseTime(uint216 indexed invoiceId, uint256 newHoldPeriod);
```

| Name            | Type      | Description                                                        |
| :-------------- | :-------: | :----------------------------------------------------------------- |
| `invoiceId`     | `uint216` | The unique identifier of the invoice whose release time was modified. |
| `newHoldPeriod` | `uint256` | The updated escrow hold duration in seconds.                       |

### Errors

| Error | Description |
| :---- | :---------- |
| `UnsupportedToken()` | Thrown when a payment is attempted with a token not supported by the processor. |
| `InvoiceExpired()` | Thrown when a payment is attempted on an invoice that has passed its expiry timestamp. |
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
