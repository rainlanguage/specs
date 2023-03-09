# CAS

Content Addressable Storage (CAS).

> Content-addressable storage (CAS), also referred to as content-addressed
> storage or fixed-content storage, is a way to store information so it can be
> retrieved based on its content, not its name or location. It has been used for
> high-speed storage and retrieval of fixed content, such as documents stored for
> compliance with government regulations. Content-addressable storage is similar
> to content-addressable memory.
>
> CAS systems work by passing the content of the file through a cryptographic
> hash function to generate a unique key, the "content address". The file
> system's directory stores these addresses and a pointer to the physical storage
> of the content. Because an attempt to store the same file will generate the
> same key, CAS systems ensure that the files within them are unique, and because
> changing the file will result in a new key, CAS systems provide assurance that
> the file is unchanged.
>
> Wikipedia
> https://en.wikipedia.org/wiki/Content-addressable_storage

## Name and location independence

The import system for .rain documents relies on hashes to define the content to
import. This implies and requires a CAS implementation to function.

The benefit is that the content can be located anywhere and still be valid
according to cryptographic checks.

It also means that many locations can be checked concurrently to search for any
given content.

Different devices can use different locations to search and get equivalent
results provided the content being retrieved is in at least 1 location known to
each device.

This makes local, cloud, DHT, onchain, cache all one and the same for the sake
of both writing and reading .rain files.

Provided that authors and reader have at least one shared content repository,
such as an onchain testnet, then they are free to cache and fetch arbitrarily
according to the needs of their content.

This allows for a fully decentralised content authoring, reading and reauthoring
model where _any_ onchain expression or metadata can be rebound and redeployed
by anyone.

## Verify before store

We want to discourage the "copy and paste" model where diffs between two
contracts can only be achieved by text analysis (e.g. per-line changes) and allow
for a "rebind and extend" model where the composition of .rain files is native.
For example, an invalid rebinding should be rejected by _all_ tooling at the
composition level, it should not require users to first pull an invalid code
state and then run additional commands, likely between several different binaries
to compile and discover invalidity piecemeal. Of course, with permissionlessly
extensible metadata, the reality is that advanced analysis will require
additional tooling, but the baseline deployability of a .rain file should be
ensured by the CAS itself.

Exactly as an invalid hash for some content would be rejected by a CAS before the
content is stored under the hash, any CAS implementation MUST ensure the
integrity of a .rain file according to the .rain spec before storing it under its
hash. This simplifies all tooling downstream of the CAS and ensures that certain
types of malicious/buggy data is filtered out as early as possible between author
and reader.

## Locations are undefined

As the content integrity is assured by the hash and we accept a many location
reality, there is no need to overspecify locations in fragile ways.

We DO NOT specify locations in .rain files as any possible location could become
defunct, even some blockchain could become difficult to access geopolitically.
Assumptions about the indefinite existence of a location are dubious enough
before we even overgeneralise to _individual access_ to a _specific_ location.

Some _likely_ locations that would be specified for common usage:

- Blockchain events, both testnets and mainnets
- IPFS and other DHTs
- Bittorrent
- URLs
- Local device storage/cache

The CAS implementation MUST support the user providing/discovering many locations
and treating all equally, subject to normal technical concerns such as latency,
bandwidth, reliability, etc. A typical CAS would fan out or cascade through the
lowest latency locations first then attempt more remote locations.

## Content is self describing

One challenge for accepting all content found on the other side of a hash is that
all context for interpreting the data is lost. There's nothing about the process
of fetching the data, or the surrounding application context, or its location or
author or anything else that suggests what the data represents.

This means that all the data needs to be self describing such that tooling can
efficiently in O(1) choose to discard it as noise, or attempt to process it as
a signal. The spec for metadata describes a CBOR sequence thin wrapper that can
be applied to any payload to make it self describing, modelled off the headers
that HTTP itself uses to do the same.

Generally though, the metadata spec outlined can't be assumed to always be
implemented but an equivalently self describing mechanism SHOULD be used, such
that tooling MAY discard anything that isn't _immediately_ O(1) apparent how to
process it.

The general case is that it isn't really good enough to assume some data is JSON
or otherwise and try to read clues from JSON keys, because by the time the tool
has reached that point it has already dedicated resources to parsing, etc.

## Hashes should be self describing

Astute readers of the .rain document will note that the hashing algorithm or even
representation of a .rain file was never specified.

The multihash format should probably be used
https://github.com/multiformats/multihash as it has been popularised by several
p2p networks already, including IPFS.

One advantage of multihash is that the hashes themselves adopt a self describing
approach, which allows ethereum standard keccak256 hashes to be specified, but
other hashing schemes are also supported.

Ultimately, whatever hashing scheme is used will map to simple keys in the CAS.
If the .rain document specifies a hash that tooling can't support then it won't
(can't) be adopted.

By adding support for multihashes in common CAS implementations, this allows
.rain files to be confident that they will be future proof beyond the immediate
use cases and limitations presented by the EVM itself.

By not specifying that hashes MUST be multihash, it allows the development of
.rain documents to evolve independently of any specific non-rain data type, much
like the metadata on the other side of the hash itself is unspecified and
unrestricted.

