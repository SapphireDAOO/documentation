# EscrowFactory.sol

An abstract factory for deploying [Escrow.sol](escrow.sol.md) contracts deterministically via CREATE2, so an escrow's address can be predicted before it's deployed. It is inherited by a payment processor contract rather than deployed standalone. You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/EscrowFactory.sol).

### Functions

#### computeSalt

Computes a unique salt for deterministic escrow deployment via CREATE2. The salt is derived from the seller, buyer, and invoice ID, ensuring each escrow has a unique and reproducible address across all invoice types.

```solidity
function computeSalt(address _seller, address _buyer, uint216 _invoiceId) public pure returns (bytes32 salt);
```

**Parameters**

|    Name     |    Type   |                     Description                    |
| :----------: | :-------: | :----------------------------------------------------: |
|  `_seller`  | `address` |        The address of the invoice seller.        |
|  `_buyer`   | `address` |        The address of the invoice buyer.         |
| `_invoiceId` | `uint216` |         The unique nonce of the invoice.          |

**Returns**

|  Name  |   Type    |               Description               |
| :-----: | :-------: | :-----------------------------------------: |
| `salt` | `bytes32` | The computed salt for the escrow deployment. |

#### getPredictedAddress

Predicts the deterministic address of an escrow contract for a given salt. The invoice ID is required because it is encoded into the escrow's constructor args, which feeds into the CREATE2 bytecode hash.

```solidity
function getPredictedAddress(bytes32 _salt, uint216 _invoiceId) public view returns (address predictedAddress);
```

**Parameters**

|     Name     |    Type   |                          Description                         |
| :-----------: | :-------: | :----------------------------------------------------------------: |
|    `_salt`   | `bytes32` |        The unique salt value used for the contract deployment.       |
| `_invoiceId` | `uint216` | The invoice ID encoded into the escrow's constructor args. |

**Returns**

|        Name        |   Type    |             Description            |
| :--------------------: | :-------: | :------------------------------------: |
| `predictedAddress` | `address` | The predicted escrow address. |

#### \_create

Deploys a new Escrow contract deterministically using CREATE2. Internal — called by the inheriting processor contract, not part of the public ABI. Uses a salt derived from the seller, buyer, and invoice ID. The `Escrow` constructor receives only the invoice ID and the payment processor address (`this`); seller and buyer are used solely for salt derivation, not passed as constructor args. For ERC20 payments, `value` is forced to zero and tokens are transferred to the escrow separately.

```solidity
function _create(EscrowCreationParams memory _params) internal returns (address escrow);
```

**Parameters**

|   Name    |          Type          |                Description                |
| :--------: | :--------------------: | :-------------------------------------------: |
| `_params` | `EscrowCreationParams` | See [EscrowCreationParams](#escrowcreationparams). |

**Returns**

|   Name   |   Type    |                    Description                   |
| :-------: | :-------: | :--------------------------------------------------: |
| `escrow` | `address` | The address of the newly deployed Escrow contract. |

### Structs

#### EscrowCreationParams

Parameters required to initialize an escrow contract.

```solidity
struct EscrowCreationParams {
    address seller;
    address buyer;
    uint216 invoiceId;
    uint256 value;
    address paymentToken;
}
```

|      Field      |    Type   |                                              Description                                             |
| :---------------: | :-------: | :--------------------------------------------------------------------------------------------------------: |
|     `seller`     | `address` |                                  The address of the seller.                                   |
|      `buyer`     | `address` |                              The address of the buyer (payer).                               |
|   `invoiceId`   | `uint216` |                       The unique identifier associated with the invoice.                     |
|      `value`     | `uint256` |          The total amount to be held in escrow, denominated in the payment token or native currency.       |
|  `paymentToken` | `address` | The address of the token used for payment. Use `address(0)` for native currency (e.g., ETH). |

### Events

#### EscrowCreated

Emitted when a new escrow contract is created.

```solidity
event EscrowCreated(uint216 indexed invoiceId, address indexed escrow);
```

|    Name    |    Type   |                       Description                      |
| :---------: | :-------: | :----------------------------------------------------------: |
| `invoiceId` | `uint216` | The unique ID of the invoice associated with the escrow. |
|   `escrow`  | `address` |       The address of the newly created escrow contract.      |
