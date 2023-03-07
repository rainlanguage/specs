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
Rain