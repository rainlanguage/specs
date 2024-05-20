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


## Pool Finder Design 
A pool address can be calculated using `create2` logic by having 2 token addresses, factory address and `initCodeHash`, however the calculated pool address might not be deployed onchain, ie a direct pool for a token pair A/B is not available, so this brings in the usage of intermediate pools/tokens that will be used to be able to perform the swap between token A/B. 
Intermediate tokens (called bases throughout this document) can be supplied at runtime or can have a default per chain as most univ2/v3 pools are deployed against well-known, highly adopted, commonly traded, etc tokens such as USDT, USDC, WETH, WBTC, DAI.

So when trying to find pools for token pair A/B, the bases can be supplied as option (or use defaults), calculate the pool addresses for each combination and then check if the calculated addresses actually exist onchain or not, the ones that do exist will be stored into a struct field with their token details. the ones that do not exist, will be stored into the struct `blacklist` field, so next time there is new request for a new token pair, same process is repeated but before checking if the calculated addresses exist onchain or not, they are checked against the `blacklist` and those that already are known to exist.
This will gurantee no same pool address is checked against onchain data twice during runtime as well as reducing the need for rpc calls as more pools are discovered.
So they will be available throughout the `router` program execution once there is a request to find a route for a token pair.

The `blacklist` needs to be purged once in while, the reason for this is that some pools might get deployed after they have gone into the blacklist, or there might have been some https/rpc error when a pool was initially checked that it exists onchain or not. 
A simple logic can implemented to check the blacklist pools once in a while and if they happen to exist onchain, they will get removed from the blacklist, another suggestion can be to have a `pending blacklist`, ie not put pools directly into the blacklist for the first time they return error when read from onchain, but put them in a pending state, and if they happen to return error a few more times, then they will go into the blacklist.

There should be methods to export/import the results, so the database of the found pools can be stored on disk and loaded from, to lower the rpc consumption as much as possible, after all once a pool is found, there is no need to find it again if the results are stored on a disk that can be loaded again at runtime.

Since the database can grow overtime, it is logical to store them sorted, resulting in increasing performance on serach actions. the goal here is performance, so any logic or external library can be used to achieve this, not just limited to a sorted set/map.

This struct will be per chain basis, but a wrapper struct with a `map` of chain ids against it can be implemented to cover multichain implementation.

## Router Design 
The ultimate goal here is to find a good balance between performance/rpc consumption and how good a route is at a ceratin block.
The overal design for calculating a good enough route for token pair A/B requires knowing the reserves balances of the involved pools (direct and indirect), and for univ3 it would require some extra data ontop (ticklense as an example). once these data are available then a route can be calculated that can be consumed by the `RouteProcessor` contract.
The pools (direct and indirect) that their data need to be fetched are being provided by the `PoolFinder`, the found route can be used for the token pair A/B for some time (like 20 minutes) until there is a need to refetch the reserves data again, this is because calculating the best route on each block will come at the expense of more rpc calls and lower performance, so at the end of the day it wont actually serve the purpose of the `arb-bot`, unless calculating a route on per block basis can be achieved in a way that doesnt eliminate any of the mentioned goals and requirements.
