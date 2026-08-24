# SimplePaymentProcessor.sol

The payment processor Solidity smart contract is the main user interface contract. Most users will interact with the SapphireDao platform via the `SimplePaymentProcessor.sol` contract. It shows invoice creation, management, payments, and escrow functionality on the blockchain.

Contract Address: [0xd4a9e5ac9f54beccd7c12ca6bd7bd026bbf0058d](https://sepolia.etherscan.io/address/0xd4a9e5ac9f54beccd7c12ca6bd7bd026bbf0058d)

You can find the full implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/SimplePaymentProcessor.sol)

Scheduled invoices are kept in an internal min-heap and processed by `processDueTasks`, which the registered [PaymentAutomation.sol](paymentautomation.sol.md) adapter calls on behalf of a keeper network (Chainlink CRE or Gelato). This contract holds no keeper configuration of its own: no forwarder address, no workflow owner, no CRE report handling; all of that now lives in the `PaymentAutomation` adapter.

Every value-moving entrypoint (everything except `cancelInvoice`, which moves no funds) reverts with `ContractPaused` while [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md#pause) reports the system paused.

`PaymentProcessor.sol` grants users access to

- Create invoice
- Payment of invoice
- Accept payment
- Reject payment
- Release invoice
- Invoice status

### State Variables

The invoice status codes and retry/fee constants below are plain file-level constants imported from `constants/Simple.sol`, not `public` members of the contract itself; there is no on-chain getter like `SimplePaymentProcessor.CREATED()`.

#### CREATED

Status code representing that an invoice has been created and is awaiting payment.

```solidity
uint8 constant CREATED = 1;
```

#### PAID

Status code representing that an invoice has been paid by the buyer.

```solidity
uint8 constant PAID = 2;
```

#### ACCEPTED

Status code representing that a payment has been accepted by the seller.

```solidity
uint8 constant ACCEPTED = 3;
```

#### REJECTED

Status code representing that a payment has been rejected by the seller.

```solidity
uint8 constant REJECTED = 4;
```

#### CANCELED

Status code representing that an invoice has been canceled by the seller.

```solidity
uint8 constant CANCELED = 5;
```

#### REFUNDED

Status code representing that a payment has been refunded to the payer.

```solidity
uint8 constant REFUNDED = 6;
```

#### RELEASED

Status code representing that a payment has been successfully released to the seller.

```solidity
uint8 constant RELEASED = 7;
```

#### BURNED

Invoice's escrowed funds were burned to `address(0)` after all automated withdrawal retries failed.

```solidity
uint8 constant BURNED = 8;
```

#### BASIS_POINTS

Basis points denominator used for percentage calculations (1% = 100).

```solidity
uint256 constant BASIS_POINTS = 10_000;
```

#### SELLER_DEFAULT_DECISION_WINDOW

Default decision period for the seller after an invoice is paid.

```solidity
uint256 constant SELLER_DEFAULT_DECISION_WINDOW = 6 hours;
```

#### MAX_WITHDRAWAL_RETRIES

Maximum number of automated withdrawal retry attempts before the escrowed funds are burned (see [refundBuyer](#refundbuyer)).

```solidity
uint8 constant MAX_WITHDRAWAL_RETRIES = 3;
```

#### ppStorage

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public immutable ppStorage
```

#### weth

Wrapped native token the platform fee is paid in. See [release](#release).

```solidity
IWETH public immutable weth
```

### Functions

#### constructor

Initializes the payment processor with its storage, notes, and WETH contract references.

Sets `ppStorage`, `notes`, and `weth`, initializes `decisionWindow` to `SELLER_DEFAULT_DECISION_WINDOW`, and assigns `minimumInvoiceValue` directly. Fee rate and fee receiver live in `ppStorage`, not here.

The minimum invoice value is assigned directly rather than via `setMinimumInvoiceValue`: this contract is deployed (via `MasterDeployer`) against a predicted storage address before `PaymentProcessorStorage` actually exists, so the setter's `onlyAuthorized` check, which calls into `ppStorage`, would revert at construction time.

```solidity
constructor(
    address _paymentProcessorStorageAddress,
    uint256 _minimumInvoicePrice,
    address _notesAddress,
    address _wethAddress
);
```

**Parameters**

|               Name                |   Type    |                          Description                          |
| :-------------------------------: | :-------: | :-----------------------------------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The address of the shared payment processor storage contract. |
|      `_minimumInvoicePrice`       | `uint256` |    The new minimum default invoice value to set (in wei).     |
|          `_notesAddress`          | `address` |     Address of the notes contract used for invoice notes.     |
|          `_wethAddress`           | `address` |     Address of the wrapped native token the platform fee is paid in.     |

#### receive

Accepts the native fee pulled out of an escrow on its way to being wrapped into WETH.

Reverts with `UnexpectedNativeTransfer` for any other incoming transfer, so native currency cannot be stranded on this contract; only accepts value while a fee is in flight during [`release`](#release) or the automated release path.

```solidity
receive() external payable;
```

#### createInvoice

Creates a new invoice with a specified price and escrow hold period.

```solidity
function createInvoice(uint256 _price, uint32 _holdPeriod, bytes memory _storageRef, bool _share)
    public
    whenNotPaused
    returns (uint216 invoiceId);
```

**Parameters**

|     Name      |   Type    |                                                     Description                                                      |
| :-----------: | :-------: | :------------------------------------------------------------------------------------------------------------------: |
|   `_price`    | `uint256` |                                           The price of the invoice in wei.                                           |
| `_holdPeriod` | `uint32`  | How long (in seconds) funds stay in escrow after the seller accepts payment. `0` releases immediately on acceptance. |
| `_storageRef` |  `bytes`  |                                A bytes-encoded reference to the user's notes storage.                                |
|   `_share`    |  `bool`   |                                     Whether the note is shared with non-authors.                                     |

**Returns**

|    Name     |   Type    |                 Description                 |
| :---------: | :-------: | :-----------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the newly created invoice. |

#### pay

Pays for an existing invoice and optionally updates the user's notes storage reference.

The caller must send enough ETH to cover the invoice price.

```solidity
function pay(uint216 _invoiceId, bytes memory _storageRef, bool _share)
    public
    payable
    whenNotPaused
    returns (address escrowAddress);
```

**Parameters**

|     Name      |   Type    |                       Description                        |
| :-----------: | :-------: | :------------------------------------------------------: |
| `_invoiceId`  | `uint216` |            The ID of the invoice being paid.             |
| `_storageRef` |  `bytes`  | A bytes-encoded reference to the caller's notes storage. |
|   `_share`    |  `bool`   |       Whether the note is shared with non-authors.       |

**Returns**

|      Name       |   Type    |                         Description                          |
| :-------------: | :-------: | :----------------------------------------------------------: |
| `escrowAddress` | `address` | The address of the escrow contract created for this payment. |

#### acceptPayment

Marks the specified invoice as accepted.

This function updates the status of the invoice to `ACCEPTED` and emits the `InvoiceAccepted` event, carrying the computed `releaseAt`. Only callable by the invoice's seller, and only while the invoice is `PAID` and within the acceptance window; reverts with `AcceptanceWindowExceeded` once that window has passed. `releaseAt` is set to `block.timestamp + escrowHoldPeriod`, using the hold period fixed on the invoice at creation, and the heap entry is rescheduled from `sellerActionDeadline` to `releaseAt`. `_feeReceiver` is recorded on the invoice and paid the platform fee on release, so it must be authorized by the fee signer via `_data`: reverts with `InvalidFeeReceiver` if `_feeReceiver` is the zero address, or `InvalidFeeAuthorization` if `_data` isn't a valid signature from [`PaymentProcessorStorage`'s fee signer](paymentprocessorstorage.sol.md#setfeesigner) over this invoice and receiver (see [FeeAuthorizationLib](../library/feeauthorizationlib.sol.md)).

```solidity
function acceptPayment(uint216 _invoiceId, address _feeReceiver, bytes memory _data) public whenNotPaused;
```

**Parameters**

|      Name      |   Type    |                              Description                              |
| :-------------: | :-------: | :----------------------------------------------------------------------: |
|  `_invoiceId`  | `uint216` |                  The key of the invoice being accepted.                  |
| `_feeReceiver` | `address` |                The address to pay this invoice's platform fee to.        |
|     `_data`    |  `bytes`  | The fee signer's 65-byte ECDSA signature over the authorization digest. |

#### rejectPayment

Marks the specified invoice as rejected and refunds the payer.

This function updates the invoice status to `REJECTED`, refunds the payer via the escrow contract, and emits the `InvoiceRejected` event. Same access-control/window rules as `acceptPayment` (seller only, within the acceptance window).

```solidity
function rejectPayment(uint216 _invoiceId) public whenNotPaused;
```

**Parameters**

|     Name     |   Type    |                        Description                        |
| :----------: | :-------: | :-------------------------------------------------------: |
| `_invoiceId` | `uint216` | The key of the invoice being rejected. address and payer. |

#### cancelInvoice

Cancels an existing invoice.

Only callable by the invoice seller.

```solidity
function cancelInvoice(uint216 _invoiceId) external;
```

**Parameters**

|     Name     |   Type    |           Description            |
| :----------: | :-------: | :------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to cancel. |

#### release

Releases the funds held in escrow for a specific invoice to the seller.

Only callable by the seller. Invoice must be in `ACCEPTED` state (reverts `InvalidInvoiceState` otherwise) and `releaseAt` must have passed (reverts `HoldPeriodHasNotBeenExceeded` otherwise). Deducts the platform fee before transferring the net amount to the seller, using the fee rate captured on the invoice at creation (`feeRate`), not the current global rate, so a later change to the global rate never affects an already-created invoice. The seller is paid in native currency; the fee is pulled from escrow as native currency, wrapped into `weth`, and sent to the fee receiver as WETH, so a receiver that rejects native transfers is still paid. The fee receiver is the one authorized at acceptance (`feeReceiver`), falling back to [`PaymentProcessorStorage`'s global fee receiver](paymentprocessorstorage.sol.md#getfeereceiver) for invoices accepted before this feature existed.

```solidity
function release(uint216 _invoiceId) public whenNotPaused;
```

**Parameters**

|     Name     |   Type    |                     Description                     |
| :----------: | :-------: | :-------------------------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice for which funds are released. |

#### refundBuyer

Refunds the buyer of a specific invoice when the seller fails to act in time.

Invoice must be in `PAID` state or retry state after seller release action fails multiple times, otherwise reverts with `InvoiceNotEligibleForRefund`. Attempts to withdraw the price to the buyer; on success the invoice transitions to `REFUNDED`, is removed from the heap, and its balance is zeroed. On withdrawal failure, the retry counter is incremented and the invoice stays `PAID` for a future retry; once retries would exceed `MAX_WITHDRAWAL_RETRIES`, the escrowed funds are instead burned to `address(0)` and the invoice transitions to `BURNED`. This is terminal and unrecoverable; there is no `releaseLocked`-style recovery path anymore. Guarded by `nonReentrant`.

```solidity
function refundBuyer(uint216 _invoiceId) public nonReentrant whenNotPaused;
```

**Parameters**

|     Name     |   Type    |              Description              |
| :----------: | :-------: | :-----------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to be refunded. |

#### hasDueTasks

Returns whether any scheduled invoice task is due for processing. Read by the registered [PaymentAutomation.sol](paymentautomation.sol.md) adapter to decide whether a keeper should trigger processing.

```solidity
function hasDueTasks() external view returns (bool dueTasksExist);
```

**Returns**

|      Name       |  Type  |                  Description                  |
| :-------------: | :----: | :-------------------------------------------: |
| `dueTasksExist` | `bool` | True when the earliest scheduled task is due. |

#### processDueTasks

Processes due invoice tasks (auto-release and auto-refund) within the gas threshold. Callable by the owner as a manual fallback, or by the registered automation adapter. Processing stops once remaining gas drops below the configured gas threshold, so leftover tasks are picked up on the next call.

Only callable by the owner or by `automation` (reverts with `NotAuthorized` otherwise). Guarded by `nonReentrant`. The CRE/Gelato-specific entrypoints and forwarder/workflow-owner configuration that used to live here have moved to [PaymentAutomation.sol](paymentautomation.sol.md); this contract now only exposes the bare `hasDueTasks`/`processDueTasks` pair and trusts nothing but the `automation` address.

```solidity
function processDueTasks() external nonReentrant whenNotPaused;
```

#### calculateFee

Calculates the fee based on the provided amount and the _current_ global fee rate.

Fee rate is expressed in basis points (1% = 100). This quotes the rate that would be captured by an invoice created right now; it does **not** reflect what a given existing invoice will actually be charged on release, since `release`/`refundBuyer`/the automated release path all use the fee rate snapshotted on the invoice at creation (`feeRate`), not the current global rate.

```solidity
function calculateFee(uint256 _amount) public view returns (uint256 feeValue);
```

**Parameters**

|   Name    |   Type    |              Description              |
| :-------: | :-------: | :-----------------------------------: |
| `_amount` | `uint256` | The amount to calculate the fee from. |

**Returns**

|    Name    |   Type    |        Description         |
| :--------: | :-------: | :------------------------: |
| `feeValue` | `uint256` | The calculated fee amount. |

#### setMinimumInvoiceValue

Updates the minimum allowed invoice value required for creating an invoice.

Only callable by the owner or the storage contract.

```solidity
function setMinimumInvoiceValue(uint256 _newMinimumInvoiceValue) public onlyAuthorized;
```

**Parameters**

|           Name            |   Type    |                  Description                   |
| :-----------------------: | :-------: | :--------------------------------------------: |
| `_newMinimumInvoiceValue` | `uint256` | The new minimum invoice value to set (in wei). |

#### setAutomation

Updates the automation adapter allowed to drain due tasks on a keeper network's behalf.

Only callable by the owner or the storage contract. The adapter (see [PaymentAutomation.sol](paymentautomation.sol.md)) holds the Chainlink CRE and Gelato entrypoints; this processor trusts nothing but its address. Setting it to the zero address leaves the owner as the only caller of `processDueTasks`.

```solidity
function setAutomation(address _automationAddress) external onlyAuthorized;
```

**Parameters**

|         Name         |   Type    |                Description                 |
| :------------------: | :-------: | :----------------------------------------: |
| `_automationAddress` | `address` | The new automation adapter address to set. |

#### setDecisionWindow

Updates the decision window sellers have to accept payments after buyer payment.

Only callable by the owner or the storage contract. Reverts with `InvalidDecisionWindow` if `_newDecisionWindow` is zero.

```solidity
function setDecisionWindow(uint256 _newDecisionWindow) external onlyAuthorized;
```

**Parameters**

|         Name         |   Type    |             Description             |
| :------------------: | :-------: | :---------------------------------: |
| `_newDecisionWindow` | `uint256` | The new decision window in seconds. |

#### getAutomation

Returns the address of the registered automation adapter.

```solidity
function getAutomation() external view returns (address automationAddress);
```

**Returns**

|        Name         |   Type    |                Description                 |
| :-----------------: | :-------: | :----------------------------------------: |
| `automationAddress` | `address` | The configured automation adapter address. |

#### getDecisionWindow

Returns the window sellers have to accept or reject a payment after the buyer pays.

```solidity
function getDecisionWindow() external view returns (uint256 decisionWindowValue);
```

**Returns**

|         Name          |   Type    |               Description               |
| :-------------------: | :-------: | :-------------------------------------: |
| `decisionWindowValue` | `uint256` | The current decision window in seconds. |

#### getNextInvoiceNonce

Gets the current invoice nonce counter.

```solidity
function getNextInvoiceNonce() external view returns (uint216 nextInvoiceNonceValue);
```

**Returns**

|          Name           |   Type    |          Description          |
| :---------------------: | :-------: | :---------------------------: |
| `nextInvoiceNonceValue` | `uint216` | The next invoice nonce value. |

#### getInvoiceData

Retrieves detailed data for a specific invoice.

```solidity
function getInvoiceData(uint216 _invoiceId) public view returns (Invoice memory i);
```

**Parameters**

|     Name     |   Type    |      Description       |
| :----------: | :-------: | :--------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice. |

**Returns**

| Name |   Type    |    Description    |
| :--: | :-------: | :---------------: |
| `i`  | `Invoice` | The invoice data. |

#### getMinimumInvoiceValue

Returns the minimum allowed invoice value required for invoice creation.

```solidity
function getMinimumInvoiceValue() external view returns (uint256 minimumValue);
```

**Returns**

|      Name      |   Type    |            Description             |
| :------------: | :-------: | :--------------------------------: |
| `minimumValue` | `uint256` | The minimum allowed invoice value. |

#### getItems

Returns a list of all task IDs currently in the heap.

Retrieves the uint216 task identifiers extracted from the internal encoded heap structure.

```solidity
function getItems() external view returns (uint216[] memory items);
```

**Returns**

|  Name   |    Type     |    Description     |
| :-----: | :---------: | :----------------: |
| `items` | `uint216[]` | Array of task IDs. |

### Structs

#### Invoice

Represents an invoice between a buyer and seller, with escrow, timestamps, and status tracking.

```solidity
struct Invoice {
    uint216 invoiceNonce;
    uint40 createdAt;
    uint40 paidAt;
    uint40 releaseAt;
    uint40 expiresAt;
    uint40 sellerActionDeadline;
    uint32 escrowHoldPeriod;
    uint8 state;
    uint8 withdrawalRetries;
    uint16 feeRate;
    address seller;
    address buyer;
    address escrow;
    address feeReceiver;
    uint256 price;
    uint256 balance;
}
```

|         Field          |   Type    |                                                                                   Description                                                                                    |
| :---------------------: | :-------: | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|     `invoiceNonce`     | `uint216` |                                                      A unique identifier assigned to this invoice, typically sequentially.                                                       |
|       `createdAt`      | `uint40`  |                                                                 The Unix timestamp when the invoice was created.                                                                 |
|        `paidAt`        | `uint40`  |                                                                The Unix timestamp when the payment was completed.                                                                |
|       `releaseAt`      | `uint40`  |                                                        The timestamp when funds in escrow can be released to the seller.                                                         |
|       `expiresAt`      | `uint40`  |                                                            The timestamp after which the invoice can no longer be paid.                                                          |
| `sellerActionDeadline` | `uint40`  |                                    The timestamp after which the seller can no longer take action (accept/reject), and the buyer is refunded.                                    |
|   `escrowHoldPeriod`   | `uint32`  |           Escrow hold duration (in seconds) set by the seller at creation, counted from acceptance. `0` means funds are releasable as soon as the payment is accepted.           |
|         `state`        |  `uint8`  |                                                                        The current state of the invoice.                                                                         |
| `withdrawalRetries` |  `uint8`  |                                Number of failed `IEscrow.withdraw` attempts by the automation path. Packed with `state` in the same storage slot.                                |
|      `feeRate`      | `uint16`  | The platform fee rate (in basis points) captured at invoice creation. Releases always charge this rate, so later changes to the global fee rate do not affect existing invoices. |
|      `seller`       | `address` |                                                                    The address of the seller of the invoice.                                                                     |
|       `buyer`       | `address` |                                                                     The address of the buyer of the invoice.                                                                     |
|      `escrow`       | `address` |                                                     The address of the escrow contract managing the funds for this invoice.                                                      |
|    `feeReceiver`    | `address` |                          Address that receives the platform fee for this invoice, authorized by the fee signer when the seller accepted the payment.                            |
|       `price`       | `uint256` |                                                                      The total price of the invoice in wei.                                                                      |
|      `balance`      | `uint256` |                                    The current amount held in escrow, net of any fees deducted upon acceptance. Zeroed on release or refund.                                     |

### Events

#### InvoiceCreated

Emitted when a new invoice is created.

```solidity
event InvoiceCreated(uint216 indexed invoiceId, Invoice invoice);
```

|    Name     |   Type    |                                    Description                                    |
| :---------: | :-------: | :-------------------------------------------------------------------------------: |
| `invoiceId` | `uint216` |                  The unique identifier for the created invoice.                   |
|  `invoice`  | `Invoice` | The full invoice struct containing buyer, price, timestamps, state, and metadata. |

#### InvoicePaid

Emitted when an invoice payment is made.

```solidity
event InvoicePaid(
    uint216 indexed invoiceId, address indexed buyer, uint256 indexed amountPaid, uint40 sellerActionDeadline
);
```

|         Name          |   Type    |                                              Description                                              |
| :--------------------: | :-------: | :---------------------------------------------------------------------------------------------------: |
|      `invoiceId`      | `uint216` |                                  The unique ID of the paid invoice.                                   |
|        `buyer`        | `address` |                                  The address of the buyer who paid.                                   |
|      `amountPaid`     | `uint256` |                              The amount paid towards the invoice in wei.                              |
| `sellerActionDeadline` | `uint40`  | The timestamp by which the seller must accept or reject; after this the buyer is eligible for refund. |

#### InvoiceRejected

Emitted when an invoice is rejected by the seller.

```solidity
event InvoiceRejected(uint216 indexed invoiceId, uint256 amount);
```

|    Name     |   Type    |                   Description                    |
| :---------: | :-------: | :----------------------------------------------: |
| `invoiceId` | `uint216` |      The unique ID of the rejected invoice.      |
|  `amount`   | `uint256` | The escrow balance refunded to the buyer in wei. |

#### InvoiceRefunded

Emitted when an invoice is refunded to the buyer.

```solidity
event InvoiceRefunded(uint216 indexed invoiceId, uint256 amount);
```

|    Name     |   Type    |                   Description                    |
| :---------: | :-------: | :----------------------------------------------: |
| `invoiceId` | `uint216` |      The unique ID of the refunded invoice.      |
|  `amount`   | `uint256` | The escrow balance refunded to the buyer in wei. |

#### InvoiceAccepted

Emitted when an invoice is accepted by the seller.

```solidity
event InvoiceAccepted(uint216 indexed invoiceId, address indexed feeReceiver, uint40 releaseAt);
```

|    Name     |   Type    |                        Description                        |
| :---------: | :-------: | :---------------------------------------------------------: |
| `invoiceId` | `uint216` |             The unique ID of the accepted invoice.           |
| `feeReceiver` | `address` | The address recorded to be paid this invoice's platform fee on release. |
| `releaseAt` | `uint40`  | The timestamp when the escrowed funds become releasable to the seller. |

#### InvoiceCanceled

Emitted when an invoice is canceled.

```solidity
event InvoiceCanceled(uint216 indexed invoiceId);
```

|    Name     |   Type    |              Description               |
| :---------: | :-------: | :------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the canceled invoice. |

#### InvoiceReleased

Emitted when an invoice is released (funds disbursed from escrow).

```solidity
event InvoiceReleased(uint216 indexed invoiceId, uint256 sellerAmount, uint256 fee);
```

|      Name      |   Type    |                       Description                       |
| :------------: | :-------: | :-----------------------------------------------------: |
|  `invoiceId`   | `uint216` |         The unique ID of the released invoice.          |
| `sellerAmount` | `uint256` |  The net amount transferred to the seller, after fees.  |
|     `fee`      | `uint256` | The platform fee deducted and sent to the fee receiver. |

#### WithdrawalRetried

Emitted when an automated withdrawal attempt fails and is retried.

```solidity
event WithdrawalRetried(uint216 indexed invoiceId, address indexed recipient, uint256 amount, uint8 attempt);
```

|    Name     |   Type    |                     Description                     |
| :---------: | :-------: | :-------------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the invoice whose withdrawal was retried. |
| `recipient` | `address` |    The address the withdrawal was attempted to.     |
|  `amount`   | `uint256` |         The amount that failed to transfer.         |
|  `attempt`  |  `uint8`  |              The retry attempt number.              |

#### AutomationUpdated

Emitted when the automation adapter authorized to call `processDueTasks` is updated.

```solidity
event AutomationUpdated(address indexed automation);
```

|     Name     |   Type    |             Description             |
| :----------: | :-------: | :---------------------------------: |
| `automation` | `address` | The new automation adapter address. |

#### TransferFailed

Emitted when a transfer from the escrow fails. Best-effort for fee transfers, which stay in escrow. On the final refund attempt it precedes `PaymentBurned`.

```solidity
event TransferFailed(uint216 indexed invoiceId, address indexed recipient, uint256 amount);
```

|    Name     |   Type    |                     Description                     |
| :---------: | :-------: | :-------------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the invoice associated with the transfer. |
| `recipient` | `address` |     The address the transfer was attempted to.      |
|  `amount`   | `uint256` |         The amount that failed to transfer.         |

#### PaymentBurned

Emitted when an invoice's escrowed funds are burned to `address(0)`. The funds are permanently destroyed; there is no recovery path.

```solidity
event PaymentBurned(uint216 indexed invoiceId, uint256 amount);
```

|    Name     |   Type    |                  Description                  |
| :---------: | :-------: | :-------------------------------------------: |
| `invoiceId` | `uint216` | The invoice whose escrowed funds were burned. |
|  `amount`   | `uint256` |    The amount of ETH sent to `address(0)`.    |

### Errors

|                           Error                            |                                                 Description                                                 |
| :--------------------------------------------------------: | :---------------------------------------------------------------------------------------------------------: |
|                     `NotAuthorized()`                      |                        Thrown when the caller lacks the required role or permission.                        |
|                     `ValueIsTooLow()`                      |                     Thrown when the provided value is lower than the required minimum.                      |
|                  `InvalidHeapPosition()`                   |                                 Thrown when a task's heap index is invalid.                                 |
|                 `InvalidDecisionWindow()`                  |                   Thrown when the decision window value provided is invalid (e.g., zero).                   |
| `IncorrectPaymentAmount(uint256 _sent, uint256 _expected)` |               Thrown when the payment amount sent does not match the expected invoice price.                |
|                  `InvoiceAlreadyExists()`                  |                        Thrown when trying to create an invoice that already exists.                         |
|        `InvalidInvoiceState(uint256 _invoiceState)`        |                  Thrown when the invoice is in an invalid state for the requested action.                   |
|                 `InvoiceIsNoLongerValid()`                 |         Thrown when a payment is attempted after the invoice's payment validity window has expired.         |
|                `AcceptanceWindowExceeded()`                |    Thrown when the seller attempts to take action on an invoice after the acceptance window has expired.    |
|              `SellerCannotPayOwnedInvoice()`               |                 Thrown when the seller of an invoice attempts to pay for their own invoice.                 |
|              `InvoiceNotEligibleForRefund()`               | Thrown when a refund to the buyer cannot be issued (invoice not `PAID` or decision window not yet elapsed). |
|              `HoldPeriodHasNotBeenExceeded()`              |                    Thrown when the hold period for an invoice has not yet been exceeded.                    |
|                `InvalidFeeAuthorization()`                 |         Thrown when the fee receiver is not authorized by a signature from the configured fee signer.        |
|                   `InvalidFeeReceiver()`                   |                       Thrown when the zero address is supplied as the fee receiver.                          |
|               `UnexpectedNativeTransfer()`                 |         Thrown when native currency is sent to the processor outside of a fee being wrapped into WETH.       |
|                  `EscrowWithdrawFailed()`                  |             Thrown when the escrow withdrawal fails during a manual release, reject, or refund.             |
|                     `ContractPaused()`                     |                 Thrown when a value-moving entrypoint is called while the system is paused.                 |
