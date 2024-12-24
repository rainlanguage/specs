# st0x Server API Specification V1

## Goal

Provide signed price quotes for specific trading pairs that can be used by solvers in Raindex orders. Each endpoint handles a single trading pair and returns a signed context containing quote information.

## Magic Numbers

- Web data metadata magic number: `0xff5dcce9b571ba42`
- st0x API magic number: `0xff5d577062616972` ("st0x pair" in hex)

## Server Advertisement

Servers advertise price quote endpoints on metaboard with:
- 8 byte magic number for st0x API
- Remaining bytes contain UTF-8 encoded URL where the API is hosted for a specific trading pair

Each trading pair requires its own advertisement. Orders reference these advertisements by their hash to save gas.

## Request Format

Clients (solvers) POST raw octet streams to the server URL with the following binary format:

```
[ 65 byte ECDSA signature ]  # Signs the hash of all following bytes
[ 32 byte keccak256 hash ]   # Hash of web data from metaboard
[ 8 byte unix timestamp ]    # Seconds since epoch
[ 32 byte nonce ]           # Random bytes for uniqueness
```

No additional payload needed since each endpoint is pair-specific.

## Server Response

The server returns a SignedContextV1 structure containing:

```solidity
struct SignedContextV1 {
    address signer;           // Account that signed the context
    uint256[] context;        // The signed data array
    bytes signature;          // EIP-191 compliant signature
}
```

The `context` array contains:
```
[
    uint256 ioRatio,        // Price ratio for the trading pair
    uint256 pairHash,       // keccak256 hash of the pair name (e.g. "TSLAUSD")
    uint256 validFrom,      // Timestamp in seconds when quote becomes valid
    uint256 expiry          // Timestamp in seconds when quote expires
]
```

The signature MUST be over the packed encoded bytes of the context array (concatenated without length prefix), hashed, and handled per EIP-191.

## Server Requirements

The server MUST:
1. Validate the signature
2. Verify the web data hash matches its own advertisement for this specific pair
3. Check timestamp is not expired
4. Ensure nonce hasn't been used in a non-expired request
5. Set appropriate validFrom and expiry times for the quote
6. Only accept requests if the signing key has a nonzero L1 balance (optional anti-sybil measure)

## Client Usage

1. Solver retrieves the signed context from the appropriate pair endpoint
2. Solver provides the signed context to Raindex orders during take/clear operations
3. The order can then validate and use the quote information according to its logic:
   - Verify quote hasn't expired
   - Check validFrom time has passed
   - Validate pair hash matches expected pair
   - Use ioRatio for price calculations

## Security Considerations

- Each endpoint serves a single pair to reduce complexity
- All quotes include validity timeframes
- Binary format reduces attack surface
- Hash references save gas
- Nonces prevent replay attacks
- Optional L1 balance checks can prevent sybil attacks
- Support for both EOA and EIP-1271 contract signatures
- Contract signatures can be revoked at any time

## Extensions

The protocol intentionally omits:
- Complex order types
- Multi-pair quotes
- Account management
- Complex rate limiting

Orders implement their own logic for using the quote data.