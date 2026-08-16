# PaymentProcessorStorage.sol

The PaymentProcessorStorage Solidity smart contract serves as the core state and configuration layer for the SapphireDao platform. It manages invoice ID sequencing, fee parameters, and access permissions. Other contracts, such as [SimplePaymentProcessor.sol](simplepaymentprocessor.sol.md) and [IntermediatedPaymentProcessor.sol](intermediatedpaymentprocessor.sol.md), rely on it for global settings and controlled state updates.

Contract Address: [0xeb57f1f77f873d8481510c1f5ee44de340dc93fe](https://sepolia.etherscan.io/address/0xeb57F1F77F873d8481510c1f5Ee44dE340Dc93fe)

You can find the full implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/PaymentProcessorStorage.sol)

PaymentProcessorStorage.sol enables:

- Sequential invoice ID management
- System-wide configuration for fees and gas thresholds
- Access control for privileged contract calls
- State-sharing across other contracts in the SapphireDao ecosystem
- Pausing both payment processors, either indefinitely (owner) or temporarily without owner involvement (emergency pauser)

### State Variables

#### DEFAULT_PAYMENT_VALIDITY_PERIOD

Default time window during which a created invoice remains valid for payment.

```solidity
uint256 public constant DEFAULT_PAYMENT_VALIDITY_PERIOD = 7 days
```

#### BASIS_POINTS

Total basis points used for percentage calculations. 10_000 = 100%.

```solidity
uint256 public constant BASIS_POINTS = 10_000
```

#### EMERGENCY_PAUSE_DURATION

How long an emergency pause holds without owner approval before it lapses automatically.

```solidity
uint256 public constant EMERGENCY_PAUSE_DURATION = 24 hours
```

### Functions

#### constructor

Initializes the contract with the given configuration.

Sets the contract owner, stores the initial configuration parameters, and initializes the invoice nonce counter. Also fetches the addresses to authorize from its deployer: `msg.sender` must implement [`IAuthorizedAddressProvider`](masterdeployer.sol.md#related-interface-iauthorizedaddressprovider) (in practice, [MasterDeployer.sol](masterdeployer.sol.md)), and this contract calls `authorizedAddresses()` on it once, at construction, emitting `AuthorizationUpdated` for each address returned. Keeping this list out of the constructor arguments keeps it out of the CREATE2 init code, so this contract's address is predictable before the authorized processors are deployed. Authorization is fixed here, at deployment, and cannot be changed afterwards; there is no setter.

```solidity
constructor(Configuration memory _configuration) ;
```

**Parameters**

|       Name       |      Type       |                                      Description                                      |
| :--------------: | :-------------: | :-----------------------------------------------------------------------------------: |
| `_configuration` | `Configuration` | The initial configuration parameters including owner, fee settings, and gas threshold. |

#### updateInvoiceNonce

Updates the invoice nonce counter.

Increments the internal nonce by the provided amount.

```solidity
function updateInvoiceNonce(uint216 _by) external onlyAuthorized returns (uint216 totalInvoices);
```

**Parameters**

| Name  |   Type    |                  Description                  |
| :---: | :-------: | :-------------------------------------------: |
| `_by` | `uint216` | The amount to increment the invoice nonce by. |

**Returns**

|      Name       |   Type    |                  Description                  |
| :-------------: | :-------: | :-------------------------------------------: |
| `totalInvoices` | `uint216` | The updated total number of invoices created. |

#### setFeeReceiver

Sets the address that will receive fees collected from transactions.

Callable only by the contract owner.

```solidity
function setFeeReceiver(address _feeReceiverAddress) external onlyOwner;
```

**Parameters**

|         Name          |   Type    |              Description              |
| :-------------------: | :-------: | :-----------------------------------: |
| `_feeReceiverAddress` | `address` | The address to receive protocol fees. |

#### setFeeRate

Updates the fee rate for seller payouts.

Callable only by the contract owner. Reverts with `InvalidFeeRate` if the rate exceeds `BASIS_POINTS` (10,000 = 100%).

```solidity
function setFeeRate(uint96 _newFeeRate) external onlyOwner;
```

**Parameters**

|     Name      |   Type   |                        Description                        |
| :-----------: | :------: | :-------------------------------------------------------: |
| `_newFeeRate` | `uint96` | The new fee rate in basis points (1% = 100 basis points). |

#### setGasThreshold

Updates the gas threshold used in automated task processing.

Only callable by the contract owner. This threshold determines the minimum gas required to continue processing during `SimplePaymentProcessor.processDueTasks`, called either directly or via the `PaymentAutomation` adapter's `onReport` (Chainlink CRE) / `processDueTasks` (Gelato) entrypoints.

```solidity
function setGasThreshold(uint96 _newGasThreshold) external onlyOwner;
```

**Parameters**

|        Name        |   Type   |                  Description                   |
| :----------------: | :------: | :--------------------------------------------: |
| `_newGasThreshold` | `uint96` | The new gas threshold value (in units of gas). |

#### setPaymentValidityDuration

Updates the payment validity duration: how long after creation an invoice can still be paid before it expires unpaid (reverts with `InvoiceIsNoLongerValid` on a payment attempt after that window).

Only callable by the contract owner. Applies to invoices created after the update; the validity window on an already-created invoice is fixed to the value in effect at creation.

```solidity
function setPaymentValidityDuration(uint256 _newValidityDuration) external onlyOwner;
```

**Parameters**

|          Name          |   Type    |                                                   Description                                                   |
| :--------------------: | :-------: | :-------------------------------------------------------------------------------------------------------------: |
| `_newValidityDuration` | `uint256` | The new payment window in seconds: how long an unpaid invoice remains payable after creation before it expires. |

#### setIntermediatedPlatformsOperator

Updates the sole platform operator wallet authorized to call privileged `IntermediatedPaymentProcessor` functions (creating invoices, triggering releases and refunds, and resolving disputes). Stored internally as `intermediatedPlatformsOperator`; emits `IntermediatedPlatformsOperatorUpdated`.

Callable only by the contract owner.

```solidity
function setIntermediatedPlatformsOperator(address _intermediatedPlatformsOperatorWallet) external onlyOwner;
```

**Parameters**

|                  Name                   |   Type    |               Description               |
| :-------------------------------------: | :-------: | :-------------------------------------: |
| `_intermediatedPlatformsOperatorWallet` | `address` | The new platform operator wallet address. |

#### pause

Halts every value-moving entrypoint on both payment processors.

Only callable by the contract owner. Stays in effect until `unpause`. Reverts with `AlreadyPaused` if the system is already paused (whether by owner pause or an active emergency pause).

```solidity
function pause() external onlyOwner;
```

#### unpause

Lifts a pause and clears any unresolved emergency pause.

Only callable by the contract owner. Reverts with `NotPaused` if neither an owner pause nor an emergency pause is active.

```solidity
function unpause() external onlyOwner;
```

#### emergencyPause

Halts both payment processors for `EMERGENCY_PAUSE_DURATION` (24 hours) without owner involvement.

Only callable by the `emergencyPauser` address (reverts with `NotAuthorized` otherwise). Lapses automatically unless the owner calls `approveEmergencyPause` within the window. The pauser may trigger a fresh one once it lapses. Reverts with `AlreadyPaused` if the system is already paused.

```solidity
function emergencyPause() external;
```

#### approveEmergencyPause

Converts an active emergency pause into an indefinite pause (equivalent to `pause`).

Only callable by the contract owner, and only while the emergency pause has not expired; reverts with `NoActiveEmergencyPause` otherwise.

```solidity
function approveEmergencyPause() external onlyOwner;
```

#### setEmergencyPauser

Sets the address allowed to call `emergencyPause`.

Only callable by the contract owner. Set to `address(0)` to revoke.

```solidity
function setEmergencyPauser(address _emergencyPauser) external onlyOwner;
```

**Parameters**

|        Name        |   Type    |            Description            |
| :----------------: | :-------: | :-------------------------------: |
| `_emergencyPauser` | `address` | The new emergency pauser address. |

#### isPaused

Returns whether the payment processors are currently paused.

True for an owner pause, or an emergency pause that has not yet expired.

```solidity
function isPaused() public view returns (bool pausedState);
```

**Returns**

|     Name      |  Type  |    Description    |
| :-----------: | :----: | :---------------: |
| `pausedState` | `bool` | True when paused. |

#### getEmergencyPauser

Returns the address allowed to call `emergencyPause`.

```solidity
function getEmergencyPauser() external view returns (address emergencyPauserAddress);
```

**Returns**

|           Name           |   Type    |          Description          |
| :----------------------: | :-------: | :---------------------------: |
| `emergencyPauserAddress` | `address` | The emergency pauser address. |

#### getEmergencyPauseExpiry

Returns the timestamp at which an unresolved emergency pause lapses.

```solidity
function getEmergencyPauseExpiry() external view returns (uint256 expiry);
```

**Returns**

|   Name   |   Type    |                          Description                           |
| :------: | :-------: | :------------------------------------------------------------: |
| `expiry` | `uint256` | The expiry timestamp, or 0 when no emergency pause is pending. |

#### getPaymentValidityDuration

Returns the current payment window: how long a newly created invoice stays payable before it expires unpaid.

```solidity
function getPaymentValidityDuration() external view returns (uint256 validDuration);
```

**Returns**

|      Name       |   Type    |                Description                |
| :-------------: | :-------: | :---------------------------------------: |
| `validDuration` | `uint256` | The payment validity duration in seconds. |

#### getNextInvoiceNonce

Returns the nonce that will be assigned to the next invoice.

```solidity
function getNextInvoiceNonce() external view returns (uint216 nextInvoiceNonceValue);
```

**Returns**

|          Name           |   Type    |          Description          |
| :---------------------: | :-------: | :---------------------------: |
| `nextInvoiceNonceValue` | `uint216` | The next invoice nonce value. |

#### totalInvoiceCreated

Returns the total number of unique invoices created.

```solidity
function totalInvoiceCreated() public view returns (uint216 totalInvoices);
```

**Returns**

|      Name       |   Type    |              Description              |
| :-------------: | :-------: | :-----------------------------------: |
| `totalInvoices` | `uint216` | The total number of invoices created. |

#### getFeeRate

Returns the current platform fee rate in basis points.

```solidity
function getFeeRate() external view returns (uint256 feeRate);
```

**Returns**

|   Name    |   Type    |              Description               |
| :-------: | :-------: | :------------------------------------: |
| `feeRate` | `uint256` | The platform fee rate in basis points. |

#### getFeeReceiver

Returns the address that receives collected platform fees.

```solidity
function getFeeReceiver() external view returns (address feeReceiver);
```

**Returns**

|     Name      |   Type    |        Description        |
| :-----------: | :-------: | :-----------------------: |
| `feeReceiver` | `address` | The fee receiver address. |

#### getIntermediatedPlatformsOperator

Returns the address of the authorized Intermediated Platforms Operator.

```solidity
function getIntermediatedPlatformsOperator() external view returns (address intermediatedPlatformsOperator);
```

**Returns**

|              Name                |   Type    |                  Description                  |
| :-------------------------------: | :-------: | :--------------------------------------------: |
| `intermediatedPlatformsOperator` | `address` | The Intermediated Platforms Operator address. |

#### getGasThreshold

Returns the current gas threshold used to limit the execution loop in automated task processing.

This threshold is typically used to prevent out-of-gas errors during batch operations triggered by the Chainlink CRE workflow.

```solidity
function getGasThreshold() external view returns (uint256 gasThreshold);
```

**Returns**

|      Name      |   Type    |           Description            |
| :------------: | :-------: | :------------------------------: |
| `gasThreshold` | `uint256` | The current gas threshold value. |

### Structs

#### Configuration

Holds core configuration parameters for the contract.

```solidity
struct Configuration {
    address owner;
    uint96 feeRate;
    address feeReceiver;
    address intermediatedPlatformsOperator;
    uint96 gasThreshold;
}
```

|        Field        |   Type    |                                       Description                                       |
| :-----------------: | :-------: | :-------------------------------------------------------------------------------------: |
|       `owner`       | `address` |               The address authorized to modify configuration parameters.                |
|      `feeRate`      | `uint96`  |        Platform fee rate in basis points (BPS). 100 BPS = 1%; 10,000 BPS = 100%.        |
|    `feeReceiver`    | `address` |                          Address that receives platform fees.                           |
| `intermediatedPlatformsOperator` | `address` | Address authorized to interact with invoice creation and specific management functions. |
|   `gasThreshold`    | `uint96`  |        The minimum amount of gas that must remain to continue processing tasks.         |

### Events

#### ConfigurationInitialized

Emitted once at construction with the initial configuration parameters.

```solidity
event ConfigurationInitialized(Configuration config);
```

|   Name   |      Type       |                     Description                      |
| :------: | :-------------: | :--------------------------------------------------: |
| `config` | `Configuration` | The configuration the contract was initialized with. |

#### AuthorizationUpdated

Emitted when an address is granted or revoked authorization.

```solidity
event AuthorizationUpdated(address indexed account, bool authorized);
```

|     Name     |   Type    |                   Description                   |
| :----------: | :-------: | :---------------------------------------------: |
|  `account`   | `address` | The address whose authorization status changed. |
| `authorized` |  `bool`   |          The new authorization status.          |

#### FeeReceiverUpdated

Emitted when the fee receiver address is updated.

```solidity
event FeeReceiverUpdated(address indexed feeReceiver);
```

|     Name      |   Type    |          Description          |
| :-----------: | :-------: | :---------------------------: |
| `feeReceiver` | `address` | The new fee receiver address. |

#### IntermediatedPlatformsOperatorUpdated

Emitted when the Intermediated Platforms Operator address is updated.

```solidity
event IntermediatedPlatformsOperatorUpdated(address indexed intermediatedPlatformsOperator);
```

|              Name                |   Type    |                   Description                    |
| :-------------------------------: | :-------: | :-----------------------------------------------: |
| `intermediatedPlatformsOperator` | `address` | The new Intermediated Platforms Operator address. |

#### FeeRateUpdated

Emitted when the platform fee rate is updated.

```solidity
event FeeRateUpdated(uint96 feeRate);
```

|   Name    |   Type   |            Description            |
| :-------: | :------: | :-------------------------------: |
| `feeRate` | `uint96` | The new fee rate in basis points. |

#### GasThresholdUpdated

Emitted when the automated task-processing gas threshold is updated.

```solidity
event GasThresholdUpdated(uint96 gasThreshold);
```

|      Name      |   Type   |         Description          |
| :------------: | :------: | :--------------------------: |
| `gasThreshold` | `uint96` | The new gas threshold value. |

#### PaymentValidityDurationUpdated

Emitted when the payment validity duration is updated.

```solidity
event PaymentValidityDurationUpdated(uint256 validityDuration);
```

|        Name        |   Type    |                 Description                 |
| :----------------: | :-------: | :-----------------------------------------: |
| `validityDuration` | `uint256` | The new payment validity window in seconds. |

#### Paused

Emitted when the owner pauses the payment processors.

```solidity
event Paused(address indexed account);
```

|   Name    |   Type    |      Description       |
| :-------: | :-------: | :--------------------: |
| `account` | `address` | The owner that paused. |

#### Unpaused

Emitted when the owner lifts a pause.

```solidity
event Unpaused(address indexed account);
```

|   Name    |   Type    |       Description        |
| :-------: | :-------: | :----------------------: |
| `account` | `address` | The owner that unpaused. |

#### EmergencyPaused

Emitted when the emergency pauser halts the payment processors.

```solidity
event EmergencyPaused(address indexed account, uint256 expiry);
```

|   Name    |   Type    |                           Description                           |
| :-------: | :-------: | :-------------------------------------------------------------: |
| `account` | `address` |                      The emergency pauser.                      |
| `expiry`  | `uint256` | The timestamp at which the pause lapses without owner approval. |

#### EmergencyPauseApproved

Emitted when the owner converts an emergency pause into an indefinite pause.

```solidity
event EmergencyPauseApproved(address indexed account);
```

|   Name    |   Type    |       Description        |
| :-------: | :-------: | :----------------------: |
| `account` | `address` | The owner that approved. |

#### EmergencyPauserUpdated

Emitted when the emergency pauser address is updated.

```solidity
event EmergencyPauserUpdated(address indexed emergencyPauser);
```

|       Name        |   Type    |            Description            |
| :---------------: | :-------: | :-------------------------------: |
| `emergencyPauser` | `address` | The new emergency pauser address. |

### Errors

|           Error            |                                           Description                                           |
| :------------------------: | :---------------------------------------------------------------------------------------------: |
|     `NotAuthorized()`      |           Thrown when a caller attempts an action without the required authorization.           |
|     `InvalidFeeRate()`     |   Thrown when the provided fee rate exceeds the maximum allowed (10,000 basis points = 100%).   |
|     `AlreadyPaused()`      | Thrown when pausing a system that is already paused, or that has an unresolved emergency pause. |
|       `NotPaused()`        |                       Thrown when unpausing a system that is not paused.                        |
| `NoActiveEmergencyPause()` |           Thrown when approving an emergency pause that is absent or already expired.           |
