# Escrow.sol

This contract holds the amount of value sent by the payer. It is created by the Escrow factory contract in the Invoice contract. You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/Escrow.sol).

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

Initializes the escrow contract with invoice details and deposits the funds.

This constructor sets the invoice ID, creator, payer, and payment processor addresses, and records the sent Ether as the balance.

```solidity
constructor(uint216 _invoiceId, address _creator, address _payer, address _paymentProcessorAddress) payable;
```

**Parameters**

|            Name            |   Type    |                             Description                             |
| :------------------------: | :-------: | :-----------------------------------------------------------------: |
|        `_invoiceId`        | `uint216` |  The unique identifier of the invoice associated with this escrow.  |
|         `_creator`         | `address` |                 The address of the invoice creator.                 |
|          `_payer`          | `address` |              The address of the payer for the invoice.              |
| `_paymentProcessorAddress` | `address` | The address of the payment processor contract managing the invoice. |

#### withdraw

Withdraws ETH or ERC20 tokens from the escrow contract to a specified receiver.

Only callable by the payment processor. Transfers ETH if `token` is the zero address, otherwise transfers ERC20 tokens.

```solidity
function withdraw(address _token, address _receiver, uint256 _amount) external;
```

**Parameters**

|    Name     |   Type    |                          Description                           |
| :---------: | :-------: | :------------------------------------------------------------: |
|  `_token`   | `address` | The address of the token to withdraw (use address(0) for ETH). |
| `_receiver` | `address` |         The address that receives the withdrawn funds.         |
|  `_amount`  | `uint256` |            The amount of ETH or tokens to transfer.            |

### Events

#### FundsDeposited

Emitted when funds are deposited into the escrow for an invoice.

```solidity
event FundsDeposited(uint216 indexed invoiceId, uint256 indexed value);
```

| Name        |   Type    | Description                                                |
| :----------: | :-------: | :---------------------------------------------------------: |
| `invoiceId` | `uint216` | The unique key of the invoice associated with the deposit. |
| `value`     | `uint256` | The amount of funds deposited in wei.                      |

### Errors

| Error            | Description                                                                  |
| :---------------: | :---------------------------------------------------------------------------: |
| `Unauthorized()` | Thrown when an unauthorized address attempts to perform a restricted action. |
