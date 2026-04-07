---
lip: 34
title: External Owner Source
author: Fabian Vogelsteller <fabian@universaleverything.io>, Thomas Beard <thomas@universaleverything.io>
discussions-to: https://t.me/+PjX_Awnpjh8xYWE0
status: Draft
type: LSP
created: 2025-03-20
requires: ERC725Y
---

## Simple Summary

A single [ERC725Y] data key that lets a contract derive its owner from another contract — typically an [LSP8] tokenId.

## Abstract

This standard defines **one** data key, `LSP34OwnershipSource`, that points to an external contract (and optionally a tokenId) from which the implementing contract resolves its `owner()`.

Instead of storing an owner locally, the contract reads it from the referenced source. This is used, for example, by an [LSP7] track token in [LSP33] Music NFTs to derive its owner from the track's tokenId owner in an [LSP8] release.

## Motivation

Some contracts are logically owned by "whoever owns something else". Manually transferring ownership whenever the upstream owner changes is fragile. LSP34 replaces that with a single declarative pointer: *"my owner lives over there."*

## Specification

### ERC725Y Data Key

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

**Value encoding:** `abi.encode(address sourceContract, bytes32 tokenId)`

- `sourceContract` — the contract to query for ownership.
- `tokenId` — the tokenId to query on `sourceContract`. If `bytes32(0)`, query `owner()` on `sourceContract` instead of `tokenOwnerOf(tokenId)`.

### Owner Resolution

A contract implementing LSP34 MUST override `owner()` to:

1. Read `LSP34OwnershipSource` from its own ERC725Y storage.
2. If unset, fall back to local [ERC173] behavior.
3. If `tokenId != bytes32(0)`, return `ILSP8(sourceContract).tokenOwnerOf(tokenId)`.
4. If `tokenId == bytes32(0)`, return `IERC173(sourceContract).owner()`.

### Ownership Transfer

When `LSP34OwnershipSource` is set, `transferOwnership(address)` and `renounceOwnership()` MUST revert — ownership is transferred by moving the upstream token/owner.

When the key is not set, both functions behave as standard [ERC173].

### Access Control

`LSP34OwnershipSource` is protected by standard ERC725Y access control — only the current `owner()` can set it.

## Rationale

LSP34 is intentionally minimal: one data key, one resolution rule. The `bytes32` tokenId field handles both cases (LSP8 tokenId owner, or plain ERC173 `owner()`) without a second key.

`bytes32(0)` is reserved as the "call owner() instead" sentinel. Implementations using sequential `uint256` tokenIds SHOULD start from `1`.

## Implementation

Reference implementation: [lukso-network/lsp-smart-contracts#1088](https://github.com/lukso-network/lsp-smart-contracts/pull/1088).

ERC725Y JSON Schema `LSP34ExternalOwnership`:

```json
[
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

[ERC173]: https://eips.ethereum.org/EIPS/eip-173
[ERC725Y]: https://github.com/ERC725Alliance/ERC725/blob/develop/docs/ERC-725.md#erc725y
[LSP7]: ./LSP-7-DigitalAsset.md
[LSP8]: ./LSP-8-IdentifiableDigitalAsset.md
[LSP33]: ./LSP-33-MusicNFT.md
