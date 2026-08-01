# PaymentProcessorStorage.sol

The PaymentProcessorStorage Solidity smart contract serves as the core state and configuration layer for the SapphireDao platform. It manages invoice ID sequencing, fee parameters, hold periods, and access permissions. Other contracts, such as [SimplePaymentProcessor.sol](simplepaymentprocessor.sol.md) and [IntermediatedPaymentProcessor.sol](intermediatedpaymentprocessor.sol.md), rely on it for global settings and controlled state updates.

Contract Address: [0xeb57f1f77f873d8481510c1f5ee44de340dc93fe](https://sepolia.etherscan.io/address/0xeb57F1F77F873d8481510c1f5Ee44dE340Dc93fe)

You can find the full implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/PaymentProcessorStorage.sol)

PaymentProcessorStorage.sol enables:

* Sequential invoice ID management
* System-wide configuration for fees, hold periods, and gas thresholds
* Access control for privileged contract calls
* State-sharing across other contracts in the SapphireDao ecosystem

### State Variables

#### DEFAULT\_PAYMENT\_VALIDITY\_PERIOD

Default time window during which a created invoice remains valid for payment.

```solidity
uint256 public constant DEFAULT_PAYMENT_VALIDITY_PERIOD = 7 days
```

#### BASIS\_POINTS

Total basis points used for percentage calculations. 10\_000 = 100%.

```solidity
uint256 public constant BASIS_POINTS = 10_000
```

### Functions

#### constructor

Initializes the contract with the given configuration.

Sets the contract owner, stores the initial configuration parameters, and initializes the invoice nonce counter. Also fetches the addresses to authorize from its deployer: `msg.sender` must implement [`IAuthorizedAddressProvider`](masterdeployer.sol.md#related-interface-iauthorizedaddressprovider) (in practice, [MasterDeployer.sol](masterdeployer.sol.md)), and this contract calls `authorizedAddresses()` on it once, at construction, emitting `AuthorizationUpdated` for each address returned. Keeping this list out of the constructor arguments keeps it out of the CREATE2 init code, so this contract's address is predictable before the authorized processors are deployed. Authorization is fixed here, at deployment, and cannot be changed afterwards — there is no setter.

```solidity
constructor(Configuration memory _configuration) ;
```

**Parameters**

|       Name       |       Type      |                                      Description                                      |
| :--------------: | :-------------: | :-----------------------------------------------------------------------------------: |
| `_configuration` | `Configuration` | The initial configuration parameters including owner, gas threshold, and hold period. |

#### updateInvoiceNonce

Updates the invoice nonce counter.

Increments the internal nonce by the provided amount.

```solidity
function updateInvoiceNonce(uint216 _by) external onlyAuthorized returns (uint216 totalInvoices);
```

**Parameters**

|  Name |    Type   |                  Description                  |
| :---: | :-------: | :-------------------------------------------: |
| `_by` | `uint216` | The amount to increment the invoice nonce by. |

**Returns**

|       Name      |    Type   |                  Description                  |
| :-------------: | :-------: | :-------------------------------------------: |
| `totalInvoices` | `uint216` | The updated total number of invoices created. |

#### setFeeReceiver

Sets the address that will receive fees collected from transactions.

Callable only by the contract owner.

```solidity
function setFeeReceiver(address _feeReceiverAddress) external onlyOwner;
```

**Parameters**

|          Name         |    Type   |              Description              |
| :-------------------: | :-------: | :-----------------------------------: |
| `_feeReceiverAddress` | `address` | The address to receive protocol fees. |

#### setFeeRate

Updates the fee rate for seller payouts.

Callable only by the contract owner. Reverts with `InvalidFeeRate` if the rate exceeds `BASIS_POINTS` (10,000 = 100%).

```solidity
function setFeeRate(uint96 _newFeeRate) external onlyOwner;
```

**Parameters**

|      Name     |    Type   |                Description               |
| :-----------: | :-------: | :--------------------------------------: |
| `_newFeeRate` | `uint96` | The new fee rate in basis points (1% = 100 basis points). |

#### setGasThreshold

Updates the gas threshold used in automated task processing.

Only callable by the contract owner. This threshold determines the minimum gas required to continue processing during `SimplePaymentProcessor.processDueTasks` — called either directly or via the `PaymentAutomation` adapter's `onReport` (Chainlink CRE) / `processDueTasks` (Gelato) entrypoints.

```solidity
function setGasThreshold(uint96 _newGasThreshold) external onlyOwner;
```

**Parameters**

|        Name         |    Type   |                   Description                  |
| :-----------------: | :-------: | :--------------------------------------------: |
| `_newGasThreshold`  | `uint96` | The new gas threshold value (in units of gas). |

#### setPaymentValidityDuration

Updates the payment validity duration.

Only callable by the contract owner.

```solidity
function setPaymentValidityDuration(uint256 _newValidityDuration) external onlyOwner;
```

**Parameters**

|          Name          |    Type   |              Description              |
| :--------------------: | :-------: | :-----------------------------------: |
| `_newValidityDuration` | `uint256` | The new validity duration in seconds. |

#### setDefaultHoldPeriod

Updates the default hold period for all new invoices.

Only callable by the contract owner. Reverts with `HoldPeriodCanNotBeZero` if `_newDefaultHoldPeriod` is zero.

```solidity
function setDefaultHoldPeriod(uint96 _newDefaultHoldPeriod) public onlyOwner;
```

**Parameters**

|           Name          |    Type   |               Description               |
| :---------------------: | :-------: | :-------------------------------------: |
| `_newDefaultHoldPeriod` | `uint96` | The new default hold period in seconds. |

#### setMarketplaceAddress

Updates the intermediated platform address allowed to perform privileged operations.

Callable only by the contract owner.

```solidity
function setMarketplaceAddress(address _marketplaceAddress) external onlyOwner;
```

**Parameters**

|          Name         |    Type   |          Description         |
| :-------------------: | :-------: | :--------------------------: |
| `_marketplaceAddress` | `address` | The new intermediated platform address. |

#### getPaymentValidityDuration

Returns the duration for which a payment remains valid.

```solidity
function getPaymentValidityDuration() external view returns (uint256 validDuration);
```

**Returns**

|       Name      |    Type   |                Description                |
| :-------------: | :-------: | :---------------------------------------: |
| `validDuration` | `uint256` | The payment validity duration in seconds. |

#### getNextInvoiceNonce

Returns the nonce that will be assigned to the next invoice.

```solidity
function getNextInvoiceNonce() external view returns (uint216 nextInvoiceNonceValue);
```

**Returns**

|           Name          |    Type   |          Description          |
| :---------------------: | :-------: | :---------------------------: |
| `nextInvoiceNonceValue` | `uint216` | The next invoice nonce value. |

#### totalInvoiceCreated

Returns the total number of unique invoices created.

```solidity
function totalInvoiceCreated() public view returns (uint216 totalInvoices);
```

**Returns**

|       Name      |    Type   |              Description              |
| :-------------: | :-------: | :-----------------------------------: |
| `totalInvoices` | `uint216` | The total number of invoices created. |

#### getFeeRate

Returns the current platform fee rate in basis points.

```solidity
function getFeeRate() external view returns (uint256 feeRate);
```

**Returns**

|    Name   |    Type   |               Description              |
| :-------: | :-------: | :------------------------------------: |
| `feeRate` | `uint256` | The platform fee rate in basis points. |

#### getFeeReceiver

Returns the address that receives collected platform fees.

```solidity
function getFeeReceiver() external view returns (address feeReceiver);
```

**Returns**

|      Name     |    Type   |        Description        |
| :-----------: | :-------: | :-----------------------: |
| `feeReceiver` | `address` | The fee receiver address. |

#### getMarketplace

Returns the address of the authorized intermediated platform.

```solidity
function getMarketplace() external view returns (address marketplace);
```

**Returns**

|      Name     |    Type   |        Description       |
| :-----------: | :-------: | :----------------------: |
| `marketplace` | `address` | The intermediated platform address. |

#### getDefaultHoldPeriod

Gets the default hold period for invoices.

```solidity
function getDefaultHoldPeriod() external view returns (uint256 defaultHoldPeriod);
```

**Returns**

|         Name        |    Type   |             Description             |
| :-----------------: | :-------: | :---------------------------------: |
| `defaultHoldPeriod` | `uint256` | The default hold period in seconds. |

#### getGasThreshold

Returns the current gas threshold used to limit the execution loop in automated task processing.

This threshold is typically used to prevent out-of-gas errors during batch operations triggered by the Chainlink CRE workflow.

```solidity
function getGasThreshold() external view returns (uint256 gasThreshold);
```

**Returns**

|      Name      |    Type   |            Description           |
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
    uint96 defaultHoldPeriod;
    address marketplace;
    uint96 gasThreshold;
}
```

|        Field        |    Type   |                                          Description                                         |
| :-------------------: | :-------: | :---------------------------------------------------------------------------------------------: |
|       `owner`        | `address` |                    The address authorized to modify configuration parameters.                   |
|      `feeRate`       |  `uint96` |          Platform fee rate in basis points (BPS). 100 BPS = 1%; 10,000 BPS = 100%.               |
|     `feeReceiver`    | `address` |                              Address that receives platform fees.                                |
| `defaultHoldPeriod`  |  `uint96` |                The default hold period for funds in escrow, measured in seconds.                |
|     `marketplace`    | `address` |         Address authorized to interact with invoice creation and specific management functions. |
|    `gasThreshold`    |  `uint96` |                 The minimum amount of gas that must remain to continue processing tasks.        |

### Events

#### ConfigurationInitialized

Emitted once at construction with the initial configuration parameters.

```solidity
event ConfigurationInitialized(Configuration config);
```

|   Name   |         Type        |                       Description                      |
| :-------: | :------------------: | :--------------------------------------------------------: |
| `config` | `Configuration` | The configuration the contract was initialized with. |

#### AuthorizationUpdated

Emitted when an address is granted or revoked authorization.

```solidity
event AuthorizationUpdated(address indexed account, bool authorized);
```

|     Name     |    Type   |                     Description                    |
| :-----------: | :-------: | :------------------------------------------------------: |
|   `account`   | `address` | The address whose authorization status changed. |
| `authorized` |   `bool`  |            The new authorization status.           |

#### FeeReceiverUpdated

Emitted when the fee receiver address is updated.

```solidity
event FeeReceiverUpdated(address indexed feeReceiver);
```

|      Name      |    Type   |            Description           |
| :-------------: | :-------: | :----------------------------------: |
| `feeReceiver` | `address` | The new fee receiver address. |

#### MarketplaceUpdated

Emitted when the intermediated platform address is updated.

```solidity
event MarketplaceUpdated(address indexed marketplace);
```

|      Name      |    Type   |          Description         |
| :-------------: | :-------: | :------------------------------: |
| `marketplace` | `address` | The new intermediated platform address. |

#### FeeRateUpdated

Emitted when the platform fee rate is updated.

```solidity
event FeeRateUpdated(uint96 feeRate);
```

|   Name   |   Type   |                Description               |
| :-------: | :------: | :------------------------------------------: |
| `feeRate` | `uint96` | The new fee rate in basis points. |

#### GasThresholdUpdated

Emitted when the automated task-processing gas threshold is updated.

```solidity
event GasThresholdUpdated(uint96 gasThreshold);
```

|      Name      |   Type   |            Description           |
| :-------------: | :------: | :----------------------------------: |
| `gasThreshold` | `uint96` | The new gas threshold value. |

#### DefaultHoldPeriodUpdated

Emitted when the default hold period is updated.

```solidity
event DefaultHoldPeriodUpdated(uint96 defaultHoldPeriod);
```

|          Name          |   Type   |                 Description                |
| :---------------------: | :------: | :--------------------------------------------: |
| `defaultHoldPeriod` | `uint96` | The new default hold period in seconds. |

#### PaymentValidityDurationUpdated

Emitted when the payment validity duration is updated.

```solidity
event PaymentValidityDurationUpdated(uint256 validityDuration);
```

|        Name        |    Type   |                    Description                    |
| :-------------------: | :-------: | :----------------------------------------------------: |
| `validityDuration` | `uint256` | The new payment validity window in seconds. |

### Errors

| Error | Description |
| :----: | :----------: |
| `NotAuthorized()` | Thrown when a caller attempts an action without the required authorization. |
| `HoldPeriodCanNotBeZero()` | Thrown when the hold period provided is zero, which is invalid. |
| `InvalidFeeRate()` | Thrown when the provided fee rate exceeds the maximum allowed (10,000 basis points = 100%). |
