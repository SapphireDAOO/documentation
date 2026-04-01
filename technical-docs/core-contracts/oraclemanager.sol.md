# OracleManager.sol

The OracleManager contract manages Chainlink price feeds and sequencer uptime checks used by the [AdvancedPaymentProcessor.sol](advancedpaymentprocessor.sol.md) to convert USD-denominated invoice prices into the equivalent payment token amounts.

It is deployed as a standalone contract and referenced by the AdvancedPaymentProcessor via the `IOracleManager` interface. Write access is restricted to the owner of the linked [PaymentProcessorStorage.sol](paymentprocessorstorage.sol.md) contract.

You can find the full code implementation [here](https://github.com/SapphireDAOO/payment-processor/blob/v2/src/OracleManager.sol)

### State Variables

#### ppStorage

Shared storage contract whose owner controls oracle configuration writes.

```solidity
PaymentProcessorStorage public immutable ppStorage
```

#### DEFAULT\_DECIMAL

Default number of decimals used for internal fixed-point arithmetic (e.g., 1e18 = 1.0).

```solidity
uint8 public constant DEFAULT_DECIMAL = 18
```

#### SEQUENCER\_GRACE\_PERIOD

Minimum time (in seconds) to wait after the L2 sequencer restarts before trusting price data. Protects against stale prices that accumulated while the sequencer was offline.

```solidity
uint256 public constant SEQUENCER_GRACE_PERIOD = 1 hours
```

### Functions

#### constructor

Deploys the OracleManager and sets the initial sequencer uptime feed.

```solidity
constructor(address _paymentProcessorStorageAddress, address _sequencerUptimeFeed);
```

**Parameters**

|               Name                |    Type   |                                              Description                                             |
| :-------------------------------: | :-------: | :--------------------------------------------------------------------------------------------------: |
| `_paymentProcessorStorageAddress` | `address` |              The shared storage contract whose owner governs oracle updates.                         |
|      `_sequencerUptimeFeed`       | `address` | Address of the Chainlink sequencer uptime feed. Pass `address(0)` to disable the sequencer check.   |

#### getUsdPerToken

Fetches the Chainlink USD price for a payment token and validates feed freshness.

Performs three layers of validation: sequencer uptime check (if feed is set), round completeness, and heartbeat staleness. Reverts if any check fails.

```solidity
function getUsdPerToken(address _paymentToken) external view returns (uint256);
```

**Parameters**

|       Name       |    Type   |                                       Description                                       |
| :--------------: | :-------: | :-------------------------------------------------------------------------------------: |
| `_paymentToken`  | `address` | The token address to price (use `address(0)` for native ETH). |

**Returns**

| Name |    Type   |                              Description                             |
| :--: | :-------: | :------------------------------------------------------------------: |
|      | `uint256` | The token's USD price with 8 decimals as returned by the Chainlink aggregator. |

#### setPriceFeed

Sets the Chainlink price feed configuration for a specific payment token.

Only callable by the owner of `ppStorage`. Setting `_config.aggregator` to `address(0)` removes the token from accepted payment methods.

```solidity
function setPriceFeed(address _token, PriceFeedConfig memory _config) external;
```

**Parameters**

|   Name    |        Type        |                                       Description                                      |
| :-------: | :----------------: | :------------------------------------------------------------------------------------: |
| `_token`  |     `address`      | The payment token address, or `address(0)` for native currency.                        |
| `_config` | `PriceFeedConfig`  | The price feed configuration containing the aggregator address and heartbeat interval. |

#### setSequencerUptimeFeed

Sets the Chainlink L2 sequencer uptime feed address.

Only callable by the owner of `ppStorage`. Set to `address(0)` to disable the sequencer check (e.g. on L1 deployments or local testnets).

```solidity
function setSequencerUptimeFeed(address _sequencerUptimeFeed) external;
```

**Parameters**

|          Name           |    Type   |                                  Description                                  |
| :---------------------: | :-------: | :---------------------------------------------------------------------------: |
| `_sequencerUptimeFeed`  | `address` | The sequencer uptime feed address, or `address(0)` to disable the check. |

#### getSequencerUptimeFeed

Returns the configured sequencer uptime feed address.

```solidity
function getSequencerUptimeFeed() external view returns (address feed);
```

**Returns**

|  Name  |    Type   |                                Description                               |
| :----: | :-------: | :----------------------------------------------------------------------: |
| `feed` | `address` | The sequencer uptime feed address, or `address(0)` if the check is disabled. |

### Structs

#### PriceFeedConfig

Configuration for a Chainlink price feed associated with a payment token.

```solidity
struct PriceFeedConfig {
    address aggregator;
    uint96 heartbeat;
}
```

| Field        |    Type   |                                                          Description                                                         |
| :-----------: | :-------: | :---------------------------------------------------------------------------------------------------------------------------: |
| `aggregator` | `address` | Address of the Chainlink AggregatorV3 contract. Set to `address(0)` to disable the token.                                   |
| `heartbeat`  |  `uint96` | Maximum acceptable age (in seconds) of a price update before it is considered stale. Should match the feed's update interval. |

### Errors

| Error | Description |
| :----: | :----------: |
| `NotAuthorized()` | Thrown when the caller is not the owner of the storage contract. |
| `UnsupportedToken()` | Thrown when a payment token has no configured price feed. |
| `StalePrice()` | Thrown when the Chainlink round is incomplete (`answeredInRound < roundId`). |
| `SequencerDown()` | Thrown when the L2 sequencer is down or still within the post-restart grace period. |
| `StalePriceFeed()` | Thrown when the price feed update is older than the configured heartbeat. |
| `InvalidPrice()` | Thrown when the Chainlink price feed returns a zero or negative answer. |
