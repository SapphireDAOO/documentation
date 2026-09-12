# Notes.sol

The `Notes.sol` Solidity contract stores encrypted order notes and tracks per-user opened state for SapphireDao payment flows. It is designed to be called by authorized payment processors (for example, `SimplePaymentProcessor.sol` and `IntermediatedPaymentProcessor.sol`). The allowlist of who may write notes is fixed once, at construction; there is no setter to change it afterwards.

Notes.sol enables:

- Encrypted, append-only notes per order
- Optional sharing for non-authors
- Per-user opened state tracking
- Allowlist-based write access, fixed at deployment
- Write-once public key registration, so others know which key to encrypt notes to

Contract Address: [0xE818dA06Ceed4Ac6c6d4871a5Fc0226B8032834e](https://sepolia.basescan.org/address/0xE818dA06Ceed4Ac6c6d4871a5Fc0226B8032834e)

You can find the full code implementation [here](http://github.com/SapphireDAOO/payment-processor/blob/main/src/Notes.sol)

### State Variables

#### NOT_ALLOWED

Authorization flag indicating access is denied.

```solidity
uint256 public constant NOT_ALLOWED = 0
```

#### ALLOWED

Authorization flag indicating access is granted.

```solidity
uint256 public constant ALLOWED = 1
```

#### PP_STORAGE

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public immutable PP_STORAGE
```

#### CURRENT_VERSION

Active note encryption version used for newly created notes. Fixed at compile time; there is no setter.

```solidity
uint8 private constant CURRENT_VERSION = 1
```

### Functions

#### constructor

Initializes the Notes contract with a payment processor storage reference, and fixes the write allowlist for good.

Fetches the addresses to authorize from its deployer: `msg.sender` must implement [`IAuthorizedAddressProvider`](masterdeployer.sol.md#related-interface-iauthorizedaddressprovider) (in practice, [MasterDeployer.sol](masterdeployer.sol.md)), and this contract calls `authorizedAddresses()` on it once, at construction. Unlike [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md#constructor), this does not emit a per-address event; the allowlist is simply populated silently. There is no setter, so the set of addresses allowed to write notes can never change after deployment.

```solidity
constructor(address _paymentProcessorStorageAddress) ;
```

**Parameters**

|               Name                |   Type    |             Description              |
| :-------------------------------: | :-------: | :----------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The address of the storage contract. |

#### createNote

Create a note under an order.

```solidity
function createNote(uint216 _invoiceId, address _author, bytes calldata _encryptedContent, bool _share)
    external
    onlyAuthorized
    returns (uint256 noteId);
```

**Parameters**

|        Name         |   Type    |                 Description                  |
| :-----------------: | :-------: | :------------------------------------------: |
|    `_invoiceId`     | `uint216` |             Invoice identifier.              |
|      `_author`      | `address` |                 Note author.                 |
| `_encryptedContent` |  `bytes`  |           Encrypted note payload.            |
|      `_share`       |  `bool`   | Whether the note is shared with non-authors. |

**Returns**

|   Name   |   Type    |      Description       |
| :------: | :-------: | :--------------------: |
| `noteId` | `uint256` | Newly created note id. |

#### setOpened

Mark a note as opened for an account.

Only authorized callers can update opened state. Reverts with Unauthorized if the note is not shared; opened state can only be tracked for shared notes.

```solidity
function setOpened(uint216 _invoiceId, address _account, uint256 _noteId) external onlyAuthorized;
```

**Parameters**

|     Name     |   Type    |              Description               |
| :----------: | :-------: | :------------------------------------: |
| `_invoiceId` | `uint216` |          Invoice identifier.           |
|  `_account`  | `address` | Account whose opened state is updated. |
|  `_noteId`   | `uint256` |            Note identifier.            |

#### getNoteCount

Get the total number of notes for an order.

```solidity
function getNoteCount(uint216 _invoiceId) external view returns (uint256 totalNotes);
```

**Parameters**

|     Name     |   Type    |     Description     |
| :----------: | :-------: | :-----------------: |
| `_invoiceId` | `uint216` | Invoice identifier. |

**Returns**

|     Name     |   Type    |                 Description                  |
| :----------: | :-------: | :------------------------------------------: |
| `totalNotes` | `uint256` | Total number of notes created for the order. |

#### isOpened

Check if a note is opened for a specific user.

```solidity
function isOpened(uint216 _invoiceId, uint256 _noteId, address _user) external view returns (bool isOpen);
```

**Parameters**

|     Name     |   Type    |     Description     |
| :----------: | :-------: | :-----------------: |
| `_invoiceId` | `uint216` | Invoice identifier. |
|  `_noteId`   | `uint256` |  Note identifier.   |
|   `_user`    | `address` |  Address to check.  |

**Returns**

|   Name   |  Type  |               Description                |
| :------: | :----: | :--------------------------------------: |
| `isOpen` | `bool` | True if the note is opened for the user. |

#### getNote

Get a single note if visible to the caller.

```solidity
function getNote(uint216 _invoiceId, uint256 _noteId)
    external
    view
    returns (address author, bool share, bytes memory content, bool openedStatus, uint8 version);
```

**Parameters**

|     Name     |   Type    |     Description     |
| :----------: | :-------: | :-----------------: |
| `_invoiceId` | `uint216` | Invoice identifier. |
|  `_noteId`   | `uint256` |  Note identifier.   |

**Returns**

|      Name      |   Type    |               Description               |
| :------------: | :-------: | :-------------------------------------: |
|    `author`    | `address` |              Note author.               |
|    `share`     |  `bool`   |       Whether the note is shared.       |
|   `content`    |  `bytes`  |         Encrypted note content.         |
| `openedStatus` |  `bool`   | Whether the caller has opened the note. |
|   `version`    |  `uint8`  |          Note schema version.           |

#### setPublicKey

Registers the caller's wallet public key, so others can encrypt notes to it.

An account registers under its own slot, so a caller can only ever set its own key. The key is not verified against the caller beyond a 64-byte length check; reverts with `InvalidPublicKey` otherwise. The key is stored alongside the note encryption version active at registration, so a reader knows which scheme the key was published for. Write-once: reverts with `PublicKeyAlreadySet` if the caller already registered a key.

```solidity
function setPublicKey(bytes calldata _publicKey) external;
```

**Parameters**

|      Name      |  Type   |            Description            |
| :-------------: | :-----: | :------------------------------------: |
| `_publicKey` | `bytes` | The caller's 64-byte public key. |

#### getPublicKey

Returns the public key an account registered.

```solidity
function getPublicKey(address _account) external view returns (PublicKey memory publicKey);
```

**Parameters**

|    Name    |   Type    |        Description        |
| :---------: | :-------: | :----------------------------: |
| `_account` | `address` | The account to look up. |

**Returns**

|    Name     |    Type     |                                       Description                                       |
| :----------: | :---------: | :-------------------------------------------------------------------------------------------: |
| `publicKey` | `PublicKey` | The registered key and the note version it was registered under. The `key` is empty when the account has not registered one. |

#### getCurrentVersion

Returns the active note encryption version. Always returns `CURRENT_VERSION`; there is no setter.

```solidity
function getCurrentVersion() external view returns (uint8 v);
```

**Returns**

| Name |  Type   |        Description        |
| :--: | :-----: | :-----------------------: |
| `v`  | `uint8` | The current note version. |

### Structs

#### Note

Stored note data.

```solidity
struct Note {
    address author;
    bool share;
    bool exists;
    uint8 version;
    bytes content;
}
```

|   Field   |   Type    |                   Description                    |
| :-------: | :-------: | :----------------------------------------------: |
| `author`  | `address` |                 The note author.                 |
|  `share`  |  `bool`   | Whether the note is shared with the other party. |
| `exists`  |  `bool`   |             Whether the note exists.             |
| `version` |  `uint8`  |             The note schema version.             |
| `content` |  `bytes`  |           The encrypted note content.            |

#### PublicKey

A public key registered by an account, so others can encrypt notes to it.

```solidity
struct PublicKey {
    bytes key;
    uint8 version;
}
```

|  Field   |  Type   |                          Description                          |
| :------: | :-----: | :------------------------------------------------------------: |
|  `key`   | `bytes` |                  The registered public key.                  |
| `version` | `uint8` | The note encryption version that was active when the key was registered. |

### Events

#### NoteCreated

Emitted when a new note is created for an invoice.

```solidity
event NoteCreated(uint216 indexed invoiceId, uint256 indexed noteId, address indexed author, bool share, bytes encryptedContent);
```

|        Name        |   Type    |                            Description                            |
| :----------------: | :-------: | :---------------------------------------------------------------: |
|    `invoiceId`     | `uint216` | The unique identifier of the invoice the note is associated with. |
|      `noteId`      | `uint256` |            The unique identifier of the created note.             |
|      `author`      | `address` |         The address of the account that created the note.         |
|      `share`       |  `bool`   |     Indicates whether the note is shared with other parties.      |
| `encryptedContent` |  `bytes`  |                The encrypted contents of the note.                |

#### PublicKeySet

Emitted once when an account registers its public key.

Never emitted twice for the same account: registration is write-once.

```solidity
event PublicKeySet(address indexed account, bytes publicKey, uint8 version);
```

|     Name      |   Type    |                     Description                     |
| :------------: | :-------: | :-----------------------------------------------------: |
|   `account`   | `address` |        The account that registered the key.        |
| `publicKey`  |  `bytes`  |         The public key that was registered.         |
|   `version`   |  `uint8`  | The note encryption version active at registration. |

#### NoteStateChanged

Emitted when a user changes their opened state for a note.

```solidity
event NoteStateChanged(uint216 indexed invoiceId, uint256 indexed noteId, address indexed user, bool opened);
```

|    Name     |   Type    |                        Description                        |
| :---------: | :-------: | :-------------------------------------------------------: |
| `invoiceId` | `uint216` | The unique identifier of the invoice the note belongs to. |
|  `noteId`   | `uint256` |            The unique identifier of the note.             |
|   `user`    | `address` |   The address of the user whose note state was updated.   |
|  `opened`   |  `bool`   | Whether the note is marked as opened or not by the user.  |

### Errors

|      Error       |                         Description                          |
| :--------------: | :----------------------------------------------------------: |
| `Unauthorized()` | Thrown when the caller is not authorized to access the note. |
| `EmptyContent()` |       Thrown when creating a note with empty content.        |
| `NoteNotFound()` |        Thrown when the requested note does not exist.        |
| `InvalidPublicKey()` |     Thrown when the supplied public key is not 64 bytes.     |
| `PublicKeyAlreadySet()` | Thrown when an account that already registered a public key tries to register another. |
