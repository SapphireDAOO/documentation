# MasterDeployer.sol

Deploys the full payment processor system (`MultiSig`, `SimplePaymentProcessor`, [PaymentAutomation.sol](paymentautomation.sol.md), `OracleManager`, `IntermediatedPaymentProcessor`, [Sweeper.sol](sweeper.sol.md), `Notes`, and finally [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md)) deterministically via CREATE2.

Deployment runs in two transactions, [deployCore](#deploycore) then [deploySystem](#deploysystem), because deploying all eight contracts at once needs roughly 15.6M gas and RPC providers reject a transaction whose gas limit exceeds 16,777,216. The two heaviest contracts (`SimplePaymentProcessor` and `IntermediatedPaymentProcessor`) are split across the two calls to keep each well clear of that ceiling.

Two circular dependencies are broken by predicting addresses before deploying:

- `PaymentProcessorStorage`'s constructor needs to know which processor addresses to authorize, but those processors don't exist until *after* they're deployed, and `PaymentProcessorStorage`'s own address needs to be predictable *before* it's deployed (so the processors can be pointed at it). `MasterDeployer` solves this by implementing `IAuthorizedAddressProvider`: `PaymentProcessorStorage`'s constructor calls back into its deployer (`msg.sender`, i.e. this contract) to fetch the list of addresses to authorize, rather than taking that list as a constructor argument. Keeping the authorized-address list out of the constructor args keeps it out of the CREATE2 init code, so `PaymentProcessorStorage`'s address depends only on its `Configuration` struct (and creation code) and can be predicted (via [predictStorageAddress](#predictstorageaddress)) before the processors exist. Authorization is fixed at that one deployment-time callback and can never be changed afterward; there is no setter. `Notes` uses the same `IAuthorizedAddressProvider` callback for its own write-allowlist (see [Notes.sol](notes.sol.md#constructor)).
- `SimplePaymentProcessor` and `PaymentAutomation` hold each other's address as immutables, which is its own cycle: neither's init code can name the other's address without knowing it first. This is broken by prediction too: `SimplePaymentProcessor` is constructed against `PaymentAutomation`'s CREATE2-predicted address (computed from init code that only needs the predicted storage address, `_params.forwarder`, and `_params.workflowOwner`, so it never depends on the processor). `PaymentAutomation` is then deployed for real and reads the processor's now-real address back from the deployer via `IPendingProcessorProvider.pendingProcessor()`, rather than taking it as a constructor argument. [deployCore](#deploycore) reverts with `AddressMismatch` if the real `PaymentAutomation` address doesn't match the prediction `SimplePaymentProcessor` was built against.

`Notes`'s address is predicted the same way in [deployCore](#deploycore) (recorded as [predictedNotes](#predictednotes)) so `SimplePaymentProcessor` can hold it as an immutable, but `Notes` itself isn't deployed until [deploySystem](#deploysystem): its write-allowlist must name both processors, and `IntermediatedPaymentProcessor` doesn't exist until the second phase.

Every child contract's creation code is passed in by the caller (via the [CoreInitCodes](#coreinitcodes) and [SystemInitCodes](#systeminitcodes) structs) rather than imported and embedded in `MasterDeployer` itself; importing all eight contracts directly would push this contract's own bytecode past the EIP-170 size limit. `MasterDeployer` appends the ABI-encoded constructor arguments to each creation code blob itself before calling `Create2.deploy`.

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/MasterDeployer.sol), and the interface at [`IMasterDeployer.sol`](https://github.com/SapphireDAOO/payment-processor/blob/main/src/interface/IMasterDeployer.sol).

### State Variables

#### DEPLOYER

The only address allowed to trigger [deployCore](#deploycore) and [deploySystem](#deploysystem). Passed explicitly to the constructor (rather than derived from `msg.sender`) because `MasterDeployer` is itself deployed via a CREATE2 factory, so `msg.sender` during its own construction is that factory, not the intended deployer.

```solidity
address public immutable DEPLOYER
```

#### multiSig

The deployed `MultiSig` contract. Zero address until [deployCore](#deploycore) runs; also used as the "core already deployed" guard (`deployCore` reverts with `AlreadyDeployed` if this is already set, and `deploySystem` reverts with `CoreNotDeployed` if it isn't).

```solidity
MultiSig public multiSig
```

#### ppStorage

The deployed `PaymentProcessorStorage` contract. Zero address until [deploySystem](#deploysystem) runs; also used as the "system already deployed" guard (`deploySystem` reverts with `AlreadyDeployed` if this is already set).

```solidity
PaymentProcessorStorage public ppStorage
```

#### notes

The deployed `Notes` contract. Zero address until [deploySystem](#deploysystem) runs.

```solidity
Notes public notes
```

#### simplePaymentProcessor

The deployed `SimplePaymentProcessor` contract. Zero address until [deployCore](#deploycore) runs.

```solidity
SimplePaymentProcessor public simplePaymentProcessor
```

#### paymentAutomation

The deployed `PaymentAutomation` adapter driving the `SimplePaymentProcessor`'s due-task queue. Zero address until [deployCore](#deploycore) runs.

```solidity
PaymentAutomation public paymentAutomation
```

#### oracleManager

The deployed `OracleManager` contract. Zero address until [deploySystem](#deploysystem) runs.

```solidity
OracleManager public oracleManager
```

#### intermediatedPaymentProcessor

The deployed `IntermediatedPaymentProcessor` contract. Zero address until [deploySystem](#deploysystem) runs.

```solidity
IntermediatedPaymentProcessor public intermediatedPaymentProcessor
```

#### sweeper

The deployed `Sweeper` contract. Zero address until [deploySystem](#deploysystem) runs.

```solidity
Sweeper public sweeper
```

#### predictedNotes

The address `Notes` will be deployed at, recorded by [deployCore](#deploycore).

`SimplePaymentProcessor` is constructed against this before `Notes` exists: `Notes` is deployed last (in [deploySystem](#deploysystem)) so its authorization list can name both processors.

```solidity
address public predictedNotes
```

#### predictedStorage

The address `PaymentProcessorStorage` will be deployed at, recorded by [deployCore](#deploycore). Carries the prediction across the two deployment transactions so both phases construct against the same address.

```solidity
address public predictedStorage
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

Returns the addresses `PaymentProcessorStorage` (and `Notes`) should authorize at construction. Only returns a non-empty list during an in-progress [deploySystem](#deploysystem) call; see `deploySystem`.

```solidity
function authorizedAddresses() external view returns (address[] memory authorized);
```

**Returns**

|     Name     |     Type      |                     Description                    |
| :-----------: | :-------------: | :---------------------------------------------------: |
| `authorized` | `address[]` | The list of addresses to authorize (empty outside of an in-progress `deploySystem` call). |

#### predictStorageAddress

Predicts the `PaymentProcessorStorage` address for a given salt, configuration, and creation code, before it's deployed.

```solidity
function predictStorageAddress(
    bytes32 _salt,
    IPaymentProcessorStorage.Configuration memory _config,
    bytes memory _ppStorageCreationCode
) public view returns (address predicted);
```

**Parameters**

|            Name            |                        Type                       |                       Description                      |
| :---------------------------: | :------------------------------------------------: | :--------------------------------------------------------: |
|           `_salt`           |                     `bytes32`                     |                  The CREATE2 salt.                    |
|          `_config`          | `IPaymentProcessorStorage.Configuration` | The storage configuration (part of the init code). |
| `_ppStorageCreationCode` |                      `bytes`                      | `PaymentProcessorStorage` creation code without constructor args. |

**Returns**

|    Name     |   Type    |                          Description                         |
| :----------: | :-------: | :---------------------------------------------------------------: |
| `predicted` | `address` | The address `PaymentProcessorStorage` will be deployed at. |

#### pendingProcessor

Returns the processor the `PaymentAutomation` currently being constructed should drive. Implements `IPendingProcessorProvider`; called by `PaymentAutomation`'s constructor during [deployCore](#deploycore), after `SimplePaymentProcessor` has already been deployed in that same call.

```solidity
function pendingProcessor() external view returns (address processor);
```

**Returns**

|    Name     |   Type    |                    Description                    |
| :----------: | :-------: | :----------------------------------------------------: |
| `processor` | `address` | The deployed `SimplePaymentProcessor` address. |

#### deployCore

First deployment phase: `MultiSig`, `SimplePaymentProcessor`, and `PaymentAutomation`.

Callable once, by `DEPLOYER` only (reverts with `NotDeployer` otherwise; reverts with `AlreadyDeployed` if `multiSig` is already set). Predicts the `PaymentProcessorStorage`, `Notes`, and `PaymentAutomation` addresses (via CREATE2, since none of their init codes name each other), records the storage and `Notes` predictions for [deploySystem](#deploysystem) to reuse, then deploys `MultiSig`, `SimplePaymentProcessor` (constructed against the predicted storage, `Notes`, and `PaymentAutomation` addresses), and finally `PaymentAutomation` itself, which reads its processor back through [pendingProcessor](#pendingprocessor) rather than taking it as a constructor argument. Reverts with `AddressMismatch` if the deployed `PaymentAutomation` address doesn't match the prediction `SimplePaymentProcessor` was built against. Emits `CoreDeployed`.

```solidity
function deployCore(Params calldata _params, CoreInitCodes calldata _initCodes)
    external
    returns (address predictedStorageAddress);
```

**Parameters**

|      Name      |          Type          |                    Description                   |
| :---------------: | :--------------------: | :---------------------------------------------------: |
|   `_params`    |        `Params`        |              The deployment parameters.            |
| `_initCodes` | `CoreInitCodes` | The creation code of each contract this phase needs (also used to predict `Notes` and `PaymentAutomation`'s addresses). |

**Returns**

|          Name           |    Type   |                    Description                   |
| :----------------------: | :-------: | :----------------------------------------------------: |
| `predictedStorageAddress` | `address` | The address `PaymentProcessorStorage` will be deployed at. |

#### deploySystem

Second deployment phase: `OracleManager`, `IntermediatedPaymentProcessor`, `Sweeper`, `Notes`, and finally `PaymentProcessorStorage` at its predicted address with both processors authorized.

Callable once, by `DEPLOYER` only, and only after [deployCore](#deploycore) (reverts with `CoreNotDeployed` if `multiSig` is unset; reverts with `AlreadyDeployed` if `ppStorage` is already set). `Sweeper` takes only the predicted `PaymentProcessorStorage` address as its constructor argument. `Notes` is authorized to name `SimplePaymentProcessor`, `IntermediatedPaymentProcessor`, and `DEPLOYER` (the deployer also gets write access, so it can attach post-deploy notes; `PaymentProcessorStorage` deliberately does not); reverts with `AddressMismatch` if the deployed `Notes` address doesn't match the prediction from `deployCore`. The pending-authorized list is then reset to just the two processors for `PaymentProcessorStorage`'s own authorization, and deleted once that deployment completes, so `authorizedAddresses()` only ever returns a non-empty list during this call (`Sweeper` is never added to it; it authorizes callers by checking `PaymentProcessorStorage`'s owner directly, not the `onlyAuthorized` allowlist). Reverts with `StorageAddressMismatch` if the deployed storage address doesn't match the prediction. Ownership of the storage contract is left with `_params.config.owner`; post-deploy wiring (setting the fee signer, price feeds, ownership transfer to the `MultiSig`) is the deployer's responsibility, not something this function does. Emits `SystemDeployed`.

```solidity
function deploySystem(Params calldata _params, SystemInitCodes calldata _initCodes)
    external
    returns (address ppStorageAddress);
```

**Parameters**

|      Name      |          Type          |                    Description                   |
| :---------------: | :--------------------: | :---------------------------------------------------: |
|   `_params`    |        `Params`        |     The deployment parameters. Must match those passed to `deployCore`.       |
| `_initCodes` | `SystemInitCodes` | The creation code of each contract this phase needs. |

**Returns**

|        Name        |    Type   |                    Description                   |
| :------------------: | :-------: | :----------------------------------------------------: |
| `ppStorageAddress` | `address` | The deployed `PaymentProcessorStorage` address. |

### Structs

#### Params

Parameters for the full system deployment, shared by both `deployCore` and `deploySystem`.

```solidity
struct Params {
    bytes32 salt;
    IPaymentProcessorStorage.Configuration config;
    uint32 escrowHoldPeriod;
    address sequencerUptimeFeed;
    address forwarder;
    address workflowOwner;
    address[] multiSigSigners;
    uint256 multiSigThreshold;
}
```

|          Field          |                        Type                       |                                Description                               |
| :------------------------: | :--------------------------------------------------: | :--------------------------------------------------------------------------: |
|          `salt`          |                     `bytes32`                     |               The CREATE2 salt used for every deployment.                |
|         `config`         | `IPaymentProcessorStorage.Configuration` |         The initial `PaymentProcessorStorage` configuration.          |
|    `escrowHoldPeriod`    |                     `uint32`                       | Seconds a `SimplePaymentProcessor` escrow holds a payment before release. Fixed on the processor at deployment; must be non-zero. |
| `sequencerUptimeFeed`   |                     `address`                      | Chainlink sequencer uptime feed; `address(0)` disables the check. |
|       `forwarder`        |                     `address`                      | CRE forwarder allowed to deliver reports to `PaymentAutomation`. |
|     `workflowOwner`      |                     `address`                      | CRE workflow owner carried in report metadata. |
|    `multiSigSigners`    |                    `address[]`                     |                     Initial `MultiSig` signers.                       |
|   `multiSigThreshold`   |                     `uint256`                      |                Initial `MultiSig` approval threshold.                |

`weth` moved into `config`: the WETH address now lives on [PaymentProcessorStorage](paymentprocessorstorage.sol.md#weth), where both processors read it, instead of being passed to `SimplePaymentProcessor`'s constructor. `minimumInvoiceValue` is not a field here either: the fee rate, gas threshold, and minimum invoice value are all compile-time constants on the contracts, not deploy-time parameters.

#### CoreInitCodes

Creation code (without constructor args) for the contracts [deployCore](#deploycore) deploys. Supplied by the caller so the deployer contract does not embed the system's bytecode, which would put it far past the EIP-170 size limit; the deployer appends the ABI-encoded constructor args itself.

```solidity
struct CoreInitCodes {
    bytes multiSig;
    bytes notes;
    bytes simplePaymentProcessor;
    bytes paymentAutomation;
    bytes ppStorage;
}
```

|              Field              |  Type   |                    Description                   |
| :---------------------------------: | :-----: | :----------------------------------------------------: |
|            `multiSig`             | `bytes` |              `MultiSig` creation code.            |
|              `notes`              | `bytes` |               `Notes` creation code. Not deployed in this phase; used to predict the address `SimplePaymentProcessor` is constructed against.              |
|    `simplePaymentProcessor`     | `bytes` |    `SimplePaymentProcessor` creation code.    |
|       `paymentAutomation`       | `bytes` |       `PaymentAutomation` creation code.       |
|            `ppStorage`            | `bytes` |     `PaymentProcessorStorage` creation code. Not deployed in this phase; needed to predict the storage address the other contracts are constructed against.      |

#### SystemInitCodes

Creation code (without constructor args) for the contracts [deploySystem](#deploysystem) deploys.

```solidity
struct SystemInitCodes {
    bytes oracleManager;
    bytes intermediatedPaymentProcessor;
    bytes sweeper;
    bytes notes;
    bytes ppStorage;
}
```

|              Field              |  Type   |                    Description                   |
| :---------------------------------: | :-----: | :----------------------------------------------------: |
|          `oracleManager`          | `bytes` |          `OracleManager` creation code.           |
| `intermediatedPaymentProcessor` | `bytes` | `IntermediatedPaymentProcessor` creation code. |
|             `sweeper`             | `bytes` |              `Sweeper` creation code.             |
|              `notes`              | `bytes` |               `Notes` creation code. Must match the one passed to `deployCore`, otherwise `Notes` lands away from the predicted address and the call reverts with `AddressMismatch`.               |
|            `ppStorage`            | `bytes` |     `PaymentProcessorStorage` creation code. Must match the one passed to `deployCore`, otherwise the storage contract lands away from the predicted address and the call reverts with `StorageAddressMismatch`.      |

### Events

#### CoreDeployed

Emitted once [deployCore](#deploycore) completes.

```solidity
event CoreDeployed(address multiSig, address notes, address simplePaymentProcessor, address paymentAutomation);
```

|              Name              |   Type    |                    Description                    |
| :-------------------------------: | :-------: | :---------------------------------------------------: |
|            `multiSig`            | `address` |          The deployed `MultiSig` address.          |
|             `notes`              | `address` |           The predicted `Notes` address (not yet deployed).           |
|    `simplePaymentProcessor`     | `address` |  The deployed `SimplePaymentProcessor` address.  |
|       `paymentAutomation`       | `address` |    The deployed `PaymentAutomation` adapter address.   |

#### SystemDeployed

Emitted once [deploySystem](#deploysystem) completes and the full system has been deployed.

```solidity
event SystemDeployed(
    address multiSig,
    address ppStorage,
    address notes,
    address simplePaymentProcessor,
    address paymentAutomation,
    address oracleManager,
    address intermediatedPaymentProcessor,
    address sweeper
);
```

|              Name              |   Type    |                    Description                    |
| :-------------------------------: | :-------: | :---------------------------------------------------: |
|            `multiSig`            | `address` |          The deployed `MultiSig` address.          |
|           `ppStorage`            | `address` |  The deployed `PaymentProcessorStorage` address.  |
|             `notes`              | `address` |           The deployed `Notes` address.           |
|    `simplePaymentProcessor`     | `address` |  The deployed `SimplePaymentProcessor` address.  |
|       `paymentAutomation`       | `address` |    The deployed `PaymentAutomation` adapter address.   |
|         `oracleManager`         | `address` |       The deployed `OracleManager` address.       |
| `intermediatedPaymentProcessor` | `address` | The deployed `IntermediatedPaymentProcessor` address. |
|             `sweeper`            | `address` |            The deployed `Sweeper` address.            |

### Errors

| Error | Description |
| :----: | :----------: |
| `NotDeployer()` | Thrown when `deployCore` or `deploySystem` is called by an address other than `DEPLOYER`. |
| `AlreadyDeployed()` | Thrown when a deployment phase that has already run is called again. |
| `CoreNotDeployed()` | Thrown when `deploySystem` is called before `deployCore`. |
| `StorageAddressMismatch(address predicted, address deployed)` | Thrown when the deployed `PaymentProcessorStorage` address does not match the prediction. |
| `AddressMismatch(address predicted, address deployed)` | Thrown when `PaymentAutomation` or `Notes` deploys away from the address the other contracts were built against. |

### Related interface: IAuthorizedAddressProvider

`IMasterDeployer` extends a small standalone interface, `IAuthorizedAddressProvider`, that any contract deploying `PaymentProcessorStorage` (or `Notes`) must implement:

```solidity
interface IAuthorizedAddressProvider {
    function authorizedAddresses() external view returns (address[] memory authorized);
}
```

`PaymentProcessorStorage` and `Notes` each call this on their deployer (`msg.sender`) during construction to fetch the addresses to authorize. Authorization can only be granted this way, at deployment time; see [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md#constructor) and [Notes.sol](notes.sol.md#constructor).

### Related interface: IPendingProcessorProvider

`IMasterDeployer.sol` also declares a second standalone interface, implemented by contracts that deploy `PaymentAutomation`:

```solidity
interface IPendingProcessorProvider {
    function pendingProcessor() external view returns (address processor);
}
```

`PaymentAutomation` calls this on its deployer (`msg.sender`) during construction to learn the processor it drives. Passing the processor as a constructor argument instead would make `PaymentAutomation`'s init code depend on the processor's address while the processor's init code depends on the adapter's, a cycle no CREATE2 prediction can break. Fetching it here keeps the adapter's address predictable, so the processor can hold it as an immutable; see [PaymentAutomation.sol](paymentautomation.sol.md#constructor).
