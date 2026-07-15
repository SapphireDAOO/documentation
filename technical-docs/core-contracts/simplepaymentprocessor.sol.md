# SimplePaymentProcessor.sol



The payment processor Solidity smart contract is the main user interface contract. Most users will interact with the SapphireDao platform via the `SimplePaymentProcessor.sol` contract. It shows invoice creation, management, payments, and escrow functionality on the blockchain.

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

The invoice status codes and retry/fee constants below are plain file-level constants imported from `constants/Simple.sol`, not `public` members of the contract itself — there is no on-chain getter like `SimplePaymentProcessor.CREATED()`.

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

#### LOCKED

Invoice is permanently locked after all automated withdrawal retries failed.

```solidity
uint8 constant LOCKED = 8;
```

#### BASIS\_POINTS

Basis points denominator used for percentage calculations (1% = 100).

```solidity
uint256 constant BASIS_POINTS = 10_000;
```

#### SELLER\_DEFAULT\_DECISION\_WINDOW

Default decision period for the seller after an invoice is paid.

```solidity
uint256 constant SELLER_DEFAULT_DECISION_WINDOW = 6 hours;
```

#### MAX\_WITHDRAWAL\_RETRIES

Maximum number of automated withdrawal retry attempts before falling back (see [refundBuyer](#refundbuyer) and the automated-upkeep retry logic).

```solidity
uint8 constant MAX_WITHDRAWAL_RETRIES = 3;
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

Initializes the payment processor with its storage and notes contract references.

Sets `ppStorage` and `notes`, initializes `decisionWindow` to `SELLER_DEFAULT_DECISION_WINDOW`, and assigns `minimumInvoiceValue` directly. Fee rate, fee receiver, and the default escrow hold period all live in `ppStorage`, not here.

The minimum invoice value is assigned directly rather than via `setMinimumInvoiceValue`: this contract is deployed (via `MasterDeployer`) against a predicted storage address before `PaymentProcessorStorage` actually exists, so the setter's `onlyAuthorized` check — which calls into `ppStorage` — would revert at construction time.

```solidity
constructor(address _paymentProcessorStorageAddress, uint256 _minimumInvoicePrice, address _notesAddress);
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
    public
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
    public
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

|      Name       |    Type   |                             Description                             |
| :---------------: | :-------: | :-----------------------------------------------------------------: |
| `escrowAddress`  | `address` | The address of the escrow contract created for this payment. |

#### acceptPayment

Marks the specified invoice as accepted.

This function updates the status of the invoice to `ACCEPTED` and emits the `InvoiceAccepted` event. Only callable by the invoice's seller, and only while the invoice is `PAID` and within the acceptance window (`expiresAt`); reverts with `AcceptanceWindowExceeded` once that window has passed.

```solidity
function acceptPayment(uint216 _invoiceId) public;
```

**Parameters**

|     Name     |    Type   |               Description              |
| :----------: | :-------: | :------------------------------------: |
| `_invoiceId` | `uint216` | The key of the invoice being accepted. |

#### rejectPayment

Marks the specified invoice as rejected and refunds the payer.

This function updates the invoice status to `REJECTED`, refunds the payer via the escrow contract, and emits the `InvoiceRejected` event. Same access-control/window rules as `acceptPayment` (seller only, within the acceptance window).

```solidity
function rejectPayment(uint216 _invoiceId) public;
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

Only callable by the seller. Invoice must be in `ACCEPTED` state (reverts `InvalidInvoiceState` otherwise) and `releaseAt` must have passed (reverts `HoldPeriodHasNotBeenExceeded` otherwise). Deducts the platform fee before transferring the net amount to the seller, using the fee rate captured on the invoice at creation (`feeRate`) — not the current global rate, so a later change to the global rate never affects an already-created invoice.

```solidity
function release(uint216 _invoiceId) public;
```

**Parameters**

|     Name     |    Type   |                     Description                     |
| :----------: | :-------: | :-------------------------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice for which funds are released. |

#### refundBuyer

Refunds the buyer of a specific invoice when the seller fails to act in time.

Invoice must be in `PAID` state and the decision window (`expiresAt`) must have elapsed, otherwise reverts with `InvoiceNotEligibleForRefund`. Attempts to withdraw the price to the buyer; on success the invoice transitions to `REFUNDED`, is removed from the heap, and its balance is zeroed. On withdrawal failure, the retry counter is incremented and the invoice stays `PAID` for a future retry; once retries would exceed `MAX_WITHDRAWAL_RETRIES`, the invoice is instead removed from the heap and transitions to `LOCKED` (recoverable only via [releaseLocked](#releaselocked)). Guarded by `nonReentrant`.

```solidity
function refundBuyer(uint216 _invoiceId) public nonReentrant;
```

**Parameters**

|     Name     |    Type   |              Description              |
| :----------: | :-------: | :-----------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to be refunded. |

#### releaseLocked

Recovers funds from a permanently locked invoice by sending them to a specified recipient.

Only callable by an authorized address (owner or storage contract). Valid only for invoices in the `LOCKED` state. Withdraws the caller-supplied `_amount` from escrow to `_recipient` and decrements the invoice's tracked `balance` by that amount — the caller is responsible for supplying the correct amount (use `getInvoiceData` to check the remaining escrow balance).

**Note:** the code only flips the invoice's `state` to `RELEASED` on an in-memory copy when `_amount` exactly equals the remaining `balance`, and that assignment is never written back to storage — so on-chain, the invoice's persisted `state` remains `LOCKED` even after a full recovery. Only `balance` is actually updated in storage.

```solidity
function releaseLocked(uint216 _invoiceId, address _recipient, uint256 _amount) external onlyAuthorized;
```

**Parameters**

|     Name     |    Type   |                           Description                          |
| :----------: | :-------: | :------------------------------------------------------------: |
| `_invoiceId` | `uint216` |             The ID of the locked invoice.                      |
| `_recipient` | `address` | The address to receive the recovered funds.                    |
|  `_amount`   | `uint256` | The amount to transfer from the escrow.                        |

#### hasDueTasks

Returns whether any scheduled invoice task is due for processing. Read by the CRE workflow on each cron tick to decide whether to submit a report onchain — the offchain analogue of the old `checkUpkeep`.

```solidity
function hasDueTasks() external view returns (bool dueTasksExist);
```

**Returns**

|       Name       |  Type  |                        Description                       |
| :----------------: | :----: | :-----------------------------------------------------------: |
| `dueTasksExist` | `bool` | True when the earliest scheduled task is due. |

#### onReport

Handles a verified report delivered by the Chainlink CRE (Keystone) forwarder and processes due invoice tasks. This is the CRE replacement for the old Chainlink Automation `performUpkeep` entry point. The report payload itself is ignored — delivery of a verified report is itself the trigger.

Only callable by the configured `forwarder` (reverts with `NotAuthorized` otherwise). Also validates that the report metadata carries the configured `workflowOwner`, reverting with `UnauthorizedWorkflowOwner` if it doesn't — so a workflow deployed by a different owner can't trigger processing through a shared forwarder. Guarded by `nonReentrant`.

```solidity
function onReport(bytes calldata _metadata, bytes calldata _report) external nonReentrant;
```

**Parameters**

|    Name     |   Type  |                                                        Description                                                       |
| :-----------: | :-----: | :--------------------------------------------------------------------------------------------------------------------------: |
| `_metadata` | `bytes` | Workflow identity data: `workflowId` (32 bytes), `workflowName` (10 bytes), `workflowOwner` (20 bytes), `reportId` (2 bytes), tightly packed. |
|   `_report`  | `bytes` | The ABI-encoded report payload produced by the workflow. Unused by this handler. |

#### processDueTasks

Processes due invoice tasks (auto-release and auto-refund) within the gas threshold. Owner-only manual fallback for the CRE workflow path (`onReport`), useful if the CRE workflow or forwarder is unavailable.

Only callable by the owner; reverts with `NotAuthorized` otherwise. Guarded by `nonReentrant`.

```solidity
function processDueTasks() external nonReentrant;
```

#### supportsInterface

ERC-165 introspection, so the CRE forwarder can confirm this contract implements `IReceiver` before delivering a report.

```solidity
function supportsInterface(bytes4 _interfaceId) external pure returns (bool supported);
```

**Parameters**

|      Name      |   Type   |            Description           |
| :---------------: | :------: | :----------------------------------: |
| `_interfaceId` | `bytes4` | The interface ID to check. |

**Returns**

|    Name    |  Type  |                                     Description                                    |
| :-----------: | :----: | :---------------------------------------------------------------------------------: |
| `supported` | `bool` | True for `IReceiver` and `IERC165` interface IDs; false otherwise. |

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

Calculates the fee based on the provided amount and the *current* global fee rate.

Fee rate is expressed in basis points (1% = 100). This quotes the rate that would be captured by an invoice created right now — it does **not** reflect what a given existing invoice will actually be charged on release, since `release`/`refundBuyer`/the automated release path all use the fee rate snapshotted on the invoice at creation (`feeRate`), not the current global rate.

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
function setMinimumInvoiceValue(uint256 _newMinimumInvoiceValue) public onlyAuthorized;
```

**Parameters**

|            Name               |    Type   |               Description               |
| :---------------------------: | :-------: | :-------------------------------------: |
| `_newMinimumInvoiceValue`     | `uint256` | The new minimum invoice value to set (in wei). |

#### setForwarderAddress

Updates the address of the CRE (Keystone) forwarder contract that delivers workflow reports via `onReport`.

Only callable by the owner or the storage contract. Only the configured forwarder may call `onReport`.

```solidity
function setForwarderAddress(address _forwarderAddress) external onlyAuthorized;
```

**Parameters**

|         Name        |    Type   |                  Description                  |
| :-----------------: | :-------: | :-------------------------------------------: |
| `_forwarderAddress` | `address` | The new forwarder contract address to be set. |

#### setWorkflowOwner

Updates the CRE workflow owner authorized to trigger `onReport`. Reports whose metadata carries a different workflow owner are rejected, so workflows deployed by other owners can't trigger task processing through the shared forwarder.

Only callable by the owner or the storage contract.

```solidity
function setWorkflowOwner(address _workflowOwner) external onlyAuthorized;
```

**Parameters**

|       Name       |    Type   |                     Description                    |
| :---------------: | :-------: | :----------------------------------------------------: |
| `_workflowOwner` | `address` | The address that owns the authorized CRE workflow. |

#### setDecisionWindow

Updates the decision window sellers have to accept payments after buyer payment.

Only callable by the owner or the storage contract. Reverts with `InvalidDecisionWindow` if `_newDecisionWindow` is zero.

```solidity
function setDecisionWindow(uint256 _newDecisionWindow) external onlyAuthorized;
```

**Parameters**

|         Name         |    Type   |             Description             |
| :------------------: | :-------: | :---------------------------------: |
| `_newDecisionWindow` | `uint256` | The new decision window in seconds. |

#### getForwarder

Returns the address of the configured CRE forwarder contract.

```solidity
function getForwarder() external view returns (address forwarderAddress);
```

**Returns**

|        Name        |    Type   |            Description            |
| :----------------: | :-------: | :-------------------------------: |
| `forwarderAddress` | `address` | The configured forwarder address. |

#### getWorkflowOwner

Returns the CRE workflow owner authorized to trigger `onReport`.

```solidity
function getWorkflowOwner() external view returns (address workflowOwnerAddress);
```

**Returns**

|          Name          |    Type   |              Description              |
| :------------------------: | :-------: | :----------------------------------------: |
| `workflowOwnerAddress` | `address` | The authorized workflow owner address. |

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
function getInvoiceData(uint216 _invoiceId) public view returns (Invoice memory i);
```

**Parameters**

|     Name     |    Type   |       Description      |
| :----------: | :-------: | :--------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice. |

**Returns**

| Name |    Type   |    Description    |
| :--: | :-------: | :---------------: |
| `i`  | `Invoice` | The invoice data. |

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

### Structs

#### Invoice

Represents an invoice between a buyer and seller, with escrow, timestamps, and status tracking.

```solidity
struct Invoice {
    uint216 invoiceNonce;
    uint40 createdAt;
    uint40 paidAt;
    uint40 releaseAt;
    uint40 invalidateAt;
    uint40 expiresAt;
    uint8 state;
    uint8 withdrawalRetries;
    uint16 feeRate;
    address seller;
    address buyer;
    address escrow;
    uint256 price;
    uint256 balance;
}
```

|        Field        |    Type   |                                                              Description                                                             |
| :---------------------: | :-------: | :--------------------------------------------------------------------------------------------------------------------------------------: |
|    `invoiceNonce`    | `uint216` |                                A unique identifier assigned to this invoice, typically sequentially.                                |
|      `createdAt`     |  `uint40` |                                       The Unix timestamp when the invoice was created.                                        |
|       `paidAt`       |  `uint40` |                                       The Unix timestamp when the payment was completed.                                      |
|      `releaseAt`     |  `uint40` |                              The timestamp when funds in escrow can be released to the seller.                                |
|    `invalidateAt`    |  `uint40` |                        The timestamp after which the invoice is considered invalid if unpaid.                                 |
|      `expiresAt`     |  `uint40` | The timestamp after which the seller can no longer take action (accept/reject), and the buyer is refunded. |
|        `state`       |  `uint8`  |                                          The current state of the invoice.                                                    |
| `withdrawalRetries`  |  `uint8`  |     Number of failed `IEscrow.withdraw` attempts by the automation path. Packed with `state` in the same storage slot.        |
|       `feeRate`      | `uint16`  | The platform fee rate (in basis points) captured at invoice creation. Releases always charge this rate, so later changes to the global fee rate do not affect existing invoices. |
|       `seller`       | `address` |                                     The address of the seller of the invoice.                                                 |
|        `buyer`       | `address` |                                     The address of the buyer of the invoice.                                                  |
|       `escrow`       | `address` |                    The address of the escrow contract managing the funds for this invoice.                                    |
|        `price`       | `uint256` |                                       The total price of the invoice in wei.                                                  |
|       `balance`      | `uint256` |                    The current amount held in escrow, net of any fees deducted upon acceptance. Zeroed on release or refund.   |

### Events

#### InvoiceCreated

Emitted when a new invoice is created.

```solidity
event InvoiceCreated(uint216 indexed invoiceId, Invoice invoice);
```

| Name        | Type      | Description                                                                          |
| :----------: | :-------: | :-----------------------------------------------------------------------------------: |
| `invoiceId` | `uint216` | The unique identifier for the created invoice.                                       |
| `invoice`   | `Invoice` | The full invoice struct containing buyer, price, timestamps, state, and metadata.    |

#### InvoicePaid

Emitted when an invoice payment is made.

```solidity
event InvoicePaid(uint216 indexed invoiceId, address indexed buyer, uint256 indexed amountPaid, uint40 expiresAt);
```

| Name        | Type      | Description                                                                                         |
| :----------: | :-------: | :--------------------------------------------------------------------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the paid invoice.                                                                  |
| `buyer`     | `address` | The address of the buyer who paid.                                                                  |
| `amountPaid`| `uint256` | The amount paid towards the invoice in wei.                                                         |
| `expiresAt` | `uint40`  | The timestamp by which the seller must accept or reject; after this the buyer is eligible for refund. |

#### InvoiceRejected

Emitted when an invoice is rejected by the seller.

```solidity
event InvoiceRejected(uint216 indexed invoiceId, uint256 amount);
```

| Name        | Type      | Description                             |
| :----------: | :-------: | :--------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the rejected invoice.  |
| `amount`    | `uint256` | The escrow balance refunded to the buyer in wei. |

#### InvoiceRefunded

Emitted when an invoice is refunded to the buyer.

```solidity
event InvoiceRefunded(uint216 indexed invoiceId, uint256 amount);
```

| Name        | Type      | Description                             |
| :----------: | :-------: | :--------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the refunded invoice.  |
| `amount`    | `uint256` | The escrow balance refunded to the buyer in wei. |

#### InvoiceAccepted

Emitted when an invoice is accepted by the seller.

```solidity
event InvoiceAccepted(uint216 indexed invoiceId);
```

| Name        | Type      | Description                             |
| :----------: | :-------: | :--------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the accepted invoice.  |

#### InvoiceCanceled

Emitted when an invoice is canceled.

```solidity
event InvoiceCanceled(uint216 indexed invoiceId);
```

| Name        | Type      | Description                             |
| :----------: | :-------: | :--------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the canceled invoice.  |

#### InvoiceReleased

Emitted when an invoice is released (funds disbursed from escrow).

```solidity
event InvoiceReleased(uint216 indexed invoiceId, uint256 sellerAmount, uint256 fee);
```

| Name           | Type      | Description                             |
| :--------------: | :-------: | :--------------------------------------: |
| `invoiceId`    | `uint216` | The unique ID of the released invoice.  |
| `sellerAmount` | `uint256` | The net amount transferred to the seller, after fees. |
| `fee`          | `uint256` | The platform fee deducted and sent to the fee receiver. |

#### UpdateHoldPeriod

Emitted when the hold period of a given invoice is updated to a new timestamp.

```solidity
event UpdateHoldPeriod(uint216 indexed invoiceId, uint256 indexed releaseDueTimestamp);
```

| Name                   | Type      | Description                                                  |
| :---------------------: | :-------: | :-----------------------------------------------------------: |
| `invoiceId`            | `uint216` | The key of the invoice whose hold period was updated.        |
| `releaseDueTimestamp`  | `uint256` | The new hold period expressed as a UNIX timestamp.           |

#### WithdrawalRetried

Emitted when an automated withdrawal attempt fails and is retried.

```solidity
event WithdrawalRetried(uint216 indexed invoiceId, address indexed recipient, uint256 amount, uint8 attempt);
```

| Name        | Type      | Description                                         |
| :----------: | :-------: | :--------------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the invoice whose withdrawal was retried. |
| `recipient` | `address` | The address the withdrawal was attempted to.        |
| `amount`    | `uint256` | The amount that failed to transfer.                 |
| `attempt`   | `uint8`   | The retry attempt number.                           |

#### LockedPaymentRecovered

Emitted when a locked invoice's funds are manually recovered by an authorized address.

```solidity
event LockedPaymentRecovered(uint216 indexed invoiceId, address indexed recipient, uint256 amount);
```

| Name        | Type      | Description                                       |
| :----------: | :-------: | :------------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the locked invoice that was recovered.  |
| `recipient` | `address` | The address that received the recovered funds.    |
| `amount`    | `uint256` | The amount of funds recovered.                    |

#### TransferFailed

Emitted when a transfer from the escrow fails.

```solidity
event TransferFailed(uint216 indexed invoiceId, address indexed recipient, uint256 amount);
```

| Name        | Type      | Description                                  |
| :----------: | :-------: | :-------------------------------------------: |
| `invoiceId` | `uint216` | The ID of the invoice associated with the transfer. |
| `recipient` | `address` | The address the transfer was attempted to.   |
| `amount`    | `uint256` | The amount that failed to transfer.          |

### Errors

| Error | Description |
| :----: | :----------: |
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
| `InvoiceNotEligibleForRefund()` | Thrown when a refund to the buyer cannot be issued (invoice not `PAID` or decision window not yet elapsed). |
| `HoldPeriodHasNotBeenExceeded()` | Thrown when the hold period for an invoice has not yet been exceeded. |
| `EscrowWithdrawFailed()` | Thrown when the escrow withdrawal fails during a manual release, reject, or refund. |
| `UnauthorizedWorkflowOwner(address _workflowOwner)` | Thrown when a CRE report's metadata does not carry the authorized workflow owner. |
