# Signed Context Oracle Spec

This spec defines the protocol for a **signed context oracle server** compatible with Rain orderbook. Any server implementing this protocol can serve as a price oracle (or any external data source) for Raindex orders.

## Overview

A signed context oracle server provides signed data that is passed into `takeOrders4()` or `clear2()` as a `SignedContextV1` struct. The orderbook contract verifies the EIP-191 signature on-chain using OpenZeppelin's `SignatureChecker`, then makes the signed data available to the order's Rainlang expression via the signed context columns.

```
External Data Source → Oracle Server → SignedContextV1 JSON
                                              ↓
Client (bot/webapp) → POST request → receives signed context
                                              ↓
Orderbook Contract → verifies EIP-191 signature → Rainlang evaluation
```

## Endpoint

The oracle server MUST expose a single `POST` endpoint. The URL of this endpoint is the `oracle-url` referenced in [ob-yaml orders](./ob-yaml.md).

There is no `GET` fallback. The request body is always required.

## Request

### Method

`POST`

### Content-Type

`application/octet-stream`

### Body

The request body is the **raw ABI-encoded bytes** of the following tuple:

```solidity
abi.encode(OrderV4 order, uint256 inputIOIndex, uint256 outputIOIndex, address counterparty)
```

Where:

- **`order`** — the full `OrderV4` struct of the order being taken/cleared
- **`inputIOIndex`** — index into `order.validInputs[]` for the current IO pair
- **`outputIOIndex`** — index into `order.validOutputs[]` for the current IO pair
- **`counterparty`** — the address of the taker/clearer. Use `address(0)` when the counterparty is unknown (e.g. at quote time)

ABI encoding is used because it is canonical — there are no JSON key ordering ambiguities, and the `OrderV4` struct contains nested arrays and bytes fields that are complex to serialize otherwise.

### Solidity types

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

## Response

### Success (200)

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
| `signer` | `address` | The address that produced the signature. The order's Rainlang expression should check that the signer is a trusted address. |
| `context` | `bytes32[]` | Array of 32-byte values. These become the signed context column available to the Rainlang expression during evaluation. Values are typically encoded as Rain DecimalFloats. |
| `signature` | `bytes` | 65-byte EIP-191 signature (r ∥ s ∥ v) over `keccak256(abi.encodePacked(context[]))`. |

### Error (400)

```json
{
  "error": "<error_code>",
  "detail": "<human-readable description>"
}
```

Standard error codes:

| Code | Meaning |
|------|---------|
| `invalid_body` | The request body could not be ABI-decoded |
| `invalid_index` | `inputIOIndex` or `outputIOIndex` is out of bounds for the order |
| `unsupported_token_pair` | The input/output token pair is not supported by this oracle |

Servers MAY define additional error codes.

### Error (500)

Internal server errors (e.g. upstream data source failure) SHOULD return a 500 with a JSON body in the same `{error, detail}` format.

## Signing

The signature MUST be an EIP-191 "personal sign" signature:

1. Compute the message: `abi.encodePacked(context[0], context[1], ..., context[n])` (raw concatenation of the bytes32 values)
2. Hash the message: `hash = keccak256(packed)`
3. Sign using EIP-191: `sign("\x19Ethereum Signed Message:\n32" + hash)`

This matches how the Rain orderbook contract verifies signatures via `LibContext.build`, which uses OpenZeppelin's `ECDSA.recover` with `toEthSignedMessageHash`.

## Context Layout

The `context` array is an ordered list of `bytes32` values that the Rainlang expression reads by index. The layout is oracle-specific — different oracles may return different data.

### Recommended layout for price oracles

| Index | Value | Encoding |
|-------|-------|----------|
| 0 | Price | Rain DecimalFloat |
| 1 | Expiry timestamp | Rain DecimalFloat (unix seconds) |

**Rain DecimalFloat** is the standard numeric encoding used throughout Rain. Implementations SHOULD use the `rain_math_float` crate (or equivalent) for encoding rather than manual bit packing.

### Expiry

Oracle servers SHOULD include an expiry timestamp in the context. The order's Rainlang expression SHOULD check that the expiry has not passed:

```rainlang
oracle-expiry: signed-context<0 1>(),
:ensure(greater-than(oracle-expiry now()));
```

Short expiry windows (e.g. 5-30 seconds) are recommended to prevent stale price usage.

## Price Direction

When the oracle serves a price feed for a specific token pair (e.g. ETH/USD), the server SHOULD inspect the order's `validInputs[inputIOIndex].token` and `validOutputs[outputIOIndex].token` to determine the correct price direction:

- If input is the quote token and output is the base token → return price as-is
- If input is the base token and output is the quote token → return the inverse (1/price)

This ensures the Rainlang expression always receives the price in the correct orientation for the IO ratio, without needing to handle inversion in Rainlang.

## On-chain Discovery

When an order specifies an `oracle-url`, the tooling encodes a `RaindexSignedContextOracleV1` metadata item (magic number `0xff7a1507ba4419ca`) into the order's `RainMetaDocumentV1` at deployment time. This allows anyone — bots, frontends, indexers — to discover the oracle endpoint for any on-chain order by reading its metadata.

See the [ob-yaml spec](./ob-yaml.md) for how `oracle-url` is specified in order configuration.

## Reference Implementation

A reference implementation is available at [hardyjosh/rain-oracle-server](https://github.com/hardyjosh/rain-oracle-server). It fetches prices from Pyth Network, encodes them as Rain DecimalFloats, and signs the context using EIP-191.

## Security Considerations

- **Signer trust:** The Rainlang expression is responsible for checking the signer address. An oracle server's signer address should be published and verified out-of-band.
- **Expiry:** Always check expiry on-chain. Without expiry checks, a stale signed context could be replayed indefinitely.
- **Counterparty:** The counterparty address may be `address(0)` at quote time. Oracle servers SHOULD handle this gracefully (e.g. ignore the counterparty field for price-only oracles).
- **HTTPS:** Oracle URLs SHOULD use HTTPS in production. Tooling MAY reject non-HTTPS URLs.
- **CORS:** Oracle servers SHOULD allow cross-origin requests (permissive CORS) so that browser-based frontends can fetch signed contexts directly.
