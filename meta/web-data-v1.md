# Web data V1

## Goal

Need some context free way to specify that some relevant data can be requested
from an offchain URL.

For example, an order on raindex might accept price feeds from an offchain oracle
and needs to specify the URL of the oracle.

This spec is intended to be as minimal as possible.

## Web data content

Web data metadata has magic number `0xff5dcce9b571ba42`.

The payload:

- begins with an 8 byte magic number that specifies the API
  - Each API magic number MUST NOT be used for any other context, including other
    versions of the web data specification.
- remaining bytes is the UTF8 encoded URL string the API is located at

## Client request

Clients POST octet streams (binary data) to the server at the specified location.

```
[ 65 byte ECDSA (EVM) signature over the keccak256 hash of all following bytes ]
[ 32 byte keccak256 hash of the web data for this request ]
[ 8 byte unix timestamp ]
[ 32 byte nonce ]
[ N payload bytes ... ]
```

The signature allows the server to `ecrecover` the pubkey of the signer, which
acts as a username and authentication if the server wants to implement
authorization of the API. The signature is over the hash of the remainder of the
bytes because signing performance and resource usage degrades much worse than
hashing over larger payloads, the 32 byte payload for the signature keeps this
overhead constant.

The hash of the web data normalizes the length of webdata so that we know it is
always in the same position. Including this in the signature prevents client
signed requests being reused or used accidentally in the wrong context.

The timestamp allows the server to disallow stale requests after a server defined
expiry period. The timestamp is millisecond precision, which is supported by all
major operating systems and languages, although it is more precise than EVM
native timestamps. If millisecond precision is not supported by the client it MAY
use less precise times (e.g. seconds) but the REPRESENTATION MUST still be millis
to prevent ambiguity.

The nonce prevents requests being replayed on the server. The server MUST track
nonces and refuse requests that reuse nonces. To prevent unbounded resource
usage servers SHOULD drop nonces tracked against requests that are expired, as
they can no longer be replayed anyway. The server MAY track nonces in memory
if the expiry time of requests is significantly shorter than the expected uptime
of the server, and the server is a single process, so any downtime is highly
unlikely to allow an attack opportunity. If the server is multiprocess and/or
the API is highly sensitive to replayed requests, it MUST persist nonces, e.g. to
a database to prevent data loss leading to exploits. The 32 byte nonce length
allows high quality randomness to guarantee uniqueness, so the client does not
need to track nonces and can remain stateless.

The payload is optional, arbitrary length and contextual to the API that the
web data describes. The client and server are expected to understand what it is
useful for, if anything (e.g. specifying trading pairs, token info, etc.).

## Server validation

At a minimum the server MUST validate:

- The signature is valid and meets any authorization requirements
- The web data hash is expected
  - Each API must know and check the web data hash that it is serving
- The unix timestamp is not expired
  - Server decides expiry rules, several minutes should be adequate for most use
    cases
- The nonce is unique across all known non-expired requests
  - Including from different signers
- Any additional contextual validation rules imposed by the API

## Intended workflow

1. Some transaction includes a data reference, i.e. the hash of some metadata
  posted to metaboard.
2. The referenced web data is found in metaboard event logs.
3. Web client signs request authentication and POSTs it to server with optional
  contextual payload.
4. Web server responds with contextual data.