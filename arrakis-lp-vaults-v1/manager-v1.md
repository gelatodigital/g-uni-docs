# Manager V1

The manager account within the vault is completely permissionless, anyone and everyone can be a manager, however the vault deployer has to specify who the manager should be.

The manager can specify the range that should be used, they can specify how to shift the liquidity of the vault around, in order to do this they simply specify the new price bounds, and also specify the amount they want to swap in order to rebalance the inventory so that more funds fit into the token ratio needed for the new position.&#x20;

## **Bene Gesserit Strategy:**

This is the Arrakis DAO generated strategy for the V1 vaults.

For any project that wants deep liquidity and does not want to match the vaults themselves we offer the Bene Gesserit Strategy. First of all we discuss with the project what their goals are in terms of how long they want their liquidity, how deep they want it, and whether they are looking to deploy their own funds or want to run a liquidity mining scheme to attract outside capital. In case this sounds like you, please do reach out to us on our [discord](https://discord.gg/arrakisfinance).

There after once the goals are aligned, the Bene Gesserit run Monte-Carlo simulations of price moves over the past months (length various on the above mentioned goals), these simulations are run in order to generate price range that has a 95% probability of staying in range for at least a few months (again dependent on the goals the project has). Each week a vault rebalancing meeting is held where all vaults are looked at in regards to recent price movements, Monte Carlo simulations are re-ran to see what the price bounds would be if the liquidity providing would be started fresh, decisions would be made whether a rebalance of the LP position should be made. A rebalance is usually only opted in for when inventory becomes very one sided, and risk of going out of range is paramount. The idea is always to rebalance as gently as possible, meaning to "swap" as little as possible, so as not to be making to opinionated trading decisions.&#x20;

![Visualization of setting the range and the re-ranging](<../.gitbook/assets/Screenshot 2022-08-17 at 12.10.50.png>)



In the case of the Bene Gesserit Strategy, the manager account is governed by a multisig that can only move the range with a 3/6 sign amount.&#x20;

These Multisigners are:\
Kassandra (CTO)- [https://twitter.com/kassandraETH](https://twitter.com/kassandraETH)\
Hilmar Orth (Head Strategy)- [https://twitter.com/hilmarxo](https://twitter.com/hilmarxo)\
Luis (Legendary Member Gelato)- [https://twitter.com/gitpusha](https://twitter.com/gitpusha)\
Shankinson (Head of Bene Gesserit)\
Matthieu (Lead Smart Contract)











