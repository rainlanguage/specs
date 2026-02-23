# Signed Context Oracle Spec

## Motivation

Onchain orders often need external data — prices, trading signals, portfolio weights, or anything else that cannot be found onchain. Push oracles (e.g. Chainlink) solve this for prices but are expensive in gas and limited to data that feed operators choose to publish. Pull oracles allow order owners to specify arbitrary data sources, including data they may want to keep offchain until execution time (e.g. proprietary trading signals).

This spec defines a standard protocol for **signed context oracle servers** that serve data to Rain orderbook orders. The key benefit is the **separation of order placer and solver/taker**: an order owner deploys an order referencing an oracle URL, and any taker or solver has a standard way to discover that URL, fetch the required data, and pass it into the order at execution time. Without this standard, takers would need out-of-band coordination with each order owner to know where to get the data and how to format it.

## Protocol

### Endpoint

The oracle server MUST expose a `POST` endpoint. The URL of this endpoint is the `oracle-url` specified in the order configuration (see [ob-yaml spec](./ob-yaml.md)).

There is no `GET` fallback. The request body is always required.

### Request

**Method:** `POST`

**Content-Type:** `application/octet-stream`

**Body:** Raw ABI-encoded bytes of the following tuple:

```solidity
abi.encode(OrderV4 order, uint256 inputIOIndex, uint256 outputIOIndex, address counterparty)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `order` | `OrderV4` | The full order struct being taken or cleared |
| `inputIOIndex` | `uint256` | Index into `order.validInputs[]` for the current IO pair |
| `outputIOIndex` | `uint256` | Index into `order.validOutputs[]` for the current IO pair |
| `counterparty` | `address` | The address of the taker or clearer |

ABI encoding is used because it is canonical — there are no JSON key ordering ambiguities, and `OrderV4` contains nested arrays and bytes fields that are complex to serialize in other formats.

The Solidity types are:

```solidity
struct IOV2 {
    address token;
    bytes32 vaultId;
}

struct EvaluableV4 {
    address interpreter;
    address store;
    bytes bytecode;
}

struct OrderV4 {
    address owner;
    EvaluableV4 evaluable;
    IOV2[] validInputs;
    IOV2[] validOutputs;
    bytes32 nonce;
}
```

These are the canonical Rain orderbook types. Implementations SHOULD use generated bindings from the Rain orderbook ABI rather than manual encoding.

### Response

**Success (200)**

Content-Type: `application/json`

```json
{
  "signer": "0x<20-byte address>",
  "context": ["0x<32-byte hex>", "0x<32-byte hex>", ...],
  "signature": "0x<65-byte hex>"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `signer` | `address` | The address that produced the signature |
| `context` | `bytes32[]` | Array of 32-byte values. These become the signed context column available to the order's Rainlang expression during evaluation |
| `signature` | `bytes` | 65-byte EIP-191 signature (r ∥ s ∥ v) over `keccak256(abi.encodePacked(context[]))` |

Values in the `context` array SHOULD be encoded as Rain DecimalFloats where they represent numeric data, so that they can be read directly by Rainlang arithmetic operations.

**Error (4xx/5xx)**

Content-Type: `application/json`

```json
{
  "error": "<error_code>",
  "detail": "<human-readable description>"
}
```

### Signing

The signature MUST be an EIP-191 "personal sign" signature:

1. Concatenate the context values: `packed = abi.encodePacked(context[0], context[1], ..., context[n])`
2. Hash: `hash = keccak256(packed)`
3. Apply the EIP-191 prefix: `eth_hash = keccak256("\x19Ethereum Signed Message:\n32" ++ hash)` (this is `toEthSignedMessageHash(hash)`)
4. Sign: `(r, s, v) = ECDSA.sign(eth_hash)`

Most Web3 libraries handle steps 3-4 automatically via `personal_sign(hash)` or `sign_message(hash)`.

This matches how the Rain orderbook contract verifies signatures via `LibContext.build`, which uses OpenZeppelin's `ECDSA.recover` with `toEthSignedMessageHash`.

## Onchain Discovery

When an order specifies an `oracle-url`, the tooling MUST encode a `RaindexSignedContextOracleV1` metadata item (magic number `0xff7a1507ba4419ca`) into the order's `RainMetaDocumentV1` at deployment time.

This is how takers, solvers, bots, and frontends discover the oracle endpoint for any onchain order. Without this metadata, there would be no standard way for a third party to know where to fetch the signed context for an order they want to take.

The metadata item contains the oracle URL as a UTF-8 encoded string.

## Security Model

The order owner is the party who stands to lose funds if the oracle misbehaves — they are trusting the oracle server (identified by its signer address) with control over the data their order uses to calculate ratios, maximums, and any other logic.

It is the order owner's responsibility to:

- Choose an oracle and signer they trust
- Include any onchain protections they want in their Rainlang expression (e.g. expiry checks, zero ratio guards, price bounds, or any other validation)
- Understand that the oracle server has full knowledge of the order struct, IO pair, and counterparty for every request

The contract enforces only that the signature is valid for the declared signer address. All other validation is up to the Rainlang expression.

## Reference Implementation

A reference implementation is available at [hardyjosh/rain-oracle-server](https://github.com/hardyjosh/rain-oracle-server). It fetches prices from Pyth Network, encodes them as Rain DecimalFloats, signs the context with EIP-191, and demonstrates price direction handling based on the order's input/output tokens.
