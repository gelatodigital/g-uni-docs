---
description: >-
  G-UNI is a framework for Uniswap V3 Positions wrapped into fractionalized,
  fungible, auto-fee-compounding ERC20 tokens. Customizable LP strategies for a
  pool can be implemented via the 'manager' role.
---

# Documentation

## Contribute to the docs

Feel free to help improving the docs by doing a PR to this repo:

{% embed url="https://github.com/gelatodigital/g-uni-docs" %}

## Introduction

G-UNI is a generic ERC20 wrapper on a Uniswap V3 Position. Pools with any price bounds on any Uniswap V3 pair can be deployed via the `GUniFactory` instantiating a tokenized V3 Position. When liquidity is added into the pool, G-UNI tokens are minted and credited to the provider. Inversely, G-UNI tokens can be burned to redeem that proportion of the pool's V3 position liquidity and fees earned. Thus, G-UNI tokens represent proportional ownership (or "shares") of the underlying Uniswap V3 position. Similar to the Uniswap V2 LP experience, anyone can add liquidity to or remove liquidity from a G-UNI Pool, and can earn their portion of the fees generated just by holding the fungible tokens.\
\
There are two privileged roles assigned to a G-UNI Pool by default: `Gelato Network Executors` and `Manager` (see the `Authorized Manager Functions` section).

## GUniFactory

The `GUniFactory`[ smart contract](https://etherscan.io/address/0xea1aff9dbffd1580f6b81a3ad3589e66652db7d9) governs over the creation of G-UNI Pools. In theory, any account or smart contract can create a G-UNI Pool via the factory by calling `createPool` (in practice pool creation should be done with care by experienced parties)

### createPool

Deploy a G-UNI Pool on the Uniswap V3 pair and with the Position parameters of your choosing.

{% code title="GUniFactory.sol" %}
```bash
    function createPool(
        address tokenA,
        address tokenB,
        uint24 uniFee,
        uint16 managerFee,
        int24 lowerTick,
        int24 upperTick
    ) external returns (address pool)
```
{% endcode %}

Arguments:

* `tokenA` One of the tokens in the Uniswap V3 pair
* `tokenB` The other token in the Uniswap V3 pair
* `uniFee` Fee tier of the Uniswap V3 pair (500, 3000, 10000)
* `managerFee` The proportion of fees earned that `manager` takes as a cut in Basis Points (deployer address is initial manager). **Note: The managerFee can only be set to a non-zero value once (immutable afterwards)**
* `lowerTick` Initial lower price bound for the position, represented as a Uniswap V3 tick.
* `upperTick` Initial upper price bound for the position, represented as a Uniswap V3 tick.

{% hint style="info" %}
The `lowerTick` and `upperTick`:\
1\. MUST be set to integers between -887272 and 887272 where `upperTick > lowerTick`\
2\. MUST be integers divisible by the tickSpacing of the Uniswap pair.
{% endhint %}

Returns: Address of newly deployed G-UNI Pool ERC20 contract (proxied).\
\
To have full verification and functionality on etherscan (read/write methods) [verify the proxy contract](https://etherscan.io/proxyContractChecker). Etherscan will recognize the contract address as an ERC20 token and generate the token page after minting of the first G-UNI tokens.

## GUniRouter02

The GUniRouter02 [smart contract](https://etherscan.io/address/0x14e6d67f824c3a7b4329d3228807f8654294e4bd#code) exposes methods for adding liquidity to and removing liquidity from any G-UNI Pool. It supports a number of different methods for these operations. \_Router operations require the router address to be approved to spend the tokens forwarded by \_ `msg.sender`

### addLiquidity

Add liquidity to a G-UNI Pool and mint G-UNI tokens.

{% code title="GUniRouter.sol" %}
```bash
    function addLiquidity(
        address pool,
        uint256 amount0Max,
        uint256 amount1Max,
        uint256 amount0Min,
        uint256 amount1Min,
        address receiver
    )
        external
        returns (
            uint256 amount0,
            uint256 amount1,
            uint256 mintAmount
        )
```
{% endcode %}

Arguments:

* `pool` address of G-UNI pool to add liquidity to
* `amount0Max` maximum amount of token0 to deposit into pool
* `amount1Max` maximum amount of token1 to deposit into pool
* `amount0Min` minimum amount of token0 deposited
* `amount1Min` minimum amount of token1 deposited
* `receiver` account that receives minted G-UNI tokens

Returns:

* `amount0` actual amount of token0 deposited into G-UNI
* `amount1` actual amount of token1 deposited into G-UNI
* `mintAmount` amount of G-UNI tokens minted

{% hint style="info" %}
Min and Max amounts are specified because exact proportions of token0 and token1 are determined by the Uniswap V3 price which can move while the transaction is waiting to be mined. Minimum amounts can be used to revert the transaction if the price has moved too far.
{% endhint %}

{% hint style="info" %}
Approve router address to spend amount0Max and amount1Max before calling addLiquidity
{% endhint %}

### rebalanceAndAddLiquidity

Add Maximal Liquidity to a G-UNI Pool with any ratio of token0 and token1 relative to the G-UNI Pool's total inventory. A swap is performed before adding liquidity to rebalance the tokens forwarded so it more closely matches the Pool's inventory.

{% code title="GUniRouter.sol" %}
```bash
    function rebalanceAndAddLiquidity(
        address pool,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amountSwap,
        bool zeroForOne,
        address[] swapActions,
        bytes[] swapDatas,
        uint256 amount0Min,
        uint256 amount1Min,
        address receiver
    )
        external
        returns (
            uint256 amount0,
            uint256 amount1,
            uint256 mintAmount
        )
```
{% endcode %}

Arguments:

* `pool` address of G-UNI pool to add liquidity to
* `amount0In` amount of token0 sender forwards to router
* `amount1In` amount of token1 sender forwards to router
* `amountSwap` amount to input into 1inch swap
* `zeroForOne` direction of swap
* `swapActions` array of addresses for 1inch swap calls
* `swapDatas` array of payloads for 1inch swap calls
* `amount0Min` minimum amount of token0 supplied
* `amount1Min` minimum amount of token1 supplied
* `receiver` account that receives minted G-UNI tokens

Returns:

* `amount0` actual amount of token0 deposited into G-UNI
* `amount1` actual amount of token1 deposited into G-UNI
* `mintAmount` amount of G-UNI tokens minted

{% hint style="info" %}
In this operation the router transfers`amount0In` and`amount1In` from user and performs a swap before depositing into the G-UNI Pool. Based on the swap parameters and the current prices relative to the position, proportion of token0 and token1 may not be perfect after the swap. Any leftover tokens after maximal deposit will be returned to the `msg.sender`
{% endhint %}

{% hint style="info" %}
`swapActions` and `swapDatas` are addresses and corresponding payloads that can be generated off chain using 1inch's api. Usually this will involve encoding an "approve" call which approves 1inch to spend tokens that the router is holding and attempting to swap, and then a "swap" call which actually performs the swap. In theory, any set of calls that successfully spends the input token and receives the output token would work here.
{% endhint %}

\*\* See GUniResolver02 `getRebalanceParams` method to generate a rough estimate of the right swap parameters (zeroForOne, swapAmount, swapThreshold)

### removeLiquidity

Burn G-UNI tokens and remove underlying liquidity and earned fees.

```bash
    function removeLiquidity(
        address pool,
        uint256 burnAmount,
        uint256 amount0Min,
        uint256 amount1Min,
        address receiver
    )
        external
        returns (
            uint256 amount0,
            uint256 amount1,
            uint128 liquidityBurned
        )
```

Arguments:

* `pool` address of G-UNI Pool to remove liquidity from
* `burnAmount` amount of G-UNI tokens to burn.
* `amount0Min` minimum amount of token0 received after burn
* `amount1Min` minimum amount of token1 received after burn
* `receiver` account to receive the token0 and token1 remitted

Returns:

* `amount0` amount of token0 remitted to receiver
* `amount1` amount of token1 remitted to receiver
* `liquidityBurned` amount of liquidity removed from the underlying Uniswap V3 position

{% hint style="info" %}
Remember that `msg.sender` must approve router to spend burnAmount of G-UNI tokens for transaction not to revert
{% endhint %}

### addLiquidityETH

{% code title="GUniRouter.sol" %}
```bash
    function addLiquidityETH(
        address pool,
        uint256 amount0Max,
        uint256 amount1Max,
        uint256 amount0Min,
        uint256 amount1Min,
        address receiver
    )
        external
        payable
        returns (
            uint256 amount0,
            uint256 amount1,
            uint256 mintAmount
        )
```
{% endcode %}

Identical to `addLiquidity` method but caller spends ETH rather than WETH. Applicable for any G-UNI Pool on a Uniswap Pair with WETH as one of the tokens.

### rebalanceAndAddLiquidityETH

{% code title="GUniRouter.sol" %}
```bash
    function rebalanceAndAddLiquidityETH(
        address pool,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amountSwap,
        bool zeroForOne,
        address[] swapActions,
        bytes[] swapDatas,
        uint256 amount0Min,
        uint256 amount1Min,
        address receiver
    )
        external
        payable
        returns (
            uint256 amount0,
            uint256 amount1,
            uint256 mintAmount
        )
```
{% endcode %}

Identical to `rebalanceAndAddLiquidity` but caller spends ETH rather than WETH. Applicable for any G-UNI Pool on a Uniswap Pair with WETH as one of the tokens.

### removeLiquidityETH

```bash
    function removeLiquidityETH(
        address pool,
        uint256 burnAmount,
        uint256 amount0Min,
        uint256 amount1Min,
        address payable receiver
    )
        external
        returns (
            uint256 amount0,
            uint256 amount1,
            uint128 liquidityBurned
        )
```

Identical to `removeLiquidity` but receiver is remitted ETH rather than WETH from the liquidity withdrawn. Applicable for any G-UNI Pool on a Uniswap Pair with WETH as one of the tokens.

## GUniResolver02

The GUniResolver02 [smart contract](https://etherscan.io/address/0x0317650Af6f184344D7368AC8bB0bEbA5EDB214a#code) is a library of simple helper methods for G-UNI positions. When exposing a UI for users to addLiquidity to G-UNI one may want to allow users to enter the pool with any amount of either asset using the `rebalanceAndAddLiquidity` Router method. However the swap parameters passed as argument to this function must be carefully chosen to deposit maximal liquidity and produce the least leftover (any leftover is returned to `msg.sender` ). This resolver contract exposes a helper for just that.

### getRebalanceParams

Generate the (roughly approximated) swap parameters for a `rebalanceAndAddLiquidity` call

```bash
    function getRebalanceParams(
        address pool,
        uint256 amount0In,
        uint256 amount1In,
        uint256 price18Decimals
    )
        external
        view
        returns (
            bool zeroForOne,
            uint256 swapAmount
        )
```

Arguments:

* `pool` Address of G-UNI pool of interest
* `amount0In` amount of token0 sender forwards to router
* `amount1In` amount of token1 sender forwards to router
* `price18Decimals` price ratio of `token1/token0` disregarding (normalizing by) token decimals and then expressed as a WAD (multiplied by `10^18`).

Returns:

* `zeroForOne` direction of swap
* `swapAmount` amount of token to swap

{% hint style="info" %}
Because the price ratio often changes depending on the amount being swapped (slippage) you may want to call this method twice - first with a "reasonable guess" `price18Decimals` to get a preliminary `swapAmount`, then get a more accurate `price18Decimals` (using this `swapAmount` as input), and finally call this method again with the more accurate price to get a final and more precise `swapAmount`
{% endhint %}

## Authorized Manager Functions

There are two distinct accounts with privileged roles for any G-UNI Pool: one is `Gelato Executors` these are Gelato Network Bots handling automated managerial tasks on the pool (fee reinvestments, manager fee collection). The other is a `manager` account who can configure the Gelato Executor meta-parameters and also can control and alter the range of the underlying Uniswap V3 position.

### Gelato Executor (automated)

The Gelato Network is a decentralized bot infrastructure for trustless automated execution of smart contracts based on arbitrary conditions. Essentially, Gelato bots (executors) allow anyone to schedule automated future transactions, without needing to run any bot infrastructure themselves or trust a single central party to do so. Gelato Executors are responsible for executing scheduled tasks when specified conditions are met. One such Gelato Task is to reinvest fees of any G-UNI Pool once the cost in gas for the transaction is less than X% of the transaction fee (as specified by `gelatoRebalanceBPS`). Gelato Network also earns a hardcoded 1% commission on all fees earned for this automated fee-reinvestment service.

#### rebalance

The rebalance function is only callable by `Gelato Executors` and invests any fees earned into the G-UNI Pool's underlying position.

{% code title="GUniPool.sol" %}
```bash
    function rebalance(
        uint160 swapThresholdPrice,
        uint256 swapAmountBPS,
        bool zeroForOne,
        uint256 feeAmount,
        address paymentToken
    ) external gelatofy(feeAmount, paymentToken)
```
{% endcode %}

The rebalance function includes swap parameters because the fees earned may not be in the correct proportion of token0 and token1 to deposit maximum liquidity into the position. While executor bots set these parameters from off chain they cannot extract value from the swap because the slippage parameter is checked on chain against the TWAP to ensure it is not a malicious slippage value intended to extract value via front/back running. Similarly the fee parameter to repay gas costs is checked against on chain gas usage and acceptable gas prices (via an oracle) to make sure fees charged are legitimate and not somehow malicious.

#### withdrawManagerBalance

```bash
    function withdrawManagerBalance(
        uint256 feeAmount,
        address feeToken
    )
        external
        gelatofy(feeAmount, feeToken)
```

This function auto withdraws the manager balance to the manager address when enough tokens have been earned (as specified by `gelatoWithdrawBPS` )

### Manager

The manager is the most important and centrally trusted role in the GUniPool. It is the only role that has the power to potentially extract value from the principal invested or potentially grief the pool in a number of ways. One should only put funds into pools where the manager is either known and trusted, a decentralized or democratically controlled smart contract, or renounced altogether (renouncing the manager makes the Pool fully trustless but also the underlying position range immutable).

#### executiveRebalance

By far the most important manager function is the `executiveRebalance` method on the `GUniPool`. This permissioned method is the only way to change the price range of the underlying Uniswap V3 Position. Manager accounts who control this function are the means by which custom rebalancing strategies can be built on top of G-UNI. These strategies can be implemented by governance (slow, but decentralized) or by some central managerial party (more responsive but requiring much more trust) and in the future the manager role can be granted to a Gelato automated smart contract so that they have a sophisticated LP strategy fully automated by Gelato (both reinvesting fees and in rebalancing ranges).

{% code title="GUniPool.sol" %}
```bash
    function executiveRebalance(
        int24 newLowerTick,
        int24 newUpperTick,
        uint160 swapThresholdPrice,
        uint256 swapAmountBPS,
        bool zeroForOne
    ) external onlyManager {
```
{% endcode %}

Arguments:

* `newLowerTick` The new lower price bound of the position represented as a Uniswap V3 tick.
* `newUpperTick`The new upper price bound of the position represented as a Uniswap V3 tick.
* `swapThresholdPrice` A sqrtPriceX96 that acts as a slippage parameter for the swap that rebalances the inventory (to deposit maximal liquidity around the new price bounds).
* `swapAmountBPS` Amount of inventory to swap represented as Basis Points of the remaining amount of the token to swap.
* `zeroForOne` The direction of the rebalancing swap.

In order to generate the parameters for an executive rebalance one has to understand the flow of this operation. First, the GUNiPool removes all the liquidity and fees earned. Then, it tries to deposit as much liquidity as possible around the new price range. Next, whatever is leftover is then swapped based on the swap parameters. Finally, another deposit of maximal liquidity to the position is attempted and any leftover sits in the contract balance waiting to be reinvested.

{% hint style="info" %}
To generate the swap parameters for an executive rebalance tx, simulate the entire operation and use the `getRebalanceParams` of the GUniResolver. It works like this:\
1\. Call `getUnderlyingBalances` on the GUniPool.\
2\. Compute the amount of leftover after depositing the maximum of these underlying balances into the new range (use Uniswap LiquidityAmounts.sol library to figure out maximal deposit)\
3\. Pass the leftover amounts as amount0In and amount1In to the GUniResolver's `getRebalanceParams` to generate the zeroForOne, swapThreshold and swapAmount vars.\
4\. Compute the swapAmountBPS from swapAmount with the formula: `swapAmount * 10000 / leftoverAmount` in the swap token.
{% endhint %}

#### updateGelatoParams

Another important role of the `manager` is to configure the parameters that restrict the functionality of `Gelato Executors`. These parameters include how often Gelato bots can reinvest fees and withdraw manager fees, as well as other safety params like the slippage check. Only the manager can call `updateGelatoParams` .

{% code title="GUniPoolStorage.sol" %}
```bash
    function updateGelatoParams(
        uint16 newRebalanceBPS,
        uint16 newWithdrawBPS,
        uint16 newSlippageBPS,
        uint32 newSlippageInterval,
        address newTreasury
    ) external onlyManager {
```
{% endcode %}

Arguments:

* `newRebalanceBPS` The maximum percentage the auto fee reinvestment transaction cost can be compared to the fees earned in that feeToken. The percentage is given in Basis Points (where 10000 mean 100%). Example: if rebalanceBPS is 200 and the transaction fee is 10 USDC then the fees earned in USDC must be 500 UDSC or the transaction will revert.
* `newWithdrawBPS` The maximum percentage the auto withdraw transaction fee can be compared to the amount withdrawn of that feeToken.
* `newSlippageBPS` The maximum percentage that the rebalance slippage can be from the TWAP.
* `newSlippageInterval` length of time in seconds for to compute the TWAP (time weighted average price). I.e. 300 means a 5 minute average price for the pool.
* `newTreasury` The treasury address where manager fees are auto withdrawn.

#### transferOwnership

Standard transfer of the manager role to a new account.

{% code title="GUniPoolStorage.sol" %}
```bash
    function transferOwnership(address newOwner)
        public onlyManager
```
{% endcode %}

Arguments: `newOwner` the new account to act as manager.

#### initializeManagerFee

{% code title="GUniPoolStorage.sol" %}
```bash
function initializeManagerFee(uint16 _managerFeeBPS)
```
{% endcode %}

Arguments: `_managerFeeBPS` the percent commission of fees earned to go to the manager account.

{% hint style="warning" %}
The manager fee can only be set to a non-zero value one time and then can never be changed (unless the manager role is renounced).
{% endhint %}

#### renounceOwnership

Burn the manager role and set all manager fees to 0.

{% code title="GUniPoolStorage.sol" %}
```bash
function renounceOwnership() public onlyManager
```
{% endcode %}

## G-UNI Subgraph

The G-UNI subgraph tracks relevant data for all G-UNI Positions created through the `GUniFactory` `createPool` method. Most relevant information about about a G-UNI position is indexed and queryable via the subgraph. Query URL is: [https://api.thegraph.com/subgraphs/name/gelatodigital/g-uni](https://api.thegraph.com/subgraphs/name/gelatodigital/g-uni)\
\
Here is an example query which fetches all information about all G-UNI Positions:

```bash
      query {
        pools {
          id
          blockCreated
          manager
          address
          uniswapPool
          token0 {
            address
            name
            symbol
          }
          token1 { 
            address
            name
            symbol
          }
          feeTier
          liquidity
          lowerTick
          upperTick
          totalSupply
          positionId
          lastTouchWithoutFees
          supplySnapshots {
            id
            block
            reserves0
            reserves1
          }
          feeSnapshots {
            id
            block
            feesEarned0
            feesEarned1
          }
        }
      }
```

`id`: subgraph identifier for the pool (contract address)\
`blockCreated`: block G-UNI position was deployed\
`address`: contract address of G-UNI positions\
`uniswapPool`: contract address of Uniswap V3 pair\
`token0`: address of token0\
`token1`: address of token1\
`feeTier`: Uniswap V3 Pair fee tier (500, 3000, 1000)\
`liquidity`: amount of liquidity currently in G-UNI position\
`lowerTick`: current lower tick of G-UNI Position\
`upperTick`: current upper tick of G-UNI Position\
`totalSupply`: current total supply of G-UNI token\
`positionId`: Uniswap V3 ID of the G-UNI position\
`lastTouchWithoutFees`: start block of fee accounting\
`supplySnapshots`: snapshots of the supply when it changes\
`feeSnapshots`: snapshots of fees earned\
\
The `supplySnapshots`, `feeSnapshots`, and `lastTouchWithoutFees` can be ingested together to produce an estimated APR for fees generated by the position.\
\
APR is calculated from these values in the following way:\
\
1\. We use the `supplySnapshots` to calculate the time weighted average value of reserves `Vr` since the block when we started tracking fees (`lastTouchWithoutFees`). This value includes all fees earned.

2\. We use the `feeSnapshots`to add up all the fees earned since `lastTouchWithoutFees` to generate the total value of fees earned `Vf` .\
\
3\. We compute APR as `Vf / (Vr - Vf)` (when `Vr > Vf` ). In rare cases when `Vf > Vr` i.e. feesEarned are larger than the time weighted average reserves, one can simply use`Vf/Vr`

An npm library for these APR calculations is forthcoming but an example of the calculation is here: [https://github.com/kassandraoftroy/apr-script/blob/main/index.ts](https://github.com/superarius/apr-script/blob/main/index.ts)\
\\

## GUniPool

{% hint style="danger" %}
Methods on the `GUniPool`contract are low level and are **NOT RECOMMENDED** for direct interaction by end users unless they know why and what they are doing. For standard mint/burn interaction with G-UNI see `GUniRouter`
{% endhint %}

The `GUniPool` [smart contract](https://etherscan.io/address/0x6dfc8b880d6c1043bebb6eb2346913185ce1b48b) is the core implementation that powers the G-UNI protocol (all G-UNI token proxy contracts point to this implementation). It is an extension of the ERC20 interface so all normal ERC20 token functions apply, as well as a number of custom methods particular to G-UNI which will be outlined below. Most of the methods on the GUniPool are low level and end users should likely be interacting with peripheral contracts rather than directly with these low level methods. Nevertheless these methods are publicly exposed and functional.

### mint

```bash
    function mint(uint256 mintAmount, address receiver)
        external
        nonReentrant
        returns (
            uint256 amount0,
            uint256 amount1,
            uint128 liquidityMinted
        )
```

### burn

```bash
    function burn(uint256 burnAmount, address receiver)
        external
        nonReentrant
        returns (
            uint256 amount0,
            uint256 amount1,
            uint128 liquidityBurned
        )
```

### getMintAmounts

View method to compute the amount of G-UNI tokens minted (and exact amounts of token0 and token1 forwarded) from and `amount0Max` and `amount1Max`

```bash
    function getMintAmounts(uint256 amount0Max, uint256 amount1Max)
        external
        view
        returns (
            uint256 amount0,
            uint256 amount1,
            uint256 mintAmount
        )
```

### getUnderlyingBalances

get the current underlying balances of the entire G-UNI Pool

```bash
    function getUnderlyingBalances()
        public
        view
        returns (
            uint256 amount0Current,
            uint256 amount1Current
        )
    
```

### getUnderlyingBalancesAtPrice

Get the current underlying balances given a custom price (not simply taking the current Uniswap price which can be manipulated).

```bash
    function getUnderlyingBalancesAtPrice(
        uint160 sqrtRatioX96
    )
        external
        view
        returns (
            uint256 amount0Current,
            uint256 amount1Current
        )
```

This function is useful for getting a fair unmanipulatable price of a G-UNI token for things like lending protocols (simply pass a time weighted average sqrtPrice to this function to get the unmanipulated underlying balances).

### getPositionId

```bash
function getPositionID() 
    external view returns (bytes32 positionID) 
```

get the Identifier of the G-UNI position on the Uniswap V3 pair (useful for fetching data about the Position with the `positions` method on `UniswapV3Pool.sol`).

## Contact

Questions about integrating G-UNI? Join the Gelato [Telegram](https://t.me/therealgelatonetwork) or [Discord](https://discord.gg/ApbA39BKyJ) and find a dev or community manager to talk to! Email team@gelato.digital.
