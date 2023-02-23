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
- The cbor-seq items also specify a magic number to act as a domain separator
  that can be read by tooling in O(1) as a signal of intent for the payload
  _without_ needing to first parse (and possibly decompress) the entire payload

### Magic number

The magic number facilitates portability allowing tooling to identify with
high confidence that some data is intended to be interpreted as meta
regardless of where it may be emitted onchain in some event or found
elsewhere.

The magic number mimics ERC165 interface function signature calculation
using 8 bytes instead of 4 bytes for additional collision resistance. The
string used for the hashing is utf-8 `rain-meta-v1` and can be calculated as
```
bytes8(keccak256(abi.encodePacked("rain-meta-v1")));
```

Tooling that wishes to read meta MUST discard/ignore all binary data that
does not begin with the magic number.

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
  felt that compression would help they SHOULD apply the compression directly and
  specify the compression algorithm in the encoding header (see below).
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

| Index | Body                            |
| ---   | ---                             |
| 0     | Payload as CBOR binary data     |
| 1     | Magic number / Domain Separator |
| 2     | Alias for `Content-Encoding`    |
| 3     | Alias for `Content-Type`        |
| 4     | Alias for `Content-Language`    |

