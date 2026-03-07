# SimplePaymentProcessor.sol



The payment processor Solidity smart contract is the main user interface contract. Most users will interact with the SapphireDao platform via the `PaymentProcessor.sol` contract. It shows invoice creation, management, payments, and escrow functionality on the blockchain.&#x20;

Contract Address: [0xd4a9e5ac9f54beccd7c12ca6bd7bd026bbf0058d](https://sepolia.etherscan.io/address/0xd4a9e5ac9f54beccd7c12ca6bd7bd026bbf0058d)

You can find the full implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/SimplePaymentProcessor.sol)&#x20;

`PaymentProcessor.sol` grants users access to

* Create invoice
* Payment of invoice
* Accept payment
* Reject payment
* Release invoice
* Invoice status

### State Variables

#### CREATED

Status code representing that a payment or transaction has been created.

```solidity
uint8 public constant CREATED = 1
```

#### PAID

Status code representing that a payment has been completed.

```solidity
uint8 public constant PAID = CREATED + 1
```

#### ACCEPTED

Status code representing that a payment or transaction has been accepted.

```solidity
uint8 public constant ACCEPTED = PAID + 1
```

#### REJECTED

Status code representing that a payment or transaction has been rejected.

```solidity
uint8 public constant REJECTED = ACCEPTED + 1
```

#### CANCELED

Status code representing that a payment or transaction has been canceled.

```solidity
uint8 public constant CANCELED = REJECTED + 1
```

#### REFUNDED

Status code representing that a payment has been refunded to the payer.

```solidity
uint8 public constant REFUNDED = CANCELED + 1
```

#### RELEASED

Status code representing that a payment has been successfully released to the payee.

```solidity
uint8 public constant RELEASED = REFUNDED + 1
```

#### BASIS\_POINTS

Basis points denominator used for percentage calculations (1% = 100).

```solidity
uint256 public constant BASIS_POINTS = 10_000
```

#### DEFAULT\_SELLER\_DECISION\_WINDOW

Default decision period for the seller after an invoice is paid.

```solidity
uint256 public constant DEFAULT_SELLER_DECISION_WINDOW = 6 hours
```

#### ppStorage

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public immutable ppStorage
```

#### decisionWindow

The window of time allowed for accepting a transaction after creation.

```solidity
uint256 public decisionWindow
```

### Functions

#### constructor

Initializes the payment processor with owner, fee settings, and default hold period.

Sets the fee receiver address, the fee rate (in basis points), and the default escrow hold time.

```solidity
constructor(address _paymentProcessorStorageAddress, uint256 _minimumInvoicePrice, address _notesAddress) ;
```

**Parameters**

|                Name               |    Type   |                          Description                          |
| :-------------------------------: | :-------: | :-----------------------------------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The address of the shared payment processor storage contract. |
|       `_minimumInvoicePrice`      | `uint256` |     The new minimum default invoice value to set (in wei).    |
|          `_notesAddress`          | `address` |     Address of the notes contract used for invoice notes.     |

#### createInvoice

Creates a new invoice with a specified price.

Optionally stores a reference to the user's off-chain notes file.

```solidity
function createInvoice(uint256 _price, bytes memory _storageRef, bool _share)
    external
    returns (uint216 invoiceId);
```

**Parameters**

|      Name     |    Type   |                       Description                      |
| :-----------: | :-------: | :----------------------------------------------------: |
| `_price`      | `uint256` |            The price of the invoice in wei.            |
|  `_storageRef`  |  `bytes`  | A bytes-encoded reference to the user's notes storage. |
|     `_share`    |   `bool`  |      Whether the note is shared with non-authors.      |

**Returns**

|     Name    |    Type   |                 Description                 |
| :---------: | :-------: | :-----------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the newly created invoice. |

#### pay

Pays for an existing invoice and optionally updates the user's notes storage reference.

The caller must send enough ETH to cover the invoice price.

```solidity
function pay(uint216 _invoiceId, bytes memory _storageRef, bool _share)
    external
    payable
    returns (address escrowAddress);
```

**Parameters**

|      Name     |    Type   |                        Description                       |
| :-----------: | :-------: | :------------------------------------------------------: |
|  `_invoiceId` | `uint216` |             The ID of the invoice being paid.            |
| `_storageRef` |  `bytes`  | A bytes-encoded reference to the caller's notes storage. |
|    `_share`   |   `bool`  |       Whether the note is shared with non-authors.       |

**Returns**

|    Name   |    Type   |                             Description                             |
| :-------: | :-------: | :-----------------------------------------------------------------: |
| `escrow`  | `address` | The address of the escrow contract created for this payment. |

#### acceptPayment

Marks the specified invoice as accepted.

This function updates the status of the invoice to `ACCEPTED` and emits the `InvoiceAccepted` event. It is expected that the creator is approving the payment for the invoice.

```solidity
function acceptPayment(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |               Description              |
| :----------: | :-------: | :------------------------------------: |
| `_invoiceId` | `uint216` | The key of the invoice being accepted. |

#### rejectPayment

Marks the specified invoice as rejected and refunds the payer.

This function updates the invoice status to `REJECTED`, refunds the payer via the escrow contract, and emits the `InvoiceRejected` event.

```solidity
function rejectPayment(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |                        Description                        |
| :----------: | :-------: | :-------------------------------------------------------: |
| `_invoiceId` | `uint216` | The key of the invoice being rejected. address and payer. |

#### cancelInvoice

Cancels an existing invoice.

Only callable by the invoice seller.

```solidity
function cancelInvoice(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |            Description           |
| :----------: | :-------: | :------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to cancel. |

#### release

Releases the funds held in escrow for a specific invoice to the seller.

```solidity
function release(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |                     Description                     |
| :----------: | :-------: | :-------------------------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice for which funds are released. |

#### refundBuyer

Refunds the buyer of a specific invoice when the seller fails to act in time.

Invoice must be in PAID state and the decision window (`expiresAt`) must have elapsed. The invoice transitions to REFUNDED, removes it from the heap, zeroes the balance, and returns funds to the buyer.

```solidity
function refundBuyer(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |    Type   |              Description              |
| :----------: | :-------: | :-----------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to be refunded. |

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

#### setInvoiceReleaseTime

Sets a custom hold period for a specific invoice.

Only callable by the owner. Invoice must be in ACCEPTED state. The new release time is computed as `block.timestamp + _holdPeriod`. The invoice's heap entry is rescheduled to the new release time.

```solidity
function setInvoiceReleaseTime(uint216 _invoiceId, uint40 _holdPeriod) external;
```

**Parameters**

|      Name     |    Type   |           Description           |
| :-----------: | :-------: | :-----------------------------: |
|  `_invoiceId` | `uint216` |      The ID of the invoice.     |
| `_holdPeriod` |  `uint40` | The hold period from now in seconds. |

#### calculateFee

Calculates the fee based on the provided amount and current fee rate.

Fee rate is expressed in basis points (1% = 100).

```solidity
function calculateFee(uint256 _amount) public view returns (uint256 feeValue);
```

**Parameters**

|    Name   |    Type   |              Description              |
| :-------: | :-------: | :-----------------------------------: |
| `_amount` | `uint256` | The amount to calculate the fee from. |

**Returns**

|    Name    |    Type   |         Description        |
| :--------: | :-------: | :------------------------: |
| `feeValue` | `uint256` | The calculated fee amount. |

#### setMinimumInvoiceValue

Updates the minimum allowed invoice value required for creating an invoice.

Only callable by the owner or the storage contract.

```solidity
function setMinimumInvoiceValue(uint256 _minimumInvoiceValue) external;
```

**Parameters**

|            Name           |    Type   | Description |
| :-----------------------: | :-------: | :---------: |
| `_minimumInvoiceValue` | `uint256` | The new minimum invoice value to set (in wei). |

#### setForwarderAddress

Updates the address of the forwarder contract used for relayed or automated calls.

```solidity
function setForwarderAddress(address _forwarderAddress) external;
```

**Parameters**

|         Name        |    Type   |                  Description                  |
| :-----------------: | :-------: | :-------------------------------------------: |
| `_forwarderAddress` | `address` | The new forwarder contract address to be set. |

#### setDecisionWindow

Updates the decision window sellers have to accept payments after buyer payment.

```solidity
function setDecisionWindow(uint256 _newDecisionWindow) external onlyAuthorized;
```

**Parameters**

|         Name         |    Type   |             Description             |
| :------------------: | :-------: | :---------------------------------: |
| `_newDecisionWindow` | `uint256` | The new decision window in seconds. |

#### getForwarder

Returns the address of the configured forwarder contract.

```solidity
function getForwarder() external view returns (address forwarderAddress);
```

**Returns**

|        Name        |    Type   |            Description            |
| :----------------: | :-------: | :-------------------------------: |
| `forwarderAddress` | `address` | The configured forwarder address. |

#### getNextInvoiceNonce

Gets the current invoice nonce counter.

```solidity
function getNextInvoiceNonce() external view returns (uint216 nextInvoiceNonceValue);
```

**Returns**

|           Name          |    Type   |          Description          |
| :---------------------: | :-------: | :---------------------------: |
| `nextInvoiceNonceValue` | `uint216` | The next invoice nonce value. |

#### getInvoiceData

Retrieves detailed data for a specific invoice.

```solidity
function getInvoiceData(uint216 _invoiceId) external view returns (Invoice memory invoiceDetails);
```

**Parameters**

|     Name     |    Type   |       Description      |
| :----------: | :-------: | :--------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice. |

**Returns**

|  Name |    Type   |    Description    |
| :---: | :-------: | :---------------: |
|  `i`  | `Invoice` | The invoice data. |

#### getMinimumInvoiceValue

Returns the minimum allowed invoice value required for invoice creation.

```solidity
function getMinimumInvoiceValue() external view returns (uint256 minimumValue);
```

**Returns**

|      Name      |    Type   |             Description            |
| :------------: | :-------: | :--------------------------------: |
| `minimumValue` | `uint256` | The minimum allowed invoice value. |

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

| Name        | Type      | Description                                                                          |
| :---------- | :-------: | :----------------------------------------------------------------------------------- |
| `invoiceId` | `uint216` | The unique identifier for the created invoice.                                       |
| `invoice`   | `Invoice` | The full invoice struct containing buyer, price, timestamps, state, and metadata.    |

#### InvoicePaid

Emitted when an invoice payment is made.

```solidity
event InvoicePaid(uint216 indexed invoiceId, address indexed buyer, uint256 indexed amountPaid, uint40 expiresAt);
```

| Name        | Type      | Description                                                                                         |
| :---------- | :-------: | :-------------------------------------------------------------------------------------------------- |
| `invoiceId` | `uint216` | The unique ID of the paid invoice.                                                                  |
| `buyer`     | `address` | The address of the buyer who paid.                                                                  |
| `amountPaid`| `uint256` | The amount paid towards the invoice in wei.                                                         |
| `expiresAt` | `uint40`  | The timestamp by which the seller must accept or reject; after this the buyer is eligible for refund. |

#### InvoiceRejected

Emitted when an invoice is rejected by the seller.

```solidity
event InvoiceRejected(uint216 indexed invoiceId);
```

| Name        | Type      | Description                             |
| :---------- | :-------: | :-------------------------------------- |
| `invoiceId` | `uint216` | The unique ID of the rejected invoice.  |

#### InvoiceRefunded

Emitted when an invoice is refunded to the buyer.

```solidity
event InvoiceRefunded(uint216 indexed invoiceId);
```

| Name        | Type      | Description                             |
| :---------- | :-------: | :-------------------------------------- |
| `invoiceId` | `uint216` | The unique ID of the refunded invoice.  |

#### InvoiceAccepted

Emitted when an invoice is accepted by the seller.

```solidity
event InvoiceAccepted(uint216 indexed invoiceId);
```

| Name        | Type      | Description                             |
| :---------- | :-------: | :-------------------------------------- |
| `invoiceId` | `uint216` | The unique ID of the accepted invoice.  |

#### InvoiceCanceled

Emitted when an invoice is canceled.

```solidity
event InvoiceCanceled(uint216 indexed invoiceId);
```

| Name        | Type      | Description                             |
| :---------- | :-------: | :-------------------------------------- |
| `invoiceId` | `uint216` | The unique ID of the canceled invoice.  |

#### InvoiceReleased

Emitted when an invoice is released (funds disbursed from escrow).

```solidity
event InvoiceReleased(uint216 indexed invoiceId);
```

| Name        | Type      | Description                             |
| :---------- | :-------: | :-------------------------------------- |
| `invoiceId` | `uint216` | The unique ID of the released invoice.  |

#### UpdateHoldPeriod

Emitted when the hold period of a given invoice is updated to a new timestamp.

```solidity
event UpdateHoldPeriod(uint216 indexed invoiceId, uint256 indexed releaseDueTimestamp);
```

| Name                   | Type      | Description                                                  |
| :--------------------- | :-------: | :----------------------------------------------------------- |
| `invoiceId`            | `uint216` | The key of the invoice whose hold period was updated.        |
| `releaseDueTimestamp`  | `uint256` | The new hold period expressed as a UNIX timestamp.           |

### Errors

| Error | Description |
| :---- | :---------- |
| `NotAuthorized()` | Thrown when the caller lacks the required role or permission. |
| `ValueIsTooLow()` | Thrown when the provided value is lower than the required minimum. |
| `InvalidHeapPosition()` | Thrown when a task's heap index is invalid. |
| `InvalidDecisionWindow()` | Thrown when the decision window value provided is invalid (e.g., zero). |
| `IncorrectPaymentAmount(uint256 _sent, uint256 _expected)` | Thrown when the payment amount sent does not match the expected invoice price. |
| `InvoiceAlreadyExists()` | Thrown when trying to create an invoice that already exists. |
| `InvalidInvoiceState(uint256 _invoiceState)` | Thrown when the invoice is in an invalid state for the requested action. |
| `InvoiceIsNoLongerValid()` | Thrown when a payment is attempted after the invoice's payment validity window has expired. |
| `AcceptanceWindowExceeded()` | Thrown when the seller attempts to take action on an invoice after the acceptance window has expired. |
| `SellerCannotPayOwnedInvoice()` | Thrown when the seller of an invoice attempts to pay for their own invoice. |
| `InvoiceNotEligibleForRefund()` | Thrown when a refund to the buyer cannot be issued. |
| `HoldPeriodHasNotBeenExceeded()` | Thrown when the hold period for an invoice has not yet been exceeded. |
