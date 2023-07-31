# PoolTogether V5: Part Deux audit details
- Total Prize Pool: $42,000 USDC 
  - HM awards: 29,287.50 USDC 
  - Analysis awards: $1,775 USDC USDC 
  - QA awards: $887.50 USDC 
  - Bot Race awards: $2,662.50 USDC 
  - Gas awards: $887.50 USDC 
  - Judge awards: $3,600 USDC 
  - Lookout awards: $2,400 USDC 
  - Scout awards: $500 USDC 
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2023-08-pooltogether-v5-part-deux/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts August 02, 2023 20:00 UTC 
- Ends August 07, 2023 20:00 UTC 

## Automated Findings / Publicly Known Issues

Automated findings output for the audit can be found [here](add link to report) within 24 hours of audit opening.

*Note for C4 wardens: Anything included in the automated findings output is considered a publicly known issue and is ineligible for awards.*

# Overview

This contest scope completes the PoolTogether V5 audit. The majority of the protocol was audited [by C4 on July 7](https://github.com/code-423n4/2023-07-pooltogether).

This scopes includes several critical pieces:

- Liquidation Pair. This is the contract that liquidates yield for prize tokens.
- Draw Auction. [docs](https://dev.pooltogether.com/protocol/next/design/draw-auction) This suite of contracts auctions off RNG requests and then auctions off the bridge transaction to L2.
- Remote Owner. A contract that builds on [ERC-5164](https://github.com/code-423n4/2022-12-pooltogether) and allows a contract on one chain to contral a contract on another.
- Vault Boost. A contract that allows anyone to liquidate any token and contribute to the prize pool on a vault's behalf.

# Scope

| Contract | SLOC | Purpose | Libraries used |  
| ----------- | ----------- | ----------- | ----------- |
| [./pt-v5-cgda-liquidator/src/LiquidationPair.sol](pt-v5-cgda-liquidator/src/LiquidationPair.sol) | 233 | This contract facilitates Periodic Continuous Gradual Dutch Auctions for yield | [`prb-math`](https://github.com/PaulRBerg/prb-math) |
| [./pt-v5-cgda-liquidator/src/LiquidationPairFactory.sol](pt-v5-cgda-liquidator/src/LiquidationPairFactory.sol) | 63 | This contract creates new LiquidationPairs | |
| [./pt-v5-cgda-liquidator/src/LiquidationRouter.sol](pt-v5-cgda-liquidator/src/LiquidationRouter.sol) | 47 | This contract is the user-facing interface for LiquidationPairs | [`openzeppelin`](https://github.com/openzeppelin/openzeppelin-contracts) |
| [./pt-v5-cgda-liquidator/src/libraries/ContinuousGDA.sol](pt-v5-cgda-liquidator/src/libraries/ContinuousGDA.sol) | 57 | This library implements the CGDA formulas | [`prb-math`](https://github.com/PaulRBerg/prb-math) |
| [./pt-v5-draw-auction/src/RngAuction.sol](pt-v5-draw-auction/src/RngAuction.sol) | 197 | This contract auctions off an RNG request |[`prb-math`](https://github.com/PaulRBerg/prb-math), [`openzeppelin`](https://github.com/openzeppelin/openzeppelin-contracts), [`owner-manager-contracts`](https://github.com/pooltogether/owner-manager-contracts), [`pt-v5-rng-contracts`](https://github.com/GenerationSoftware/pt-v5-rng-contracts) |
| [./pt-v5-draw-auction/src/RngAuctionRelayerDirect.sol](pt-v5-draw-auction/src/RngAuctionRelayerDirect.sol) | 24 | This contract relays RNG request auction results to a listener ||
| [./pt-v5-draw-auction/src/RngAuctionRelayerRemoteOwner.sol](pt-v5-draw-auction/src/RngAuctionRelayerRemoteOwner.sol) | 38 | This contract relays RNG request auction results over an ERC-5164 bridge to a listener | [`ERC5164`](https://github.com/generationsoftware/ERC5164) |
| [./pt-v5-draw-auction/src/RngRelayAuction.sol](pt-v5-draw-auction/src/RngRelayAuction.sol) | 136 | This contract auctions off the RNG result relay | [`ERC5164`](https://github.com/generationsoftware/ERC5164) |
| [./pt-v5-draw-auction/src/libraries/RewardLib.sol](pt-v5-draw-auction/src/libraries/RewardLib.sol) | 52 | Implements Parabolic Fractional Dutch Auction math | [`prb-math`](https://github.com/PaulRBerg/prb-math) |
| [./pt-v5-draw-auction/src/interfaces/IRngAuctionRelayListener.sol](pt-v5-draw-auction/src/interfaces/IRngAuctionRelayListener.sol) | 11 | Listener interface for the RNG auction relay | |
| [./pt-v5-draw-auction/src/interfaces/IAuction.sol](pt-v5-draw-auction/src/interfaces/IAuction.sol) | 12 | Common Auction functions | |
| [./pt-v5-draw-auction/src/abstract/RngAuctionRelayer.sol](pt-v5-draw-auction/src/abstract/RngAuctionRelayer.sol) | 22 | Base class for relayers | |
| [./pt-v5-draw-auction/src/abstract/AddressRemapper.sol](pt-v5-draw-auction/src/abstract/AddressRemapper.sol) | 19 | Allows addresses to remap themselves for relayers | |
| [./pt-v5-vault-boost/src/VaultBoost.sol](pt-v5-vault-boost/src/VaultBoost.sol) | 56 | Allows anyone to liquidate tokens to boost the chances of a Vault winning | |
| [./pt-v5-vault-boost/src/VaultBoostFactory.sol](pt-v5-vault-boost/src/VaultBoostFactory.sol) | 11 | Creates new Vault Boost contracts | |
| [./remote-owner/src/RemoteOwner.sol](remote-owner/src/RemoteOwner.sol) | 11 | Allows a contract on one chain to control a contract on another using ERC-5164 | [`ERC5164`](https://github.com/generationsoftware/ERC5164) |
| [./remote-owner/src/libraries/RemoteOwnerCallEncoder.sol](remote-owner/src/libraries/RemoteOwnerCallEncoder.sol) | 12 | Helps encode calls to a Remote Owner contract | |

# Additional Context

## Mathematical Models

**Continuous Gradual Dutch Auction**

The LiquidationPair prices yield liquidations using a periodic [Continuous Gradual Dutch Auction](https://www.paradigm.xyz/2022/04/gda). It's periodic in the sense that the auction runs in periods that will be aligned with the prize pool periods. At the beginning of each period, the CGDA adjusts the emissions rate and target price so that it adapts to changing market conditions.

**Parabolic Fractional Dutch Auction**

For the RNG auction we employ a novel auction we call a [Parabolic Fractional Dutch Auction](https://dev.pooltogether.com/protocol/next/design/draw-auction#parabolic-fractional-dutch-auction-pfda). This allows us to have an auction on L1 and have the price as a fraction of an L2 Prize Pool's reserve.

## Deployment Configuration

**LiquidationPair**

There will be a LiquidationPair for each Vault. The pair's period length and offset will match the prize pool draw length and offset. This is to try to ensure there is an auction every draw. The target first sale time will be halfway through the auction, as that configuration works at a variety of TVLs in our simulations.

The minimum auction amount will be configured to ensure a minimum efficiency of the liquidation pair.  If liquidation gas costs are $0.10 and we want a minimum liquidation efficiency of 95%, then the minimum auction amount should be $2.00 worth of tokens.

The maximum decay constant is approximately 0.00015.

**RNGAuction**

The RNG auction's sequence period and offset will be daily and match all prize pools on L2. The auction duration will be less than the sequence period, to ensure that the VRGDA Claimer has sufficient time to claim prizes.

## Unique Concerns

In addition to concerns around the security of funds, we also have a number of domain-specific concerns:

**Liquidation Pair getting bricked or hacked**. Our goal is for each Liquidation Pair to run indefinitely. The pair supports price volatility in several orders of magnitude around the last sale price, but is it possible for the pair to break permanently? Can it be bricked deliberately or accidentally?

**Draw Auction performing badly**. The RNG Auction and RNG Relay Auctions are a novel mechanism that auction using a fraction of the Prize Pool reserve, rather than a specific price. Is there a significant risk of draining the reserve? Can the auction deviate from expectations and become bricked or unusable?

**Losing control of the Remote Owner**. The Remote Owner is a unique contract in that it will allow us to extend our control from one chain to another. Can someone usurp us and take control of our Remote?

## Scoping Details 
```
- If you have a public code repo, please share it here:  https://github.com/GenerationSoftware/pt-v5-draw-auction and https://github.com/GenerationSoftware/pt-v5-cgda-liquidator
- How many contracts are in scope?:  17 
- Total SLoC for these contracts?:  1001
- How many external imports are there?: 6 
- How many separate interfaces and struct definitions are there for the contracts within scope?:  5
- Does most of your code generally use composition or inheritance?:  Composition
- How many external calls?:   7
- What is the overall line coverage percentage provided by your tests?: 100%
- Is this an upgrade of an existing system?: False
- Check all that apply (e.g. timelock, NFT, AMM, ERC20, rollups, etc.): 
- Is there a need to understand a separate part of the codebase / get context in order to audit this part of the protocol?:  True
- Please describe required context:   PoolTogether V5: Part I (the July 7, 2023 audit)
- Does it use an oracle?:  No
- Describe any novel or unique curve logic or mathematical models your code uses: There are two variations of dutch auctions. One is a continuous gradual dutch auction, and the other is a fractional dutch auction
- Is this either a fork of or an alternate implementation of another project?:   False
- Does it use a side-chain?: False
- Describe any specific areas you would like addressed: We want to make sure that the auctions can't be manipulated or bricked, and that the auction fees are competitive or priced correctly.
```

# Setup

Clone using the `--recurse` option or update the repository with `git submodule update --init --recursive`

# Tests

Within each git submodule, you can run `forge test` to run all tests. You can run `forge coverage` to see the coverage report.

One-Liner for tests: `cd claimer && forge test && cd .. && cd prize-pool && forge test && cd .. && cd twab-controller && forge test && cd .. && cd vault && forge test && cd ..`

Note: the Prize Pool must be compiled with `--optimize --via-ir` in order to fit within contract size limits.
