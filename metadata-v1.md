# Metadata V1 design

## Design goals

- Reinvent as little as possible, lean on existing standards
- Gas efficient enough to at least emit data as onchain events and
  potentially even some scope for onchain reads
- Rich enough for complex offchain data processing
- Permissionlessly extensible as new encodings, content types, meta types,
  serializations are invented and adopted
- Portable enough to not be coupled to a specific event type, could be
  emitted anywhere by any contract, saved in a file, etc. and still be
  standalone parseable
- Native support of unambiguous concatenations of a sequence of metas within
  a single sequence of bytes
- Opt-in tooling support for various encodings and meta/content types
- Support a graph structure between meta such that meta can be about other
  meta, e.g. to support community driven translations of content between
  different human languages such as English and German
- Tooling can efficiently O(1) drop/ignore meta that it does not need or
  support decoding and parsing for

# V1 Design.

Meta is binary data, the body of `bytes` in solidity but whatever binary
representation is appropriate elsewhere. E.g. the leading `uint256` length
from Solidity `bytes` is NOT considered part of the meta, nor are any other
representation concerns such as ABI encoding for inclusion in some onchain
event etc.

## Self describing

Meta is a self describing document in three ways:

- The first 8 bytes MUST be the rain meta magic number
- The list of metas is represented as an RFC8742 cbor-seq of maps where each
  map includes standard HTTP representation headers `Content-Type`,
  `Content-Encoding`, and optionally `Content-Length`.
  https://www.rfc-editor.org/rfc/rfc8742.html
  https://developer.mozilla.org/en-US/docs/Glossary/Representation_header
- The cbor-seq items also specify a magic number that can be read by tooling in
  O(1) as a signal of intent for the payload _without_ needing to first parse
  (and possibly decompress) the entire payload

### Magic numbers

Magic numbers facilitate portability allowing tooling to identify with high
confidence that some data is intended to be interpreted in some way regardless of
where it may be found in the wild and what anyone else on the planet might be
doing with their data.

Spiritually similar to UUIDs and cryptographic hashes the basic intuition is
that if some value is sufficiently unpredictable in the face of someone
deliberately trying to brute force it, then it is even more unlikely to collide
with people merely accidentally building similar systems concurrently.

It's important to understand that simply hashing some value doesn't provide
any collision resistance at all relative to the raw strings/data if it's likely
that other people will be writing similar code with the same hashing algorithm.
In the context of Solidity this is very likely as `keccak256` is enshrined in an
onchain EVM opcode.

The [pidgeonhole principle](https://en.wikipedia.org/wiki/Pigeonhole_principle)
tells us that any hashing scheme actually makes collisions _more_ likely than
merely preserving all possible unique inputs.
The [birthday problem](https://en.wikipedia.org/wiki/Birthday_problem) tells us
that the chance of there being _any_ collision within a set of randomly
distributed values is far higher than intuitively seems possible, e.g. there is
a 50% that two people will share the same birthday with only 23 people sampled.

So why does anyone ever hash data to produce magic numbers if the original data
was _more_ collision resistant?

- Hashes reliably fill a known number of bytes e.g. keccak256 -> 32 bytes always.
- Arbitrarily truncating/sampling a high quality hash to fewer bytes doesn't
  introduce statistical biases, unlike e.g. UUIDs that have fixed values at
  certain bits to denote UUID version etc.

In this case however, there is no reason to try to _derive_ some magic number at
all (c.f. ERC165 4 byte interface function signatures). We get the best possible
guarantee of collision resistance by simply generating random bytes of the
desired size and publishing it alongside the standard. This isn't a cryptographic
primitive, there's nothing to "backdoor", it's a mere convention that tells a
decoder at a glance that some data MAY be relevant.

By starting the binary form of the magic number with `0xFF` we can ensure it is
NOT a valid utf-8 byte sequence, which MAY futher aid arbitrary decoders to drop
rain metadata unless it intentionaly wants to handle it. However, it is
NOT RECOMMENDED to point decoders at user data on a decentralised ledger if that
data is not self describing as this has a high probability of misinterpreting
the intent of the sender.

The magic number for the first 8 bytes of rain meta is

```
0xff0a89c674ee7874
```

Which was obtained by running

```
openssl rand -hex 8
```

Then manually setting the leading bytes to `0xff`.

Don't believe me? Doesn't matter. It's a convention, not a security guarantee.

A quick google shows zero results, so it's unlikely to collide with any other
value used elsewhere.

Tooling that wishes to read meta MUST discard/ignore all binary data that
does not begin with the magic number.

#### Trustlessness and extensibility of magic numbers

There is no registry of magic numbers for rain meta. If you want to prefix valid
meta with some other magic number when you encode it, go for it, but very likely
no decoder will accept it.

The degree to which any magic number or other convention in this document is
"official" or enforceable essentially comes down to

- The tooling that the community uses needs _some_
  [schelling point](https://en.wikipedia.org/wiki/Focal_point_(game_theory))
  out of pure practicality, otherwise opportunities to compose efforts will be
  lost in the sea of all possible binary data strings
- If there's a deliberate or accidental collision then tooling will tend to break
  in subtle ways. Devs _hate_ it when their code breaks in subtle ways, so tool
  maintainers will probably outright reject one or both encodings by _social_
  consensus (e.g. they will talk to each other and figure it out, or they will
  simply drop anything they don't personally want to support)

CBOR and IANA already maintain enough centralised structures for lightweight and
flexible encoders and decoders to be written (and have already been written in
every major language). We should focus on curation of 8-byte magic numbers at the
social layer to allow actual applications to be written on a globally shared data
repository (the blockchain).

Here is a table of magic numbers that the tooling maintained by the authors of
this document are already handling.

| Number             | Interpretation                            |
| ---                | ---                                       |
| 0xff0a89c674ee7874 | Prefixes every rain meta document         |
| 0xffe5ffb4a3ff2cde | Solidity ABIv2                            |
| 0xffe5282f43e495b4 | Ops meta v1                               |
| 0xffc21bbf86cc199b | Contract meta v1                          |
| 0xffe9e3a02ca8e235 | Authoring meta v1                         |
| 0xff1c198cec3b48a7 | Rainlang v1                               |
| 0xffdac2f2f37be894 | Dotrain v1                                |
| 0xffdb988a8cd04d32 | ExpressionDeployerV2 bytecode v1          |
| 0xff13109e41336ff2 | Rainlang source meta v1

This document will be updated as new numbers become known but also feel free to
build systems and applications with your own numbers and interpretations.

### RFC8742 CBOR sequence (uncompressed)

A CBOR sequence simply concatenates the raw binary bytes of each CBOR item
with no additional separators or other bytes. This is possible because each
CBOR item is unambiguous either because it has an explicit length or an
indefinite length followed by an explicit "break" byte. RFC8742 explains that
this property allows CBOR data to be directly concatentated within a binary
file, unlike e.g. JSON that is ambiguous and so requires additional
separators and processing logic to handle sequences correctly.

Typical usage will result in binary data that has a high probability of being
comparable or even _larger_ size if compressed.

- There is a high likelihood that the binary payload of each metadata item itself
  is already compressed data. Compressing compressed data usually either has no
  benefit or _increases_ the size of each payload. If the encoder of the payload
  feels that compression would help they SHOULD apply the compression directly
  and specify the compression algorithm in the encoding header (see below).
- Through the use of domain specific aliasing of common keys and values
  (see below), much of the potential benefits of a general purpose compression
  algorithm are already realised by convention and so compression is likely to
  _increase_ the overall size of the CBOR sequence.

An additional consideration is that decompression algorithms may be too complex
and/or memory intensive to be supported onchain. If we were to put the CBOR
sequence behind compression it would make it impossible for smart contracts to
inspect user-provided data and sanity check that compatible/complete metadata has
been provided. CBOR was explicitly designed for low code complexity and resource
consumption in decoding so we should preserve/forward this design goal downstream
to consumers of our metadata.

For these reasons the CBOR sequence itself MUST NOT be compressed although
individual binary payloads SHOULD be compressed if it allows significant gas
cost efficiencies.

### HTTP representation headers

A subset of HTTP representation headers are supported, specifically those
that are describe content rather than behaviour. E.g. `Content-Location` is
NOT supported because it describes how to retrieve data rather than how to
interpret it. Meta headers ONLY concern themselves with describing the
content itself as the inteded use is for the data to be made available either
onchain or from some other p2p system such as IPFS where the practicalities
of the system typically handle data integrity and availability concerns.
E.g. There is no danger that a CBOR seq will be truncated accidentally due to
some network failure as the block or IPFS hash itself provides cryptographic
proof of the data integrity. Similarly `Content-Length` is NOT supported as the
CBOR sequence itself unambiguously specifies its own member items and the lengths
of each payload.

#### Header name aliases (CBOR map keys)

As the HTTP header names are each quite long (12-16 bytes each) and we DO NOT
compress the CBOR sequence data (see above), we instead alias each supported
header to a single byte integer (range 0-255). Actually, for very small integers
CBOR encodes both the size and value of the integer into a single byte, so the
we can save a full ~30 bytes per CBOR item just by aliasing the keys.

The following table describes the canonical index/alias integers that tooling
MUST implement. I.e. the HTTP string representations of the keys such as
`'Content-Encoding'` are NOT supported as this would allow encoders to produce
data that decoders are explicitly trying to avoid the complexity of reading.
E.g. consider the runtime and code size deployment gas costs of supporting BOTH
strings and integer keys onchain for redundant information.

| Index | Body                         |
| ---   | ---                          |
| 0     | Payload as CBOR binary data  |
| 1     | Magic number                 |
| 2     | Alias for `Content-Type`     |
| 3     | Alias for `Content-Encoding` |
| 4     | Alias for `Content-Language` |

Indexes 0-1 inclusive are MANDATORY and any CBOR item that omits these keys MUST
be treated as unexpected (cbor terminology) and dropped/ignored. Type, encoding
and language are optional. No encoding means the payload is to be read literally
as per `Content-Type`.

This structure as presented could have been more concisely represented as a
sequence rather than a key/value map but this would have several disadvantages

- It is awkward to express optional values in a sequence
- It would be more difficult to represent new indexes that we MAY want to add in
  the future
- It would be more difficult to remove/deprecate indexes in the future
- It would be difficult or impossible to represent large valued indexes such as
  an 8-byte magic number or a string key

In summary the map structure is chosen to facilitate future modifications to the
conventions in this document in a way that tooling can adopt (or not) in a
backwards compatible way.

The main disadvantage of the map structure is some additional complexity in
decoders to parse out the magic numbers that they support, relative to a
hypothetical algorithm that would always read from the same sequence offset.
However, any generic CBOR decoder can trivially read the map keys as outlined
above, the only scenario this might come up would be some kind of custom onchain
decoding and handling.

### Example

- Consider some JSON ABIv2 document produced by solc then deflated
- A contract meta that references parts of the ABI and provides additional data
  that a GUI can use to better describe the contract operation to a human, this
  meta is encoded with cbor but provided as-is uncompressed

The broad structure of the meta document would be

```
0xff0a89c674ee7874<ABI cbor item><contract cbor item>
```

Where the ordering of the cbor items is arbitrary, this spec doesn't have an
opinion on ordering, which means outputs are NON DETERMINISTIC.

As per CBOR spec, if the encoder wants to produce deterministic outputs it is
up to the encoder to specify how it will achieve that.

Assuming the deflated JSON ABIv2 data is `0x12345678` the CBOR data would be

```
{
  0: h'12345678',
  1: 0xffe5ffb4a3ff2cde,
  2: "application/json",
  3: "deflate"
}
```

Which encodes to 40 bytes excluding the payload bytes (from https://cbor.me).

```
A4                                     # map(4)
   00                                  # unsigned(0)
   44                                  # bytes(4)
      12345678                         # "\u00124Vx"
   01                                  # unsigned(1)
   1B FFE5FFB4A3FF2CDE                 # unsigned(18439425400648969438)
   02                                  # unsigned(2)
   70                                  # text(16)
      6170706C69636174696F6E2F6A736F6E # "application/json"
   03                                  # unsigned(3)
   67                                  # text(7)
      6465666C617465                   # "deflate"
```

Assuming contract meta data of `0x11223344` the CBOR data would be

```
{
  0: h'11223344',
  1: 0xffc21bbf86cc199b,
  2: "application/cbor"
}
```

Which encodes to 31 bytes excluding the payload bytes

```
A3                                     # map(3)
   00                                  # unsigned(0)
   44                                  # bytes(4)
      11223344                         # "\u0011\"3D"
   01                                  # unsigned(1)
   1B FFC21BBF86CC199B                 # unsigned(18429323134567717275)
   02                                  # unsigned(2)
   70                                  # text(16)
      6170706C69636174696F6E2F63626F72 # "application/cbor"
```

So the final document is

```
0xff0a89c674ee7874a4001a12345678011bffe5ffb4a3ff2cde02706170706c69636174696f6e2f6a736f6e03676465666c617465a3001a11223344011bffc21bbf86cc199b02706170706c69636174696f6e2f63626f72
```

Which is 79 bytes total excluding both payloads.
