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

## Specification Version

To handle potential breaking changes in the YAML format itself, a top-level `spec-version` field should be included. This allows tooling to identify the expected structure and handle different versions appropriately.

Required fields:
  - `spec-version`

### Example

```yaml
spec-version: 1.0.0
```

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

### Using networks from

To support prepopulating the app with networks we support `using-networks-from`
which fetches well known lists of networks with well known formats and maps them
internally to what we need/expect.

Required fields:
  - url
  - format

#### Supported formats

##### ChainID

Format: `chainid`

The json as per https://chainid.network/chains.json

Maps the `shortName` to the name of the network. Other fields are extracted as
needed.

### Example

```
using-networks-from:
  chainid:
    url: https://chainid.network/chains.json
    format: chainid
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

# Metaboards

Metaboard is an onchain contract for posting Rain meta about a subject (an address). This meta is indexed by a metaboard subgraph. Currently these are 1-1 with networks.

Metaboards have no fields, they're merely a name for a url string.

```
metaboards:
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

## Accounts

Accounts are optional filters that can be used to filter the orderbook to only show orders and vaults that belong to that account.

Account aliases are mapped to account addresses.

```
accounts:
  my-account: 0x...
  my-other-account: 0x...
```

## Sentry Analytics

The app will optionally collect analytics data to send to Sentry. This functionality is opt-out and enabled by default. This data helps us identify UI bugs and provide the best possible user experience.

Optional fields:
- `sentry` (defaults to 'true')

```
sentry: false
```

## Front matter yaml

This yaml is NOT arbitrary across the GUI. It only makes sense when coupled to some specific rainlang, which also makes the most sense when provided as frontmatter that can be directly parsed, composed and bound by the `dotrain` tool.

### Front matter orders

Top level element `orders` in the front matter.

Used to define a set of named orders that can be deployed onchain using `addOrder` on an orderbook contract.

Requires all the deployment components to be defined already somehow in the GUI as per the above yamls.
Network will be taken from inputs/outputs token's network, and they must match as well as `deployer` and `orderbook` if they are specified.

Required fields:
- `inputs`
- `outputs`

Optional fields:
- `deployer` (defaults to network deployer if unambiguous, otherwise required)
- `orderbook` (defaults to network orderbook if unambiguous, otherwise required)

```
orders:
  dca-eth:
    inputs:
      - token: eth-weth
        vault-id: 1
    outputs:
      - token: eth-usdc
        vault-id: 1
      - token: eth-dai
        vault-id: 0x123
      - token: eth-usdt
        vault-id: 2
  dca-eth-polygon:
    orderbook: polygon-experimental
    deployer: polygon-experimental
    inputs:
      - token: polygon-weth
        vault-id: 1
    outputs:
      - token: polygon-usdc
        vault-id: 2
      - token: polygon-dai
        vault-id: 99
      - token: polygon-usdt
        vault-id: 0xabcd
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
- `blocks` a block range for the fuzzer to iterate over. if runs is specified, the fuzzer will perform that number of runs for each block (not specifying is the same as `runs: 0`.  If `blocks` is specified as a mapping it may also take an `interval` field.
- `entrypoint` is a custom entrypoint to use for the scenario. only charts can use this entrypoint, deployments are not affected by this setting.

```yaml
blocks: [a..b]

blocks:
  range: [a..b]
  interval: 5

blocks:
  range: [a..] # block a to latest

blocks:
  range: [..b] # genesis to block b
```

A complete scenario could look like:

```yaml
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
    runs: 10
    blocks:
      range: [10000..]
      interval: 1000
    # bar intentionally not set so it would be fuzzed
    bindings:
      foo: ...
      bing: ...
```

### front matter charts

Any scenario can be charted as every concrete set of bindings can be treated as
a data point. For a single concrete set, a single data point is produced, for a
fuzzer or similar, a set of data points will be produced.

Front matter supports both scalar `metrics` and `plots` for charts with x/y axes.

#### Metrics

Metrics are list of single scalar values each rendered with additional text.

Required `metrics` keys per item:
- `label` The title of the metric
- `value` The stack path of the value to render for the metric

Optional `metrics` keys per item

- `description` Longer description of the item to render as text

Example:

```yaml
metrics:
  - label: Initial price
    value: 0.5
    description: The price this will be at launch
  - label: Max price
    value: 0.3
```

#### Plots

For ease of implementation we simply name the scenario to plot and then expose
observable plot config directly to the yaml. Plot is a mostly declarative DSL
from the creators of d3 https://observablehq.com/plot/features/plots

Required chart fields:

- `<key>` named plots, need at least 1 for a chart

Optional chart fields:

- `scenario` name of the scenario to draw data from, falls back to the name of the chart if not set
- `plots` mapping of charts to create

Required fields for each mapping under `plots`:

- `marks`: list of marks to draw in the plot according to the observable lib

Required fields for each mapping under `marks`:

- `type`: Can be `line`, `dot` or `recty` as per observable
- `options`: May be keys `x` and `y` that specify a stack path to a value to plot, or a `transform` (see examples below).

Example:

```yaml
charts:
  fuzzer:
    scenario: some-scenario.foo
    plots:
      Normalized amount vs. Denormalized amount:
        marks:
          - type: line
            options:
              x: 0.1.4
              y: 0.6
      Final price vs. oracle price:
        marks:
          - type: dot
            options:
              x: 0.1.2
              y: 0.7
```

Histogram example:
See https://observablehq.com/plot/transforms/bin#bin-options

```yaml
charts:
  histogram-example:
    scenario: some.foo
    plots:
      my-histogram:
        marks:
          - type: recty
            options:
              transform:
                type: binx
                content:
                  outputs:
                    y: "count"
                  options:
                    x: "0.5" # the stack item being binned
                    thresholds: 25 # the number of bins
```

Hexbin example:
See https://observablehq.com/plot/transforms/hexbin

```yaml
charts:
  plots:
    my-hexbin:
      marks:
      - type: dot
        options:
          transform:
              type: hexbin
              content:
                  outputs:
                      fill: count
                  options:
                      x: 0.0
                      y: 0.1
                      bin-width: 50
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
