This is part of the build/response to the overall arb-bot spec
https://hackmd.io/u_7pJgpjSdmbdpIfKlINXA?both


# Router
A library that finds a good enough route on an evm chain of onchain liquidity providers (decentralized exchanges) for a pair of tokens that is consumable by `sushiswap RouteProcessor` contracts to execute a swap between that pair.
This will be a component of `arb-bot` that ultimately uses the found route to clear rain orderbook orders against onchain liquidity.

# Pool Finder
A library that finds (uniswap v2 and v3 pools (possibly other protocols)) on an evm chain for pair of tokens, as a component of a `router` library.

## goals
- should be as cheap as possible in terms of rpc consumption
- should be as performant as possible since there is a very tight margin on each block when arb bot is running on per block basis
- should be reproduceble, ie once the needed data is available, routes can be calculated for any token amount and any token combination that is available in the data.


The ultimate goal here is to find a good balance between performance/rpc consumption and how good a route is at a ceratin block. a found route can be used for some time until a recalculation process is triggered again.


## 1- Pool Finder
finding available pools for token pair A/B, with additional tokens (called bases in this doc), so any paired combinations of tokens A, B and bases can be generated (using create2 logic for every available dex on the operatiing chain), then they can be checked onchain that if they exist or not, those that dont will be filtered out, and those that do exist, will be included as the final result as all the available pools of all available dexes on the operating chain, once pools are found, their required data (such as reserves balances, ticklens, etc) can be fetched (from onchain).
another approach for finding pools is to use indexers, ie check each token pair combination on the indexer and get their data.
This process/functionalities can be wrapped with a struct with a hashmap to implement a multichain functionality.

## 2 - find routes:
once the pool data is available, for each pool with token A, the `priceImpact` and `amountOut` can be calculated with replicating the math logic that happens onchain on their contract. (this would contain either direct pool of A/B or any pool with A/* for finding a multi hop route, in this case 1 hop max)
for 1 hop routes, the result of previous step (amountOut of previous) can be used to calculated `priceImpact` and `amountOut` of the pools with the in-between token by doing the same process, and then the same process will be repeated for any pools with token B and the output token of previous step.
once these values are calculated, they can be compared and the best outcome can be chosen as the route for A/B (it can end up being a direct route or a multi route with 1 hop).(ideally this design should be expandable, ie in a way that in future can be upgraded to be able to do more hops, so most probably a recursive or looped based design)
once the route and its legs are ready, the final route code can be built by following the logic that sushi lib has  for each type of pool, the logic is pretty straight forward, as our case doesnt involve any non uniswap based dex, so we either have univ2 or v3, (they each have RP based code and some simple logic to set the direction), once each leg data has been constructed, they will be merged to form the final route code that can be used to submit onchain on a RP contract.
