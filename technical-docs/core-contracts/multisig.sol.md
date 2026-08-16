# MultiSig.sol

The MultiSig contract is a multisignature governance contract for privileged payment processor administration. It replaces a single-owner key with collective authorization across a defined signer set. Administrative calls (fee updates, decision windows, locked fund recovery, oracle updates, etc.) to [SimplePaymentProcessor.sol](simplepaymentprocessor.sol.md), [IntermediatedPaymentProcessor.sol](intermediatedpaymentprocessor.sol.md), and [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md) must pass through this contract via propose → approve → execute.

Signer management and threshold updates are self-referential: `addSigner`, `removeSigner`, `updateThreshold`, and `cancelTransaction` are only callable by the MultiSig contract itself, so they can only be triggered as the executed result of a transaction proposed and approved against the MultiSig contract, going through the same flow as any other admin call.

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/MultiSig.sol)

### State Variables

#### PROPOSED / APPROVED / EXECUTED / CANCELED

Numeric lifecycle codes used by `Transaction.status`, imported from `constants/MultiSig.sol`.

```solidity
uint8 constant PROPOSED = 1;
uint8 constant APPROVED = 2;
uint8 constant EXECUTED = 3;
uint8 constant CANCELED = 4;
```

#### MINIMUM\_THRESHOLD

Minimum approval threshold enforced at deployment and on `updateThreshold`.

```solidity
uint8 constant MINIMUM_THRESHOLD = 2;
```

#### MINIMUM\_SIGNERS

Minimum number of signers required at deployment.

```solidity
uint8 constant MINIMUM_SIGNERS = 2;
```

### Functions

#### constructor

Deploys the multisig with an initial signer set and approval threshold.

```solidity
constructor(address[] memory _initialSigners, uint256 _initialThreshold);
```

**Parameters**

|         Name         |     Type    |                                     Description                                    |
| :-------------------: | :---------: | :---------------------------------------------------------------------------------: |
|   `_initialSigners`   | `address[]` |          Array of addresses to register as signers (minimum 2).                     |
| `_initialThreshold`   |  `uint256`  | Minimum approvals required for execution; must be >= 2 and <= `_initialSigners.length`. |

#### proposeTransaction

Proposes a new administrative transaction targeting a payment processor. Only callable by a registered signer. The proposer's approval is recorded immediately, so a fresh proposal starts with an `approvalCount` of 1.

```solidity
function proposeTransaction(address _target, uint256 _value, bytes calldata _data) external returns (bytes32 txHash);
```

**Parameters**

|   Name    |   Type    |                            Description                            |
| :-------: | :-------: | :-----------------------------------------------------------------: |
| `_target` | `address` | Payment processor address (SimplePaymentProcessor or IntermediatedPaymentProcessor). |
| `_value`  | `uint256` |               ETH to forward; must be 0 for admin calls.             |
| `_data`   |  `bytes`  |             ABI-encoded payment processor admin function call.      |

**Returns**

|   Name   |   Type    |                        Description                       |
| :------: | :-------: | :---------------------------------------------------------: |
| `txHash` | `bytes32` | keccak256 hash of the transaction, used as its identifier. |

#### approveTransaction

Records the caller's approval for a proposed transaction. Only callable by a registered signer who has not already approved. Automatically transitions the transaction to `APPROVED` (emitting `TransactionApproved`) once the approval count meets the threshold.

```solidity
function approveTransaction(bytes32 _txHash) external;
```

**Parameters**

|   Name    |   Type    |                    Description                    |
| :-------: | :-------: | :--------------------------------------------------: |
| `_txHash` | `bytes32` | Identifier of the transaction to approve. |

#### executeTransaction

Executes an `APPROVED` transaction by forwarding the encoded call to the target contract. Only callable by a registered signer.

```solidity
function executeTransaction(bytes32 _txHash) external returns (bytes memory);
```

**Parameters**

|   Name    |   Type    |                     Description                    |
| :-------: | :-------: | :---------------------------------------------------: |
| `_txHash` | `bytes32` | Identifier of the approved transaction to execute. |

**Returns**

| Name |   Type   |                 Description                |
| :--: | :------: | :-------------------------------------------: |
|      | `bytes`  | The raw bytes returned by the target call. |

#### cancelTransaction

Cancels a `PROPOSED` or `APPROVED` transaction, preventing execution. Only callable by the multisig contract itself via an executed transaction.

```solidity
function cancelTransaction(bytes32 _txHash) external;
```

**Parameters**

|   Name    |   Type    |                  Description                 |
| :-------: | :-------: | :---------------------------------------------: |
| `_txHash` | `bytes32` | Identifier of the transaction to cancel. |

#### addSigner

Adds a new signer to the authorized set. Only callable by the multisig contract itself via an executed transaction.

```solidity
function addSigner(address _signer) external;
```

**Parameters**

|   Name    |   Type    |                     Description                    |
| :-------: | :-------: | :----------------------------------------------------: |
| `_signer` | `address` | Address to register; must not already be a signer. |

#### removeSigner

Removes a signer from the authorized set. Only callable by the multisig contract itself via an executed transaction. Reverts if removal would leave fewer signers than the current threshold.

```solidity
function removeSigner(address _signer) external;
```

**Parameters**

|   Name    |   Type    |                Description               |
| :-------: | :-------: | :-----------------------------------------: |
| `_signer` | `address` | Address to deregister; must be a current signer. |

#### updateThreshold

Updates the minimum approval threshold. Only callable by the multisig contract itself via an executed transaction.

```solidity
function updateThreshold(uint256 _newThreshold) external;
```

**Parameters**

|      Name       |   Type    |                              Description                             |
| :--------------: | :-------: | :---------------------------------------------------------------------: |
| `_newThreshold`  | `uint256` | New required approval count; must be >= `MINIMUM_THRESHOLD` and <= signer count. |

#### getTransaction

Returns the full `Transaction` struct for a given hash.

```solidity
function getTransaction(bytes32 _txHash) external view returns (Transaction memory);
```

**Parameters**

|   Name    |   Type    |            Description           |
| :-------: | :-------: | :---------------------------------: |
| `_txHash` | `bytes32` | The transaction identifier. |

#### hasApproved

Returns whether a specific signer has approved a transaction.

```solidity
function hasApproved(bytes32 _txHash, address _signer) external view returns (bool);
```

**Parameters**

|   Name    |   Type    |               Description              |
| :-------: | :-------: | :----------------------------------------: |
| `_txHash` | `bytes32` |        The transaction identifier.      |
| `_signer` | `address` | The signer address to check. |

#### isSigner

Returns whether an address is a registered signer.

```solidity
function isSigner(address _account) external view returns (bool);
```

**Parameters**

|    Name    |   Type    |         Description        |
| :---------: | :-------: | :----------------------------: |
| `_account` | `address` | The address to check. |

#### getThreshold

Returns the current approval threshold.

```solidity
function getThreshold() external view returns (uint256);
```

#### getSignerCount

Returns the total number of registered signers.

```solidity
function getSignerCount() external view returns (uint256);
```

#### getNonce

Returns the current nonce value, equal to the total number of transactions proposed.

```solidity
function getNonce() external view returns (uint256);
```

### Structs

#### Transaction

Represents a proposed administrative transaction awaiting or having completed multisig approval.

```solidity
struct Transaction {
    address target;
    uint256 value;
    bytes data;
    uint256 nonce;
    uint8 status;
    uint256 approvalCount;
}
```

|      Field      |   Type    |                                       Description                                      |
| :---------------: | :-------: | :----------------------------------------------------------------------------------------: |
|     `target`     | `address` |                       Payment processor contract to call.                                |
|      `value`     | `uint256` |                        ETH value to forward (0 for admin calls).                          |
|       `data`     |  `bytes`  |                            ABI-encoded admin function call.                                |
|      `nonce`     | `uint256` |                       Unique identifier preventing replay.                                |
|      `status`    |  `uint8`  | Current lifecycle state: `PROPOSED` (1), `APPROVED` (2), `EXECUTED` (3), or `CANCELED` (4). |
| `approvalCount`  | `uint256` |               Cumulative approval count for this transaction hash.                        |

### Events

#### TransactionProposed

Emitted when a new transaction is proposed by a signer.

```solidity
event TransactionProposed(bytes32 indexed txHash, address indexed target, uint256 value, bytes data, uint256 nonce, address indexed proposer);
```

|    Name    |    Type   |                    Description                   |
| :---------: | :-------: | :--------------------------------------------------: |
|  `txHash`  | `bytes32` | The keccak256 hash identifying this transaction. |
|  `target`  | `address` |        The payment processor contract address.   |
|   `value`  | `uint256` |             ETH value to forward.                |
|    `data`  |  `bytes`  |              ABI-encoded calldata.                |
|   `nonce`  | `uint256` |             Unique proposal nonce.                |
| `proposer` | `address` |     Address of the signer who proposed.           |

#### ApprovalAdded

Emitted when a signer records their approval for a transaction.

```solidity
event ApprovalAdded(bytes32 indexed txHash, address indexed approver, uint256 approvalCount);
```

|      Name       |    Type   |                Description               |
| :---------------: | :-------: | :------------------------------------------: |
|     `txHash`     | `bytes32` |    The transaction hash that was approved.   |
|    `approver`    | `address` |          The signer who approved.            |
| `approvalCount`  | `uint256` |        The updated total approval count.     |

#### TransactionApproved

Emitted when a transaction accumulates enough approvals to reach `APPROVED` status.

```solidity
event TransactionApproved(bytes32 indexed txHash);
```

|   Name   |    Type   |                       Description                      |
| :-------: | :-------: | :--------------------------------------------------------: |
| `txHash` | `bytes32` | The transaction hash that reached the approval threshold. |

#### TransactionExecuted

Emitted when an approved transaction is executed.

```solidity
event TransactionExecuted(bytes32 indexed txHash, address indexed executor);
```

|    Name    |    Type   |                Description               |
| :---------: | :-------: | :------------------------------------------: |
|  `txHash`  | `bytes32` |    The transaction hash that was executed.   |
| `executor` | `address` | The signer who triggered execution.          |

#### TransactionCanceled

Emitted when a `PROPOSED` or `APPROVED` transaction is canceled via a multisig-executed transaction.

```solidity
event TransactionCanceled(bytes32 indexed txHash);
```

|   Name   |    Type   |             Description            |
| :-------: | :-------: | :-------------------------------------: |
| `txHash` | `bytes32` | The transaction hash that was canceled. |

#### SignerAdded

Emitted when a new signer is added (at deployment, or via a multisig-executed transaction).

```solidity
event SignerAdded(address indexed signer);
```

|   Name   |    Type   |             Description            |
| :-------: | :-------: | :------------------------------------: |
| `signer` | `address` | The address added as a signer. |

#### SignerRemoved

Emitted when a signer is removed via a multisig-executed transaction.

```solidity
event SignerRemoved(address indexed signer);
```

|   Name   |    Type   |               Description              |
| :-------: | :-------: | :----------------------------------------: |
| `signer` | `address` | The address removed from the signer set. |

#### ThresholdUpdated

Emitted when the approval threshold is updated via a multisig-executed transaction.

```solidity
event ThresholdUpdated(uint256 oldThreshold, uint256 newThreshold);
```

|      Name       |    Type   |             Description            |
| :---------------: | :-------: | :------------------------------------: |
| `oldThreshold`   | `uint256` |      The previous threshold value.     |
| `newThreshold`   | `uint256` |          The new threshold value.      |

### Errors

| Error | Description |
| :----: | :----------: |
| `NotSigner()` | Thrown when the caller is not a registered signer. |
| `NotSelf()` | Thrown when the caller is not the multisig contract itself. |
| `TransactionDoesNotExist()` | Thrown when the referenced transaction hash does not exist. |
| `TransactionNotProposedOrApproved()` | Thrown when a cancel is attempted on a transaction that is not `PROPOSED` or `APPROVED`. |
| `TransactionNotProposed()` | Thrown when an operation requires `PROPOSED` status but the transaction is not proposed. |
| `TransactionNotApproved()` | Thrown when execution is attempted but the transaction has not reached `APPROVED` status. |
| `AlreadyApproved()` | Thrown when approval is attempted on a transaction that is no longer `PROPOSED`. |
| `AlreadyApprovedByThisSigner()` | Thrown when a signer attempts to approve a transaction they have already approved. |
| `AlreadyExecuted()` | Thrown when the transaction has already been executed. |
| `ThresholdCannotBeZero()` | Thrown when a threshold update of zero is attempted. |
| `ExecutionFailed()` | Thrown when the low-level call to the payment processor fails. |
| `InvalidThreshold()` | Thrown when a provided threshold is zero, exceeds signer count, or is otherwise invalid. |
| `InvalidTarget()` | Thrown when the target address is the zero address. |
| `SignerCountBelowThreshold()` | Thrown when removing a signer would bring the signer count below the current threshold, or when a new threshold exceeds the signer count. |
| `AlreadyASigner()` | Thrown when the provided address is already a registered signer. |
| `NotASigner()` | Thrown when the provided address is not a registered signer. |
| `InsufficientSigners()` | Thrown when the initial signer array has fewer than two entries. |
