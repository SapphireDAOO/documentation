# Sweeper.sol

Collects ERC20 tokens held by many addresses into a single destination in one transaction. Moves tokens with `transferFrom`, so each holder must have approved this contract first; the sweeper can never take more than a holder allowed. The destination is chosen per call, so only the [PaymentProcessorStorage](paymentprocessorstorage.sol.md) owner may sweep: any other caller could otherwise send approved balances to themselves.

Deployed standalone, independently of [MasterDeployer](masterdeployer.sol.md); it only needs the shared `PaymentProcessorStorage` address to know who its owner is.

The holders it sweeps from are typically per-invoice stealth addresses that pre-approved it via an EIP-7702 delegation; see [Fee Receiver Privacy](../fee-receiver-privacy.md) for the full design.

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/Sweeper.sol)

### State Variables

#### ppStorage

Reference to the external Payment Processor storage contract, which holds the owner.

```solidity
IPaymentProcessorStorage public immutable ppStorage
```

### Functions

#### constructor

Initializes the sweeper against the shared payment processor storage.

Reverts with `InvalidAddress` if `_paymentProcessorStorageAddress` is the zero address.

```solidity
constructor(address _paymentProcessorStorageAddress);
```

**Parameters**

|                Name                |   Type    |                       Description                      |
| :---------------------------------: | :-------: | :--------------------------------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The storage contract whose owner may sweep. |

#### sweep

Pulls `_amounts[i]` of `_token` from each `_from[i]` into `_destination`.

Callable only by the `PaymentProcessorStorage` owner, which is what stops anyone else from redirecting the balances holders approved to this contract. Every holder must have approved at least its amount beforehand; the sweep is atomic, so one failing transfer reverts the batch. Reverts with `LengthMismatch` if `_from` and `_amounts` aren't the same length, `EmptySweep` if `_from` is empty, or `InvalidAddress` if `_token` or `_destination` is the zero address. Entries with a zero amount are skipped.

```solidity
function sweep(address _token, address[] calldata _from, uint256[] calldata _amounts, address _destination)
    external
    onlyOwner
    returns (uint256 total);
```

**Parameters**

|      Name      |     Type    |                       Description                      |
| :-------------: | :---------: | :---------------------------------------------------------: |
|     `_token`    |   `address`   |            The ERC20 token to sweep.            |
|      `_from`    | `address[]` |               The holders to pull from.               |
|    `_amounts`   | `uint256[]` | The amount to pull from each holder, index-aligned with `_from`. |
| `_destination`  |   `address`   |    The address to send the collected tokens to.    |

**Returns**

|  Name   |   Type    |               Description               |
| :-----: | :-------: | :----------------------------------------: |
| `total` | `uint256` | The total amount swept into `_destination`. |

#### sweepable

Returns how much of `_token` a sweep can currently pull from `_holder`.

The lesser of the holder's balance and the allowance it granted this contract.

```solidity
function sweepable(address _token, address _holder) external view returns (uint256 amount);
```

**Parameters**

|   Name    |   Type    |         Description        |
| :-------: | :-------: | :---------------------------: |
|  `_token`  | `address` |   The ERC20 token to check.   |
| `_holder`  | `address` |      The holder to check.     |

**Returns**

|  Name    |   Type    |        Description       |
| :------: | :-------: | :--------------------------: |
| `amount` | `uint256` | The sweepable amount. |

### Events

#### Swept

Emitted for each holder a sweep pulls tokens from.

```solidity
event Swept(address indexed token, address indexed from, address indexed destination, uint256 amount);
```

|     Name      |   Type    |              Description              |
| :------------: | :-------: | :----------------------------------------: |
|    `token`     | `address` |        The ERC20 token swept.         |
|     `from`     | `address` |    The holder the tokens came from.   |
| `destination`  | `address` | The address the tokens were sent to.  |
|    `amount`    | `uint256` |     The amount transferred.           |

### Errors

|            Error            |                              Description                              |
| :--------------------------: | :------------------------------------------------------------------------: |
|      `NotAuthorized()`       |     Thrown when the caller is not the PaymentProcessorStorage owner.     |
|      `LengthMismatch()`      |  Thrown when the source and amount arrays are not the same length.  |
|        `EmptySweep()`        |     Thrown when a sweep is requested with no source addresses.      |
|       `InvalidAddress()`     | Thrown when the zero address is supplied as the token or the destination. |
