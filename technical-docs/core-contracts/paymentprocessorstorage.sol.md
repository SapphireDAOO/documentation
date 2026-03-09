# PaymentProcessorStorage.sol

The PaymentProcessorStorage Solidity smart contract serves as the core state and configuration layer for the SapphireDao platform. It manages invoice ID sequencing, fee parameters, hold periods, and access permissions. Other contracts, such as [SimplePaymentProcessor.sol](./) and [AdvancedPaymentProcessor.sol,](advancedpaymentprocessor.sol.md) rely on it for global settings and controlled state updates.&#x20;

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

Sets the contract owner, stores the initial configuration parameters, and initializes the invoice nonce counter.

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

#### setAuthorizedAddress

Sets or revokes authorization for a specific address.

Only callable by the contract owner.

```solidity
function setAuthorizedAddress(address _authorizedAddress, bool _authorized) external onlyOwner;
```

**Parameters**

|         Name         |    Type   |                Description               |
| :------------------: | :-------: | :--------------------------------------: |
| `_authorizedAddress` | `address` | The address to authorize or deauthorize. |
|     `_authorized`    |   `bool`  |    Whether the address is authorized.    |

#### setFeeRate

Updates the fee rate for seller payouts.

Callable only by the contract owner.

```solidity
function setFeeRate(uint256 _newFeeRate) external onlyOwner;
```

**Parameters**

|      Name     |    Type   |                Description               |
| :-----------: | :-------: | :--------------------------------------: |
| `_newFeeRate` | `uint256` | The new fee rate in basis points (1% = 100 basis points). |

#### setGasThreshold

Updates the gas threshold used in automated upkeep logic.

Only callable by the contract owner. This threshold determines the minimum gas required to continue processing during `performUpkeep`.

```solidity
function setGasThreshold(uint256 _newGasThreshold) external onlyOwner;
```

**Parameters**

|        Name         |    Type   |                   Description                  |
| :-----------------: | :-------: | :--------------------------------------------: |
| `_newGasThreshold`  | `uint256` | The new gas threshold value (in units of gas). |

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

Only callable by the contract owner.

```solidity
function setDefaultHoldPeriod(uint256 _newDefaultHoldPeriod) public onlyOwner;
```

**Parameters**

|           Name          |    Type   |               Description               |
| :---------------------: | :-------: | :-------------------------------------: |
| `_newDefaultHoldPeriod` | `uint256` | The new default hold period in seconds. |

#### setMarketplaceAddress

Updates the marketplace address allowed to perform privileged operations.

Callable only by the contract owner.

```solidity
function setMarketplaceAddress(address _marketplaceAddress) external onlyOwner;
```

**Parameters**

|          Name         |    Type   |          Description         |
| :-------------------: | :-------: | :--------------------------: |
| `_marketplaceAddress` | `address` | The new marketplace address. |

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

Returns the address of the authorized marketplace contract.

```solidity
function getMarketplace() external view returns (address marketplace);
```

**Returns**

|      Name     |    Type   |        Description       |
| :-----------: | :-------: | :----------------------: |
| `marketplace` | `address` | The marketplace address. |

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

Returns the current gas threshold used to limit the execution loop in automated upkeep.

This threshold is typically used to prevent out-of-gas errors during batch operations in Chainlink Automation.

```solidity
function getGasThreshold() external view returns (uint256 gasThreshold);
```

**Returns**

|      Name      |    Type   |            Description           |
| :------------: | :-------: | :------------------------------: |
| `gasThreshold` | `uint256` | The current gas threshold value. |
