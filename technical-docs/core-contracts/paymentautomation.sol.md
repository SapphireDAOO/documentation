# PaymentAutomation.sol

Keeper adapter that triggers automated release and refund of due invoices on [SimplePaymentProcessor.sol](simplepaymentprocessor.sol.md). This contract owns no queue, no invoice state, and no funds: the scheduling heap and every state transition stay in `SimplePaymentProcessor`; this adapter only reads `hasDueTasks()` and calls `processDueTasks()` on it, wrapping that pair in the entrypoints two keeper networks expect:

- **Chainlink CRE**: `onReport`, called by the Keystone forwarder with a DON-signed report. The forwarder confirms this contract advertises `IReceiver` over ERC-165 before delivering.
- **Gelato Web3 Functions**: `checker`, polled offchain, which names `processDueTasks` as the exec target.

Only one keeper network is meant to be active at a time; the second is redundancy, to be switched on if the primary stalls or is decommissioned. Both paths converge on the same processor call, so running both at once is still safe: whichever fires first drains the queue and the other finds nothing due. `SimplePaymentProcessor` holds this adapter's address as an immutable set at deployment, so the pairing is fixed; changing it means redeploying both contracts (see [MasterDeployer.sol](masterdeployer.sol.md#deploycore)).

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/PaymentAutomation.sol)

### State Variables

#### PROCESSOR

The payment processor whose due-task queue this contract drives.

```solidity
ISimplePaymentProcessor public immutable PROCESSOR
```

#### PP_STORAGE

Reference to the external Payment Processor storage contract, used for owner checks.

```solidity
IPaymentProcessorStorage public immutable PP_STORAGE
```

`PROCESSOR`, `PP_STORAGE`, `FORWARDER`, and `WORKFLOW_OWNER` are all immutable. There is no way to re-point any of them after deployment; changing one means redeploying this contract (and, since `PROCESSOR` is fixed on both sides, `SimplePaymentProcessor` too).

### Functions

#### constructor

Wires the adapter to the storage contract and the keeper identities it trusts.

Every value here is fixed at construction. The processor is not a constructor argument: it is read back from the deployer via `IPendingProcessorProvider.pendingProcessor()`, which keeps this contract's init code free of the processor's address so its own address stays predictable via CREATE2. The deployer must therefore implement that interface (in practice, [MasterDeployer.sol](masterdeployer.sol.md), which resolves it to the already-deployed `SimplePaymentProcessor`). Reverts with `InvalidAddress` if the storage address is the zero address, or if the resolved processor address is the zero address.

```solidity
constructor(address _paymentProcessorStorageAddress, address _forwarderAddress, address _workflowOwner);
```

**Parameters**

|              Name              |    Type   |                          Description                          |
| :-------------------------------: | :-------: | :-----------------------------------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The address of the shared payment processor storage contract. |
|       `_forwarderAddress`       | `address` | The CRE forwarder allowed to deliver reports to `onReport`. |
|         `_workflowOwner`        | `address` | The CRE workflow owner carried in report metadata. |

#### onReport

Handles a verified report delivered by the CRE forwarder and processes due invoice tasks. The report payload is ignored; delivery of a verified report is itself the trigger.

Only callable by the configured `FORWARDER` (reverts with `NotAuthorized` otherwise). Also validates that the report metadata carries the configured `WORKFLOW_OWNER`, reverting with `UnauthorizedWorkflowOwner` if it doesn't, so a workflow deployed by a different owner can't trigger processing through a shared forwarder. On success, calls `PROCESSOR.processDueTasks()` and emits `DueTasksProcessed` tagged with `CRE_SOURCE`. This function does not itself check whether the system is paused; if it is, the downstream `PROCESSOR.processDueTasks()` call reverts with `ContractPaused` rather than no-op'ing (unlike `hasDueTasks`/`checker`, which report no work so a well-behaved keeper never gets this far).

```solidity
function onReport(bytes calldata _metadata, bytes calldata _report) external;
```

**Parameters**

|    Name     |   Type  |                                                        Description                                                       |
| :-----------: | :-----: | :--------------------------------------------------------------------------------------------------------------------------: |
| `_metadata` | `bytes` | Workflow identity data: `workflowId` (32 bytes), `workflowName` (10 bytes), `workflowOwner` (20 bytes), `reportId` (2 bytes), tightly packed. |
|   `_report`  | `bytes` | The ABI-encoded report payload produced by the workflow. Unused by this handler. |

#### processDueTasks

Drains the processor's due invoice tasks (auto-release and auto-refund). Permissionless: this is the Gelato exec target named by `checker`, and doubles as the manual fallback when neither keeper network is running. The call is a no-op when nothing is due, and the processor enforces every state and authorization rule itself, so an arbitrary caller can only pay for work the keeper would have done anyway. Reverts with `ContractPaused` (from the processor) if the system is paused; this function doesn't check pause state itself, it just relays the call.

Calls `processor.processDueTasks()` and emits `DueTasksProcessed` tagged with `GELATO_SOURCE`.

```solidity
function processDueTasks() external;
```

#### checker

Gelato Web3 Function resolver: reports whether the processor has work to do. Gelato calls this offchain each polling interval and submits `execPayload` to this contract when `canExec` is true. `canExec` is also false while [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md#pause) reports the system paused, so Gelato doesn't spend gas on a call that would revert.

```solidity
function checker() external view returns (bool canExec, bytes memory execPayload);
```

**Returns**

|      Name      |    Type   |                     Description                     |
| :---------------: | :-------: | :------------------------------------------------------: |
|    `canExec`    |   `bool`  | True when the processor's earliest scheduled task is due. |
| `execPayload` |  `bytes`  |        Calldata for `processDueTasks` on this contract.        |

#### hasDueTasks

Returns whether the processor has any scheduled invoice task due for processing. Read by the CRE workflow each cron tick to decide whether to submit a report onchain. Passes through to the processor so keepers only need this contract's address, except while [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md#pause) reports the system paused, in which case this always returns `false` regardless of the processor's actual queue state, so keepers don't spend gas on calls that would revert.

```solidity
function hasDueTasks() external view returns (bool dueTasksExist);
```

**Returns**

|       Name       |  Type  |                        Description                       |
| :----------------: | :----: | :-----------------------------------------------------------: |
| `dueTasksExist` | `bool` | True when the processor's earliest scheduled task is due. |

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

#### getForwarder

Returns the address of the configured CRE forwarder contract. Fixed at construction as an immutable; there is no setter, so enabling or rotating the CRE keeper path requires redeploying this contract.

```solidity
function getForwarder() external view returns (address forwarderAddress);
```

**Returns**

|        Name        |    Type   |            Description            |
| :----------------: | :-------: | :-------------------------------: |
| `forwarderAddress` | `address` | The configured forwarder address. |

#### getWorkflowOwner

Returns the CRE workflow owner authorized to trigger `onReport`. Fixed at construction as an immutable; there is no setter.

```solidity
function getWorkflowOwner() external view returns (address workflowOwnerAddress);
```

**Returns**

|          Name          |    Type   |              Description              |
| :------------------------: | :-------: | :----------------------------------------: |
| `workflowOwnerAddress` | `address` | The authorized workflow owner address. |

### Events

#### DueTasksProcessed

Emitted whenever the due-task queue is drained through this adapter.

```solidity
event DueTasksProcessed(address indexed caller, bytes32 indexed source);
```

|   Name   |    Type   |                                Description                               |
| :--------: | :-------: | :--------------------------------------------------------------------------: |
| `caller` | `address` |                  The address that triggered processing.                 |
| `source` | `bytes32` | The keeper path used: `CRE_SOURCE` for `onReport`, `GELATO_SOURCE` for `processDueTasks`. |

### Errors

| Error | Description |
| :----: | :----------: |
| `NotAuthorized()` | Thrown when the caller lacks the required role or permission. |
| `InvalidAddress()` | Thrown when the storage address supplied to the constructor is the zero address, or the processor resolved via `pendingProcessor()` is the zero address. |
| `UnauthorizedWorkflowOwner(address _workflowOwner)` | Thrown when a CRE report's metadata does not carry the authorized workflow owner. |
