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

#### heap

Internal min-heap used to efficiently manage scheduled invoice tasks by release time.

```solidity
TaskQueueLib.Heap private heap
```

#### forwarder

Address of the forwarder contract responsible for calling performUpkeep.

```solidity
address private forwarder
```

#### ppStorage

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public ppStorage
```

#### nextMetaInvoiceNonce

The next available meta-invoice ID to be assigned.

```solidity
uint216 private nextMetaInvoiceNonce
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

#### invoices

Mapping from unique invoice ID to its invoice data.

Used for standalone invoices (not part of a meta-invoice).

```solidity
mapping(uint216 invoiceId => Invoice invoiceData) private invoices
```

#### metaInvoices

Mapping from meta-invoice ID to its aggregate meta-invoice data.

Stores metadata for grouped payments consisting of multiple sub-invoices. Each MetaInvoice contains the total price and all associated sub-invoice IDs.

```solidity
mapping(uint216 metaInvoiceId => MetaInvoice invoice) private metaInvoices
```

#### priceFeed

Mapping of payment tokens to their Chainlink price feed aggregator.

Used for converting USD prices to the appropriate payment token amounts.

```solidity
mapping(address token => address aggregator) private priceFeed
```

#### index

Maps task or invoice ID to its 1-based index position in the heap.

A value of 0 means the task is not present in the heap

```solidity
mapping(uint216 invoiceId => uint256 key) private index
```

### Functions

#### onlyMarketplace

Restricts function access to the authorized marketplace address.

Reverts with NotAuthorized() if the caller is not the marketplace.

```solidity
modifier onlyMarketplace() ;
```

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

#### payMetaInvoice

Pays all sub-invoices in a meta invoice using native ETH or ERC20.

Caller must be the buyer of all sub-invoices. Use `address(0)` for native payment.

```solidity
function payMetaInvoice(uint216 _invoiceId, address _paymentToken) external payable;
```

**Parameters**

|       Name      |    Type   |                          Description                          |
| :-------------: | :-------: | :-----------------------------------------------------------: |
|   `_invoiceId`  | `uint216` |                The meta invoice ID to be paid.                |
| `_paymentToken` | `address` | The token address used for payment (or zero address for ETH). |

#### createDispute

Creates a dispute for an invoice.

Callable only by the buyer of the invoice. Only valid for invoices in the ACCEPTED state and within the dispute window.

```solidity
function createDispute(uint216 _invoiceId) external onlyMarketplace;
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

Callable only by the marketplace. Only valid for invoices in the ACCEPTED state and only after `releaseAt` timestamp has been reached.

```solidity
function release(uint216 _invoiceId) external onlyMarketplace;
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

Cancels a single invoice.

Callable only by the seller of the invoice. Only valid for invoices in the PAID state.

```solidity
function cancelInvoice(uint216 _invoiceId) public onlyMarketplace;
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
| `upkeepNeeded` |  `bool` |     Data to pass to `performUpkeep` if upkeep is needed.     |
|  `performData` | `bytes` | Boolean indicating whether `performUpkeep` should be called. |

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

Callable only by the storage contract (via `execute()`), not directly by users or marketplace.

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

#### getForwarder

Returns the address of the configured forwarder contract.

```solidity
function getForwarder() external view returns (address forwarderAddress);
```

**Returns**

|        Name        |    Type   |            Description            |
| :----------------: | :-------: | :-------------------------------: |
| `forwarderAddress` | `address` | The configured forwarder address. |

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

#### \_isReleasable

Checks if an invoice state is eligible for release.

```solidity
function _isReleasable(Invoice memory _inv) internal view returns (bool isReleasable);
```

**Returns**

|      Name      |  Type  |              Description             |
| :------------: | :----: | :----------------------------------: |
| `isReleasable` | `bool` | True if the invoice can be released. |

#### \_release

Attempts to release the payment for a given invoice.

Performs validation checks before releasing. If successful, updates invoice state, removes it from the heap, processes seller payout, and emits a PaymentReleased event.

```solidity
function _release(uint216 _invoiceId) internal returns (uint256 status);
```

**Parameters**

|     Name     |    Type   |            Description            |
| :----------: | :-------: | :-------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to release. |

**Returns**

|   Name   |    Type   |                                  Description                                 |
| :------: | :-------: | :--------------------------------------------------------------------------: |
| `status` | `uint256` | The release status code (SUCCESSFUL, ERROR, or NOT\_ELIGIBLE\_FOR\_RELEASE). |

#### \_invoicePayment

Handles payment for an invoice, performs validation, and initializes escrow.

* Converts the USD price to payment token amount using Chainlink oracles.
* Validates invoice state, sender, and expiration.
* Creates an escrow contract and updates the invoice with payment info.
* Transfers tokens to escrow if the payment is in ERC20.

```solidity
function _invoicePayment(Invoice memory _inv, uint216 _invoiceId, address _paymentToken, uint256 _value) internal;
```

**Parameters**

|       Name      |    Type   |                               Description                               |
| :-------------: | :-------: | :---------------------------------------------------------------------: |
|      `_inv`     | `Invoice` |                         The invoice to be paid.                         |
|   `_invoiceId`  | `uint216` |                    The key of the invoice being paid.                   |
| `_paymentToken` | `address` | The address of the payment token (use address(0) for the native token). |
|     `_value`    | `uint256` |          The amount of native token sent with the transaction.          |

#### \_createInvoice

Creates a new invoice and stores it in contract state.

```solidity
function _createInvoice(uint216 _nonce, uint216 _metaInvoiceId, InvoiceCreationParam memory _param)
    internal
    returns (uint216 invoiceId);
```

**Parameters**

|       Name       |          Type          |                          Description                          |
| :--------------: | :--------------------: | :-----------------------------------------------------------: |
|     `_nonce`     |        `uint216`       |          The unique ID to assign to the new invoice.          |
| `_metaInvoiceId` |        `uint216`       | The associated meta-invoice ID, or 0 for standalone invoices. |
|     `_param`     | `InvoiceCreationParam` |         The parameters required to create the invoice.        |

**Returns**

|     Name    |    Type   |                   Description                   |
| :---------: | :-------: | :---------------------------------------------: |
| `invoiceId` | `uint216` | The keccak256 hash representing the invoice ID. |

#### \_applyBasisPoints

Calculates a portion of an amount using basis points.

```solidity
function _applyBasisPoints(uint256 _amount, uint256 _basisPoints) internal pure returns (uint256 value);
```

**Parameters**

|      Name      |    Type   |                      Description                      |
| :------------: | :-------: | :---------------------------------------------------: |
|    `_amount`   | `uint256` |      The base amount to apply the percentage to.      |
| `_basisPoints` | `uint256` | The percentage value in basis points (1 BPS = 0.01%). |

**Returns**

|   Name  |    Type   |                    Description                   |
| :-----: | :-------: | :----------------------------------------------: |
| `value` | `uint256` | The resulting value after applying basis points. |

#### \_distributeFunds

Distributes the remaining invoice balance between the seller and the buyer.

Transfers the buyer's refund (if any) and the seller's payout based on the given share. The refund is only processed if the seller is not entitled to the full balance.

```solidity
function _distributeFunds(Invoice memory _inv, uint256 _sellerShare)
    internal
    returns (uint256 sellerReceivingValue, uint256 buyerReceivingValue);
```

**Parameters**

|      Name      |    Type   |                                   Description                                  |
| :------------: | :-------: | :----------------------------------------------------------------------------: |
|     `_inv`     | `Invoice` |               The invoice containing payment and escrow details.               |
| `_sellerShare` | `uint256` | The portion of the invoice balance (in basis points) to be sent to the seller. |

**Returns**

|          Name          |    Type   |                            Description                           |
| :--------------------: | :-------: | :--------------------------------------------------------------: |
| `sellerReceivingValue` | `uint256` |                  The amount sent to the seller.                  |
|  `buyerReceivingValue` | `uint256` | The amount refunded to the buyer (zero if sellerShare == 10000). |

#### \_processSellerPayout

Distributes the seller's payout from the escrow, applying platform fees.

```solidity
function _processSellerPayout(Invoice memory _inv, uint256 _sellerReceivingValue)
    internal
    returns (uint256 sellerNetAmount);
```

**Parameters**

|           Name          |    Type   |                       Description                      |
| :---------------------: | :-------: | :----------------------------------------------------: |
|          `_inv`         | `Invoice` | The invoice data containing escrow and recipient info. |
| `_sellerReceivingValue` | `uint256` |    The gross amount owed to the seller before fees.    |

**Returns**

|        Name       |    Type   |                       Description                       |
| :---------------: | :-------: | :-----------------------------------------------------: |
| `sellerNetAmount` | `uint256` | The amount the seller receives after fees are deducted. |

#### \_computeMetaInvoiceId

Computes a deterministic ID for a meta-invoice based on the sub-invoice range and a salt.

The hash is based on the contract address, the sub-invoice ID range \[lower, upper], and a salt (e.g., a sequence number or counter). This prevents collisions when multiple meta-invoices share the same buyer and invoice range.

```solidity
function _computeMetaInvoiceId(uint256 _lower, uint256 _upper, uint256 _salt)
    internal
    view
    returns (uint216 metaInvoiceId);
```

**Parameters**

|   Name   |    Type   |                                          Description                                         |
| :------: | :-------: | :------------------------------------------------------------------------------------------: |
| `_lower` | `uint256` |                           The starting sub-invoice ID in the group.                          |
| `_upper` | `uint256` |                            The ending sub-invoice ID in the group.                           |
|  `_salt` | `uint256` | A user-provided or system-generated value (e.g., nextMetaInvoiceNonce) to ensure uniqueness. |

**Returns**

|       Name      |    Type   |                               Description                              |
| :-------------: | :-------: | :--------------------------------------------------------------------: |
| `metaInvoiceId` | `uint216` | A keccak256 hash representing the deterministic meta-invoice order ID. |

#### \_owner

Returns the owner of the PaymentProcessorStorage contract.

This helper reads the owner directly from the linked PaymentProcessorStorage instance.

```solidity
function _owner() internal view returns (address ownerAddress);
```

**Returns**

|      Name      |    Type   |                              Description                              |
| :------------: | :-------: | :-------------------------------------------------------------------: |
| `ownerAddress` | `address` | The address that currently owns the PaymentProcessorStorage contract. |

#### \_onlyMarketplace

Ensures that the caller is the registered marketplace address.

Reverts with `NotAuthorized` if `msg.sender` is not equal to the marketplace address stored in `ppStorage`.

```solidity
function _onlyMarketplace() internal view;
```

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

Returns the ID that will be assigned to the next meta-invoice.

```solidity
function getNextMetaInvoiceNonce() external view returns (uint216 nextMetaInvoiceId);
```

**Returns**

|         Name        |    Type   |             Description            |
| :-----------------: | :-------: | :--------------------------------: |
| `nextMetaInvoiceId` | `uint216` | The next meta-invoice nonce value. |

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
