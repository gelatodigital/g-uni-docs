---
description: >-
  Arrakis is a decentralized market making platform that specializes in
  concentrated & active liquidity management to create deep liquidity for
  projects on decentralized exchanges.
---

# Arrakis Finance

## Introduction

Arrakis v1 is a generic ERC20 wrapper on a Uniswap V3 Position. Pools with any price bounds on any Uniswap V3 pair can be deployed via the `ArrakisFactory` instantiating a tokenized V3 Position. When liquidity is added into the pool, Arrakis vault tokens are minted and credited to the provider. Inversely, Arrakis tokens can be burned to redeem that proportion of the pool's V3 position liquidity and fees earned. Thus, Arrakis tokens represent proportional ownership (or "shares") of the underlying Uniswap V3 position. Similar to the Uniswap V2 LP experience, anyone can add liquidity to or remove liquidity from a Arrakis Pool, and can earn their portion of the fees generated just by holding the fungible tokens.\
\
Some Arrakis vaults may have a special privileged `manager` role (see the `createManagedVault` and  `Manager Functions` sections).

## GUniFactory

The `ArrakisFactory`[ smart contract](https://etherscan.io/address/0xea1aff9dbffd1580f6b81a3ad3589e66652db7d9) governs over the creation of Arrakis Vaults. In theory, any account or smart contract can create a Arrakis Pool via the factory by calling `createVault`. This creates a tokenized UniswapV3 Position with a given immutable price range on the token pair and fee tier of your choice. Anyone can now participate as an LP in that range, by adding liquidity into this position and minting Arrakis tokens.

### createPool

Deploy a Arrakis Vault on the Uniswap V3 pair and with the Position parameters of your choosing.

{% code title="GUniFactory.sol" %}
```bash
    function createVault(
        address tokenA,
        address tokenB,
        uint24 uniFee,
        int24 lowerTick,
        int24 upperTick
    ) external returns (address pool)
```
{% endcode %}

Arguments:

* `tokenA` One of the tokens in the Uniswap V3 pair
* `tokenB` The other token in the Uniswap V3 pair
* `uniFee` Fee tier of the Uniswap V3 pair (500, 3000, 10000)
* `lowerTick` Initial lower price bound for the position, represented as a Uniswap V3 tick.
* `upperTick` Initial upper price bound for the position, represented as a Uniswap V3 tick.

{% hint style="info" %}
The `lowerTick` and `upperTick`:\
1\. MUST be set to integers between -887272 and 887272 where `upperTick > lowerTick`\
2\. MUST be integers divisible by the tickSpacing of the Uniswap pair.
{% endhint %}

Returns: Address of newly deployed Arrakis Pool ERC20 contract (proxied).\
\
To have full verification and functionality on etherscan (read/write methods) [verify the proxy contract](https://etherscan.io/proxyContractChecker). Etherscan will recognize the contract address as an ERC20 token and generate the token page after minting of the first Arrakis tokens.

### createManagedVault&#x20;

The simplest type of Arrakis vault to create is via the `createVault` method above, which creates an immutable Arrakis vault that no account has special privileges for. If you create a pool via the [sorbet UI](https://sorbet.finance) this is indeed the type of pool you create. However Arrakis Vaults are further customizable and can be used to implement tokenized _dynamic strategies_ on top of Uniswap V3. To do this one needs to create a "managed" Arrakis Vault, which creates a pool with a privileged manager role with the ability to `executiveRebalance` the position moving the liquidity to a new range on that Uniswap V3 token pair. Via this manager functionality arbitrary rebalancing strategies can be implemented.

{% code title="GUniFactory.sol" %}
```
    function createManagedPool(
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
* `managerFee` The proportion of fees earned that `manager` takes as a cut in Basis Points (deployer address is initial manager). **Note: The managerFee can only be set to a non-zero value once (immutable afterwards)**&#x20;
* `lowerTick` Initial lower price bound for the position, represented as a Uniswap V3 tick.
* `upperTick` Initial upper price bound for the position, represented as a Uniswap V3 tick.

## GUniResolver02

The GUniResolver02 [smart contract](https://etherscan.io/address/0x0317650Af6f184344D7368AC8bB0bEbA5EDB214a#code) is a library of simple helper methods for Arrakis positions. When exposing a UI for users to addLiquidity to Arrakis one may want to allow users to enter the pool with any amount of either asset using the `rebalanceAndAddLiquidity` Router method. However the swap parameters passed as argument to this function must be carefully chosen to deposit maximal liquidity and produce the least leftover (any leftover is returned to `msg.sender` ). This resolver contract exposes a helper for just that.

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

* `pool` Address of Arrakis vault of interest
* `amount0In` amount of token0 sender forwards to router
* `amount1In` amount of token1 sender forwards to router
* `price18Decimals` price ratio of `token1/token0` disregarding (normalizing by) token decimals and then expressed as a WAD (multiplied by `10^18`).

Returns:

* `zeroForOne` direction of swap
* `swapAmount` amount of token to swap

{% hint style="info" %}
Because the price ratio often changes depending on the amount being swapped (slippage) you may want to call this method twice - first with a "reasonable guess" `price18Decimals` to get a preliminary `swapAmount`, then get a more accurate `price18Decimals` (using this `swapAmount` as input), and finally call this method again with the more accurate price to get a final and more precise `swapAmount`
{% endhint %}

## Manager Functions

Pools created with `createManagedPool` are assigned a `manager` account who can configure the Gelato Executor meta-parameters and also can control and alter the range of the underlying Uniswap V3 position.

The manager is the most important and centrally trusted role in the GUniPool. It is the only role that has the power to potentially extract value from the principal invested or potentially grief the pool in a number of ways. One should only put funds into "managed" pools if they have some information about the manager account: manager could be fully controlled by a DAO (token voting), or could simply be a project's multi-sig, or be locked in a smart contract that automates rebalances under a certain codified strategy, or be trusted by the user for some other reason.

#### executiveRebalance

By far the most important manager function is the `executiveRebalance` method on the `GUniPool`. This permissioned method is the only way to change the price range of the underlying Uniswap V3 Position. Manager accounts who control this function are the means by which custom rebalancing strategies can be built on top of Arrakis. These strategies can be implemented by governance (slow, but decentralized) or by some central managerial party (more responsive but requiring much more trust) and in the future the manager role can be granted to a Gelato automated smart contract so that they have a sophisticated LP strategy fully automated by Gelato (both reinvesting fees and in rebalancing ranges).

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
To generate the swap parameters for an executive rebalance tx, simulate the entire operation. It works like this:

\
1\. Call `getUnderlyingBalances` on the GUniPool to obtain `amount0Current` and `amount1Current`\
2\. Compute `amount0Liquidity` and `amount1Liquidity` using LiquidityAmounts.sol library, the new position bounds, the current price and the current amounts from step 1.

3\. Compute `amount0Leftover` and `amount1Leftover` with formula `amount0Leftover = amount0Current - amount0Liquidity`. In most cases one of these values will be 0 (or very close to 0).\
4\. Use `amount0Liquidity` and `amount1Liquidity` to compute the current proportion of each asset needed.\
5\. Use the `amount0Leftover` and `amount1Leftover` and the proportion from previous step to compute which token to swap and the swapAmount.

6\. Convert swapAmount to swapAmountBPS by doing `swapAmount * 10000 / amountLeftover` for the token being swapped.



A complete example of this calculation can be found in this [test file](https://github.com/gelatodigital/g-uni-v1-core/blob/feat/executive-rebalance-test/test/GUniPool.test.ts)
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

## Arrakis Subgraph

The Arrakis subgraph tracks relevant data for all Arrakis Positions created through the `GUniFactory` `createPool` method. Most relevant information about about a Arrakis position is indexed and queryable via the subgraph. Query URL is: [https://api.thegraph.com/subgraphs/name/gelatodigital/g-uni](https://api.thegraph.com/subgraphs/name/gelatodigital/g-uni)\
\
Here is an example query which fetches all information about all Arrakis Positions:

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
          latestInfo {
            sqrtPriceX96
            reserves0
            reserves1
            leftover0
            leftover1
            unclaimedFees0
            unclaimedFees1
            block
          }
        }
      }
```

`id`: subgraph identifier for the pool (contract address)\
`blockCreated`: block Arrakis position was deployed\
`address`: contract address of Arrakis positions\
`uniswapPool`: contract address of Uniswap V3 pair\
`token0`: address of token0\
`token1`: address of token1\
`feeTier`: Uniswap V3 Pair fee tier (500, 3000, 1000)\
`liquidity`: amount of liquidity currently in Arrakis position\
`lowerTick`: current lower tick of Arrakis Position\
`upperTick`: current upper tick of Arrakis Position\
`totalSupply`: current total supply of Arrakis token\
`positionId`: Uniswap V3 ID of the Arrakis position\
`supplySnapshots`: snapshots of the supply when it changes\
`feeSnapshots`: snapshots of fees earned&#x20;

`latestInfo:` has up to date info about the position in a recent block.\
\
The `supplySnapshots`, `feeSnapshots`, and `latestInfo` can be ingested together to produce an estimated APR for fees generated by the position.\
\
APR is calculated from these values in the following way:\
\
1\. We use the `supplySnapshots` to calculate the time weighted average value of reserves `Vr`. This value includes all fees earned.

2\. We use the `feeSnapshots`to add up all the fees earned to generate the total value of fees earned `Vf` .\
\
3\. We compute APR as `Vf / (Vr - Vf)` (when `Vr > Vf` ). In rare cases when `Vf > Vr` i.e. feesEarned are larger than the time weighted average reserves, one can simply use`Vf/Vr`

An npm library for these APR calculations is forthcoming but an example of the calculation is here: [https://github.com/kassandraoftroy/apr-script/blob/main/index.ts](https://github.com/superarius/apr-script/blob/main/index.ts)\


## GUniPool

{% hint style="danger" %}
Methods on the `GUniPool`contract are low level and are **NOT RECOMMENDED** for direct interaction by end users unless they know why and what they are doing. For standard mint/burn interaction with Arrakis see `ArrakisRouter`
{% endhint %}

The `ArrakisVault` [smart contract](https://etherscan.io/address/0x6dfc8b880d6c1043bebb6eb2346913185ce1b48b) is the core implementation that powers the Arrakis protocol (all Arrakis token proxy contracts point to this implementation). It is an extension of the ERC20 interface so all normal ERC20 token functions apply, as well as a number of custom methods particular to Arrakis which will be outlined below. Most of the methods on the ArrakisVault are low level and end users should likely be interacting with peripheral contracts rather than directly with these low level methods. Nevertheless these methods are publicly exposed and functional.

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

View method to compute the amount of Arrakis tokens minted (and exact amounts of token0 and token1 forwarded) from and `amount0Max` and `amount1Max`

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

get the current underlying balances of the entire Arrakis Pool

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

This function is useful for getting a fair unmanipulatable price of a Arrakis token for things like lending protocols (simply pass a time weighted average sqrtPrice to this function to get the unmanipulated underlying balances).

### getPositionId

```bash
function getPositionID() 
    external view returns (bytes32 positionID) 
```

get the Identifier of the Arrakis position on the Uniswap V3 pair (useful for fetching data about the Position with the `positions` method on `UniswapV3Pool.sol`).

## Contact

Questions about integrating G-UNI? Join the Arrakis [Telegram](https://app.gitbook.com/o/-MGYxBRvIEmcAGZbDyDG/s/EupJ9ymQK5fwIT2mGucc/) or [Discord](https://discord.gg/arrakisfinance) and find a dev or community manager to talk to!&#x20;
