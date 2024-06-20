This is part of the build/response to the overall arb-bot spec
https://hackmd.io/u_7pJgpjSdmbdpIfKlINXA?both


# Router
A standalone library that finds a good enough route on an evm chain of onchain liquidity providers (decentralized exchanges) for a pair of tokens that is consumable by `sushiswap RouteProcessor` contracts to execute a swap between that pair.
This will be a component of `arb-bot` that ultimately uses the found route to clear rain orderbook orders against onchain liquidity.

# Pool Finder
A standalone library that finds (uniswap v2 and v3 pools (possibly other protocols)) on an evm chain for pair of tokens, as a component of a `router` library.

## goals
- should be as cheap as possible in terms of rpc consumption
- should be as performant as possible since there is a very tight margin on each block when arb bot is running on per block basis
- should be reproduceble, ie once the needed data is available, routes can be calculated for any token amount and any token combination that is available in the data.

The ultimate goal here is to find a good balance between performance/rpc consumption and how good a route is at a ceratin block. calculating the route on each block is ofc the ideal case as it ends up providing the most accurate/reliable result, but a found route can be used for some time until it is recalculation again.


## 1 - Find Pools:
Finding available pools for token pair `A/B`, with custom additional tokens (called bases in this doc, custom tokens are mostly high liq and widely adopted tokens such as `USDT`, `USDC`, `DAI`, `WETH`, `WBTC`), so any paired combinations of tokens `A`, `B` and bases can be generated (using create2 logic for every available dex on the operatiing chain), then they can be checked onchain that if they exist or not, those that dont will be filtered out, and those that do exist, will be included as the final result as all the available pools of all available dexes on the operating chain, once pools are found, their required data (such as reserves balances, ticklens, etc) can be fetched (from onchain).

Another approach for finding pools is to use indexers, ie check each token pair combination on the indexer and get their data.

This process/functionalities can be wrapped with a struct with a hashmap to provide a multichain functionality.

## 2 - Find Routes:
Once the pool data is available (from pool finder), for each pool with token `A` (`A/*`), the `priceImpact` and `amountOut` can be calculated with replicating the math logic that happens onchain on the contract (this would contain either direct pool of `A/B` or any pool with `A/*` for finding a multiroute, in this case 1 hop max).

For 1 hop routes, the result of previous step (`amountOut` of previous step) can be used to calculated `priceImpact` and `amountOut` of the pools with the in-between token (`*/B`) by doing the same process.

Once these values are calculated, they can be compared and the best outcome can be chosen as the route for `A/B` (it can end up being a direct route or a multiroute with 1 hop; ideally this design should be expandable (`A/*`, (`*/* x n`), `*/B`), ie in a way that in future can be upgraded to be able to do more hops, so most probably a recursive or looped based design).

Once the route and its legs are ready, the final route code can be built by following the logic that sushi lib has for each type of pool, the logic is pretty straight forward, as our case doesnt involve any non uniswap based dex, so we either have `univ2` or `univ3`, (they each have RP based code and some simple logic to set the direction, etc), once each leg's data has been constructed, they will be merged together to form the final route code that can be used to submit a tx onchain on a RP contract.
