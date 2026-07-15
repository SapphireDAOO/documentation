# MasterDeployer.sol

Deploys the full payment processor system — `MultiSig`, `Notes`, `SimplePaymentProcessor`, `OracleManager`, `IntermediatedPaymentProcessor`, and finally [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md) — deterministically via CREATE2, in a single transaction.

The key trick: `PaymentProcessorStorage`'s constructor needs to know which processor addresses to authorize, but those processors don't exist until *after* they're deployed — and `PaymentProcessorStorage`'s own address needs to be predictable *before* it's deployed (so the processors can be pointed at it). `MasterDeployer` solves this by implementing `IAuthorizedAddressProvider`: `PaymentProcessorStorage`'s constructor calls back into its deployer (`msg.sender`, i.e. this contract) to fetch the list of addresses to authorize, rather than taking that list as a constructor argument. Keeping the authorized-address list out of the constructor args keeps it out of the CREATE2 init code, so `PaymentProcessorStorage`'s address depends only on its `Configuration` struct and can be predicted (via `predictStorageAddress`) before the processors exist. Authorization is fixed at that one deployment-time callback and can never be changed afterward — there is no setter.

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/MasterDeployer.sol), and the interface at [`IMasterDeployer.sol`](https://github.com/SapphireDAOO/payment-processor/blob/main/src/interface/IMasterDeployer.sol).

### State Variables

#### deployer

The only address allowed to trigger `deployAll`. Passed explicitly to the constructor (rather than derived from `msg.sender`) because `MasterDeployer` is itself deployed via a CREATE2 factory, so `msg.sender` during its own construction is that factory, not the intended deployer.

```solidity
address public immutable deployer
```

#### multiSig

The deployed `MultiSig` contract. Zero address until `deployAll` runs.

```solidity
MultiSig public multiSig
```

#### ppStorage

The deployed `PaymentProcessorStorage` contract. Zero address until `deployAll` runs; also used as the "already deployed" guard (`deployAll` reverts with `AlreadyDeployed` if this is already set).

```solidity
PaymentProcessorStorage public ppStorage
```

#### notes

The deployed `Notes` contract.

```solidity
Notes public notes
```

#### simplePaymentProcessor

The deployed `SimplePaymentProcessor` contract.

```solidity
SimplePaymentProcessor public simplePaymentProcessor
```

#### oracleManager

The deployed `OracleManager` contract.

```solidity
OracleManager public oracleManager
```

#### intermediatedPaymentProcessor

The deployed `IntermediatedPaymentProcessor` contract.

```solidity
IntermediatedPaymentProcessor public intermediatedPaymentProcessor
```

### Functions

#### constructor

Sets the address allowed to run the deployment.

```solidity
constructor(address _deployer);
```

**Parameters**

|    Name     |   Type    |                                                Description                                                |
| :---------: | :-------: | :--------------------------------------------------------------------------------------------------------: |
| `_deployer` | `address` | The deployer address. Passed explicitly since `msg.sender` here is the CREATE2 factory, not the deployer. |

#### authorizedAddresses

Returns the addresses `PaymentProcessorStorage` should authorize at construction. Only returns a non-empty list during the `deployAll` call itself — see `deployAll`.

```solidity
function authorizedAddresses() external view returns (address[] memory authorized);
```

**Returns**

|     Name     |     Type      |                     Description                    |
| :-----------: | :-------------: | :---------------------------------------------------: |
| `authorized` | `address[]` | The list of addresses to authorize (empty outside of an in-progress `deployAll` call). |

#### predictStorageAddress

Predicts the `PaymentProcessorStorage` address for a given salt and configuration, before it's deployed.

```solidity
function predictStorageAddress(bytes32 _salt, IPaymentProcessorStorage.Configuration memory _config)
    public
    view
    returns (address predicted);
```

**Parameters**

|   Name    |                        Type                       |                       Description                      |
| :--------: | :------------------------------------------------: | :--------------------------------------------------------: |
|  `_salt`  |                     `bytes32`                     |                  The CREATE2 salt.                    |
| `_config` | `IPaymentProcessorStorage.Configuration` | The storage configuration (part of the init code). |

**Returns**

|    Name     |   Type    |                          Description                         |
| :----------: | :-------: | :---------------------------------------------------------------: |
| `predicted` | `address` | The address `PaymentProcessorStorage` will be deployed at. |

#### deployAll

Deploys the full system in one transaction: `MultiSig`, `Notes`, `SimplePaymentProcessor`, `OracleManager`, `IntermediatedPaymentProcessor`, and finally `PaymentProcessorStorage` at its predicted address with both processors authorized.

Callable once, by `deployer` only (reverts with `NotDeployer` otherwise; reverts with `AlreadyDeployed` if `ppStorage` is already set). All child contracts are deployed via `Create2.deploy` using the same `_params.salt`. `SimplePaymentProcessor` and `IntermediatedPaymentProcessor` are pushed onto the pending-authorized list before `PaymentProcessorStorage` is deployed; that list is deleted immediately after, so `authorizedAddresses()` only ever returns a non-empty list during this call. Reverts with `StorageAddressMismatch` if the deployed storage address doesn't match the prediction. Ownership of the storage contract is left with `_params.config.owner`; post-deploy wiring (notes authorization, price feeds, ownership transfer to the `MultiSig`) is the deployer's responsibility, not something this function does.

```solidity
function deployAll(Params calldata _params) external returns (address ppStorageAddress);
```

**Parameters**

|    Name    |    Type    |            Description           |
| :---------: | :--------: | :----------------------------------: |
| `_params` | `Params` | The deployment parameters. |

**Returns**

|        Name        |    Type   |                    Description                   |
| :------------------: | :-------: | :----------------------------------------------------: |
| `ppStorageAddress` | `address` | The deployed `PaymentProcessorStorage` address. |

### Structs

#### Params

Parameters for the full system deployment.

```solidity
struct Params {
    bytes32 salt;
    IPaymentProcessorStorage.Configuration config;
    uint256 minimumInvoiceValue;
    address sequencerUptimeFeed;
    address[] multiSigSigners;
    uint256 multiSigThreshold;
}
```

|          Field          |                        Type                       |                                Description                               |
| :------------------------: | :--------------------------------------------------: | :--------------------------------------------------------------------------: |
|          `salt`          |                     `bytes32`                     |               The CREATE2 salt used for every deployment.                |
|         `config`         | `IPaymentProcessorStorage.Configuration` |         The initial `PaymentProcessorStorage` configuration.          |
| `minimumInvoiceValue`   |                     `uint256`                      | Minimum invoice value (in wei) for the `SimplePaymentProcessor`. |
| `sequencerUptimeFeed`   |                     `address`                      | Chainlink sequencer uptime feed; `address(0)` disables the check. |
|    `multiSigSigners`    |                    `address[]`                     |                     Initial `MultiSig` signers.                       |
|   `multiSigThreshold`   |                     `uint256`                      |                Initial `MultiSig` approval threshold.                |

### Events

#### SystemDeployed

Emitted once the full system has been deployed.

```solidity
event SystemDeployed(
    address multiSig,
    address ppStorage,
    address notes,
    address simplePaymentProcessor,
    address oracleManager,
    address intermediatedPaymentProcessor
);
```

|              Name              |   Type    |                    Description                    |
| :-------------------------------: | :-------: | :---------------------------------------------------: |
|            `multiSig`            | `address` |          The deployed `MultiSig` address.          |
|           `ppStorage`            | `address` |  The deployed `PaymentProcessorStorage` address.  |
|             `notes`              | `address` |           The deployed `Notes` address.           |
|    `simplePaymentProcessor`     | `address` |  The deployed `SimplePaymentProcessor` address.  |
|         `oracleManager`         | `address` |       The deployed `OracleManager` address.       |
| `intermediatedPaymentProcessor` | `address` | The deployed `IntermediatedPaymentProcessor` address. |

### Errors

| Error | Description |
| :----: | :----------: |
| `NotDeployer()` | Thrown when `deployAll` is called by an address other than `deployer`. |
| `AlreadyDeployed()` | Thrown when `deployAll` is called more than once. |
| `StorageAddressMismatch(address predicted, address deployed)` | Thrown when the deployed storage address does not match the prediction. |

### Related interface: IAuthorizedAddressProvider

`IMasterDeployer` extends a small standalone interface, `IAuthorizedAddressProvider`, that any contract deploying `PaymentProcessorStorage` must implement:

```solidity
interface IAuthorizedAddressProvider {
    function authorizedAddresses() external view returns (address[] memory authorized);
}
```

`PaymentProcessorStorage` calls this on its deployer (`msg.sender`) during construction to fetch the addresses to authorize. Authorization can only be granted this way, at deployment time — see [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md#constructor).
