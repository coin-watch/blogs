---
title: "Ethereum CEX-DEX Arbitrage: What Searcher Concentration Means for Crypto Markets"
date: "2026-10-22"
description: "An Ethereum Foundation study estimates $233.8 million in CEX-DEX arbitrage revenue and finds that three searchers captured three-quarters of the measured volume and value. Here's how the market works and why it matters."
tags: ["Ethereum", "MEV", "Algorithmic Trading", "Market Structure", "Crypto Arbitrage", "DeFi"]
---

# Ethereum CEX-DEX Arbitrage: What Searcher Concentration Means for Crypto Markets

![Ethereum CEX-DEX arbitrage market structure with centralized exchange order books connected to decentralized liquidity pools, automated searcher bots, block builder pipeline, dark background #121613, neon green accents #C4E855, clean editorial financial illustration](https://image.pollinations.ai/prompt/Ethereum%20CEX-DEX%20arbitrage%20market%20structure%20with%20centralized%20exchange%20order%20books%20connected%20to%20decentralized%20liquidity%20pools%2C%20automated%20searcher%20bots%2C%20block%20builder%20pipeline%2C%20dark%20background%20%23121613%2C%20neon%20green%20accents%20%23C4E855%2C%20clean%20editorial%20financial%20illustration?width=800&height=400&nologo=true)

Crypto arbitrage is often described as a simple race: buy an asset where it is cheap, sell where it is expensive. On Ethereum, the mechanics are more demanding. A searcher has to spot the price gap, submit a transaction, pay for block inclusion, and compete with other bots before the opportunity disappears.

A 2025 study from the Ethereum Foundation's Robust Incentives Group examined this market using on-chain data and a method for estimating arbitrage revenue without access to traders' centralized-exchange activity. For transactions identified between August 2023 and March 2025, the researchers estimated $233.8 million in revenue from 7,203,560 CEX-DEX arbitrages involving 19 major searchers. Their analysis found that three searchers accounted for three-quarters of both measured volume and extracted value.[1]

Those figures describe the study's sample and methodology, not every arbitrage trade across Ethereum or the entire crypto market. Still, the results offer a useful look at how algorithmic trading, block production, and market access intersect.

## What is CEX-DEX arbitrage?

A centralized exchange (CEX) matches orders through an operator's trading system. A decentralized exchange (DEX), by contrast, executes swaps through smart contracts. Since these venues update at different speeds and attract different flows, their prices can diverge temporarily.

A CEX-DEX arbitrage searcher monitors prices and tries to trade against that gap. The strategy can help bring prices back into line across venues. But an Ethereum transaction must compete for block space, and the searcher cannot assume that a profitable-looking opportunity will still exist when the transaction executes.

Ethereum's documentation describes maximal extractable value (MEV) as value arising from the inclusion, exclusion, or ordering of transactions in a block beyond the standard block reward and gas fees.[2] Arbitrage is one common source of MEV. Searchers use software to identify opportunities and submit transactions; their bids and fees compete for inclusion.[2]

## Why integration with builders matters

The Ethereum study reports that searcher profitability depends on how closely searchers are integrated with block builders. It also identifies exclusive searcher-builder relationships and finds that accounting for vertically integrated builders changes estimates of builder profitability.[1]

This connection is important because a transaction's outcome depends on more than the price signal. Searchers need reliable execution inside a block, while builders assemble candidate blocks and compete to supply them to proposers. Ethereum's explanation of proposer-builder separation describes builders as creating blocks and offering them to a proposer, who selects a profitable proposal.[3]

For a trading firm, better connectivity and coordination can influence the chance of getting a trade included in time. That creates an operational advantage distinct from the quality of the trading signal itself. A model that detects a price difference but cannot land its transaction reliably may lose to a competitor with tighter infrastructure or a stronger builder relationship.

## The concentration finding, in context

The study's headline result is concentration: three of the 19 major searchers captured three-quarters of measured CEX-DEX volume and extracted value across the dataset.[1] This does not prove that all Ethereum arbitrage is controlled by three firms. The sample covers identified CEX-DEX arbitrages and major searchers over a specified 19-month window; it does not represent every form of MEV, every trading venue, or every participant.

The distinction matters for interpreting market structure. A highly competitive race can still produce concentrated results if the winners have better infrastructure, information, capital, or access to block production. The paper's findings point to integration with builders as one factor associated with searcher profitability, rather than treating the strategy as a level playing field.[1]

## What this means for algorithmic traders

For quant teams, the paper is a reminder that execution design belongs in the model, not as an afterthought. Backtests that compare exchange prices without accounting for transaction costs, timing, inclusion probability, and competition can overstate the returns a deployable strategy might capture.

It also suggests that arbitrage performance should be measured across the whole execution path: opportunity detection, transaction construction, bidding, inclusion, and realized settlement. A visible spread is only a candidate trade. It is not guaranteed profit.

For users swapping on-chain, arbitrage and MEV have mixed effects. Arbitrage can reduce price differences across markets, while transaction ordering can also expose users to adverse outcomes such as sandwich attacks. Ethereum's documentation distinguishes these forms of MEV and notes that some extraction can come at a user's expense.[2] Wallet protections, slippage limits, and transaction privacy tools address different parts of that risk, but none removes every execution uncertainty.

## What to watch next

Future research and market monitoring can test whether the observed concentration persists across newer periods, venues, and types of arbitrage. It is also worth tracking how changes to Ethereum's block-building arrangements affect searcher-builder relationships and the distribution of revenue. The current study documents a specific dataset and period; it does not settle how the market will evolve.[1]

For now, the evidence supports a narrower conclusion: Ethereum CEX-DEX arbitrage is a substantial, infrastructure-intensive market in which measured results were concentrated among a small group of searchers. Anyone evaluating crypto algorithmic trading strategies should account for access and execution, not just price discrepancies.

## Sources

[1] [Measuring CEX-DEX Extracted Value and Searcher Profitability: The Darkest of the MEV Dark Forest — Ethereum Foundation Robust Incentives Group](https://rig.ethereum.org/paper/measuring-cex-dex-extracted-value-and-searcher-profitability-the-darkest-of-the-mev-dark-forest)

[2] [Maximal extractable value (MEV) — ethereum.org](https://ethereum.org/en/developers/docs/mev)

[3] [Proposer-builder separation — ethereum.org](https://ethereum.org/roadmap/pbs)
