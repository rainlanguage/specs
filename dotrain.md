# .rain files

## Design goals

### Primary goals

A portable, extensible and composable format for describing Rainlang fragments.

.rain files serve as a wrapper/container/medium for Rainlang to be shared and
audited simply in a permissionless and adversarial environment such as a public
blockchain.

.rain files MUST facilitate "overlay style" code audits where composition of
metadata itself can be shared at a social layer to create an audit trail of audit
trails. Without this it will be more difficult to build an explicit and
incremental model of trust, where participants earn/lose trust through producing
onchain artifacts that can be used to justify their good or bad reputation.

Composition MUST natively support iterative development practises such as
"this is exactly that with only these changes". While we don't want to reinvent
git, there are some aspects of git such as hashed "commits" that reference an
unambiguous _state_ which all existing audit trails and tooling relies upon. The
.rain composition MUST ALSO be amenable to standard development practises such as
git, the two systems should be complementary in nature.

Much as the bytecode syntax of Rainlang is 1:1 with bytecode, a .rain file is
1:1 with a cbor-seq of metadata as described by the metadata spec. That is to
say, given a compatible cbor-seq of metadata it is possible to recover an
equivalent .rain file to the one that produced the metadata, give or take some
incidental concerns such as insignificant whitespace.

It follows from the above that some properties of .rain are desirable:

- The ability to unambiguously import immutable data to support composition and
  auditability, mutable/ambiguous imports leads to "works on my machine"
  situations that tooling cannot mitigate without heroic effort (e.g. all of nix)
- Clear extension points for tooling to provide all imaginable computer assisted
  analysis of code, to give readers/consumers every possible edge over malicious
  authors (both bytecode authors and metadata authors).
- Location independent imports, such that tooling knows the integrity of an
  import regardless of where the content was discovered, much as blocks in a
  blockchain and data in IPFS are hashed so that they can be handled p2p.
- Native support for namespacing to support disambiguation of the remaining
  ambiguity that unavoidably arise in uncoordinated human-readable data
  (two different computer names with the same human name)
- The document structure should be as minimal as possible to facilitate stability
  and extensibility and allow for simple tooling implementations. We want to make
  as few decisions as possible that we might regret in the future, such as
  hardcoding meaning into syntax, where that meaning might foreseeably change in
  the future, or even be deprecated.

### Secondary goals

We SHOULD adopt common conventions because conventions often embody a kind of
evolutionary wisdom. We SHOULD NOT change things for the sake of change and the
_feeling_ of "innovation" but ONLY due to first principles reasoning that we
explicitly state such that others may disagree and discuss, and we may reassess
decisions in the light of new information.

We (the people trying to write a spec for .rain) accept that we have blind spots
in our understanding of absolutely every imaginable topic.

Given a choice between two seemingly arbitrary and equivalent options, in our
opinion, we should adopt the choice that seems most similar to comparable choices
made by other people, in our observation.

This gives us some ability to make better decisions without being aware of the
underlying reasoning.

An example of this is selection of some ascii character to represent some syntax,
often `#` is just as valid as `%` or several other characters, yet we MUST pick
something. To avoid analysis paralysis, it MAY be helpful to simply pin the
decision to something else commonly found in the wild.

However, often choices are NOT arbitrary and equivalent, so that's why convention
is a secondary goal. In any case where conventions takes us further from the
primary goals, we give ourselves permission to break from convention.

### Non goals

#### Familiarity

Familiarity of concepts and syntax for authors and readers relative to assumed
knowledge.

While it is always nice to shortcut the learning curve for something new by
referencing existing concepts this has potential severe pitfalls.

Firstly, almost all concepts in programming are heavily overloaded and come with
baggage. Even something as seemingly simple as "function" or "constant" means
subtly different things in every programming language. The _subtle_ differences
are exactly where bugs love to hide. If an author or a reader goes about .rain
or Rainlang and _assumes_ that familiar looking things work _exactly_ like they
do in `<insert language here>` then they will introduce a bug every time this
exactness breaks down. The obvious differences aren't half so dangerous as
everyone knows and loudly complains about them.

We SHOULD adopt a very high bar of how similar two concepts are before stating or
even implying that they _are_ the same concept. For example, Rainlang itself
stops short of defining any kind of "function" because the onchain behaviour
doesn't necessarily provide the kinds of guarantees a function might provide in
some (most) other languages. Another example is the `eager-if` word in the
reference interpreter that explicitly calls out the _subtle difference_ between
the "if" it defines and the alternative implementation of "if" that would be
lazy, and is probably taken as granted by many programmers who never really
thought too much about the tradeoffs of a lazy/eager "if".

> Everyone has an individual background. Someone may come from Python, someone
> else may come from Perl, and they may be surprised by different aspects of the
> language. Then they come up to me and say, "I was surprised by this feature of
> the language, so therefore Ruby violates the principle of least surprise."
> Wait. Wait. The principle of least surprise is not for _you_ only. The
> principle of least surprise means principle of least _my_ surprise. And it
> means the principle of least surprise after you learn Ruby very well. For
> example, I was a C++ programmer before I started designing Ruby. I programmed
> in C++ exclusively for two or three years. And after two years of C++
> programming, it still surprised me.
>
> - Yukihiro Matsumoto, creator of Ruby
> https://www.artima.com/articles/the-philosophy-of-ruby

Secondly, if we make "familiarity" an explicit goal then we have to also define
"for who?", and this definition will inevitably reference some existing tool or
language that embodies the concepts we want people to magically already be
familiar with somehow, like "JavaScript devs" or "Spreadsheet jockeys".

The problem with tighly coupling Rain systems to other systems is that Rain
operates in a very specific context that comes with severe restrictions due to
the nature of the operating environment. It's very likely that we _can't_ exactly
implement a feature commonly found in X, because that feature relies on compute
or threat models that Rain don't have.

Further, it couples Rain's reach to a subset of the reach of the reference
system. It's very different to state "the target audience is someone who _could_
understand spreadsheets" than "the target audience is someone who _does_ know
excel". The former is a statement about broad technical aptitude and the second
is a statement about a specific skillset.

#### No-tooling

While Rainlang has a design goal to be simply and visually readable and
writeable in entirety by a relatively (to assembly/Solidity) non technical user,
.rain documents DO NOT have that goal.

.rain files explicitly import/reference metadata for tooling such as information
about how every word in an interpreter moves a stack at runtime. The default
state for an empty .rain file already contains too much information for human
comprehension the moment it binds to a DISpair.

The goals of .rain documents are intented as stated _to allow tooling to exist_,
whether a classic IDE, or some more specialised interface.

Notably the details of how to produce and manage hashes of imports
(discussed below) are never going to be produced by hand by a human.

Similarly the composition of Rainlang fragments into a Rainlang document involves
several steps that cannot be done mentally by a human:

- Checking hashes
- Name bindings and namespaces
- Expansion of fragments into a document
- Applying metadata such as opmeta to the final document to build a deployable
  artifact

Rainlang has the goal that bytecode syntax can be recovered to a legible document
from the DISpair alone but this doesn't mean the original .rain document tree
is recoverable directly from deployed binary.

However, it SHOULD be trivial for an author (or anyone else) to provide:

- Deployed expression
- DISpair
- A single hash

And unambiguously find full metadata that can be recovered to a .rain document.

Of course, tooling will be needed to perform that recovery.

## Encoding

For humans to write code they need a medium in which to write the code in.

Typically code is written in files/documents that are binary data representing
strings of printable characters and whitespace for humans to read and write.

There's the usual concerns such as

- Character encoding: ascii, utf8, utf16, et.
- Compression algorithms over binary data
- Handling whitespace such as end of line characters

Once we get past these and have higher level abstractions over the binary data
like

- Character
- End of line
- Indentation

Then we can meaningfully talk about the structure of a .rain file as a human
experiences it. Moving from binary data to a human legible file is outside the
scope of this spec, and typically should be considered an implementation detail.
For example a utf8 encoded .rain document on a file system and a utf16 javascript
string containing the same document should be considered equivalent and valid.

## Allowed characters

ONLY ascii printable and whitespace characters are allowed in .rain files.

As a regex this is `[\s -~]`.

utf8 is backwards compatible with ascii so the codepoints are the same in either
encoding.

## Additional characters

Beyond the basic Rainlang fragment syntax, .rain documents introduce several new
characters/syntax described below:

- `#` binds a name to a fragment
- `!` is a fragment error
- `.` delimits paths in a namespace
- `'` quotes names
- `@` imports .rain/metadata

At the time of writing none of these characters are in use in Rainlang proper but
even if they were their usage is unlikely to cause ambiguity with .rain and
Rainlang fragments.

- `#` and `@` only appear at the root of a .rain document
- `!` only appears immediately following `#`
- `.` and `'` MAY be found within Rainlang fragments but are unambiguous
  extensions of names/words in Rainlang provided they are not included within
  names/words with some conflicting meaning in a future version of Rainlang

## Self describing

@TODO

Perhaps we want .rain files to be self describing.

This would be some syntax that tooling can use to autodetect with high confidence
that the document is a .rain document.

This process should be O(1) efficient, not relying on tooling to first attempt to
parse an arbitrary file found in the wild to then decide if it is indeed .rain.

For binary data this is relatively easy, we can generate a magic number and start
the binary with that magic number. Tooling can check the first few bytes in O(1)
and discard any binary wholesale that isn't a match.

Serialization formats often self describe as their problem domain is representing
data in a portable context. This typically looks like some kind of versioning
perhaps with a link to the relevant specification, see `<!DOCTYPE>` tag in html
for an example.

Source code is a little messier, for whatever reason it is common for the source
code to NOT version or describe itself and rely on the compiler or interpreter's
own version and configuration to control the handling of the source code.

Solidity has `pragma solidity ^0.5.2;` which is a lot better than nothing but
isn't formally (as far as I know) intended for tooling to _discover_ that a file
is Solidity. It is an instruction to the `solc` compiler to check _compatibility_
once the file type is already known. The fact that a solidity `pragma` has the
word `solidity` in it is our only clue, and we can't read it in O(1) as there is
no requirement that the document starts with this `pragma` statement.

We are dealing with printable ascii in a .rain file so we can't have a real magic
number, unless we do something that is not noob friendly.

## Rainlang fragment

Rainlang fragments are analogous to XML fragments, they are part of a valid
Rain AST and can be validated e.g. `<1 2>` is a valid fragment but `<1(2` is not.

A Rainlang document, much like an XML document, is one that could be deployed to
an expression deployer and pass integrity checks according to the associated
opmeta. (Rainlang document != .rain document)

This implies that the difference between a document and a fragment is somewhat
coupled to metadata imported alongside the .rain file.

## Relationship to metadata

A .rain file itself is NOT metadata, but it can be round tripped to metadata and
back in a mechanical way. This is analogous to
[.wasm and .wat files](https://developer.mozilla.org/en-US/docs/WebAssembly/Text_format_to_wasm).

## Structure

A .rain file consists entirely of

- named rainlang fragments
- import statements

These both introduce syntax not found in Rainlang proper, i.e. it is NOT valid to
include either of these within a Rainlang fragment.

### Named fragments

Named fragments take the form of `#<name><whitespace><fragment>`.

We can say the fragment is _lexically bound_ to the name in the  current .rain
document, which acts as as anonymous top level _namespace_.

Namespaces aren't named until they are imported with a name (discussed below).

The top level of the .rain file is only named fragments.

The choice of `#` is inspired by markdown's header syntax, EDN's tagging, Rust's
raw literals, etc. It is supposed to convey some kind of feeling of "here is some
domain specific content and it has a name".

Anonymous fragments are NOT SUPPORTED.

An example consisting of two fragments, `slow-three` and `multiline-example`.

```✅
#slow-three add(1 3)
#multiline-example
a _: slow-three 3
_: mul(a 5)
```

#### Named fragments must be valid

Tooling MUST reject any .rain documents that contain _any_ invalid fragments
even if the invalid fragments are not ultimately bound to a Rainlang document.

#### Bound and unbound names within fragments

The previous example includes `slow-three` as both a fragment name and appearing
within a fragment.

This is binding the name _in_ one fragment to the name _of_ another fragment.

```✅
#a 1
#b a
```

Both `a` and `b` expand to `1` when processed by tooling.

Circular references are disallowed. Tooling SHOULD reject any circular bindings
including transitive circular references.

```❌
#a b
#b a
```

```❌
#a b
#b c
#c a
```

Unbound names are NOT allowed on the RHS although are allowed on the LHS as per
standard Rainlang rules about input values on the stack for sub-calls.

Template/macro-like logic can be achieved by first binding all the names within
a namespace then _explicitly rebinding_ names as desired upon import (see below).

Fragments in .rain files that are intended to be a base for reuse should either
set sane defaults that can be optionally rebound, or bind to an error fragment
(discussed below) so that the fragment cannot be built into a document without a
mandatory rebind.

```❌
#a b
```

#### Error fragments

Names can point to error fragments. This is possible in a .rain document but will
error if still present in a Rainlang document before deployment.

Error fragment syntax is `!<message>`. Tooling SHOULD provide a generic error
message that includes the name bound to the error if `<message>` is an empty
string.

The only way to build a Rainlang document from a tree of fragments that include
an error fragment is to rebind the error fragment out of the tree at the import
level (described below).

This allows for template/macro like fragments to specify mandatory rebindings
without relying on an error-prone system such as implicitly unbound names
(which could be unbound accidentally due to a mere typo).

This example will error if the tooling attempts to build a document from either
`#a` or `#b`.

```✅
#a !must bind a to calculate rate of change
#b mul(sub(1000000 now()) a)
```

The error message can be multiline, it continues until the next # character that
signifies a subsequent name/fragment binding.

```✅
#a !
`a` must be bound so that we can calculate a rate of change in seconds.
Take care not to provide millis such as is commonly found in unix timestamps.
If `a` is `0` then nothing will happen as the rate of change is `0`.

#b mul(sub(1000000 now()) a)
```

Error fragments MUST be the root of a fragment, i.e. the first character of an
error fragment MUST be `!`.

```❌
#a <!>
```

```❌
#a foo(!)
```

#### Shadowing is disallowed

Shadowing is NOT allowed.

```❌
#a 1
#a 2
```

Although our name bindings are lexically bound within a .rain document we adopt
the same rule from Rainlang that shadowing is disallowed.

Every name binding MUST be unique and collisions MUST be handled by either
renaming in situ or rebinding upon import.

#### Name bindings are unordered

As collisions, shadowing and unbound names are all disallowed the names within
a .rain file behave like a simple unordered key/value store. Namespaces can be
modelled as a value that is itself a key/value store, recursively.

Tooling MAY implement name bindings as a nested key/value store, and MAY
implement sorted or unsorted bindings internally.

```✅
#b a
#a 1
```

#### Using/quoting names directly

The `'` character can be used to quote names.

This is roughly adopted from clojure where `'` means "read without evaluating"
which in a lisp means treating your code as data, to be used elsewhere in code.
We're doing something much less featureful than that, but "quote" is fair enough
to capture the concept of the name itself rather than the thing it names.

This has two key uses:

- Renaming (rather than rebinding) on import (see below)
- Calculating the index of a call-like word in a fragment so that tooling can
  construct indexes when building a Rainlang document from a .rain document

Renaming is ONLY possible during import and so is described below.

An example of a fairly standard `call` with operand args `call<index outputs>()`.

```✅
#add-five
i:;
_: add(i 5)

#main
/* outputs 10 */
_: call<'add-five 1>(5)
```

From the example we can see that tooling will need to

- Include any quoted fragments as sources that can be called
- Substitute the quoted names found in fragments with the source index in the
  final artifact

#### Requires binding to expression to become a Rainlang document

If a .rain file only consists of named expressions it has no natural/implied
entrypoint.

Deployed expressions need entrypoints to run, e.g. "calculate order" or
"handle transfer", which have historically been handled implicitly by index only.
E.g. the calling contract runs index 0 upon transfer so therefore the rainlang
representation of the expression MUST put transfer logic at index 0.

While it's very direct and simple to do things this way, there are problems

- It becomes difficult to compose expressions without code stutter/tautology such
  as "this expression is that expression + this KYC requirement"
- It introduces a split between "header" and "body" of a .rain file if we want
  to support including fragments that aren't intended to be the final deployed
  bytecode
- It couples the structure of a .rain file to an implied but unspecified
  deployment that hasn't even happened yet during the drafting phase

Moving forward we want tooling to bind the fragments to a _specific_ DISpair
such that we can _verify and deploy_ the Rainlang fragment as a full Rainlang
document.

It's highly likely that the tooling-binding process takes strong clues from
metadata. For example, an orderbook contract may emit "contract metadata" that
states there is an entrypoint called "calculate-order" and so the tooling will
look for a fragment named `#calculate-order` in the .rain file to bind to the
indexed entrypoint in the final evaluable config to deploy onchain.

### Namespaces

Namespaces are just branches on a tree of names.

We use `.` to delimit names as it often brings a kind of "path", "drilling down"
or "chaining" feeling in the wild.

There is no concern that `.` will be mistaken for part of a Rainlang fragment as
it ONLY means "namespace" in the context of a `#` prefixed name, and there are
no fragments where `#` is a valid character. If some future iteration of Rainlang
includes `#` as a valid character then we may need to revisit concerns around `.`
and potentially being misinterpreted e.g. as a decimal number etc.

Namespacing applies equally to words as words are valid fragments, even though
they are leaves of the AST. This is especially important for extern contracts
where the implementation of a word MAY be on an onchain contract that is not the
root interpreter, and so we MAY need to disambiguate between
e.g. `add` and `my-extern.add`.

Namespaces are the standard way to handle name collisions in basically every
language and data structure.

> Namespaces are one honking great idea -- let's do more of those!
> The Zen of Python

```❌
#a 1
#a 2
```

```✅
#a 1
#b.a 2
```

### Imports

Imports take the form of `@<namespace><whitespace><hash>` OR `@<hash>`.

If the namespace is ommitted that means "import into the current namespace".

`.` also means "current namespace" where it needs to, or benefits from, being
made explicit, much like "current directory" in a file system.

Importing is a recursive process and tooling MUST build the entire tree according
to the imports but also MUST reject .rain documents that consume too many
resources to build inn their entirety.
e.g. due to a malicious structure that is very deep/broad to cheaply recurse.

Import hashes reference _metadata_ so we can colloquially say
"import a .rain file" but technically we mean "import the metadata the .rain file
maps to when compiled".

As importing references _metadata_ it can have two possible effects:

- Populating namespaces with named fragments to _build_ metadata that _generates_
  bytecode when bound to a deployment
- Making metadata available to tooling to _describe_ metadata and/or bytecode

A notable example of a single import that has both effects is op metadata.

Op metadata

- makes new words available to _build_ rainlang fragments
- _describes_ how each word functions so that tooling can perform static analysis
  like balancing LHS and RHS with inputs and outputs, etc.

#### Imports respect namespaces with explicit rebinding

Import statements respect the local lexical binding of all namespaces that are
imported (until/unless explicitly dynamically bound by the import).

Any namespace specified in an import will be prepended to all the names found in
the imported document.

For example, consider two .rain documents

A.rain

```✅
#pi
/* pi is roughly 3 */
3
```

B.rain

```✅
@math 0xdeadbeef...
#two-pies
add(math.pi math.pi)
```

Assuming that `0xdeadbeef...` is the hash of the metadata produced by compiling
A then B expands to

```✅
#math.pi
/* pi is roughly 3 */
3
#two-pies
add(math.pi math.pi)
```

#### Rebinding names

Imports allow explicit
[dynamic (re)binding](emacswiki.org/emacs/DynamicBindingVsLexicalBinding) of
values upon import.

Consider the following .rain files.

X.rain

```✅
#pi
3
#two-pies
add(pi pi)
```

Y.rain

```✅
#pi
4
@x 0x..
```

This will expand to

```✅
#pi
4
#x.pi
3
#x.two-pies
add(x.pi x.pi)
```

The author of the root namespace may want to use their version of `pi` within
the `two-pies` fragment that they imported.

To support this the import itself can dynamically rebind anything within the
content of its import.

The syntax for rebinding is a simple whitespace delimited list of key/value
pairings where the key is the name being rebound and the value is the new binding
within the current scope.

The key is relative to the namespace specified after `@` and may optionally begin
with a `.` character.

Let's consider some alternate versions of Y that rebind `x.pi` to expand to `4`,
or `pi` in the root namespace which is bound to `4`, rather than the original
`3` value.

```✅
#pi
4
@x 0x..
  pi 4
```

```✅
#pi
4
@x 0x..
  pi pi
```

```✅
#pi
4
@x 0x..
  .pi pi
```

#### Renaming names

In some edge cases there may be an unavoidable naming collisions, such as when
an imported word from an interpreter pollutes a previously clean namespace
(described below).

Names can be renamed by _rebinding a quoted name_.

Continuing our example from above

```✅
#pi
4
@x 0x..
  'pi phi
```

Expands to

```✅
#pi
4
#x.phi
3
#x.two-pies
add(x.phi x.phi)
```

Note that both the name is renamed (of course) and also all the occurances of the
name in fragments are renamed. As the goal is to maintain consistency while
resolving unavoidable naming collisions, this is necessary to preserve
original behaviours.

#### Importing words

Words are also importable and MUST be imported somewhere in the tree of imports
that is specified upon building a Rainlang document from .rain files and Rainlang
fragments.

Words are a special case as they are the ultimate leaves of fragments and their
definition is tightly coupled to the specific interpreter smart contract that
will run the compiled bytecode from a Rainlang document.

There are two places that words can be evaluated at runtime onchain:

- Internally to the interpreter that runs `eval` over the expression bytecode
- Externally to the interpreter on an "extern" contract dispatching a single word

It follows that a Rainlang document can only include a _singleton_ of _internal_
interpreter words, but MAY consist of arbitrarily many _externs_.

As the internal interpreter words are a singleton they can be referenced globally
in every namespace. E.g. `add` is just as valid as `a.add` and `b.add`.

This has two key effects

- Every namespace can write unprefixed words, which allows much more concise code
- No namespace can accidentally shadow an interpreter word, forcing an explicit
  _renaming_ anywhere there might be a collision

Note that _words cannot be renamed_ so by definition any quoted rename
(discussed above) will modify a name and NOT a word.

_External_ words do NOT populate every namespace simultaneously, rather they
follow the same namespacing conventions as names. Authors SHOULD import external
word's metadata underneath a dedicated namespace to avoid all collisions and
make it explicit and obvious where external words are used. If nothing else,
external words are more gas intensive than internal words, and are worth calling
out for the reader.

#### Tooling compatibility

.rain itself makes NO ASSUMPTIONS about what will be on the other side of an `@`
import.

At a minimum it SHOULD follow the metadata cbor conventions that would furnish
the data with a magic number and enough information about the encoding etc. to
allow tooling to parse the data somehow.

Tooling SHOULD outright reject or strongly warn users if a tree of imports
contains _any_ metadata that it does not support.

There is a natural balance/tension between permissionless extension of metadata
backed by conventions and magic numbers, and the requisite tooling support and
maintenance overhead to meaningfully apply this ocean of optionality.

If .rain forms an opinion on the interpretation of metadata then it will be
endlessly chasing new forms of metadata and tooling, naturally creating an
undesirable rift between the "official" metadata and "everything else".

What we want is to encourage an ecosystem of tooling and metadata where adoption
is based on "most useful" NOT "most official".