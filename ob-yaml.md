## Basic design principles

- Minimal
- Avoid repetition
- prefer named items (k/v sets) over unordered lists with inline ad hoc naming
  - can be referenced elsewhere unambiguously
- avoid needless depth in heirarchies
  - prefer flat, except where it would imply excessive repetition
- support multitudes of things
- strict yaml for parsing, apply explicit schemas/types for non-strings
- prefer structures that could be merged to support cascading/overriding of multiple files
- UX/UI agnostic
  - any combination of these yaml structures _could_ appear in any section of the app for editing/viewing/composing

## Networks

"Network" is the terminology used by metamask, foundry, hardhat, and implied by terms such as "mainnet" and "testnet".

Need to define many networks that can be switched between at any time. Generally we follow the format at chainlist.org

Required fields:

- `rpc`
- `chain-id`

Optional fields:

- `label`
- `network-id`
- `currency`

### Chain ID vs Network ID

Technically the chain ID and network ID can be different but _usually_ are the same by convention, but not always, we can see that eth classic has chain ID 61 and network ID 1 for example.

https://besu.hyperledger.org/23.4.0/public-networks/concepts/network-and-chain-id

### Example

```
networks:
  mainnet:
    rpc: https://eth.llamarpc.com
    chain-id: 0x1

  classic:
    label: ETH classic
    rpc: https://etc.rivet.link
    chain-id: 61
    network-id: 1
    currency: ETC
```

## Subgraphs

Currently subgraphs are 1:1 with orderbooks, but this could and probably should change in the future. It would be far better for downstream implementations if a single subgraph could handle _at least_ all bytecode identical orderbooks deployed to the same network, if not a set of known compatible bytecodes. For that reason, it might seem overkill to specify subgraphs separately but they should have a 1:many relationship with orderbooks in the mid term.

Subgraphs have no fields, they're merely a name for a url string.

```
subgraphs:
  polygon: https://...
  polygon2: https://...
  mainnet: https://...
```

## Orderbooks

Every orderbook is a contract deployed on some chain (has an address) with a subgraph that knows how to index it into the form expected by the application.

Required fields:
  - `address`

Optional fields:
  - `network` (foreign key into the known networks k/v, default is same as orderbook name)
  - `subgraph` (default is same as orderbook name)
  - `label`

```
orderbooks:
  polygon:
    address: 0x...

  polygon-experimental:
    network: polygon
    address: 0x...
    subgraph: polygon2
    label: Polygon experimental

  legacy:
    network: mainnet
    address: 0x...
```

## Vaults

From the perspective of yaml structure, vaults are much like subgraphs. There is a 1:many relationship with both orders that reference them for IO and tokens that might reference them for capital sandboxing. Each configured vault is simply a convenient name for (commonly) an opaque 32 byte hash. I.e. vaults don't have fields.

The GUI SHOULD provide an easy way for users to generate randomised 32 byte values against memorable names, so that they can create many vaults easily, similar to how web2 services spin up randomised subdomains for development websites.

```
vaults:
  # arbitrary
  foo: 0x...
  # date
  2024-02-20: 0x...
  # purpose
  dca-input: 0x...
  dca-output: 0x...
  # generated
  minute-dozen: 0x...
  held-flavor: 0x...
```

## Tokens

At the minimum a token is a network, address and decimals. While ERC20 doesn't mandate decimals on the contract, it is required in practise to allow for standardized decimal 18 math, as non-18 decimal token values need to be scaled up/down to match. If a token has `0` decimals then this implies there are no fractional values and any amounts are to be treated as literal integers.

The GUI MAY query the network rpc + address to attempt to populate the optional fields (notably decimals). Note that the ERC20 specification explicitly DOES NOT mandate that tokens implement the metadata methods, including decimals. If the contract query fails, and the relevant optional field has not been specified in the yaml, this MUST be treated as an error for the user to fix. NEVER assume decimal values that haven't been provided by the contract/user.

Required fields:

- `network`
- `address`

Optional fields:

- `decimals` (fetch from contract)
- `label` (fetch from contract, called `name` in the erc20 interface)
- `symbol` (fetch from contract)

```
tokens:
  eth-usdc:
    network: mainnet
    address: 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48
    decimals: 8
    label: USD Coin
    symbol: USDC
  eth-dai:
    network: mainnet
    address: 0x6b175474e89094c44da98b954eedeac495271d0f
```

## Deployers

Each deployer contract serves as the logical entrypoint into an entire DISpair (the associated addresses can be queried from the deployer onchain).

Typically a single DISPair will be used for many orders and even across many orderbooks.

Required fields:
  - `address`

Optional fields:
  - `network` (assume deployer name if not set)
  - `label`

```
deployers:
  mainnet:
    address: 0x...
  polygon:
    address: 0x...
  polygon-experimental:
    address: 0x...
    network: polygon
```

## Front matter yaml

This yaml is NOT arbitrary across the GUI. It only makes sense when coupled to some specific rainlang, which also makes the most sense when provided as frontmatter that can be directly parsed, composed and bound by the `dotrain` tool.

### Front matter orders

Top level element `orders` in the front matter.

Used to define a set of named orders that can be deployed onchain using `addOrder` on an orderbook contract.

Requires all the deployment components to be defined already somehow in the GUI as per the above yamls.

Required fields:
- `inputs`
- `outputs`
- `network`

Optional fields:
- `deployer` (defaults to network deployer if unambiguous, otherwise required)
- `orderbook` (defaults to network orderbook if unambiguous, otherwise required)

```
orders:
  dca-eth:
    network: mainnet
    inputs:
      - eth-weth
    outputs:
      - eth-usdc
      - eth-dai
      - eth-usdt
  dca-eth-polygon:
    network: polygon
    orderbook: polygon-experimental
    deployer: polygon-experimental
    inputs:
      - polygon-weth
    outputs:
      - polygon-usdc
      - polygon-dai
      - polygon-usdt
```

### front matter scenarios

Scenarios are a hierarchical structure that specifies bindings used by `dotrain` tooling to produce a concrete rainlang instance that can be parsed and deployed onchain, and deliberately introduces ambiguity to be iteratively disambiguated by a fuzzer, for the purpose of producing simulations.

The bindings in yaml are forwarded as-is to `dotrain` as strings, so all forms are supported including quote bindings, etc.

All fields are optional, if there exists enough unambiguous deployment components all sharing the same name, e.g. `mainnet` and there are no elided bindings in the body of the .rain file (under the front matter) then a deployment is possible, as no bindings are required.

If there is any ambiguity however, notably in the case of elided bindings, a binding set will need to be provided.

Deployments of nested scenarios are expected to be a path such as `foo.bar.baz`. Every level of the scenario path inherits its parents bindings recursively. The inheritance avoids overly verbose yaml files that would need to redeclare perhaps dozens of the same bindings redundantly just to modify one or two individual bindings. By nesting scenarios, only the accretionary/modified diff need be specified at each level.

Shadowing is disallowed, if a child specifies a binding that is already set by the parent, this is an error.

Optional fields:
- `bindings` bindings for this scenario level, each binding is a `key: value` pair, forwarded to `dotrain` tooling as strings
- `runs` an integer number of runs for the fuzzer
  - **all elided bindings are fuzzed independently** when `runs` is specified
  - `runs` is an invalid key for concrete deployments, as there is no fuzzer and so elided bindings remain elided
- `network` (fallback to network of same name if not set **root scenario only**)
- `deployer` (fallback to deployer of same name if not set **root scenario only**)

```
scenarios:
  mainnet:
    bindings:
      foo: ...
      bar: ...
    scenarios:
      sell:
        bindings:
          bing: ...
      buy:
        bindings:
          bing: ...
  fuzzer:
    network: testnet
    deployer: testnet
    orderbook: testnet
    runs: 100000
    # bar intentionally not set so it would be fuzzed
    bindings:
      foo: ...
      bing: ...
```

### front matter charts

Any scenario can be charted as every concrete set of bindings can be treated as a data point. For a single concrete set, a single data point is produced, for a fuzzer or similar, a set of data points will be produced.

For ease of implementation we simply name the scenario to plot and then expose observable plot config directly to the yaml. Plot is a mostly declarative DSL from the creators of d3 https://observablehq.com/plot/features/plots

Required chart fields:
- `<key>` named plots, need at least 1 for a chart

Optional chart fields:
- `scenario` name of the scenario to draw data from, falls back to the name of the chart if not set

Required plot fields
- `data`: stack trace paths to draw x and y data from according to the scenario
- `type`: the type of observable plot, e.g. `dot`, `ruleY`, etc.

Optional plot fields
- `<key>` keys according to the observable plot DSL for each plot type

```
charts:
  fuzzer:
    y-axis:
      type: ruleY
      domain: [0, 100]
    normalized-amount:
      type: dot
        fill: blue
      data:
        # stack trace paths
        x: 0.1.4
        y: 0.6
    denormalized-amount:
      type: dot
        fill: red
      data:
        x: 0.1.4
        y: 0.5
```

### front matter deployments

Specifies deployments that consist of `scenario` in combination with an `order` mapped to a key:

Required deployment fields:
- `scenario` name of a defined `scenario`
- `order` name of a defined `order`

```
deployments:
  first-deployment:
    scenario: scenario1
    order: order1
  second-deployment:
    scenario: scenario2
    order: order2
```