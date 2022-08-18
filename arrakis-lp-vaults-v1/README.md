---
description: Examples of what can be build using Arrakis Vaults as money legos
---

# Arrakis LP Vaults V1

The Arrakis LP Vaults are the underlying technology which make management of liquidity possible.

These Vaults allow for liquidity to be managed in an automated way on a concentrated liquidity DEX, currently these vaults are only active on Uniswap V3.

In short, the Vaults are smart contracts which allow for projects or users to deposit funds, and vault managers to activate strategies on these vaults. This makes UniswapV3 as easy to use for projects and users as UniswapV2, no longer having to think about "where should I set my LP range?".

Vaults interact with 2 users:

* **Projects/Users**-  that provide the liquidity
* **Managers**- Managers can be the Arrakis Strategies also known as the Bene-Geserit-Strategists, or external managers

When a project or user deposits funds into the Vault they receive LP tokens back which represent their share of the overall Vault LP position.

## **Bene Gesserit Strategists:**&#x20;

Arrakis' vaults allow for anyone to be a manager, the Bene Gesserit Strategists are the first ones to have made strategies.

These strategies are focused on optimising for long term sustainable deep liquidity for project tokens, to read in detail how strategies work see the [Strategy](manager-v1.md) page.
