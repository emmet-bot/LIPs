---
lip: 33
title: Music NFT
author: Fabian Vogelsteller <fabian@universaleverything.io>, Thomas Beard <thomas@universaleverything.io>
discussions-to: https://t.me/+PjX_Awnpjh8xYWE0
status: Draft
type: LSP
created: 2025-03-20
requires: ERC165, ERC725Y, LSP2, LSP4, LSP7, LSP8, LSP34
---

## Simple Summary

A composable standard for representing music releases and tracks as digital assets, using [LSP8] for releases/albums and optionally [LSP7] for ownable units of individual tracks.

## Abstract

This standard defines how to represent music as digital assets by composing existing LUKSO standards:

- An **[LSP8] collection** represents a **release or album**. Each `tokenId` represents a **single track**.
- **[LSP4Metadata][LSP4]** on the LSP8 contract describes the release/album. Per-track metadata is set via `setDataForTokenId`.
- Optionally, a track can have an associated **[LSP7] contract** for **ownable units** (e.g., collectible editions), indicated by the `LSP33OwnableTrackToken` data key.
- When an LSP7 is linked, the LSP8 acts as a **transparent router**: reads proxy to the LSP7, writes forward to the LSP7's `setData`.
- The LSP7 uses [LSP34] to derive its owner from the LSP8 `tokenOwnerOf`, so the artist retains control over minting and metadata.
- Extended music metadata (contributors, identifiers, copyright, lyrics, DDEX, stems) lives in `LSP33Metadata` — a separate data key with its own JSON file.

Tracks can exist as metadata-only entries (proving creation) or as fully ownable digital assets, all within a single composable framework.

### Ownership Model

This standard distinguishes between two layers of ownership:

| | **Artist Ownership** | **Collector Ownership** |
|---|---|---|
| **What** | LSP8 `tokenId` (the track itself) | LSP7 units (copies/shares of a track) |
| **Who** | The artist or rights holder | Fans, collectors, anyone |
| **Controls** | Metadata, minting new units, linking LSP7 | Transferring/trading their units |
| **How** | `tokenOwnerOf(tokenId)` on the LSP8 | `balanceOf(address)` on the LSP7 |

**The LSP8 tokenId always stays with the artist.** It represents authorship and creative control — not a collectible to be traded. The artist mints a tokenId for each track, sets its metadata, and optionally links an LSP7 to make it collectible.

**Collectors never touch the LSP8 directly.** They interact only with the LSP7 — buying, holding, and trading fungible units of a track. Holding LSP7 units does not grant any control over the track's metadata, minting, or identity.

```
  Artist (Universal Profile)
    │
    ├── Owns LSP8 tokenId 1 ("Track A")
    │     ├── Can set/update metadata
    │     ├── Can link an LSP7 for collectible editions
    │     └── Can mint LSP7 units to sell to fans
    │
    └── Owns LSP8 tokenId 2 ("Track B")
          └── Metadata-only (no LSP7, no collectibles)

  Collector (Fan)
    │
    └── Holds 5 LSP7 units of "Track A"
          ├── Can transfer or trade units
          ├── Cannot change track metadata
          └── Cannot mint new units
```

## Motivation

Music on the blockchain lacks a standardized way to represent relationships between releases, tracks, and collectibles. Artists need:

1. **Provenance** — prove creation at a point in time, even without selling.
2. **Composability** — use existing standards (LSP7, LSP8, LSP4) rather than new token contracts.
3. **Flexibility** — some tracks are metadata-only, others have ownable editions.
4. **Artist Control** — the artist always controls their tracks' metadata and minting, regardless of who holds collectible units.
5. **Unified Interface** — manage all track data through the LSP8 collection contract.
6. **Industry Compatibility** — bridge on-chain metadata with DDEX, ISRC, ISWC, GRid for DSP interoperability.

## Specification

### Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  LSP8 Collection (Release / Album)                              │
│  Owner: The Artist (Universal Profile)                          │
│  LSP4Metadata = release display metadata                        │
│  LSP33Metadata = release music metadata                         │
│  LSP4TokenType = 2 (Collection)                                 │
│                                                                  │
│  tokenId 1 ─── Track 1 (owned by artist)                        │
│    ├── LSP4Metadata (per-track display)                          │
│    ├── LSP33Metadata (per-track music data)                      │
│    └── LSP33OwnableTrackToken ──► LSP7 Contract                  │
│         ▲      read/write proxy        │                         │
│         └──────────────────────────────┘                         │
│                                                                  │
│  tokenId 2 ─── Track 2 (metadata only, owned by artist)         │
│    ├── LSP4Metadata                                              │
│    └── LSP33Metadata                                             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  LSP7 Ownable Token (single track)                              │
│  Controlled by: Artist (via LSP34 → LSP8 tokenOwnerOf)          │
│  Traded by: Collectors / fans (fungible units)                   │
│  LSP8ReferenceContract ──► (LSP8, tokenId)                       │
│  LSP34OwnershipSource ──► (LSP8, tokenId)                        │
│  LSP4Metadata = track display metadata                           │
│  LSP33Metadata = track music metadata                            │
│  owner() = artist (resolved from LSP8 tokenOwnerOf)              │
│  setData() = artist OR parent LSP8 (same person)                 │
│  mint() = artist only                                            │
│  balanceOf() = how many units each collector holds               │
└──────────────────────────────────────────────────────────────────┘
```

### LSP8 Contract (Release / Album)

The LSP8 contract MUST:

- Set `LSP4TokenType` to `2` (Collection).
- Use `LSP4Metadata` on the contract level for release/album metadata.
- Use `setDataForTokenId` with `LSP4Metadata` and `LSP33Metadata` for per-track metadata.
- Use sequential `uint256` values (encoded as `bytes32`) for `tokenId`. The `LSP8TokenIdFormat` SHOULD be set to `0` (uint256).
- Implement the data routing behavior described below.

### Data Routing Behavior

When `LSP33OwnableTrackToken` is set for a tokenId, the LSP8 MUST act as a transparent router for metadata data keys (`LSP4Metadata` and `LSP33Metadata`). This gives the artist a single interface for managing all track data.

```
Artist (Universal Profile)
  │
  ├──► LSP8.setData(key, value)
  │      └── Release-level data → writes locally
  │          Requires: caller is contract owner
  │
  ├──► LSP8.setDataForTokenId(tokenId, dataKey, dataValue)
  │      ├── Requires: caller is tokenOwnerOf(tokenId)
  │      ├── LSP33OwnableTrackToken set for tokenId?
  │      │
  │      │   NO ──► Write locally to LSP8 tokenId storage
  │      │
  │      │   YES ──► Is dataKey LSP4Metadata or LSP33Metadata?
  │      │           │
  │      │           NO ──► Write locally to LSP8 tokenId storage
  │      │           │
  │      │           YES ──► Forward to LSP7.setData(dataKey, dataValue)
  │      │                    └── LSP7 checks: is msg.sender
  │      │                        the LSP8 in my LSP8ReferenceContract?
  │      │                        YES ──► Write to LSP7 storage ✓
  │      │                        NO  ──► Revert
  │
  ├──► LSP7.setData(key, value)
  │      └── Direct track metadata update (also valid)
  │          Requires: caller is owner (via LSP34)
  │
  └──► LSP7.mint(to, amount, ...)
         └── Requires: caller is owner (via LSP34)


Anyone (Reader)
  │
  └──► LSP8.getDataForTokenId(tokenId, dataKey)
         ├── LSP33OwnableTrackToken set for tokenId?
         │
         │   NO ──► Return from LSP8 tokenId storage
         │
         │   YES ──► Is dataKey LSP4Metadata or LSP33Metadata?
         │           │
         │           NO ──► Return from LSP8 tokenId storage
         │           │
         │           YES ──► Call LSP7.getData(dataKey)
         │                    ├── Succeeds ──► Return LSP7 data
         │                    └── Reverts  ──► Fallback: return
         │                                     LSP8 local tokenId data
```

#### getDataForTokenId (Read Proxy)

When `getDataForTokenId(tokenId, dataKey)` is called and `LSP33OwnableTrackToken` is set for that tokenId:

- If `dataKey` is `LSP4Metadata` or `LSP33Metadata`: Read from the linked LSP7 via `getData(dataKey)`. If the call reverts, fall back to locally stored data on the LSP8.
- For all other data keys: Return from local LSP8 tokenId storage.

The fallback ensures metadata remains available even if the linked LSP7 becomes unavailable.

#### setDataForTokenId (Write Forwarding)

When `setDataForTokenId(tokenId, dataKey, dataValue)` is called and `LSP33OwnableTrackToken` is set for that tokenId:

- If `dataKey` is `LSP4Metadata` or `LSP33Metadata`: Forward to the linked LSP7 by calling `setData(dataKey, dataValue)`. If the external call reverts, the entire call MUST revert.
- For all other data keys: Write locally to LSP8 tokenId storage.

_Requirements:_

- MUST only be callable by the current owner of the specific `tokenId` (as returned by `tokenOwnerOf(tokenId)`), **not** the contract-level `owner()`. This is consistent with LSP7's access model, where `owner()` resolves to the same `tokenOwnerOf` result via LSP34. When a track is transferred, the new token owner automatically gains control over that track's metadata.

#### Event Behavior

When a metadata write is forwarded to a linked LSP7, the LSP8 does **not** emit `TokenIdDataChanged` for that operation — the data is not stored on the LSP8. Instead, the LSP7 emits its own `DataChanged` event.

Indexers and frontends SHOULD:

1. Read `LSP33OwnableTrackToken` for each tokenId to discover linked LSP7 contracts.
2. Subscribe to `DataChanged` events on linked LSP7 contracts for metadata updates.
3. Treat the LSP8 as a router — only `TokenIdDataChanged` events for locally stored data (non-metadata keys, unlinked tokenIds) are emitted by the LSP8.

### ERC725Y Data Keys

#### SupportedStandards:LSP33MusicNFT

```json
{
  "name": "SupportedStandards:LSP33MusicNFT",
  "key": "0xeafec4d89fa9619884b60000f43ae543d533ae7389b5791d7870aadfdcff2ca4",
  "keyType": "Mapping",
  "valueType": "bytes4",
  "valueContent": "0x3cd46617"
}
```

Indicates that the contract implements LSP33. MUST be set on the LSP8 contract.

#### LSP33OwnableTrackToken

```json
{
  "name": "LSP33OwnableTrackToken",
  "key": "0x869105ad93b0c40a24a19b5c3260f597737cf3625302cb5ab26a82e14468ad91",
  "keyType": "Singleton",
  "valueType": "address",
  "valueContent": "Address"
}
```

Per-tokenId data key (via `setDataForTokenId`) pointing to the [LSP7] contract representing ownable units of this track.

- **Not set**: Track is metadata-only.
- **Set**: The referenced LSP7 MUST meet the requirements in [LSP7 Contract](#lsp7-contract-ownable-track-units).

_Requirements:_

- MUST only be settable by the `tokenId` owner (via `tokenOwnerOf`).
- Before setting, the implementation SHOULD verify the [bidirectional link](#bidirectional-link-verification).
- Once set, it is RECOMMENDED not to change this value, to preserve ownership integrity for existing LSP7 token holders.

#### LSP33Metadata

```json
{
  "name": "LSP33Metadata",
  "key": "0xbecee5b67fa43fb2e96c7c6ccdc23007fffc2b24205416b2122cd7b8d8ef48fa",
  "keyType": "Singleton",
  "valueType": "bytes",
  "valueContent": "VerifiableURI"
}
```

References the extended music metadata JSON file. This is a separate file from `LSP4Metadata`, stored under its own data key.

Can be set:
- On the **LSP8 contract level** for release/album metadata.
- Per **tokenId** via `setDataForTokenId` for track-level metadata.
- On the **LSP7 contract level** for ownable track token metadata.

> **Note:** `LSP4Metadata` and `LSP33Metadata` MAY point to the same JSON file (with both as top-level properties) to save storage and fetches, but this is not required.

### LSP7 Contract (Ownable Track Units)

The LSP7 contract MUST:

- Implement [LSP34] with `LSP34OwnershipSource` pointing to `(LSP8Address, tokenId)`.
- Set [`LSP8ReferenceContract`][LSP8Ref] to `(LSP8Address, tokenId)`.
- Cache the parent LSP8 collection address as an `immutable` variable (set in the constructor). This avoids reading from ERC725Y storage on every `owner()` call and ensures the parent reference cannot be changed after deployment.
- Use `LSP4Metadata` and `LSP33Metadata` for track metadata.
- Resolve `owner()` via LSP34.
- Restrict minting to the resolved owner.
- Accept `setData` and `setDataBatch` calls from the parent LSP8 collection contract (see below).

The LSP7 contract SHOULD:

- Set `decimals()` to `0` (non-divisible).
- Set `LSP4TokenType` to `1` (NFT/NDT).

#### Parent Collection Authorization

The LSP7 MUST accept `setData` and `setDataBatch` calls from its parent LSP8 collection contract, in addition to the resolved owner. This enables the LSP8 to forward metadata writes.

When `setData` or `setDataBatch` is called, the LSP7 MUST allow the call if either:

1. `msg.sender` is the resolved `owner()` (via LSP34), OR
2. `msg.sender` matches the cached parent LSP8 collection address (set immutably in the constructor).

This is safe because the LSP8 verifies `onlyOwner` before forwarding, and both owners resolve to the same address (the artist).

#### Address Encoding

The `LSP33OwnableTrackToken` value (the linked LSP7 address) MAY be encoded as either:

- **20 bytes** (`abi.encodePacked(address)`) — compact form.
- **32 bytes** (`abi.encode(address)`) — left-padded form.

Implementations MUST handle both formats when extracting the address.

#### Bidirectional Link Verification

When setting `LSP33OwnableTrackToken` for a tokenId, the LSP8 SHOULD verify that the linked LSP7's `LSP8ReferenceContract` points back to `(address(this), tokenId)`. If verification fails, the call SHOULD revert.

### Metadata

This standard uses [LSP4] for display metadata and extends it with `LSP33Metadata` for structured music data. Each has its own data key and is typically stored as a **separate JSON file**.

- **`LSP4Metadata`**: Human-readable — name, description, artwork, links, audio assets, attributes. Defined by [LSP4].
- **`LSP33Metadata`**: Structured music data — contributors, identifiers, copyright, lyrics, preview, stems, DDEX. Defined by this standard.

The `LSP4Metadata` JSON MUST include the top-level `category` field set to `"Music"`.

> **Note:** Both files MAY be combined into a single JSON document with `LSP4Metadata` and `LSP33Metadata` as top-level properties. In that case, both data keys reference the same URI.

#### Release / Album Metadata

##### LSP4Metadata (Release)

The following `attributes` SHOULD be set on the LSP8 contract's `LSP4Metadata`:

| Attribute Key | Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `Artist` | `string` | ✓ | Primary artist or band name |
| `Release Type` | `string` | ✓ | `Single`, `EP`, `Album`, `Compilation` |
| `Release Date` | `string` | ✓ | ISO 8601 (`YYYY-MM-DD`) |
| `Primary Genre` | `string` | ✓ | Primary genre |
| `Track Count` | `string` | ✓ | Total number of tracks |
| `Label` | `string` | | Record label name |
| `Secondary Genre` | `string` | | Secondary genre |
| `Language` | `string` | | ISO 639-2 code |

##### LSP33Metadata (Release)

| Field | Type | Description |
| :--- | :---: | :--- |
| `contributors` | `array` | Contributors (see [Contributors](#contributors)) |
| `identifiers` | `object` | Industry identifiers (see [Identifiers](#identifiers)) |
| `copyright` | `object` | Copyright info (see [Copyright](#copyright)) |
| `ddex` | `object` | DDEX ERN reference (see [DDEX](#ddex)) |

#### Track Metadata

##### LSP4Metadata (Track)

The following `attributes` SHOULD be set per track:

| Attribute Key | Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `Artist` | `string` | ✓ | Track artist |
| `Track Number` | `string` | ✓ | Position in release |
| `Primary Genre` | `string` | ✓ | Primary genre |
| `Release Date` | `string` | ✓ | ISO 8601 |
| `Duration` | `string` | | e.g., `3:35` or `PT3M35S` |
| `Disc Number` | `string` | | For multi-disc releases |
| `Explicit Content` | `string` | | `Explicit`, `NotExplicit`, `Cleaned` |
| `BPM` | `string` | | Beats per minute |
| `Key` | `string` | | Musical key (e.g., `Cm`, `F#`) |
| `Secondary Genre` | `string` | | Secondary genre |
| `Language` | `string` | | ISO 639-2 code |

##### LSP33Metadata (Track)

| Field | Type | Description |
| :--- | :---: | :--- |
| `contributors` | `array` | Contributors (see [Contributors](#contributors)) |
| `identifiers` | `object` | Industry identifiers (see [Identifiers](#identifiers)) |
| `copyright` | `object` | Copyright info (see [Copyright](#copyright)) |
| `lyrics` | `object` | Lyrics data (see [Lyrics](#lyrics)) |
| `preview` | `object` | Preview clip (see [Preview](#preview)) |
| `stems` | `array` | Stem files (see [Stems](#stems)) |
| `ddex` | `object` | Track DDEX reference (see [DDEX](#ddex)) |

### LSP33Metadata Fields

#### Contributors

An ordered array of contributors. Each represents a person or entity with one or more roles, mapping to DDEX ERN's `<Contributor>` element.

```json
{
  "contributors": [
    { "name": "ledfut", "address": "0x1234...abcd", "roles": ["Producer", "Mixer"] },
    { "name": "ampy", "address": "0x5678...efgh", "roles": ["Composer", "Lyricist"] },
    { "name": "Big Boss", "roles": ["Executive Producer"] }
  ]
}
```

| Property | Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `name` | `string` | ✓ | Display name |
| `address` | `string` | | Universal Profile address |
| `email` | `string` | | Contact email |
| `roles` | `string[]` | ✓ | One or more roles |

Common roles (extensible — any role MAY be used):

| Role | DDEX Equivalent |
| :--- | :--- |
| `MainArtist` | `MainArtist` |
| `FeaturedArtist` | `FeaturedArtist` |
| `Producer` | `StudioProducer` |
| `Executive Producer` | `ExecutiveProducer` |
| `Composer` | `Composer` |
| `Lyricist` | `Lyricist` |
| `Mixer` | `MixingEngineer` |
| `Mastering Engineer` | `MasteringEngineer` |
| `Remixer` | `Remixer` |

Array ordering = intended display sequence, matching DDEX's `SequenceNumber`.

#### Identifiers

Industry standard identifiers:

```json
{
  "identifiers": {
    "isrc": "USABC1212346",
    "iswc": "T-123.456.789-0",
    "upc": "012345678905",
    "grid": "A12345A67890123456",
    "catalogueNumber": "TMPS005"
  }
}
```

| Property | Type | Level | Description |
| :--- | :---: | :---: | :--- |
| `isrc` | `string` | Track | International Standard Recording Code |
| `iswc` | `string` | Both | International Standard Musical Work Code |
| `upc` | `string` | Release | Universal Product Code |
| `grid` | `string` | Release | Global Release Identifier |
| `catalogueNumber` | `string` | Both | Label catalogue number |

#### Copyright

```json
{
  "copyright": {
    "pLine": { "year": 2026, "text": "TMPS" },
    "cLine": { "year": 2026, "text": "TMPS" }
  }
}
```

| Property | Type | Description |
| :--- | :---: | :--- |
| `pLine` | `object` | Sound recording copyright (℗): `year` + `text` |
| `cLine` | `object` | Composition copyright (©): `year` + `text` |

#### Lyrics

```json
{
  "lyrics": {
    "text": "Verse 1:\nFeel the bass drop...",
    "language": "en",
    "synced": false
  }
}
```

| Property | Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `text` | `string` | ✓ | Full lyrics |
| `language` | `string` | | ISO 639-1 code |
| `synced` | `boolean` | | Has timing data. Default: `false` |

#### Preview

```json
{ "preview": { "startMs": 30000, "durationMs": 30000 } }
```

| Property | Type | Description |
| :--- | :---: | :--- |
| `startMs` | `number` | Start time in milliseconds |
| `durationMs` | `number` | Duration in milliseconds |

Maps to DDEX ERN `<PreviewDetails>`.

#### Stems

Array of stem/multitrack files. Each follows LSP4's `assets` structure with an added `name`:

```json
{
  "stems": [
    {
      "name": "Vocals",
      "url": "ipfs://Qm.../vocals.wav",
      "fileType": "audio/wav",
      "verification": { "method": "keccak256(bytes)", "data": "0x..." }
    }
  ]
}
```

| Property | Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `name` | `string` | ✓ | Stem name (e.g., `Vocals`, `Drums`) |
| `url` | `string` | ✓ | URI to audio file |
| `fileType` | `string` | ✓ | MIME type |
| `verification` | `object` | | Same as LSP4 asset verification |

#### DDEX

Reference to a [DDEX ERN](https://ddex.net/standards/electronic-release-notification-message-suite/) XML file:

```json
{
  "ddex": {
    "url": "ipfs://Qm.../release-ern.xml",
    "version": "4.3.2",
    "verification": { "method": "keccak256(bytes)", "data": "0x..." }
  }
}
```

| Property | Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `url` | `string` | ✓ | URI to DDEX ERN XML |
| `version` | `string` | | ERN version |
| `verification` | `object` | | Same as LSP4 VerifiableURI |

### Full Metadata Examples

`LSP4Metadata` and `LSP33Metadata` have **separate data keys** and are typically stored as **separate JSON files**. Each data key points to its own [VerifiableURI].

> **Note:** The two files MAY be combined into a single JSON file with both `LSP4Metadata` and `LSP33Metadata` as top-level properties. In that case, both data keys would reference the same URI. This saves one upload and one fetch, but is not required.

#### Release / Album — LSP4Metadata File

Referenced by the LSP8 contract's `LSP4Metadata` data key:

```json
{
  "LSP4Metadata": {
    "name": "ledfut - single 20031400",
    "description": "A 2-track house single by ledfut, released on TMPS.",
    "links": [
      { "title": "Artist Website", "url": "https://ledfut.xyz" },
      { "title": "Spotify", "url": "https://open.spotify.com/album/..." }
    ],
    "icon": [
      {
        "width": 256,
        "height": 256,
        "url": "ipfs://QmIcon256.../icon.png",
        "verification": { "method": "keccak256(bytes)", "data": "0xabcd..." }
      }
    ],
    "images": [
      [
        {
          "width": 1024,
          "height": 1024,
          "url": "ipfs://QmCover1024.../cover.jpg",
          "verification": { "method": "keccak256(bytes)", "data": "0x1234..." }
        },
        {
          "width": 512,
          "height": 512,
          "url": "ipfs://QmCover512.../cover.jpg",
          "verification": { "method": "keccak256(bytes)", "data": "0x5678..." }
        }
      ]
    ],
    "assets": [],
    "attributes": [
      { "key": "Artist", "value": "ledfut", "type": "string" },
      { "key": "Release Type", "value": "Single", "type": "string" },
      { "key": "Release Date", "value": "2025-11-06", "type": "string" },
      { "key": "Primary Genre", "value": "House", "type": "string" },
      { "key": "Secondary Genre", "value": "Techno", "type": "string" },
      { "key": "Track Count", "value": "2", "type": "string" },
      { "key": "Label", "value": "TMPS", "type": "string" },
      { "key": "Language", "value": "eng", "type": "string" }
    ],
    "category": "Music"
  }
}
```

#### Release / Album — LSP33Metadata File

Referenced by the LSP8 contract's `LSP33Metadata` data key:

```json
{
  "LSP33Metadata": {
    "contributors": [
      { "name": "ledfut", "address": "0x1234...abcd", "roles": ["MainArtist", "Producer"] },
      { "name": "Studio Wizard", "address": "0xabcd...1234", "roles": ["Mastering Engineer"] }
    ],
    "identifiers": {
      "upc": "012345678905",
      "catalogueNumber": "TMPS005"
    },
    "copyright": {
      "pLine": { "year": 2025, "text": "TMPS" },
      "cLine": { "year": 2025, "text": "TMPS" }
    },
    "ddex": {
      "url": "ipfs://QmDdex.../release-ern.xml",
      "version": "4.3.2",
      "verification": { "method": "keccak256(bytes)", "data": "0x9abc..." }
    }
  }
}
```

#### Track — LSP4Metadata File

Referenced per-tokenId (via `setDataForTokenId`) or on the LSP7 contract's `LSP4Metadata` data key:

```json
{
  "LSP4Metadata": {
    "name": "20031400",
    "description": "Track 1 from 'ledfut - single 20031400'. A driving house cut.",
    "links": [],
    "icon": [
      {
        "width": 256,
        "height": 256,
        "url": "ipfs://QmTrackIcon.../icon.png",
        "verification": { "method": "keccak256(bytes)", "data": "0xdef0..." }
      }
    ],
    "images": [
      [
        {
          "width": 1024,
          "height": 1024,
          "url": "ipfs://QmTrackCover.../cover.jpg",
          "verification": { "method": "keccak256(bytes)", "data": "0x2345..." }
        }
      ]
    ],
    "assets": [
      {
        "url": "ipfs://QmAudio.../20031400.flac",
        "fileType": "audio/flac",
        "verification": { "method": "keccak256(bytes)", "data": "0x6789..." }
      }
    ],
    "attributes": [
      { "key": "Artist", "value": "ledfut", "type": "string" },
      { "key": "Track Number", "value": "1", "type": "string" },
      { "key": "Primary Genre", "value": "House", "type": "string" },
      { "key": "Release Date", "value": "2025-11-06", "type": "string" },
      { "key": "Duration", "value": "5:22", "type": "string" },
      { "key": "BPM", "value": "128", "type": "string" },
      { "key": "Key", "value": "Am", "type": "string" },
      { "key": "Explicit Content", "value": "NotExplicit", "type": "string" }
    ],
    "category": "Music"
  }
}
```

#### Track — LSP33Metadata File

Referenced per-tokenId (via `setDataForTokenId`) or on the LSP7 contract's `LSP33Metadata` data key:

```json
{
  "LSP33Metadata": {
    "contributors": [
      { "name": "ledfut", "address": "0x1234...abcd", "roles": ["MainArtist", "Producer", "Composer"] },
      { "name": "ampy", "address": "0x5678...efgh", "roles": ["Lyricist"] },
      { "name": "Studio Wizard", "address": "0xabcd...1234", "roles": ["MixingEngineer", "MasteringEngineer"] }
    ],
    "identifiers": {
      "isrc": "USABC2512345",
      "iswc": "T-123.456.789-0",
      "catalogueNumber": "TMPS005-01"
    },
    "copyright": {
      "pLine": { "year": 2025, "text": "TMPS" },
      "cLine": { "year": 2025, "text": "TMPS" }
    },
    "lyrics": {
      "text": "Verse 1:\nFeel the bass drop low tonight\nMoving through the neon light\n\nChorus:\nWe don't stop, we don't stop...",
      "language": "en",
      "synced": false
    },
    "preview": {
      "startMs": 45000,
      "durationMs": 30000
    },
    "stems": [
      {
        "name": "Drums",
        "url": "ipfs://QmStems.../drums.wav",
        "fileType": "audio/wav",
        "verification": { "method": "keccak256(bytes)", "data": "0xaaaa..." }
      },
      {
        "name": "Bass",
        "url": "ipfs://QmStems.../bass.wav",
        "fileType": "audio/wav",
        "verification": { "method": "keccak256(bytes)", "data": "0xbbbb..." }
      },
      {
        "name": "Vocals",
        "url": "ipfs://QmStems.../vocals.wav",
        "fileType": "audio/wav",
        "verification": { "method": "keccak256(bytes)", "data": "0xcccc..." }
      },
      {
        "name": "Synths",
        "url": "ipfs://QmStems.../synths.wav",
        "fileType": "audio/wav",
        "verification": { "method": "keccak256(bytes)", "data": "0xdddd..." }
      }
    ],
    "ddex": {
      "url": "ipfs://QmDdex.../track1-ern.xml",
      "version": "4.3.2"
    }
  }
}
```

### Lifecycle

#### Creating a Release

1. Deploy an LSP8 with `LSP4TokenType = 2`.
2. Upload a JSON file with both `LSP4Metadata` and `LSP33Metadata`.
3. Set `LSP4Metadata` and `LSP33Metadata` on the LSP8 pointing to the same file.
4. Set `SupportedStandards:LSP33MusicNFT`.

#### Adding a Track (Metadata Only)

1. Mint a new tokenId on the LSP8.
2. Upload a track JSON file with both metadata sections.
3. Call `setDataForTokenId(tokenId, LSP4MetadataKey, ...)` and `setDataForTokenId(tokenId, LSP33MetadataKey, ...)`.

#### Making a Track Ownable

1. Deploy an LSP7 implementing LSP34, with `LSP34OwnershipSource` = `(LSP8Address, tokenId)`.
2. Set `LSP8ReferenceContract` on the LSP7 to `(LSP8Address, tokenId)`.
3. Set `LSP4Metadata` and `LSP33Metadata` on the LSP7.
4. On the LSP8, set `LSP33OwnableTrackToken` for the tokenId to the LSP7 address.
5. From this point, the LSP8 routes metadata reads and writes to the LSP7.

#### Minting Ownable Units

1. The artist (resolved via LSP34) calls `mint(...)` on the LSP7.
2. Units can be transferred, traded, or held by collectors.
3. Metadata updates go through `LSP8.setDataForTokenId` (forwarded) or `LSP7.setData` directly.

## Rationale

### Composability Over Complexity

This standard composes existing LSP primitives rather than defining new token contracts. Existing tooling, indexers, and interfaces work out of the box.

### Separate Metadata Files

Using separate data keys and files for `LSP4Metadata` (display) and `LSP33Metadata` (structured music data) provides backwards compatibility (any LSP4-aware interface works), clean separation of concerns, independent extensibility, and the option to combine both into a single file when desired.

### Transparent Data Routing

Making the LSP8 a transparent router for linked LSP7s gives artists a unified interface, prevents metadata drift between contracts, and degrades gracefully when no LSP7 is linked.

### Parent Collection Authorization

The LSP7 trusting its parent LSP8 for `setData` calls is safe because the LSP8 verifies that the caller is the `tokenOwnerOf(tokenId)` before forwarding, both resolve to the same owner (the artist) via LSP34, and the LSP7 only trusts the specific LSP8 set immutably in its constructor.

### Artist vs Collector Ownership

This standard separates two distinct ownership layers:

- **Artist ownership** (LSP8 `tokenOwnerOf`): The artist holds the LSP8 tokenId. This grants control over the track — setting metadata, linking an LSP7, and minting collectible units. The LSP8 tokenId is **not** a tradeable collectible; it represents authorship and creative rights.
- **Collector ownership** (LSP7 `balanceOf`): Fans and collectors hold fungible LSP7 units. These represent ownership of copies or shares of a track. Holding units does **not** grant any control over metadata or minting — only the ability to transfer or trade the units themselves.

This separation ensures that the artist always controls their work, regardless of how many collectible units are in circulation or who holds them.

### Metadata-Only Tracks

Not every track needs ownable editions. A track without `LSP33OwnableTrackToken` is simply metadata on an LSP8 tokenId — useful for proving creation or making tracks discoverable.

## Implementation

An implementation can be found in [lukso-network/lsp-smart-contracts#1088](https://github.com/lukso-network/lsp-smart-contracts/pull/1088).

ERC725Y JSON Schema `LSP33MusicNFT`:

```json
[
  {
    "name": "SupportedStandards:LSP33MusicNFT",
    "key": "0xeafec4d89fa9619884b60000f43ae543d533ae7389b5791d7870aadfdcff2ca4",
    "keyType": "Mapping",
    "valueType": "bytes4",
    "valueContent": "0x3cd46617"
  },
  {
    "name": "LSP33OwnableTrackToken",
    "key": "0x869105ad93b0c40a24a19b5c3260f597737cf3625302cb5ab26a82e14468ad91",
    "keyType": "Singleton",
    "valueType": "address",
    "valueContent": "Address"
  },
  {
    "name": "LSP33Metadata",
    "key": "0xbecee5b67fa43fb2e96c7c6ccdc23007fffc2b24205416b2122cd7b8d8ef48fa",
    "keyType": "Singleton",
    "valueType": "bytes",
    "valueContent": "VerifiableURI"
  }
]
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

[ERC165]: https://eips.ethereum.org/EIPS/eip-165
[ERC725Y]: https://github.com/ERC725Alliance/ERC725/blob/develop/docs/ERC-725.md#erc725y
[LSP2]: ./LSP-2-ERC725YJSONSchema.md
[LSP4]: ./LSP-4-DigitalAsset-Metadata.md
[LSP7]: ./LSP-7-DigitalAsset.md
[LSP8]: ./LSP-8-IdentifiableDigitalAsset.md
[LSP8Ref]: ./LSP-8-IdentifiableDigitalAsset.md#lsp8referencecontract
[LSP34]: ./LSP-34-ExternalOwnership.md
