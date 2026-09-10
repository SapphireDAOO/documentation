# FeeAuthorizationLib.sol

FeeAuthorizationLib verifies that a per-invoice fee receiver was authorized by the configured fee signer ([`PaymentProcessorStorage.getFeeSigner`](../core-contracts/paymentprocessorstorage.sol.md#getfeesigner)). [SimplePaymentProcessor.sol](../core-contracts/simplepaymentprocessor.sol.md) and [IntermediatedPaymentProcessor.sol](../core-contracts/intermediatedpaymentprocessor.sol.md) call it when a buyer's chosen fee receiver is attached to an invoice, at `acceptPayment` and `payInvoice` respectively; `IntermediatedPaymentProcessor` also calls its meta-invoice overload at [`payMetaInvoiceWithValue`](../core-contracts/intermediatedpaymentprocessor.sol.md#paymetainvoicewithvalue) and [`payMetaInvoice`](../core-contracts/intermediatedpaymentprocessor.sol.md#paymetainvoice).

The signed message is an EIP-191 `personal_sign` digest over the processor address, the chain id, the invoice ID, and the fee receiver(s), so a signature is bound to one invoice on one processor on one chain. Invoices only accept a fee receiver once, in a state transition that cannot be repeated, so no separate nonce or deadline is required. A meta-invoice authorizes the whole array of sub-invoice receivers with a single signature; because the array is abi-encoded as a dynamic type its digest cannot collide with a single-receiver one.

For why the fee receiver is unique per invoice rather than a fixed treasury address, see [Fee Receiver Privacy](../fee-receiver-privacy.md).

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

#### isAuthorized (meta-invoice)

Recovers the signer of a meta-invoice fee authorization covering every sub-invoice.

```solidity
function isAuthorized(
    address _feeSigner,
    uint216 _metaInvoiceId,
    address[] memory _feeReceivers,
    bytes memory _signature
) internal view returns (bool valid);
```

**Parameters**

|       Name       |     Type    |                                Description                               |
| :---------------: | :---------: | :---------------------------------------------------------------------------: |
|   `_feeSigner`   |  `address`  |             The address expected to have produced the signature.             |
| `_metaInvoiceId` |  `uint216`  |             The meta-invoice whose sub-invoices are being paid.             |
| `_feeReceivers`  | `address[]` | The fee receivers, index-aligned with the meta-invoice's sub-invoice IDs. |
|   `_signature`   |   `bytes`   |          The 65-byte ECDSA signature over the authorization digest.          |

**Returns**

|  Name   |  Type  |                        Description                       |
| :-----: | :----: | :----------------------------------------------------------: |
| `valid` | `bool` | True when the recovered address equals `_feeSigner`. |

#### digest (meta-invoice)

Builds the EIP-191 digest authorizing every fee receiver of a meta-invoice at once.

```solidity
function digest(uint216 _metaInvoiceId, address[] memory _feeReceivers)
    internal
    view
    returns (bytes32 authorizationDigest);
```

**Parameters**

|       Name       |     Type    |                                Description                               |
| :---------------: | :---------: | :---------------------------------------------------------------------------: |
| `_metaInvoiceId` |  `uint216`  |             The meta-invoice whose sub-invoices are being paid.             |
| `_feeReceivers`  | `address[]` | The fee receivers, index-aligned with the meta-invoice's sub-invoice IDs. |

**Returns**

|         Name          |   Type    |                    Description                    |
| :--------------------: | :-------: | :--------------------------------------------------: |
| `authorizationDigest` | `bytes32` | The `personal_sign` digest to be signed. |
