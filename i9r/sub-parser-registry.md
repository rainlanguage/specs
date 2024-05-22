## problem

addresses are totally opaque to users, both for authoring and reading

for example, wanting to include the orderbook sub parser and uniswap words sub parser looks like

```
using-words-from
  0xFCe5E9F48049f3D8850C2C5fd7AD792F10B36326
  0xF1F6cC9376e4A79794BCB7AC451D79425cB381b0
```

on ethereum and

```
using-words-from
  0x23F77e7Bc935503e437166498D7D72f2Ea290E1f
  0x5Cf7d0a8c61c8dcC6b0ECB281dF1C17264C2A517
```

on arbitrum.

How do i know that?

At the time of writing they happen to be summarised in a readme at https://github.com/h20liquidity/h20.pubstrats and these were scraped from github actions runs at some arbitrary time.

- there's no safety mechanism for the sub parser contract to agree that it is what the user thinks it is
- there's no onchain registries or protections against mistakes/phishing
- the addresses are different for the same contracts on different networks
- there's no visual indications at all that the address is anything or other
- there's no easy way to recognise different versions of similar contracts (e.g. different versions of the raindex sub parser)

probably there's more problems here, it's a bad system

## Bad solution: ENS

ENS is not a permissionless system and not available on all EVM chains. Relying on ENS would limit rainlang to only those networks that ENS deploys to.

ENS is a trust based model, if we register `rainlang.eth` and give the keys to that to our CI task, then if someone hacks the private key on our github CI then they can create "official" sub parsers, and phish that way.

## Bad solution: Existing EIP registry standards

As far as i know, the introspection standards like `1820` and `165` are all focussed on interfaces. In this case _all_ the target contracts in the problem domain implement the sub parser interface.

Knowing that "uniswap words" and "raindex words" both implement the sub parser interface is exactly as useful as knowing that DAI and SHIB are both tokens.

## Bad solution: Deployer based registry

Idea being to make some registry where accounts can register contracts under themselves.

E.g. if there was a `rns` sub parser (rain name service) it might look something like this:

`[rns 0xF5B3fCB63911313968bE6D29F6b2Daa269c4CCE0/raindex-words:latest]`

similar conceptually how an org can push containers to docker hub, like `org/container:tag`

This has the benefit of being a permissionless registry, by namespacing things to the registrant there's no need to gatekeep access to the registry.

Solves some of the problems that we have

- there's something human readable that allows us to know what we're using words from
- we only have to remember one address, the deployer that we trust
- allows for some kind of concept of versioning
- the contract _could_ self report its name, allowing for sub parsing to check that the contract agrees with what the author expects

But still fails to solve some large issues

- There's no onchain enforceable/verifiable synchronisation across networks, is `raindex-words:foo` on eth the same as on arb?
- Just like ENS there's the issue of a single compromisable key on a centralised service (github actions) allowing arbitrary "official" registrations
- We still have a lot of opaque looking addresses that don't actually mean much, that authors/readers have to deal with
- There's no onchain way to know anything about deployers, if we ever have to roll the key, how might users know what the situation is with historical keys?

## Bad solution: Subgraph/indexer approach

Perhaps we can lift some heavy lifting to an indexer level concern, but it won't be available onchain, so sub parsed literals won't be able to take advantage of it, even if offchain tooling could somehow.

# Proposed ?good? solution: BIP 39 bytecode hash dynamic mnemonic registry

The idea is that there's a bytecode-oriented registry, where anyone can register any contract under its bytecode hash.

The registry stores a mapping of the first 11*n bits of the bytecode hash to the first address with that bytecode that is registered.

Immediately we can see use cases that this registry is rubbish for:

- EOAs have no bytecode
- Contracts that are designed to function with many copies of the same bytecode, but independent storage instances (e.g. proxies)
- Contracts that have admin keys, so someone could front run a deployment with their own admin keys and squat the registry entry
- Mutable contracts, e.g. metamorphic contracts, that change their bytecode after deployment

Luckily sub parsers by design are none of these things. Sub parsers should be immutable contracts generally.

The sub parsed literal form in rainlang for reading the registry would look like this:

`[@raindex cruel silver]`

- Leading `@` is the dispatch to the rain name service sub parser, mimicing the
  import character in dotrain
- `raindex` is an arbitrary string that the sub parser self identifies as
- The following 2 words `cruel silver` are the BIP 39 encoding of the first 22 bits of the sub parser's bytecode hash
  - There are 2048 BIP 39 words, so each word can encode 11 bits
  - Technically only the first 4 letters of each word are needed so `crue silv` is equivalent but less legible
  - 22 bits = 4.2 million values

The result has the following properties:

- Anyone can register any contract that is already onchain at any time permissionlessly
- The same words will map to an equivalent contract across all networks (if it exists on that network) or error if there's a collision/not found (see below for error)
  - Errors can be resolved by the author providing additional words incrementally until the ambiguity resolves
  - There is a potential phishing vector if the desired contract isn't registered on the network at all (see below)
- The registered contract can self report what it identifies as
- There are no opaque addresses visible/required in the rns literal
- Different versions of the same thing simply have the same self identification string with a different mnemonic

The overall import statement would look something like:

```
using-words-from
  [@raindex cruel silver]
  [@uniswap rather leg bachelor unit frozen music]
```

Assuming uniswap sub parser was griefed at some point, so the 6 word mnemonic is
needed to disambiguate.

### Dealing with collisions (theory)

With 4.2 million possible values, the registry should support quite a few contracts before we start seeing _accidental_ collisions. Due to the birthday paradox, it's not _that_ unlikely that we'll eventually run into a collision. Consider that we already see collisions for function selectors in openchain.xyz

If we ever see incidental collisions we can simply embed a "reroll" constant value in the contract that we can brute force until the resulting contract bytecode hash no longer collides with anything that's already in the registry. As the bytecode hashes are evenly distributed, the ratio of used to unused space is roughly how many times we need to reroll in order to find a free slot, so with just 2 words and 1 million registered contracts, our rerolling will only fail to find a free spot 1/4 of the time.

This rerolling offchain is something that can be done very quickly as it only requires modifying some bytes in the contract that aren't reachable during execution then re-hashing the result. It's roughly like the bitcoin proof of work scheme that involves finding hashes, which a single ant miner s21 can currently produce 200e12 of per second. Bitmain sells "ant space" prefab racks that can house 210 of these, producing 4.2e16 hashes per second overall.

The ease of rerolling implies a cross chain phishing attack:

1. The honest contract `cruel silver` is deployed to some network A
2. An attacker rerolls a malicious parser until they find one that also maps to `cruel silver` and deploys to network B
3. Assuming that the honest contract is not also deployed on network B, authors who attempt to use `cruel silver` on network B will be directed to the malicious parser by the registry

The main issue is that the registry on network B is simply unaware of the honest contract. As soon as the honest contract is deployed and registered to network B (by anyone, permissionlessly), the registry can recognise the collision and revert any ambiguities.

The defence against the above attack looks like:

1. The honest contract `cruel silver` is deployed to network B
2. The registry notes that 2 words are ambiguous, and all further calls to `cruel silver` REVERT
3. The registry will resolve calls to `cruel silver pear` to the honest contract, and `cruel silver cube` to the malicious contract
4. Authors see that their 2 words start reverting during parsing and so have to start providing 3 words

This downgrades the attack from a serious phishing vector to merely a griefing vector. An attacker can cause the sub parser registry to revert where it otherwise would have resolved, but it won't ever resolve honest mnemonics to a dishonest bytecode.

For attackers that aren't willing to dedicate an ant space to griefing/phishing rain sub parsers, a more realistic number of hashes per second for non specialized hardware is under 1e9 hashes per second based on bitcoin hashing as an approximation https://en.bitcoin.it/wiki/Non-specialized_hardware_comparison if we roughly equate the difficulty of sha256 to keccak256.

Each BIP 39 word adds 11 bits of entropy to the brute force attack so 3 words is 2^33 = 8.6e9 so the expected time to "crack" this is half the combinations, which is about 4.3 seconds on a modern gpu. Still a trivial griefing vector. However, adding a fourth word increases the time by 2048x so about 9000 seconds (about 2.5 hours), then a fifth word is another 2048x which is 6+ months, etc.

The exponential nature of mnemonic length to attack difficulty means that as soon as we hit ~5-6 words we're in the realm of serious disincentives for any griefing. Once we get up to 12 words we're in the realm of thermodynamics based arguments (https://en.wikipedia.org/wiki/Brute-force_attack) like "it would take 0.1% of the annual energy budget of all humanity just to flip that many bits, let alone hash them".

### Dealing with collisions (practise)

What is the actual user experience of the above?

It's probably clear that something like this will have to be tooling-assisted and semi automated to be useful.

Consider the worst case scenario where some 2 word mnemonic rainlang has already been deployed onchain, so it is immutable, and a reader, having never seen those 2 words tries to parse the rainlang and self verify it. Between deployment and now, some griefer has collided those 2 words and so the reader has no idea what the real sub parser is.

The reader could **use a full node to fork and check the registry at the time the rainlang source makes it onchain**, but there's no guarantee that the collision didn't happen between when the offchain parsing was done by the author and the onchain deploying of the resulting bytecode. For 2 word mnemonic this would be a real problem as a trivial front run straight into the mempool could achieve it, for a 5 or 6 word mnemnic it's much less likely unless a griefer has precalculated a collision over the course of months or years specifically to target some deployment.

But really, even if the author is using 2 words locally for their convenience, **their tooling should expand the mnemonic to 12 words when they provide the source onchain**. This would be similar to how dotrain tooling currently composes quote bindings to indexed calls in the final generated source, an RNS-aware dotrain tool could use the registry to expand mnemonics from what the author writes to what will be deployed onchain. At the time of expansion, the registry will always provide the correct expanded form or error if a collision exists, so the expansion itself is always safe if done _before_ deployment of the source as metadata.

With reader's forking to check the registry and authors expanding mnemonics, there's no longer much that can go wrong between the author and reader's "meeting of the minds", both have taken reasonable steps to protect themselves, with strong overall effect.

The remaining point of failure is between the author and the network itself. If the sub parser that they want to work with hasn't been deployed to the network of their choice, they are exposed to a phishing attack. In this case, RNS-aware tooling can **check the same mnemonic across many known networks** and raise an exception for the author if the mnemonic ever resolves to different bytecode on any network. This turns the cross chain problem into a 1/n setup where a single honest contract deployed to any network that the author knows about and checks (automatically via their tooling) will detect a collision. Once a collision is detected, the author will need to manually resolve it, perhaps by comparing mnemonics with a known-good rainlang expression, but they'll be protected from accidentally resolving to a malicious parser.

As an extension of checking across multiple networks, **the author can adopt a TOFU (trust on first use) policy as they are working with sub parser**, much like the "known hosts" file in the OS tracks the identities of remote devices that it interacts with on the network. If a collision is discovered after the author has been working with some sub parser for some time, the tooling can warn the user, and offer to inject disambiguating mnemonic words to resolve the collision minimally based on the version of the mnemonic that it has been tracking internally.

At this point, the choice of mnemonic length for working locally comes down to the author's own preferences, shorter mnemonics might be more convenient to work with in a collision-free environment, but longer mnemonics provide better assurances that the author never has to go and manually resolve a collision while authoring. Provided the author has access to a single network that their sub parser of choice exists on, they'll never accidentally use the wrong parser, they'll just have make their own personal tradeoffs re: their authoring experience being lazy and occasionally griefed, vs. more dilligent and reliable.

Of course, the author/reader being tricked into using a malicious subparser separately to mnemonic collisions still exists. This is an identical threat to being tricked into using entirely the wrong dispair, etc. so it's not something that is in scope for the registry. The goal is simply for the registry to not make the problem worse.

Summary:

- Reader checks registry at the same block the author deployed source at, via full node
- Author uses tooling to do mnemonic expansion upon deploy
- Author uses tooling to cross reference mnemonics against multiple networks
- Author adopts TOFU model to track known good sub parsers and restore them in future conflicts

### Implementation

Registry needs mappings between mnemonics of different lengths and associated bytecode.

There isn't much point in mapping mnemonics longer than 12 words, so we can have 11 mappings per registered bytecode, all mnemonics from length 2-12, total cost 11 * 20k gas = 220k gas per registration.

A collision can be registered as some sentinel against all the colliding levels of mnemonic, which would be a storage update rather than cold write so actually a bit cheaper to register contracts with partial collisions.

Only the normalized first 4 bytes of each BIP39 word need be stored.

For example (untested sketch only):

```solidity
string constant BIP39_WORDS = hex"...";

contract SubParserRegistry {
    function register(address registree) external {
        string memory bip39Words = BIP39_WORDS;
        uint256 wordMask = (1 << 11) - 1;
        assembly ("memory-safe") {
            // Allocate mnemonic
            let mnemonic := mload(0x40)
            mstore(0x40, add(mnemomic, 48))
            mstore(mnemonic, 48)

            // Lookup a word and store in mnemonic
            // Do this 12 times until all 48 bytes of the mnemonic are filled...
            let codehash = extcodehash(registree)
            let wordIndex = shr(sub(256, 11), codehash)
            let word = and(mload(add(bip39Words, mul(add(wordIndex, 1), 4))), wordMask)
            mstore(mnemonic, word)

            // Store mnemonic 11 times
            // First two words
            let key := keccak256(mnemonic, 8)
            let value := codehash
            if sload(key) {
                // Collision sentinel is 1
                value := 1
            }
            sstore(key, value)

            // Do for three words, then four, etc...
        }
    }

    function lookup(string memory role, string memory mnemonic) external view returns (address) {
        assembly ("memory-safe") {
            // Allocate a canonical version of the mnemonic
            let canonicalMnemonic := mload(0x40)
            mstore(0x40, add(canonicalMnemonic, 48))

            // Copy first 4 bytes into canonical
            // ...

            // Skip non-whitespace chars
            // ...

            // Skip whitespace chars
            // ...

            // Copy next 4 bytes into canonical
            // ...

            // Skip non-whitespace then whitespace as above...
            // Repeat until provided mnemonic is exhausted

            // Hash the canonical mnemonic to lookup from storage

            // If sentinel `1` error due to collision.

            // If not found `0` error due to unregistered.

            // If current bytecode hash at address differs from registered then
            // error (e.g. if metamorphic or self destruct).

            // If the contract at address doesn't agree with the role as it is
            // looked up then error.

            // Else return registree address.
        }
    }
}
```

The registry itself should be a first class citizen of the DISPair, let's call it
DISPaiR to account for the registry.

By bringing a registry under the parser directly, it avoids an awkward bootstrap
problem where the author would have to specify the registry as a sub parser in
order to use the registry to lookup sub parsers.