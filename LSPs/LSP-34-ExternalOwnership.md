---
lip: 34
title: External Ownership
author: Fabian Vogelsteller <fabian@lukso.network>
discussions-to: https://discord.gg/E2rJPP4
status: Draft
type: LSP
created: 2025-03-20
requires: ERC165, ERC725Y, LSP2
---

## Simple Summary

A standard that allows any [ERC725Y] contract to derive its owner from an external contract, such as an [LSP8] tokenId owner or any [ERC173] Ownable contract.

## Abstract

This standard defines a single [ERC725Y] data key that references an external contract (and optionally a tokenId) from which the implementing contract resolves its `owner()`. Instead of storing the owner locally, the contract reads it dynamically from the referenced source.

This enables **ownership delegation** — a pattern where one contract's ownership is derived from another contract's state. For example, an LSP7 contract can be "owned by" whoever currently owns a specific tokenId in an LSP8 collection, without requiring manual ownership synchronization.

## Motivation

In composable smart contract systems, one contract's ownership is often logically tied to another contract's state. Current approaches require manual transfers, custom hooks, or centralized coordination to keep ownership consistent — all of which are fragile, gas-expensive, and error-prone.

A declarative approach — "my owner is whoever owns tokenId X in contract Y" — is simpler, cheaper, and always consistent.

**Use cases include:**

- **Music NFTs ([LSP33])**: An LSP7 contract representing ownable track units derives its owner from the track's tokenId owner in an LSP8 release.
- **Sub-assets**: Any asset contract controlled by the owner of a parent NFT.
- **DAO-governed contracts**: A contract owned by whoever holds a specific governance token or position.
- **Delegated vaults**: An [LSP9-Vault](./LSP-9-Vault.md) controlled by the owner of a specific NFT.

## Specification

### Owner Resolution

A contract implementing LSP34 MUST override the `owner()` function (from [ERC173]) to resolve the owner dynamically:

1. Read the `LSP34OwnershipSource` data key from its own [ERC725Y] storage.
2. Decode the value as `(address sourceContract, bytes32 tokenId)`.
3. **If `tokenId != bytes32(0)`**: Call `tokenOwnerOf(tokenId)` on `sourceContract` and return the result.
4. **If `tokenId == bytes32(0)`**: Call `owner()` on `sourceContract` and return the result.
5. **If the data key is not set** (empty bytes): Fall back to the local owner state variable (standard [ERC173] behavior).

### Ownership Transfer

Since the owner is derived externally, `transferOwnership()` and `renounceOwnership()` MUST behave as follows when `LSP34OwnershipSource` is set:

- **`transferOwnership(address)`**: MUST revert. Ownership is transferred by transferring the referenced token or changing ownership on the source contract.
- **`renounceOwnership()`**: MUST revert.

If the `LSP34OwnershipSource` data key is not set, both functions SHOULD behave as standard [ERC173].

### ERC725Y Data Keys

#### SupportedStandards:LSP34ExternalOwnership

```json
{
  "name": "SupportedStandards:LSP34ExternalOwnership",
  "key": "0xeafec4d89fa9619884b600000dd104e111c91ef2cbd7b5824c859213bc599feb",
  "keyType": "Mapping",
  "valueType": "bytes4",
  "valueContent": "0x36a53360"
}
```

Indicates that the contract implements the LSP34 External Ownership standard.

#### LSP34OwnershipSource

```json
{
  "name": "LSP34OwnershipSource",
  "key": "0xa8bc5aea0671308a0920eb016db4108c486ef117a7cd18bf3a9dfcadab6232e1",
  "keyType": "Singleton",
  "valueType": "(address,bytes32)",
  "valueContent": "(Address,bytes32)"
}
```

References the external contract (and optionally a tokenId) from which this contract derives its owner.

**Value encoding:** `abi.encode(address sourceContract, bytes32 tokenId)`

- `sourceContract`: The address of the contract to query for ownership.
- `tokenId`: The tokenId to query. If `bytes32(0)`, the contract calls `owner()` on the source instead of `tokenOwnerOf(tokenId)`.

_Requirements:_

- MUST only be settable by the current owner (as resolved by `owner()`).
- When set, `owner()` MUST resolve from the external source.
- When removed (set to empty bytes), the contract SHOULD fall back to the local owner variable. The local owner SHOULD be set to the last resolved external owner before clearing, to prevent ownership loss.

_Recommendations:_

- SHOULD be set during contract deployment and not changed afterward.
- Implementations SHOULD validate that `sourceContract` is a valid contract address before accepting the value.

### Security Considerations

#### Circular Ownership

Implementations MUST guard against circular ownership references (contract A derives owner from B, B derives from A). Implementations SHOULD set a maximum resolution depth of 1 hop and revert if the resolved owner is `address(0)` or the call fails.

#### Source Contract Availability

If `sourceContract` is destroyed, the `owner()` call will return `address(0)` or revert. Implementations SHOULD handle this gracefully, falling back to the local owner variable.

#### Access Control

The `LSP34OwnershipSource` data key is protected by standard [ERC725Y] access control — only the current `owner()` can call `setData`. This creates a consistent trust chain: the entity that controls the upstream ownership also controls the LSP34 configuration.

## Rationale

### Single Data Key Design

A single data key with a tuple value `(address, bytes32)` keeps the standard minimal. The `bytes32` tokenId field handles both use cases: set it to a specific tokenId for LSP8 ownership, or to `bytes32(0)` to signal "call `owner()` instead."

### Fallback to Local Owner

When `LSP34OwnershipSource` is not set, the contract behaves as a normal [ERC173] Ownable contract. This means LSP34 is backwards-compatible — a contract can start with local ownership and later delegate to external ownership, or vice versa.

### Why Not Proxy/Delegatecall

Proxy-based approaches require compatible storage layouts. LSP34 is a pure read — it queries the external contract's state without coupling beyond the `tokenOwnerOf` or `owner()` interface, working with any existing LSP8 or ERC173 contract without modifications.

## Implementation

An implementation can be found in [TODO: link to reference implementation].

ERC725Y JSON Schema `LSP34ExternalOwnership`:

```json
[
  {
    "name": "SupportedStandards:LSP34ExternalOwnership",
    "key": "0xeafec4d89fa9619884b600000dd104e111c91ef2cbd7b5824c859213bc599feb",
    "keyType": "Mapping",
    "valueType": "bytes4",
    "valueContent": "0x36a53360"
  },
  {
    "name": "LSP34OwnershipSource",
    "key": "0xa8bc5aea0671308a0920eb016db4108c486ef117a7cd18bf3a9dfcadab6232e1",
    "keyType": "Singleton",
    "valueType": "(address,bytes32)",
    "valueContent": "(Address,bytes32)"
  }
]
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

[ERC165]: https://eips.ethereum.org/EIPS/eip-165
[ERC173]: https://eips.ethereum.org/EIPS/eip-173
[ERC725Y]: https://github.com/ERC725Alliance/ERC725/blob/develop/docs/ERC-725.md#erc725y
[LSP2]: ./LSP-2-ERC725YJSONSchema.md
[LSP8]: ./LSP-8-IdentifiableDigitalAsset.md
[LSP9]: ./LSP-9-Vault.md
[LSP33]: ./LSP-33-MusicNFT.md
