# Manager V1

The manager account within the vault is completely permissionless, anyone and everyone can be a manager, however the vault deployer has to specify who the manager should be.

The manager can specify the range that should be used, they can specify how to shift the liquidity of the vault around, in order to do this they simply specify the new price bounds, and also specify the amount they want to swap in order to rebalance the inventory so that more funds fit into the token ratio needed for the new position.&#x20;

## **Bene Gesserit Strategy:**

For any project that wants deep liquidity and does not want to manage the vaults themselves the full autonomous Bene Gesserit Strategy exists. In case you are a project that is looking for deep liquidity strategies, please do reach out to on [discord](https://discord.gg/arrakisfinance).

The full autonomous Bene Gesserit Strategies run Monte-Carlo simulations of price moves over the past months (length various on the above mentioned goals), these simulations are run in order to generate price range that has a 95% probability of staying in range for at least a few months (again dependent on the goals the project has). Each week a vault rebalancing check is done where all vaults are checked in regards to recent price movements, Monte Carlo simulations are re-ran to see what the price bounds would be if the liquidity providing would be started fresh, then a rebalance of the LP position is either made or not. A rebalance is usually only opted in for when inventory becomes very one sided, and risk of going out of range is paramount. The idea for this strategy is to rebalance as gently as possible, meaning to "swap" as little as possible, so as not to be making opinionated trading decisions.&#x20;

![Visualization of setting the range and the re-ranging](<../.gitbook/assets/Screenshot 2022-08-17 at 12.10.50.png>)
