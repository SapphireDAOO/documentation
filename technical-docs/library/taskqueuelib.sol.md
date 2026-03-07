# TaskQueueLib.sol

TaskQueueLib is a small scheduling helper that keeps track of “what should happen next” using a priority queue (a binary min-heap). Each scheduled task has an ID (for example, an invoice ID) and a due time (a timestamp). The library packs both values into a single number so it can store, compare, and sort tasks efficiently, with the earliest due time always sitting at the top of the queue.

When a new task needs to be scheduled, it is inserted with its due time. If the due time changes, the task can be rescheduled, and the heap reorders itself so the next upcoming task is still on top. Tasks can also be removed directly, which is useful if an invoice is canceled or no longer needs an automated action.

To run scheduled actions, the library checks whether the next task is due “now” and, if it is, it processes tasks in order. It does this carefully with a gas/limit safeguard so it does not try to process too many tasks at once. For each task, it calls back into the payment processor’s release logic and uses simple status codes to decide whether to remove the task, skip it, or stop if something is wrong.

Keys are encoded into `uint256` with `dueTime` in the high 40 bits and `id` in the low 216 bits.

You can find the full implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/libraries/TaskQueueLib.sol)

### State Variables

#### NOT\_ELIGIBLE\_FOR\_RELEASE

Returned when a task is not yet eligible for release (e.g. wrong status or too early).

```solidity
uint256 constant NOT_ELIGIBLE_FOR_RELEASE = 1
```

#### ERROR

Returned when an error occurs during task processing (e.g. invalid index or heap inconsistency).

```solidity
uint256 constant ERROR = 2
```

#### SUCCESSFUL

Returned when a task has been successfully released and removed from the heap.

```solidity
uint256 constant SUCCESSFUL = 3
```

### Functions

#### insert

Inserts a new task into the heap.

Reverts if the task with the same ID already exists.

```solidity
function insert(Heap storage _heap, uint216 _id, uint40 _dueTime, mapping(uint216 => uint256) storage _index)
    internal;
```

**Parameters**

|    Name    |                  Type                 |                     Description                    |
| :--------: | :-----------------------------------: | :------------------------------------------------: |
|   `_heap`  |                 `Heap`                |              The heap storage struct.              |
|    `_id`   |               `uint216`               |         The unique identifier of the task.         |
| `_dueTime` |                `uint40`               |     The due timestamp (in seconds) of the task.    |
|  `_index`  | `mapping(uint216 => uint256) storage` | Mapping from task ID to 1-based index in the heap. |

#### removeAt

Removes a task from a specific index in the heap.

```solidity
function removeAt(Heap storage _heap, uint256 _i, mapping(uint216 => uint256) storage _index) internal;
```

**Parameters**

|   Name   |                  Type                 |                     Description                    |
| :------: | :-----------------------------------: | :------------------------------------------------: |
|  `_heap` |                 `Heap`                |              The heap storage struct.              |
|   `_i`   |               `uint256`               |        The zero-based index to remove from.        |
| `_index` | `mapping(uint216 => uint256) storage` | Mapping from task ID to 1-based index in the heap. |

#### reschedule

Updates the due time of an existing task and rebalances the heap.

Reverts if task ID is not found.

```solidity
function reschedule(Heap storage _heap, uint216 _id, uint40 _newDueAt, mapping(uint216 => uint256) storage _index)
    internal;
```

**Parameters**

|     Name    |                  Type                 |                     Description                    |
| :---------: | :-----------------------------------: | :------------------------------------------------: |
|   `_heap`   |                 `Heap`                |              The heap storage struct.              |
|    `_id`    |               `uint216`               |             The task ID to reschedule.             |
| `_newDueAt` |                `uint40`               |      The new due time (timestamp in seconds).      |
|   `_index`  | `mapping(uint216 => uint256) storage` | Mapping from task ID to 1-based index in the heap. |

#### processDueTask

Iterates through the heap and attempts to release due tasks based on available gas.

This function uses a gas threshold to avoid running out of gas. It repeatedly attempts to release the current task using the provided `releaseCallback`. If the task is not eligible or an error occurs, it moves to the next task in the heap using the `index` mapping.

* `SUCCESSFUL`: The task was released and the next is processed.
* `NOT_ELIGIBLE_FOR_RELEASE`: Skips to the next task.
* `ERROR`: Aborts the loop.

```solidity
function processDueTask(
    Heap storage _heap,
    function(uint216) internal returns (uint256) _releaseCallback,
    uint256 _gasThresold
) internal;
```

**Parameters**

|        Name        |                       Type                      |                                 Description                                |
| :----------------: | :---------------------------------------------: | :------------------------------------------------------------------------: |
|       `_heap`      |                      `Heap`                     |               The heap data structure storing encoded tasks.               |
| `_releaseCallback` | `function (uint216) internal returns (uint256)` | A function that attempts to release a task by ID, returning a status code. |
|   `_gasThresold`   |                    `uint256`                    |         The minimum remaining gas required to continue processing.         |

#### due

Returns true if the next task is due based on the current block timestamp.

```solidity
function due(Heap storage _heap) internal view returns (bool isDue);
```

**Parameters**

|   Name  |  Type  |        Description       |
| :-----: | :----: | :----------------------: |
| `_heap` | `Heap` | The heap storage struct. |

**Returns**

|   Name  |  Type  |                   Description                   |
| :-----: | :----: | :---------------------------------------------: |
| `isDue` | `bool` | True if the heap has a task due now or earlier. |

#### peek

Returns the ID and due time of the next task in the heap.

Reverts if heap is empty.

```solidity
function peek(Heap storage _heap) private view returns (uint216 id, uint40 dueAt);
```

**Parameters**

|   Name  |  Type  |        Description       |
| :-----: | :----: | :----------------------: |
| `_heap` | `Heap` | The heap storage struct. |

**Returns**

|   Name  |    Type   |          Description          |
| :-----: | :-------: | :---------------------------: |
|   `id`  | `uint216` |          The task ID.         |
| `dueAt` |  `uint40` | The due timestamp in seconds. |

#### \_encode

Encodes a task's ID and due time into a 256-bit key.

```solidity
function _encode(uint216 _id, uint40 _dueTime) private pure returns (uint256 key);
```

**Parameters**

|    Name    |    Type   |        Description       |
| :--------: | :-------: | :----------------------: |
|    `_id`   | `uint216` |       The task ID.       |
| `_dueTime` |  `uint40` | The due time in seconds. |

**Returns**

|  Name |    Type   |                        Description                        |
| :---: | :-------: | :-------------------------------------------------------: |
| `key` | `uint256` | Encoded key with dueTime in high bits and id in low bits. |

#### \_decode

Decodes a 256-bit key into task ID and due time.

```solidity
function _decode(uint256 _key) private pure returns (uint216 id, uint40 dueAt);
```

**Parameters**

|  Name  |    Type   |    Description   |
| :----: | :-------: | :--------------: |
| `_key` | `uint256` | The encoded key. |

**Returns**

|   Name  |    Type   |        Description       |
| :-----: | :-------: | :----------------------: |
|   `id`  | `uint216` |       The task ID.       |
| `dueAt` |  `uint40` | The due time in seconds. |

#### \_getId

Extracts and returns the task ID from a given heap key.

```solidity
function _getId(uint256 _key) private pure returns (uint216 id);
```

**Parameters**

|  Name  |    Type   |                           Description                          |
| :----: | :-------: | :------------------------------------------------------------: |
| `_key` | `uint256` | The encoded heap key containing the task ID and due timestamp. |

**Returns**

| Name |    Type   |      Description     |
| :--: | :-------: | :------------------: |
| `id` | `uint216` | The decoded task ID. |

#### \_siftDown

Maintains the heap property by moving an item down the tree.

```solidity
function _siftDown(Heap storage _heap, uint256 _i, mapping(uint216 => uint256) storage _index) private;
```

#### \_siftUp

Maintains the heap property by moving an item up the tree.

```solidity
function _siftUp(Heap storage _heap, uint256 _i, mapping(uint216 => uint256) storage _index) private;
```

#### \_swap

Swaps two elements in the heap and updates the index mapping.

```solidity
function _swap(Heap storage _heap, mapping(uint216 => uint256) storage _index, uint256 _i, uint256 _j) private;
```

#### getItems

Returns the task IDs currently in the heap in raw order (not sorted).

```solidity
function getItems(Heap storage _heap) internal view returns (uint216[] memory items);
```

**Parameters**

|   Name  |  Type  |        Description       |
| :-----: | :----: | :----------------------: |
| `_heap` | `Heap` | The heap storage struct. |

**Returns**

|   Name  |     Type    |     Description    |
| :-----: | :---------: | :----------------: |
| `items` | `uint216[]` | Array of task IDs. |

### Errors

#### TaskNotFound

Thrown when a task with a given ID is not found in the queue.

```solidity
error TaskNotFound();
```

#### DuplicateTask

Thrown when trying to insert a task that already exists in the queue.

```solidity
error DuplicateTask();
```

### Structs

#### Heap

Min-heap struct holding the encoded task keys.

```solidity
struct Heap {
    uint256[] data;
}
```

**Properties**

|  Name  |     Type    |                Description                |
| :----: | :---------: | :---------------------------------------: |
| `data` | `uint256[]` | The heap data array of encoded task keys. |
