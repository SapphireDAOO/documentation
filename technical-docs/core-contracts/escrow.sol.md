# Escrow.sol

This contract holds the amount of value sent by the payer. It is created by the Escrow factory contract in the Invoice contract. You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/Escrow.sol).

### State Variables

#### BUYER

The address of the buyer associated with this escrow.

```solidity
address public immutable BUYER
```

#### SELLER

The address of the seller associated with this escrow.

```solidity
address public immutable SELLER
```

#### PAYMENT\_PROCESSOR

The address of the payment processor.

```solidity
address public immutable PAYMENT_PROCESSOR
```

#### INVOICE

The invoice ID associated with the escrow.

```solidity
uint256 public immutable INVOICE
```

### Functions

#### onlyPaymentProcessor

Restricts access to the payment processor contract.

Reverts with Unauthorized if the caller is not the payment processor.

```solidity
modifier onlyPaymentProcessor() ;
```

#### constructor

Initializes the escrow contract with invoice details and deposits the funds.

This constructor sets the invoice ID, creator, payer, and payment processor addresses, and records the sent Ether as the balance.

```solidity
constructor(uint216 _invoiceId, address _creator, address _payer, address _paymentProcessorAddress) payable;
```

**Parameters**

|            Name            |    Type   |                             Description                             |
| :------------------------: | :-------: | :-----------------------------------------------------------------: |
|        `_invoiceId`        | `uint216` |  The unique identifier of the invoice associated with this escrow.  |
|         `_creator`         | `address` |                 The address of the invoice creator.                 |
|          `_payer`          | `address` |              The address of the payer for the invoice.              |
| `_paymentProcessorAddress` | `address` | The address of the payment processor contract managing the invoice. |

#### withdraw

Withdraws ETH or ERC20 tokens from the escrow contract to a specified receiver.

Only callable by the payment processor. Transfers ETH if `token` is the zero address, otherwise transfers ERC20 tokens.

```solidity
function withdraw(address _token, address _receiver, uint256 _amount) external onlyPaymentProcessor;
```

**Parameters**

|     Name    |    Type   |                           Description                          |
| :---------: | :-------: | :------------------------------------------------------------: |
|   `_token`  | `address` | The address of the token to withdraw (use address(0) for ETH). |
| `_receiver` | `address` |         The address that receives the withdrawn funds.         |
|  `_amount`  | `uint256` |            The amount of ETH or tokens to transfer.            |

#### \_onlyPaymentProcessor

Ensures that the caller is the authorized payment processor.

Reverts with `Unauthorized` if `msg.sender` is not equal to `paymentProcessor`.

```solidity
function _onlyPaymentProcessor() internal view;
```
