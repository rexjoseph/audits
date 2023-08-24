<table>
  <tr>
    <td><img src="https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Fblueberry.jpg&w=96&q=75" height="250" width="250" /></td>
    <td>
      <h1>Blueberry v3 Audit Reports</h1>
      <h2>Sherlock</h2>
      <p>Prepared by: Kenzo Tenma, Rex Joseph (Fides Team)</p>
      <p>Date: August 10 to 15, 2023</p>
    </td>
  </tr>
</table>

# About **Blueberry**

Blueberry unifies the DeFi experience: Aggregating, Automating, and Boosting Capital Efficiency for top DeFi Strategies.

# About **Fides**

Fides is a two person smart contract security research team made up of Rex and Kenzo.


# Summary & Scope

The [sherlock-audit/2023-07-blueberry-fides](https://github.com/sherlock-audit/2023-07-blueberry-fides/) repository was audited.

The following contracts were in scope:
- blueberry-core/contracts/BlueBerryBank.sol
- blueberry-core/contracts/FeeManager.sol
- blueberry-core/contracts/ProtocolConfig.sol
- blueberry-core/contracts/libraries/BBMath.sol
- blueberry-core/contracts/libraries/FixedPointMathLib.sol
- blueberry-core/contracts/libraries/Paraswap/PSwapLib.sol
- blueberry-core/contracts/libraries/Paraswap/Utils.sol
- blueberry-core/contracts/libraries/UniV3/UniV3WrappedLib.sol
- blueberry-core/contracts/libraries/balancer/FixedPoint.sol
- blueberry-core/contracts/spell/AuraSpell.sol
- blueberry-core/contracts/spell/BasicSpell.sol
- blueberry-core/contracts/spell/ConvexSpell.sol
- blueberry-core/contracts/spell/IchiSpell.sol
- blueberry-core/contracts/spell/ShortLongSpell.sol
- blueberry-core/contracts/utils/BlueBerryConst.sol
- blueberry-core/contracts/utils/BlueBerryErrors.sol
- blueberry-core/contracts/utils/ERC1155NaiveReceiver.sol
- blueberry-core/contracts/utils/EnsureApprove.sol
- blueberry-core/contracts/vault/HardVault.sol
- blueberry-core/contracts/vault/SoftVault.sol
- blueberry-core/contracts/wrapper/WAuraPools.sol
- blueberry-core/contracts/wrapper/WConvexPools.sol
- blueberry-core/contracts/wrapper/WERC20.sol
- blueberry-core/contracts/wrapper/WIchiFarm.sol
- blueberry-core/contracts/oracle/AggregatorOracle.sol
- blueberry-core/contracts/oracle/BandAdapterOracle.sol
- blueberry-core/contracts/oracle/BaseAdapter.sol
- blueberry-core/contracts/oracle/BaseOracleExt.sol
- blueberry-core/contracts/oracle/ChainlinkAdapterOracle.sol
- blueberry-core/contracts/oracle/ChainlinkAdapterOracleL2.sol
- blueberry-core/contracts/oracle/CompStableBPTOracle.sol
- blueberry-core/contracts/oracle/CoreOracle.sol
- blueberry-core/contracts/oracle/CurveBaseOracle.sol
- blueberry-core/contracts/oracle/CurveStableOracle.sol
- blueberry-core/contracts/oracle/CurveTricryptoOracle.sol
- blueberry-core/contracts/oracle/CurveVolatileOracle.sol
- blueberry-core/contracts/oracle/IchiVaultOracle.sol
- blueberry-core/contracts/oracle/StableBPTOracle.sol
- blueberry-core/contracts/oracle/UniswapV2Oracle.sol
- blueberry-core/contracts/oracle/UniswapV3AdapterOracle.sol
- blueberry-core/contracts/oracle/UsingBaseOracle.sol
- blueberry-core/contracts/oracle/WeightedBPTOracle.sol
- blueberry-core/contracts/libraries/UniV3/UniV3WrappedLibContainer.sol

# Summary of Findings

| ID     | Title                        | Severity      | Fixed |
| ------ | ---------------------------- | ------------- | ----- |
| [M-01] | The `poolToken0` and `poolToken1` are incorrectly returned | Medium |   |
| [M-02] | Curve MetaPool Registry is incorrectly added in `addressProvider.get_address()` | Medium |   |
| [M-03] | Zero Address implementation is incorrect | Medium |   |
| [M-04] | The success flag of `augustusSwapper.call(data)` isn't checked | Medium |   |

# Detailed Findings

## [M-01] The poolToken0 and poolToken1 are incorrectly returned

The ternary operation is incorrect in `getPrice` function. The `poolToken` should be returned on `poolToken0 == token`

## Vulnerability Detail

In the contract `UniswapV3AdapterOracle`, the function `getPrice` is meant return the prices of two tokens `poolToken0` and `poolToken1` from UniswapV3 oracle. But in implementation to get addresses(Line 76), the Ternary Operator returns `poolToken1` when the condition is `poolToken0 == token`

## Impact

The poolTokens will return incorrect price to the protocol.

## Code Snippet

```solidity
    function getPrice(address token) external override returns (uint256) {
        /// Maximum cap of timeGap is 2 days(172,800), safe to convert
        uint32 secondsAgo = timeGaps[token].toUint32();
        if (secondsAgo == 0) revert Errors.NO_MEAN(token);

        address stablePool = stablePools[token];
        if (stablePool == address(0)) revert Errors.NO_STABLEPOOL(token);

        address poolToken0 = IUniswapV3Pool(stablePool).token0();
        address poolToken1 = IUniswapV3Pool(stablePool).token1();
        address stablecoin = poolToken0 == token ? poolToken1 : poolToken0; // get stable token address
        // @audit wrong stablecoin address assignment

        uint8 stableDecimals = IERC20Metadata(stablecoin).decimals();
        uint8 tokenDecimals = IERC20Metadata(token).decimals();

        (int24 arithmeticMeanTick, ) = UniV3WrappedLibContainer.consult(
            stablePool,
            secondsAgo
        );
        uint256 quoteTokenAmountForStable = UniV3WrappedLibContainer
            .getQuoteAtTick(
                arithmeticMeanTick,
                uint256(10 ** tokenDecimals).toUint128(),
                token,
                stablecoin
            );

        return
            (quoteTokenAmountForStable * base.getPrice(stablecoin)) /
            10 ** stableDecimals;
    }
```

## Tool Used

Manual review

## Recommendation

The mitigation steps are as follows:

```solidity
76        - address stablecoin = poolToken0 == token ? poolToken1 : poolToken0;
78        + address stablecoin = poolToken0 == token ? poolToken0 : poolToken1;
```

## [M-02] Curve MetaPool Registry is incorrectly added in `addressProvider.get_address()`

The function `_getPoolInfo` fetches the curve pool information for a given Curve LP token address. The meta pool address is wrong.

## Vulnerability Detail

In abstract contract `CurveBaseOracle`, the function `_getPoolInfo` fetches curve pool information for three different Curve LP. main curve registry and CryptoSwap curve registry addresses are implemented correctly but in case of meta curve registry the address is incorrect. (Line 108)

## Impact

The attempt of retrieval from Meta Curve Registry will fail.

## Code Snippet

```solidity
  registry = addressProvider.get_address(7); // @audit no address 7 in curve documentation
```

## Tool Used

Manual review

## Recommendation

Add `addressProvider.get_address(3)` in place of `addressProvider.get_address(7)`

```solidity
108  - registry = addressProvider.get_address(7);
108  + registry = addressProvider.get_address(3);
```

## [M-03] Zero Address implementation is incorrect

In function `getPrice`, `if(token == address(0))` then `token = token_` bypasses the zero address validation in the function.

## Vulnerability Detail

In abstract contract `ChainlinkAdapterOracle.sol`, the zero address validation check `if (token == address(0)) token = token_;` (line 105) is incorrect because if the token address is zero address, it will consider token = token_ which just bypasses the check

## Impact

`_token` address might be zero and caller won't recieve any price value from the chainlink oracle.

## Code Snippet

```solidity
  address token = remappedTokens[token_];
  if (token == address(0)) token = token_; // @audit
```

## Tool Used

Manual review

## Recommendation

The mitigation steps are as follows:

```solidity
105  - if (token == address(0)) token = token_;
106  + if (token == address(0)) revert Errors.ZERO_ADDRESS();
```

## [M-04] The success flag of `augustusSwapper.call(data)` isn't checked

The success flag of `augustusSwapper.call(data)` isn't checked. Which might fail silently returning `returndata` containing an empty bytes array (bytes memory with length 0).

## Vulnerability Detail

In library `PSwapLib`, the function `swap` uses a low level call `.call(success, returndata) = augustusSwapper.call(data);` (line 31) which might just fail without reverting silently which makes the `returndata` contain nothing.

## Impact

The low level call will just fail silently without returning any value.

## Code Snippet

```solidity
  bytes memory returndata;
  // @audit handle the failure case accordingly
  (success, returndata) = augustusSwapper.call(data);
```

## Tool Used

Manual review

## Recommendation

The mitigation steps are as follows:
Add this line to check the low level call success flag

```solidity
if (!success) {
  // Handle the case where the external function call fails
  revert("External call failed");
}
```