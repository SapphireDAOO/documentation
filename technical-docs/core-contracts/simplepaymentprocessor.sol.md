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

#### notes

Notes contract used for encrypted invoice notes.

```solidity
INotes private notes
```

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

#### CANCELLED

Status code representing that a payment or transaction has been cancelled.

```solidity
uint8 public constant CANCELLED = REJECTED + 1
```

#### REFUNDED

Status code representing that a payment has been refunded to the payer.

```solidity
uint8 public constant REFUNDED = CANCELLED + 1
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

#### heap

Internal min-heap used to efficiently manage scheduled invoice tasks by release time.

```solidity
TaskQueueLib.Heap private heap
```

#### ppStorage

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public immutable ppStorage
```

#### minimumInvoiceValue

The minimum allowed value (in wei) required to create a new invoice.

```solidity
uint256 private minimumInvoiceValue
```

#### decisionWindow

The window of time allowed for accepting a transaction after creation.

```solidity
uint256 public decisionWindow
```

#### forwarder

Address of the forwarder contract responsible for calling performUpkeep.

```solidity
address private forwarder
```

#### invoices

Stores the `Invoice` structs, keyed by a unique invoice ID.

The key is an unsigned integer representing the invoice ID, and the value is an `Invoice` struct that contains detailed information such as the creator, payer, status, amount, escrow address, timestamps, etc.

```solidity
mapping(uint216 invoiceId => Invoice invoiceData) private invoices
```

#### index

Maps task or invoice ID to its 1-based index position in the heap.

A value of 0 means the task is not present in the heap

```solidity
mapping(uint216 invoiceId => uint256 key) private index
```

### Functions

#### onlyAuthorized

Restricts access to the payment processor owner or storage contract.

Reverts with NotAuthorized if the caller is not permitted.

```solidity
modifier onlyAuthorized() ;
```

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
function createInvoice(uint256 _invoicePrice, bytes memory _storageRef, bool _share)
    external
    returns (uint216 invoiceId);
```

**Parameters**

|       Name      |    Type   |                       Description                      |
| :-------------: | :-------: | :----------------------------------------------------: |
| `_invoicePrice` | `uint256` |            The price of the invoice in wei.            |
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

|       Name      |    Type   |                             Description                             |
| :-------------: | :-------: | :-----------------------------------------------------------------: |
| `escrowAddress` | `address` | escrow The address of the escrow contract created for this payment. |

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

```solidity
function release(uint216 _invoiceId) public;
```

**Parameters**

|     Name     |    Type   |                     Description                     |
| :----------: | :-------: | :-------------------------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice for which funds are released. |

#### refundBuyer

Refunds the seller of a specific invoice.

This function allows the buyer to be refund if the acceptance window has not been exceeded and the invoice is eligible for a refund. The refund will be processed through the escrow contract.

```solidity
function refundBuyer(uint216 _invoiceId) public;
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

#### \_validateInvoiceStateForPaymentDecision

Validates that the caller can accept or reject a payment.

Ensures caller is the seller and invoice is within the decision window.

```solidity
function _validateInvoiceStateForPaymentDecision(Invoice memory _invoice) internal view;
```

**Parameters**

|    Name    |    Type   |          Description          |
| :--------: | :-------: | :---------------------------: |
| `_invoice` | `Invoice` | The invoice data to validate. |

#### \_release

Attempts to release the specified invoice if it is eligible.

This function performs all the checks required to determine whether the invoice can be released, updates the invoice status, removes it from the scheduling heap, and triggers the escrow payout.

```solidity
function _release(uint216 _invoiceId) internal returns (uint256 status);
```

**Parameters**

|     Name     |    Type   |            Description            |
| :----------: | :-------: | :-------------------------------: |
| `_invoiceId` | `uint216` | The ID of the invoice to release. |

**Returns**

|   Name   |    Type   |                                                                                                                   Description                                                                                                                   |
| :------: | :-------: | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| `status` | `uint256` | A status code from TaskQueueLib indicating the outcome: - SUCCESSFUL (3): Invoice was released and removed from heap. - NOT\_ELIGIBLE\_FOR\_RELEASE (1): Invoice not accepted or not yet due. - ERROR (2): Invalid index or heap inconsistency. |

#### \_computeInvoiceId

Computes a unique order ID for an invoice using buyer, invoice ID, timestamp, and contract address.

This function ensures the order ID is non-deterministic, even for repeated inputs.

```solidity
function _computeInvoiceId(address _buyer, uint256 _invoiceNonce) internal view returns (uint216 invoiceId);
```

**Parameters**

|       Name      |    Type   |                    Description                   |
| :-------------: | :-------: | :----------------------------------------------: |
|     `_buyer`    | `address` |             The address of the buyer.            |
| `_invoiceNonce` | `uint256` | The invoice identifier provided during creation. |

**Returns**

|     Name    |    Type   |                      Description                     |
| :---------: | :-------: | :--------------------------------------------------: |
| `invoiceId` | `uint216` | The keccak256 hash representing the unique order ID. |

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

#### \_isAuthorized

Internal function to validate whether the caller is authorized.

Reverts if the caller is not the contract owner or the PaymentProcessorStorage contract itself. Can only be called by either the owner of the PaymentProcessor contract or the storage contract address.

```solidity
function _isAuthorized() internal view;
```

#### setInvoiceReleaseTime

Sets a custom hold period for a specific invoice.

Overrides the default hold period for this invoice.

```solidity
function setInvoiceReleaseTime(uint216 _invoiceId, uint32 _holdPeriod) external;
```

**Parameters**

|      Name     |    Type   |           Description           |
| :-----------: | :-------: | :-----------------------------: |
|  `_invoiceId` | `uint216` |      The ID of the invoice.     |
| `_holdPeriod` |  `uint32` | The new hold period in seconds. |

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

Should only be callable by the contract owner or an authorized role.

```solidity
function setMinimumInvoiceValue(uint256 _newMinimumInvoiceValue) public onlyAuthorized;
```

**Parameters**

|            Name           |    Type   | Description |
| :-----------------------: | :-------: | :---------: |
| `_newMinimumInvoiceValue` | `uint256` |             |

#### setForwarderAddress

Updates the address of the forwarder contract used for relayed or automated calls.

```solidity
function setForwarderAddress(address _forwarderAddress) external onlyAuthorized;
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

|       Name       |    Type   |    Description    |
| :--------------: | :-------: | :---------------: |
| `invoiceDetails` | `Invoice` | The invoice data. |

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

