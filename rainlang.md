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

### Stack first language

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

Rainlang doesn't know which values are which, but the author does (should). The
calling contract also specifies a minimum number of values (for example an order
book needs at least two values), and if the final stack is less than the minimum
this is a deploy time error that MUST revert the deployment.

The calling contract also specifies a maximum number of values that it can
handle usefully. This allows the interpreter to save gas on returning the entire
stack. For example, there may be dozens of internal values leading up to a final
single true/false, the interpreter should only return the singular final value if
the caller specifies it has no use for excess values.

There are no initial values on the initial stack when called by an external
contract. Initial values MAY be created by words that recursively evaluate logic
and handle substacks to emulate something like function calls.

## Trust, data and extensibility model

@TODO

- Stack
    - Deterministic size/OOB
    - Highwater/immutability
    - Runtime values
- Constants
    - Expression time values
- Context
    - Contextual values
        - Contract
        - `msg.sender`
        - Third party signed
- Extern
- Call/scope
- Loop

## Bytecode syntax

The bytecode syntax is everything that is mechanically reversible from literal
bytecode onchain _without additional metadata_.

For example, if some bytes onchain represents "get token balance" then we can
format the bytes as "get token balance" in Rainlang directly and represent it
exactly with the bytecode syntax.

An example of this from another language is [`.wat` files for wasm](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format),
the lisp syntax and the assembly bytecode can be parsed and formatted 1:1
bidirectionally by a machine, depending on the desired representation at any
moment.

### Stack representation

Rainlang expressions are a sequence of statements (like most languages) that
incrementally build up the final stack.

Each line is explicitly ended by `,`. It is an error to omit the trailing `,` for
each line.

> 🤔 Unlearning 🤔
>
> Commas DO NOT delimit list items or inputs to words or anything else in
> Rainlang. If you are coming from a more functional language like a lisp or
> nix (or even bash) this may be natural. If you're working with Javascript
> or similar a lot you may need to unlearn the habit of sprinkling commas through
> your code.
>
> Every character of syntax in Rainlang has ONE unambiguous meaning, which
> simplifies learning and implementing Rainlang.
>
> Commas as separators do NOT help computers understand code, they may even be
> nothing more than a mistake waiting to happen, as is often the case with
> trailing commas in otherwise valid JSON documents.

The end of a stack is denoted by a `;` _instead of_ a `,`.

A single expression MAY consist of many stacks and each stack usually consists of
multiple lines/statements.

Unlike most languages, the structure of the language mirrors the structure of
the stack and is explicitly denoted by `:`. Everything on the left hand side (LHS)
of the `:` is a stack item and everything on the right hand side (RHS) is logic
that will run to create the items on the left.

The default case is unnamed stack items which are denoted by `_`.

> 🤔 Unlearning 🤔
>
> The end of a line in Rainlang is denoted by a comma `,` rather than newline
> whitespace characters. This allows for compact "one liners" where a single line
> of text is interpreted as several lines of Rainlang code. For CLI interfaces,
> tweets, and other space constrained environments, this may be important to
> concisely represent a stack/concept without awkward escape characters etc.
>
> EOL whitespace characters can also change based on operating system and
> environment configuration. Removing potentially ambiguous syntactically
> significant whitespace from the definition of "a statement" or even "a line"
> allows for simpler implementations and more compact expressions.
>
> To state it another way, the EOL character of the medium the Rainlang
> expression is written in (context dependent whitespace defined by environment)
> is different to the EOL character that defines LHS/RHS pairings
> (unambiguous character defined by Rainlang).

Words on the RHS _reference_ some compiled solidity code in the interpreter. The
metadata about the compiled code informs Rainlang how each word reads and writes
to the stack, and other behaviours.

As words are references to real compiled Solidity functions they follow a similar
syntax to function invocations in other languages. The syntax is the name of the
word as prefix then parens `()` with optional inputs (explained more below).

Any symbol on the RHS that is not followed by parens is considered a literal and
DOES NOT directly reference a Solidity function. How literals are handled is
explained below.

Infix notation (or any other -ix) is NOT supported. There are no "operators" such
as `+` or `-` hardcoded into Rainlang. Even "basic" arithmatic is up to the
interpreter to define and provide. There are several reasons for this:

- There is no One True Addition (let alone any other operator), consider
  overflow, saturation, Solidity compiler optimisations, Open Zepplin SafeMath,
  etc. just to name a few variants of integer addition
- Each -fix notation that a language parser needs to support increases code
  complexity and therefore reduces the surface area that Rainlang can cover as a
  hosted language
- We don't want to require that ANY specific word exists
- Removing operators also removes the need to learn and memorize and chance to
  forget and mistake operator precedence rules

For example, a simple expression that creates a stack of 2 items, with the
current block number and timestamp could look like either of the following.


✅
```
_: now(),
_: block-number();
```

✅
```
_ _: now() block-number();
```

However it is NOT valid to have unbalanced excess items on the RHS. For example
the following is NOT valid even though the overall outputs and stack items are
balanced, the individual lines are not.

❌
```
_:,
_: now() block-number();
```

The reason for this is simply that it makes visual inspection more difficult.

The following IS valid if the stack is expected to be prepopulated with some
inputs. The first value on the LHS has no associated RHS because the memory
will be written by the caller of the expression rather than the expression
itself.

✅
```
_:,
_: now();
```

Prepopulated inputs are ONLY valid in the (potential) inital value region of the
stack, they cannot follow an RHS side item. The following is NOT valid.

❌
```rain
_: now(),
_:;
```

An identity stack simply sets aside the structure for inputs and has no RHS
items. A 2-item identity stack is valid as either of the following.

✅
```rain
_:,
_:;
```

✅
```rain
_ _:;
```

An empty stack is also valid, denoted simply as `:;`.

Rainlang MUST output `0` bytes for an empty stack, so that onchain contracts can cheaply and reliably skip evaluation of empty stacks entirely.

### Named LHS values

## Pragma

### StrictYaml front matter

@TODO

### DISpairing

@TODO

### License

@TODO

## Expression metadata

Anything that is NOT 1:1 with information that can be directly read from the
expression bytecode is considered metadata.

The definition of metadata is the complement to the definition of bytecode syntax
as explained above. While bytecode syntax allows a machine to move 1:1 between
bytecode and the Rainlang formatted equivalent, expression metadata is discarded
and not present in the bytecode. Metadata by its nature and by design is only
available as an overlay _alongside_ the bytecode syntax, a "nice to have" but
never strictly necessary to read or run Rainlang code.

The metadata spec is in the same respository as this spec. It is permissionlessly
extensible so any description of metadata in this spec for an expression should
be understood as only a minimum or guideline for how metadata _could_ work.

Metadata can include things like:

- Code comments
- Descriptive naming of LHS items
- Aliasing to make sense of or reduce boilerplate

Which are all great and important sensemaking tools but are all a double edged
sword.

ALL METADATA CAN LIE.

It is important to understand that metadata is a _claim_ made by a specific party
(not necessarily the expression author) that assigns a _concept_ to hard logic.
There are several ways this can go wrong

- Author misunderstands the code and mistakenly assigns the wrong concept (bug)
- Author wants reader to misunderstand the code and intentionally assigns wrong
  concept (attack)
- Author assigns valid concept but reader misunderstands (user error)

The first two we obviously want to avoid entirely. The last point is subtle. The
question in this case is whether the damage would have been more or less had the
metadata been completely omitted. Logically we can say that a larger number of
factual statements is more information is better, but is that how human brains
operate?

Walls of text DO NOT help improve smart contract security nor encourage usage.

Subjective concepts and explanations WILL BE manipulated by attackers to trick
victims.

> 🤔 Unlearning 🤔
>
> Metadata authors SHOULD NOT include information that
> - substantially duplicates the information available directly onchain from meta
>   that is hashed into contract bytecode directly, in this case the contract is
>   the best source of truth, not an individual author
> - explain how Rainlang or underlying smart contracts themselves work unless
>   there is real expectation of unexpected behaviour
>   (e.g. there's a bug and you're documenting a workaround)
> - Paraphrase or transliterate literal RHS logic as it is written verbatim
> - Anything else with very low signal to noise ratio
>
> The last point is important. Consider the COST OF LIES and the ATTENTION SPAN
> OF YOUR READER as well as the benefits of true statements. If your data adds
> little benefit if true, but potentially creates catastrophic misunderstanding
> if false or misread or distracting, is it worth the risk?
>
> Good metadata helps the reader analyse whether code achieves/implements the
> stated/implied goals so INTENT IS SIGNAL, transliteration and repetition is
> noise.

### Decoupled multi-social-metadata

All metadata in Rainlang is explicitly decoupled from both the expression
bytecode and expression author.

By design there can be many authors of metadata for a single expression. Once can
easily imagine a high stakes high value expression deployed by some author who
provided buggy metadata for otherwise valid code. A professional auditing firm
could flag the metadata and reissue corrected metadata under their own private
key.

As this all happens onchain according to the metadata spec, readers can use the
indexer and filtering algorithm of their choice to display metadata they __trust__
and discard everything else.

This lifts the utility of metadata from the bytecode (where it is useless) to the
social layer (where humans can share context p2p).

The simplest filtering algorithm is "keep only metadata issued onchain by the
author of the expression". This algorithm is functionally equivalent to
verification of a smart contract on etherscan by the deployer, but onchain
instead of tied to the etherscan system.

Every possible more complex algorithm such as "keep author metadata and also
auditors i trust and also everything i author" adds overlays on top of bytecode
much like restaurant reviews on Google Maps. People SHOULD put at least as much
effort into reading and writing smart contract reviews as they put into reading
and writing pizza and movie reviews.

> 🤔 Unlearning 🤔
>
> In web3 the author of an arbitrary smart contract is NOT your friend. Rainlang
> expressions may have simple syntax compared to Solidity or EVM bytecode, but
> they are still smart contracts.
>
> Unlike typical FOSS projects where the author can generally be assumed to be
> at least attempting to act in your best interest, the opposite is true for
> crypto. The vast, overwhelming majority of all smart contracts everywhere are
> buggy and/or malicious, sometimes even unintentionally. Many a ponzi scheme has
> launched and imploded on Ethereum seemingly by token designers who themselves
> had no idea that their "economy" was little more than a rube goldberg ponzi
> waiting for a bank run.
>
> All metadata associated with Rainlang expressions SHOULD be treated as highly
> suspect by default. __The author of a scam doesn't want you to notice their
> infinite mint function, or they want to convince you that it is harmless
> (it isn't).__ Even a seemingly harmless naming of an LHS item could be pulling
> a sleight-of-hand switcheroo in your brain, making you read a price as an
> amount or a block number as a timestamp, etc. etc.
>
> Overly complex/nested expressions that seem to need a lot of metadata to
> comprehend SHOULD be treated as suspicious.
>
> If the author of an expression hasn't done everything in their power to make
> the _bytecode syntax_ form of an expression comprehensible, this is a red flag
> already. Either they are a beginner and so watch out for bugs or they are an
> attacker and so watch out for scams.
>
> The intended workflow for a _reader_ of Rainlang is that they add their own
> metadata to raw formatted bytecode syntax, and then republish this onchain to
> contribute back to the social layer that helps keep everyone safe.

### Comments

`/* */` is the comment syntax used in Rainlang and popular in many other langs.

This is the most explicit commonly used syntax across all languages, it doesn't
require end of line detection to implement so reduces implementation complexity.

✅
```
/* Stack the current block time */
_: now(),
/**
 * Stack the current block number.
 * Is also the last value on the stack.
 */
_: block-timestamp();
```

Common commenting syntax that relies on EOL whitespace to terminate is NOT
supported, as `,` denotes the end of a line in Rainlang and this is a very common
character to want to use in a comment.

❌
```
// A "one line" comment like this would be awkward, it contains commas, so is it
// valid or not? easier to simply NOT support this type of entirely unnecessary
// thing at all.
_: now();
```

Inline comments are also NOT supported. The naive implementation of comment
metadata as assigned to a stack position on the LHS is also the correct
implementation of all possible comment metadata.

❌
```
_: erc20-balance(/* caller */ context<0 0>());
```

Note however that like all metadata COMMENTS CAN LIE. Usually comments are
included in scope for a Solidity smart contract audit because they can very
easily misrepresent the contract logic either accidentally or maliciously.
Rainlang comments fall into the same category and produce the same dangers.

❌
```
/* Stack the current block number. */
_: now();
```

Bytecode however never lies. A well behaved interpreter includes the name and
description of every word as _onchain metadata hashed into its bytecode_.

Either state the purpose/intent of code (if not obvious from context/convention)
or say nothing at all (if obvious from context/convention).

✅
```
_: now();
```

✅
```
/**
 * We need the current timestamp because the dinglehoppers assume that our token
 * spingleringers are never stale.
 */
_: now();
```

❌
```
/* Stack the current block timestamp. */
_: now();
```

## LHS name overrides

@TODO

## Aliases

@TODO

## Context interpretation

@TODO

## Opmeta

@TODO

## Contract meta

@TODO

## Constant meta

@TODO