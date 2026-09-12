# PaymentProcessorStorage.sol

The PaymentProcessorStorage Solidity smart contract serves as the core state and configuration layer for the SapphireDao platform. It manages invoice ID sequencing, fee parameters, and access permissions. Other contracts, such as [SimplePaymentProcessor.sol](simplepaymentprocessor.sol.md) and [IntermediatedPaymentProcessor.sol](intermediatedpaymentprocessor.sol.md), rely on it for global settings and controlled state updates.

Contract Address: [0xa5a8d53D9138D17F6C94c56846D50a850aFd14c9](https://sepolia.basescan.org/address/0xa5a8d53D9138D17F6C94c56846D50a850aFd14c9)

You can find the full implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/PaymentProcessorStorage.sol)

PaymentProcessorStorage.sol enables:

- Sequential invoice ID management
- System-wide configuration for fees and gas thresholds
- Access control for privileged contract calls
- State-sharing across other contracts in the SapphireDao ecosystem
- Pausing both payment processors, either indefinitely (owner) or temporarily without owner involvement (emergency pauser)

### State Variables

Every value below is `public`, so each one is readable through its Solidity-generated getter (`FEE_RATE()`, `GAS_THRESHOLD()`, `DEFAULT_PAYMENT_VALIDITY_PERIOD()`, `FEE_RECEIVER()`, `WETH()`). The `getFeeRate`, `getGasThreshold`, `getPaymentValidityDuration`, and `getFeeReceiver` wrappers that used to front these values are gone; read them through the generated getters instead.

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

#### FEE_RATE

Platform fee rate in basis points (BPS). Fixed at compile time; there is no setter.

```solidity
uint96 public constant FEE_RATE = 500
```

#### GAS_THRESHOLD

Minimum gas that must remain to continue processing automated tasks. Fixed at compile time; there is no setter.

```solidity
uint96 public constant GAS_THRESHOLD = 100_000
```

#### FEE_RECEIVER

Address that receives collected platform fees. Fixed at construction; there is no setter.

```solidity
address public immutable FEE_RECEIVER
```

#### WETH

Wrapped native token both processors pay platform fees in. Fixed at construction and rejected if zero (`InvalidWeth`); there is no setter. Both [SimplePaymentProcessor.sol](simplepaymentprocessor.sol.md#release) and [IntermediatedPaymentProcessor.sol](intermediatedpaymentprocessor.sol.md#release) read this address when a fee comes out of a native escrow, wrap the fee, and pay the receiver in WETH.

```solidity
address public immutable WETH
```

### Functions

#### constructor

Initializes the contract with the given configuration.

Sets the contract owner, records `feeReceiver`, `weth`, and `intermediatedPlatformsOperator`, and initializes the invoice nonce counter. Reverts with `InvalidWeth` if `weth` is the zero address. Also fetches the addresses to authorize from its deployer: `msg.sender` must implement [`IAuthorizedAddressProvider`](masterdeployer.sol.md#related-interface-iauthorizedaddressprovider) (in practice, [MasterDeployer.sol](masterdeployer.sol.md)), and this contract calls `authorizedAddresses()` on it once, at construction, emitting `AuthorizationUpdated` for each address returned. Keeping this list out of the constructor arguments keeps it out of the CREATE2 init code, so this contract's address is predictable before the authorized processors are deployed. Authorization is fixed here, at deployment, and cannot be changed afterwards; there is no setter. `feeReceiver` and `weth` are likewise fixed at construction as immutables; only `intermediatedPlatformsOperator` and `feeSigner` remain settable after deployment (via [setIntermediatedPlatformsOperator](#setintermediatedplatformsoperator) and [setFeeSigner](#setfeesigner)).

```solidity
constructor(Configuration memory _configuration) ;
```

**Parameters**

|       Name       |      Type       |                                      Description                                      |
| :--------------: | :-------------: | :-----------------------------------------------------------------------------------: |
| `_configuration` | `Configuration` | The initial configuration: owner, fee receiver, intermediated platforms operator, and WETH address. |

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

#### setFeeSigner

Sets the key whose signature authorizes the fee receiver supplied when an invoice is accepted or paid.

Callable only by the contract owner. Reverts with `InvalidFeeSigner` if `_feeSigner` is the zero address. Must be an EOA: the processors recover it with ECDSA, so it cannot be the MultiSig that owns this contract.

```solidity
function setFeeSigner(address _feeSigner) external onlyOwner;
```

**Parameters**

|      Name     |   Type    |            Description            |
| :------------: | :-------: | :--------------------------------: |
| `_feeSigner` | `address` | The new fee signer address. |

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

#### getFeeSigner

Returns the key whose signature authorizes a per-invoice fee receiver.

```solidity
function getFeeSigner() external view returns (address feeSignerAddress);
```

**Returns**

|       Name       |   Type    |        Description        |
| :---------------: | :-------: | :-------------------------: |
| `feeSignerAddress` | `address` | The fee signer address. |

#### getIntermediatedPlatformsOperator

Returns the address of the authorized Intermediated Platforms Operator.

```solidity
function getIntermediatedPlatformsOperator() external view returns (address intermediatedPlatformsOperator);
```

**Returns**

|              Name                |   Type    |                  Description                  |
| :-------------------------------: | :-------: | :--------------------------------------------: |
| `intermediatedPlatformsOperator` | `address` | The Intermediated Platforms Operator address. |

### Structs

#### Configuration

Holds the addresses the contract is permanently configured with. Every field becomes an immutable or is fixed at construction; the fee rate and gas threshold are not part of this struct since they are compile-time constants (`FEE_RATE`, `GAS_THRESHOLD`) on the contract itself.

```solidity
struct Configuration {
    address owner;
    address feeReceiver;
    address intermediatedPlatformsOperator;
    address weth;
}
```

|        Field        |   Type    |                                       Description                                       |
| :-----------------: | :-------: | :-------------------------------------------------------------------------------------: |
|       `owner`       | `address` |               The address authorized to pause and to set the emergency pauser.                |
|    `feeReceiver`    | `address` |                          Address that receives platform fees.                           |
| `intermediatedPlatformsOperator` | `address` | Address authorized to interact with invoice creation and specific management functions. |
|       `weth`        | `address` |     Wrapped native token both processors pay platform fees in. Cannot be the zero address.     |

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

#### FeeSignerUpdated

Emitted when the fee signer is updated.

```solidity
event FeeSignerUpdated(address indexed feeSigner);
```

|    Name    |   Type    |          Description          |
| :---------: | :-------: | :------------------------------: |
| `feeSigner` | `address` | The new fee signer address. |

#### IntermediatedPlatformsOperatorUpdated

Emitted when the Intermediated Platforms Operator address is updated.

```solidity
event IntermediatedPlatformsOperatorUpdated(address indexed intermediatedPlatformsOperator);
```

|              Name                |   Type    |                   Description                    |
| :-------------------------------: | :-------: | :-----------------------------------------------: |
| `intermediatedPlatformsOperator` | `address` | The new Intermediated Platforms Operator address. |

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
|    `InvalidFeeSigner()`    |                    Thrown when setting the fee signer to the zero address.                       |
|      `InvalidWeth()`       |          Thrown when deploying with the zero address as the wrapped native token.                |
|     `InvalidFeeRate()`     |   Declared but never thrown; `FEE_RATE` is a fixed constant now, so there is nothing left to validate.   |
|     `AlreadyPaused()`      | Thrown when pausing a system that is already paused, or that has an unresolved emergency pause. |
|       `NotPaused()`        |                       Thrown when unpausing a system that is not paused.                        |
| `NoActiveEmergencyPause()` |           Thrown when approving an emergency pause that is absent or already expired.           |
