# FeeAuthorizationLib.sol

FeeAuthorizationLib verifies that a per-invoice fee receiver was authorized by the configured fee signer ([`PaymentProcessorStorage.getFeeSigner`](../core-contracts/paymentprocessorstorage.sol.md#getfeesigner)). Both [SimplePaymentProcessor.sol](../core-contracts/simplepaymentprocessor.sol.md) and [IntermediatedPaymentProcessor.sol](../core-contracts/intermediatedpaymentprocessor.sol.md) call it when a buyer's chosen fee receiver is attached to an invoice, at `acceptPayment` and `payInvoice` respectively.

The signed message is an EIP-191 `personal_sign` digest over the processor address, the chain id, the invoice ID, and the fee receiver, so a signature is bound to one invoice on one processor on one chain. Invoices only accept a fee receiver once, in a state transition that cannot be repeated, so no separate nonce or deadline is required.

You can find the full implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/main/src/libraries/FeeAuthorizationLib.sol)

### Functions

#### isAuthorized

Recovers the signer of a fee authorization and reports whether it matches `_feeSigner`.

```solidity
function isAuthorized(address _feeSigner, uint216 _invoiceId, address _feeReceiver, bytes memory _signature)
    internal
    view
    returns (bool valid);
```

**Parameters**

|      Name      |    Type   |                              Description                             |
| :-------------: | :-------: | :--------------------------------------------------------------------: |
|  `_feeSigner`  | `address` |         The address expected to have produced the signature.         |
|  `_invoiceId`  | `uint216` |          The invoice the fee receiver is being attached to.          |
| `_feeReceiver` | `address` |                  The fee receiver being authorized.                  |
|  `_signature`  |  `bytes`  | The 65-byte ECDSA signature over the authorization digest. |

**Returns**

|  Name   |  Type  |                        Description                       |
| :-----: | :----: | :----------------------------------------------------------: |
| `valid` | `bool` | True when the recovered address equals `_feeSigner`. |

#### digest

Builds the EIP-191 digest a fee signer must sign to authorize a fee receiver.

```solidity
function digest(uint216 _invoiceId, address _feeReceiver) internal view returns (bytes32 authorizationDigest);
```

**Parameters**

|      Name      |    Type   |                     Description                     |
| :-------------: | :-------: | :----------------------------------------------------: |
|  `_invoiceId`  | `uint216` | The invoice the fee receiver is being attached to. |
| `_feeReceiver` | `address` |             The fee receiver being authorized.             |

**Returns**

|         Name          |   Type    |                    Description                    |
| :--------------------: | :-------: | :--------------------------------------------------: |
| `authorizationDigest` | `bytes32` | The `personal_sign` digest to be signed. |
