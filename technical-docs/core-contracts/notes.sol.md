# Notes.sol

The `Notes.sol` Solidity contract stores encrypted order notes and tracks per-user opened state for SapphireDao payment flows. It is designed to be called by authorized payment processors (for example, `SimplePaymentProcessor.sol` and `AdvancedPaymentProcessor.sol`) and uses the `PaymentProcessorStorage.sol` owner to manage authorization.

Notes.sol enables:

* Encrypted, append-only notes per order
* Optional sharing for non-authors
* Per-user opened state tracking
* Allowlist-based write access controlled by the storage owner

Contract Address: [0xbe210c16e990e74a92eb85060bb33eb03418c565](https://sepolia.etherscan.io/address/0xbe210c16e990e74a92eb85060bb33eb03418c565)

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/Notes.sol)

### State Variables

#### NOT\_ALLOWED

Authorization flag indicating access is denied.

```solidity
uint256 public constant NOT_ALLOWED = 0
```

#### ALLOWED

Authorization flag indicating access is granted.

```solidity
uint256 public constant ALLOWED = 1
```

#### ppStorage

Reference to the external Payment Processor storage contract.

```solidity
IPaymentProcessorStorage public immutable ppStorage
```

### Functions

#### constructor

Initializes the Notes contract with a payment processor storage reference.

```solidity
constructor(address _paymentProcessorStorageAddress) ;
```

**Parameters**

|                Name               |    Type   |              Description             |
| :-------------------------------: | :-------: | :----------------------------------: |
| `_paymentProcessorStorageAddress` | `address` | The address of the storage contract. |

#### createNote

Create a note under an order.

```solidity
function createNote(uint216 _invoiceId, address _author, bytes calldata _encryptedContent, bool _share)
    external
    override
    onlyAuthorized
    returns (uint256 noteId);
```

**Parameters**

|         Name        |    Type   |                  Description                 |
| :-----------------: | :-------: | :------------------------------------------: |
|     `_invoiceId`    | `uint216` |               Order identifier.              |
|      `_author`      | `address` |                 Note author.                 |
| `_encryptedContent` |  `bytes`  |            Encrypted note payload.           |
|       `_share`      |   `bool`  | Whether the note is shared with non-authors. |

**Returns**

|   Name   |    Type   |       Description      |
| :------: | :-------: | :--------------------: |
| `noteId` | `uint256` | Newly created note id. |

#### setOpened

Mark a note as opened or unopened for an account.

Only authorized callers can update opened state. Reverts with Unauthorized if the note is not shared — opened state can only be tracked for shared notes.

```solidity
function setOpened(uint216 _invoiceId, address _account, uint256 _noteId, bool _open) external;
```

**Parameters**

|     Name     |    Type   |                   Description                   |
| :----------: | :-------: | :---------------------------------------------: |
| `_invoiceId` | `uint216` | Invoice identifier.                             |
| `_account`   | `address` | Account whose opened state is updated.          |
|   `_noteId`  | `uint256` | Note identifier.                                |
|    `_open`   |   `bool`  | New opened state for the account.               |

#### getNoteCount

Get the total number of notes for an order.

```solidity
function getNoteCount(uint216 _invoiceId) external view override returns (uint256 totalNotes);
```

**Parameters**

|     Name     |    Type   |    Description    |
| :----------: | :-------: | :---------------: |
| `_invoiceId` | `uint216` | Order identifier. |

**Returns**

|     Name     |    Type   |                  Description                 |
| :----------: | :-------: | :------------------------------------------: |
| `totalNotes` | `uint256` | Total number of notes created for the order. |

#### isOpened

Check if a note is opened for a specific user.

```solidity
function isOpened(uint216 _invoiceId, uint256 _noteId, address _user) external view override returns (bool isOpen);
```

**Parameters**

|     Name     |    Type   |    Description    |
| :----------: | :-------: | :---------------: |
| `_invoiceId` | `uint216` | Order identifier. |
|   `_noteId`  | `uint256` |  Note identifier. |
|    `_user`   | `address` | Address to check. |

**Returns**

|   Name   |  Type  |                Description               |
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

|     Name     |    Type   |    Description    |
| :----------: | :-------: | :---------------: |
| `_invoiceId` | `uint216` | Order identifier. |
|   `_noteId`  | `uint256` |  Note identifier. |

**Returns**

|      Name      |    Type   |               Description               |
| :------------: | :-------: | :-------------------------------------: |
|    `author`    | `address` |               Note author.              |
|     `share`    |   `bool`  |       Whether the note is shared.       |
|    `content`   |  `bytes`  |         Encrypted note content.         |
| `openedStatus` |   `bool`  | Whether the caller has opened the note. |
|    `version`   |  `uint8`  |            Note schema version.         |

#### updateVersion

Updates the active note encryption version.

This affects only notes created after the update. Existing notes retain their original version and remain decryptable using the encrypter associated with their stored version.

```solidity
function updateVersion(uint8 _newVersion) external;
```

**Parameters**

|      Name     |   Type  |                     Description                    |
| :-----------: | :-----: | :------------------------------------------------: |
| `_newVersion` | `uint8` | The new note encryption version identifier to use. |

#### setAuthorized

Updates the authorization status for a user.

```solidity
function setAuthorized(address _user, bool _enabled) external;
```

**Parameters**

|    Name    |    Type   |               Description              |
| :--------: | :-------: | :------------------------------------: |
|   `_user`  | `address` |         The address to update.         |
| `_enabled` |   `bool`  | Whether the user should be authorized. |

#### getCurrentVersion

```solidity
function getCurrentVersion() external view returns (uint8 v);
```

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

| Field     | Type      | Description                                      |
| :--------: | :-------: | :-----------------------------------------------: |
| `author`  | `address` | The note author.                                 |
| `share`   | `bool`    | Whether the note is shared with the other party. |
| `exists`  | `bool`    | Whether the note exists.                         |
| `version` | `uint8`   | The note schema version.                         |
| `content` | `bytes`   | The encrypted note content.                      |

### Events

#### NoteCreated

Emitted when a new note is created for an invoice.

```solidity
event NoteCreated(uint216 indexed invoiceId, uint256 indexed noteId, address indexed author, bool share, bytes encryptedContent);
```

| Name               | Type      | Description                                                   |
| :-----------------: | :-------: | :------------------------------------------------------------: |
| `invoiceId`        | `uint216` | The unique identifier of the invoice the note is associated with. |
| `noteId`           | `uint256` | The unique identifier of the created note.                    |
| `author`           | `address` | The address of the account that created the note.             |
| `share`            | `bool`    | Indicates whether the note is shared with other parties.      |
| `encryptedContent` | `bytes`   | The encrypted contents of the note.                           |

#### NoteStateChanged

Emitted when a user changes their opened state for a note.

```solidity
event NoteStateChanged(uint216 indexed invoiceId, uint256 indexed noteId, address indexed user, bool opened);
```

| Name        | Type      | Description                                                       |
| :----------: | :-------: | :----------------------------------------------------------------: |
| `invoiceId` | `uint216` | The unique identifier of the invoice the note belongs to.         |
| `noteId`    | `uint256` | The unique identifier of the note.                                |
| `user`      | `address` | The address of the user whose note state was updated.             |
| `opened`    | `bool`    | Whether the note is marked as opened or not by the user.          |

### Errors

| Error | Description |
| :----: | :----------: |
| `Unauthorized()` | Thrown when the caller is not authorized to access the note. |
| `EmptyContent()` | Thrown when creating a note with empty content. |
| `NoteNotFound()` | Thrown when the requested note does not exist. |
