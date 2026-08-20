# Library

List of  core contracts

|                Contracts                | Description                                                                                                                                                        |
| :-------------------------------------: | :------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| [TaskQueueLib.sol](taskqueuelib.sol.md) | is a lightweight scheduling library that keeps tasks (like invoice releases) ordered by due time using a min-heap, so the next action to run is always at the top. |
| [FeeAuthorizationLib.sol](feeauthorizationlib.sol.md) | verifies that a per-invoice fee receiver was authorized by the configured fee signer, via an EIP-191 signature check. |

