# Rainlang

## Design goals

Rainlang is a language that can be interpreted on a blockchain.

The interpreter implementation is itself a smart contract, so the implementation
of Rainlang is a smart contract.

### Learning curve

Rainlang is a new language so inevitably new concepts must be learned and some
existing habits/expectations unlearned.

The learning curve must be comparable to the learning curve for other languages
commonly used in a financial setting.

The learning curve has to include the full end to end experience of working with
Rainlang, including deployment. If an expression author needs a compiler or other
tool to deploy a smart contract, the learning curve of the tool is included in
the learning curve of Rainlang.

The learning curve includes everything required to deploy a production grade,
unhackable smart contract. Adding two numbers together doesn't cut it, we need
authors to be confident that they can sleep at night after deploying their code
to a public forum full of motivated, ruthless, well funded and intelligent
attackers.

Good examples

- Excel/Spreadsheets: Spreadsheets have over a billion monthly users globally,
  "every adult" either knows excel or someone who does (maybe their accountant).
  A few good habits that are easy to learn are all that is required to be fairly
  confident that all the values in a spreadsheet are correct.
- Pine Script/Trading View: Much more niche but still a language intended for
  broad adoption by traders, their social features show many strategies and
  charts written by the general public that seem relatively bug free
- Scientific calculators: The humble calculator (does anyone use these anymore??)
  maintains enough internal memory, buttons and screen real estate to be
  considered "a language" of sorts, the sheer simplicity and limited scope makes
  it hard for bugs to hide
- SQL: Databases are pretty much just the spreadsheet's big brother. The most
  disastrous bugs (injection attacks) are typically on the client side, the
  language itself is usually relatively easy to take from idea to production

### Decentralised extensibility

If someone wants to write their own interpreter with their own words, and it
follows the basic form and guarantees of Rainlang, then it is valid Rainlang.

Much like Solidity `.sol` files include a pragma that specifies the compiler
version, a Rainlang `.rain` file can specify any interpreter.

There is an associated set of interfaces and behaviours that a smart contract
needs to implement to be Rainlang compatible (largely out of scope for this doc).

Rainlang needs to be designed so that many different interpreter implementations
can happily coexist without being carbon copies of each other.

This also implies that the code complexity of implementing Rainlang must be low
enough that it could be handled by a university student or similar, not requiring
a well funded team many years and endless maintenance cycles to implement

- The rainlang parser and formatter that round trips strings to onchain bytes
- An interpreter as a smart contract
- A contract that deploys and uses expressions for its internal logic
- Implementations of alternative/additional metadata formats describing the above

### Pragmatic metaprogramming

Rainlang needs to be amenable to some degree of autogeneration e.g. by a widget
in a website, and also constraints based analysis.

We say "some degree" because a full "code is data" macro style system such as
found in lisp or extensive programmability such as proc macros in Rust is well
out of scope. If Rainlang itself needed to be able to program itself then it
would likely undermine the other design goals.

The intent here is that the syntax of Rainlang is minimal and regular enough that
the "dumb" implementation of code to manipulate/analyse it out of band
(e.g. string interpolation or simple pattern matching) is also the correct one.

### Gas

The bytecode of the language must be gas efficient in several ways:

- Small codesize, deployable as a real onchain contract using SSTORE2 or
  potentially even as [EIP3540](https://eips.ethereum.org/EIPS/eip-3540) data
  contracts if that becomes the standard
- Minimal compilation and deployment overhead, overall cost should be similar to
  configuring and interacting with existing major defi protocols
- Comparable runtime cost to equivalent hand-coded Solidity, if not assembly

### Safety

For a well formed runtime environment an expression should be immune to several
key classes of vulnerability by construction.

- All expressions are the equivalent of `view`, guaranteed by the interpreter
- The contract that drives the interpreter is crafted to disallow reentrancy
  that could exploit the intent of an expression
- All state changes are local to the expression (in memory) during its execution
  and explicitly persisted changes are committed to storage in bulk after the
  expression runs
- Expressions are bound to a collection of immutable contracts upon creation,
  the runtime environment has no admin keys, self destruction or upgradeable
  functionality. New functionality is achieved by the expression author removing
  (if possible) their expression (not the underlying contracts) and writing a new
  expression for a new interpreter.
- The compile time bytecode and runtime stack structure are both explicit to
  a visual inspection of Rainlang expressions
- A legible Rainlang expression can be constructed directly from onchain bytecode
  without any offchain inputs from the author
- The structure of the stack is known at deploy time, out of bounds access is a
  deploy time error enforced onchain, as are other opcode specific integrity
  checks
- Any buggy expressions only (directly) negatively impact the expression author
  and those who explicitly interact with it
- Security sensitive commentary can be attached to expressions onchain to allow
  social layer defenses against bad expressions and surfacing good expressions.
- A consistent worldview is always presented within the evaluation of an
  expression, e.g. querying a token balance returns the same value in all
  possible positions within an expression, making caching/reusing expensive
  values trivial
- Words should implement runtime safety as appropriate by default without any
  special handling by the expression author, addition should error on overflow,
  oracles should error if their values are too stale, etc.
- The runtime environment should provide an escape hatch for trapped funds in the
  case of a buggy expression. Direct refunds to participants, a 1:1 unwinding of
  funds, should be preferred over any priviledge escalation. The unwinding must
  not be sensitive to expression logic, so could be e.g. a hardcoded timeout
  after "a long time" but "not too long".
- Trust relationships should be explicit and upfront, counterparties to an
  expression need to know who they are dealing with and what control each other
  party has over the operation of the system. Obvious scams such as admin mint
  functions should be obvious to anyone who knows Rainlang.

### Portability

Currently Rainlang is implemented in Javascript and the underlying smart
contracts are implemented in Solidity for the EVM.

Ultimately these are mere implementation details. It should be possible to
implement both Rainlang and the interpreter in Rust (for example) and deploy via.
wasm to the browser and blockchain simultaneously, etc.

In this way Rainlang can be thought of as a "hosted language" as popularised by
Clojure, with major concurrent implementations in java, javascript and .NET.

## Stack first

Rainlang is not only a stack language but its raison d'etre is to build a stack
as its final output.

The most obvious implementation (if you're familiar with stacks) is a single
Solidity `uint256[]` array.

The final stack output is returned to the caller of `eval` and the usage of the
stack is entirely up to the caller. Examples include popping the final two values
as amount/price on an order book, scanning the list until a sentinel is found to
build a sub-array, taking a single value to act as a bool to gate access, etc.

Rainlang itself doesn't know anything about what the final stack does or is for,
it merely focusses on building the stack.

Conceptually each Rainlang stack can be divided into 3 zones. Any of the zones
can have 0+ values (i.e. may be empty).

`[ input/initial values ] [ internal values] [ output/used values ]`

Rainlang doesn't know which values are which, but the author does (should).

### LHS then RHS

