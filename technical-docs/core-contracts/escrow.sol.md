# Escrow.sol

This contract holds the amount of value sent by the payer. It is created by the Escrow factory contract in the Invoice contract. You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/Escrow.sol).

### State Variables

#### PAYMENT_PROCESSOR

The address of the payment processor.

```solidity
address public immutable PAYMENT_PROCESSOR
```

#### INVOICE_ID

The invoice ID associated with the escrow.

```solidity
uint216 public immutable INVOICE_ID
```

### Functions

#### constructor

Initializes the escrow contract and receives the deposited funds.

Sets the immutable invoice ID and payment processor address. Any ETH sent with deployment is held by the contract. ERC20 escrows receive tokens via a direct transfer.

```solidity
constructor(uint216 _invoiceId, address _paymentProcessorAddress) payable;
```

**Parameters**

|            Name            |   Type    |                             Description                             |
| :------------------------: | :-------: | :-----------------------------------------------------------------: |
|        `_invoiceId`        | `uint216` |  The unique identifier of the invoice associated with this escrow.  |
| `_paymentProcessorAddress` | `address` | The address of the payment processor contract managing the invoice. |

#### withdraw

Withdraws ETH or ERC20 tokens from the escrow contract to a specified receiver.

Only callable by the payment processor. Transfers ETH if `token` is the zero address, otherwise transfers ERC20 tokens. Uses a low-level call for both ETH and ERC20 transfers and does **not** revert on failure — the return value must be checked by the caller.

```solidity
function withdraw(address _token, address _receiver, uint256 _amount) external onlyPaymentProcessor returns (bool success);
```

**Parameters**

|    Name     |   Type    |                          Description                           |
| :---------: | :-------: | :------------------------------------------------------------: |
|  `_token`   | `address` | The address of the token to withdraw (use address(0) for ETH). |
| `_receiver` | `address` |         The address that receives the withdrawn funds.         |
|  `_amount`  | `uint256` |            The amount of ETH or tokens to transfer.            |

### Events

#### Deposited

Emitted when funds are deposited into the escrow for an invoice.

```solidity
event Deposited(uint216 indexed invoiceId, uint256 indexed value);
```

| Name        |   Type    | Description                                                |
| :----------: | :-------: | :---------------------------------------------------------: |
| `invoiceId` | `uint216` | The unique key of the invoice associated with the deposit. |
| `value`     | `uint256` | The amount of funds deposited in wei.                      |

#### Withdrawn

Emitted when funds are withdrawn from the escrow to a receiver.

```solidity
event Withdrawn(address token, address receiver, uint256 amount);
```

| Name       |   Type    | Description                                                  |
| :---------: | :-------: | :-------------------------------------------------------------: |
| `token`    | `address` | The address of the ERC20 token withdrawn, or `address(0)` for ETH. |
| `receiver` | `address` | The address that received the withdrawn funds.               |
| `amount`   | `uint256` | The amount of ETH (wei) or tokens transferred.                |

### Errors

| Error            | Description                                                                  |
| :---------------: | :---------------------------------------------------------------------------: |
| `Unauthorized()` | Thrown when an unauthorized address attempts to perform a restricted action. |
