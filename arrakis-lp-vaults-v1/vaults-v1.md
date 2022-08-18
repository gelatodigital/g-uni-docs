# Vaults V1

The Vaults V1 are the first iteration of our automised liquidity vaults, keeping the mission in mind of optimising DEX liquidity the focus is on easily allowing managers to deploy strategies and LPers to deploy capital into these vaults.&#x20;

The capabilities of these vaults are the following:

* ERC20 wrapper for Uniswap V3 LP Positions
* Allow deploying of vaults with any price bounds on any Uniswap V3 pair
* &#x20;Allow for manager role to control the LP range
* Allow for manager role to re-range the partial position, or total position
* Allow for liquidity mining rewards to be distributed to depositors
* Fully non-custodial
* Fully Fungible

## Setting up a Vault

1\) Launch a Uniswap V3 pair for your token and the base asset you would like to have your token trade against e.g. USDC or ETH

2\) Create a vault on Arrakis

3\) Choose a manager for the vault, and deploy the vault

3\) Depositors can add and remove liquidity

Anyone can deploy these V1 vaults, they are fully permissionless, if you would like to learn more about the code base see the developer docs [here](../developer-docs.md).
