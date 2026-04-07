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

A composable standard for representing music as digital assets on LUKSO, with two equally valid paths: a **standalone [LSP7]** for a single ownable track, or an **[LSP8] collection** where each `tokenId` is a track (optionally linked to an LSP7 for collectible units).

## Abstract

LSP33 defines how music — tracks and releases — is represented on-chain by composing existing LUKSO primitives ([LSP4], [LSP7], [LSP8], [LSP34]) rather than introducing new token contracts.

It supports two deployment paths:

1. **Standalone LSP7 track.** A single LSP7 contract represents one track. Fans hold fungible units. There is no parent collection.
2. **LSP8 collection.** An LSP8 represents a release/album. Each `tokenId` is a track. A track tokenId can be metadata-only, or linked to its own LSP7 for collectible units via the `LSP33OwnableTrackToken` data key.

In both paths:

- Display metadata lives in `LSP4Metadata` (with `category: "Music"`).
- Extended music metadata (contributors, identifiers, copyright, lyrics, preview, stems, DDEX) lives in `LSP33Metadata` — a separate data key with its own JSON file.
- The **artist** always controls metadata and minting. Collectors only hold and trade LSP7 units.

In the LSP8 collection path, linked LSP7s use [LSP34] to derive their owner from the LSP8 `tokenOwnerOf`, and the LSP8 acts as a **transparent router** for metadata reads and writes — giving the artist a single interface for everything.

### Two Ownership Layers

LSP33 separates authorship from collectibility:

| | **Artist Ownership** | **Collector Ownership** |
|---|---|---|
| **What** | LSP8 `tokenId` (Path 2) <br> or LSP7 contract `owner()` (Path 1) | LSP7 `balanceOf(address)` |
| **Who** | The artist or rights holder | Fans, collectors |
| **Controls** | Metadata, minting, linking | Transferring/trading units |
| **Tradeable?** | No — represents authorship | Yes — fungible units |

**Holding LSP7 units never grants control over metadata or minting.** That stays with the artist.

## Motivation

Music on the blockchain lacks a standardized way to represent releases, tracks, and collectibles in one framework. Artists need:

1. **Provenance** — prove creation at a point in time, even without selling.
2. **Composability** — use existing LSP standards rather than new token contracts.
3. **Flexibility** — a single track can be standalone, or part of a release. Tracks can be metadata-only or ownable.
4. **Artist Control** — metadata and minting stay with the artist, regardless of who holds units.
5. **Industry Compatibility** — bridge on-chain data with DDEX, ISRC, ISWC, GRid for DSP interoperability.

## Specification

### Path 1 — Standalone LSP7 Track

A single track deployed as its own LSP7 contract. No parent collection.

```
┌────────────────────────────────────────────────┐
│  LSP7 Track Contract                           │
│  owner() = artist (standard ERC173)            │
│  LSP4Metadata  = track display metadata        │
│  LSP33Metadata = track music metadata          │
│  LSP4TokenType = 1 (NFT/NDT)                   │
│  decimals() = 0                                │
│  SupportedStandards:LSP33MusicNFT              │
│                                                │
│  mint()     → only artist (owner)              │
│  setData()  → only artist (owner)              │
│  balanceOf(collector) → units owned            │
└────────────────────────────────────────────────┘
```

**Requirements:**

- MUST set `LSP4TokenType` to `1`.
- SHOULD set `decimals()` to `0` (non-divisible).
- MUST set `SupportedStandards:LSP33MusicNFT`.
- MUST set `LSP4Metadata` and `LSP33Metadata` on the contract.
- MUST restrict `setData` and minting to the contract `owner()`.
- MUST NOT set `LSP8ReferenceContract` or `LSP34OwnershipSource` (Path 1 has no parent).

Use this path when a track stands alone and has no album/release context.

### Path 2 — LSP8 Collection (Release / Album)

An LSP8 contract represents a release. Each `tokenId` is a track. Tracks can be metadata-only, or linked to their own LSP7 for ownable units.

```
┌──────────────────────────────────────────────────────────────────┐
│  LSP8 Release Contract                                          │
│  owner() = artist (standard ERC173)                             │
│  LSP4Metadata  = release display metadata                       │
│  LSP33Metadata = release music metadata                         │
│  LSP4TokenType = 2 (Collection)                                 │
│  LSP8TokenIdFormat = 0 (uint256)                                │
│  SupportedStandards:LSP33MusicNFT                               │
│                                                                  │
│  tokenId 1 ─── Track 1 (metadata-only)                          │
│    ├── LSP4Metadata  (per-tokenId)                               │
│    └── LSP33Metadata (per-tokenId)                               │
│                                                                  │
│  tokenId 2 ─── Track 2 (ownable, linked to LSP7)                │
│    ├── LSP4Metadata                                              │
│    ├── LSP33Metadata                                             │
│    └── LSP33OwnableTrackToken ──► LSP7 Track Contract            │
│           ▲  read/write proxy            │                       │
│           └──────────────────────────────┘                       │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│  Linked LSP7 Track Contract                                     │
│  owner() = artist (via LSP34 → LSP8.tokenOwnerOf(tokenId))      │
│  LSP8ReferenceContract ──► (LSP8 address, tokenId)               │
│  LSP34OwnershipSource  ──► (LSP8 address, tokenId)               │
│  LSP4Metadata  = track display metadata                          │
│  LSP33Metadata = track music metadata                            │
│                                                                  │
│  mint()    → only artist (resolved owner)                        │
│  setData() → artist OR parent LSP8 (for forwarded writes)        │
│  balanceOf(collector) → units owned                              │
└──────────────────────────────────────────────────────────────────┘
```

**LSP8 requirements:**

- MUST set `LSP4TokenType` to `2` (Collection).
- SHOULD set `LSP8TokenIdFormat` to `0` (uint256). TokenIds SHOULD be sequential `uint256` starting from `1`.
- MUST set `SupportedStandards:LSP33MusicNFT`.
- MUST set `LSP4Metadata` and `LSP33Metadata` on the contract for release data.
- Per-track data is set via `setDataForTokenId`.
- MUST implement the routing behavior below for tokenIds that have `LSP33OwnableTrackToken` set.

**Linked LSP7 requirements (optional per track):**

- MUST implement [LSP34] with `LSP34OwnershipSource` = `(LSP8 address, tokenId)`.
- MUST set `LSP8ReferenceContract` to `(LSP8 address, tokenId)`.
- MUST cache the parent LSP8 address as an `immutable` variable (set in constructor). This avoids repeated ERC725Y reads on every `owner()` call and ensures the parent reference cannot be changed.
- MUST resolve `owner()` via LSP34.
- MUST restrict minting to the resolved owner.
- MUST accept `setData` and `setDataBatch` calls from either the resolved owner OR the cached parent LSP8 address (see [Parent Collection Authorization](#parent-collection-authorization)).
- SHOULD set `decimals()` to `0`.
- SHOULD set `LSP4TokenType` to `1`.

Where a track is **metadata-only**, only per-tokenId `LSP4Metadata` / `LSP33Metadata` are set on the LSP8 — no LSP7, no `LSP33OwnableTrackToken`.

### Data Routing (LSP8 ↔ Linked LSP7)

When `LSP33OwnableTrackToken` is set for a tokenId, the LSP8 MUST act as a transparent router for the metadata data keys `LSP4Metadata` and `LSP33Metadata`. All other data keys are read/written locally on the LSP8's tokenId storage.

#### Reads: `getDataForTokenId`

```
getDataForTokenId(tokenId, dataKey)
  │
  ├── LSP33OwnableTrackToken set?
  │     │
  │     NO  ──► return LSP8 local tokenId storage
  │     │
  │     YES ──► is dataKey LSP4Metadata or LSP33Metadata?
  │              │
  │              NO  ──► return LSP8 local tokenId storage
  │              │
  │              YES ──► call LSP7.getData(dataKey)
  │                        ├── success ──► return LSP7 value
  │                        └── revert  ──► fallback to LSP8 local tokenId storage
```

The fallback keeps metadata available if the linked LSP7 becomes unreachable.

#### Writes: `setDataForTokenId`

```
setDataForTokenId(tokenId, dataKey, value)
  │
  ├── require: msg.sender == tokenOwnerOf(tokenId)   // NOT contract owner()
  │
  ├── LSP33OwnableTrackToken set?
  │     │
  │     NO  ──► write to LSP8 local tokenId storage, emit TokenIdDataChanged
  │     │
  │     YES ──► is dataKey LSP4Metadata or LSP33Metadata?
  │              │
  │              NO  ──► write to LSP8 local tokenId storage, emit TokenIdDataChanged
  │              │
  │              YES ──► call LSP7.setData(dataKey, value)
  │                        ├── success ──► LSP7 emits DataChanged; LSP8 emits NO event
  │                        └── revert  ──► whole call reverts
```

**Access control.** `setDataForTokenId` is gated by `tokenOwnerOf(tokenId)`, **not** the contract-level `owner()`. Since the linked LSP7 resolves `owner()` via LSP34 to the same `tokenOwnerOf(tokenId)`, both paths converge on the artist.

**Events.** When a write is forwarded, the LSP8 does **not** emit `TokenIdDataChanged` — the data is not stored on the LSP8. The LSP7 emits its own `DataChanged`. Indexers reading per-tokenId metadata MUST follow the `LSP33OwnableTrackToken` link and subscribe to `DataChanged` on the LSP7. `TokenIdDataChanged` events from the LSP8 only reflect locally stored data (non-metadata keys, or tokenIds without a linked LSP7).

#### Parent Collection Authorization

A linked LSP7 MUST accept `setData` / `setDataBatch` calls if **either**:

1. `msg.sender == owner()` (the resolved artist via LSP34), **or**
2. `msg.sender == immutableParentLSP8` (the cached parent address set in the constructor).

This is safe because the LSP8 already enforces `tokenOwnerOf(tokenId)` before forwarding, and both paths resolve to the same artist address. The LSP7 only trusts the specific LSP8 set immutably at deployment.

#### Bidirectional Link Verification

When setting `LSP33OwnableTrackToken` for a tokenId, the LSP8 SHOULD verify that the linked LSP7's `LSP8ReferenceContract` points back to `(address(this), tokenId)`. If verification fails, the call SHOULD revert.

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

Indicates that the contract implements LSP33. MUST be set on a standalone LSP7 (Path 1) or on the LSP8 collection (Path 2).

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

**Path 2 only.** Set per-tokenId via `setDataForTokenId` on the LSP8. Points to an LSP7 contract representing ownable units of that track.

- **Not set** → the tokenId is metadata-only.
- **Set** → the referenced LSP7 MUST satisfy the linked LSP7 requirements above.

_Requirements:_

- MUST only be settable by `tokenOwnerOf(tokenId)`.
- SHOULD be verified bidirectionally before being set.
- SHOULD NOT be changed once set, to preserve ownership integrity for existing LSP7 holders.

The value MAY be encoded as either 20 bytes (`abi.encodePacked(address)`) or 32 bytes (`abi.encode(address)`). Implementations MUST handle both.

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

References the extended music metadata JSON file. A separate file from `LSP4Metadata`, stored under its own data key.

Can be set:

- On a **standalone LSP7 contract** (Path 1, track-level).
- On the **LSP8 contract** (Path 2, release-level).
- Per **tokenId** via `setDataForTokenId` on the LSP8 (Path 2, track-level).
- On a **linked LSP7 contract** (Path 2, track-level — accessed via the LSP8 router).

### Metadata Model

LSP33 uses **two separate data keys** for metadata:

- **`LSP4Metadata`** (from [LSP4]) — human-readable: name, description, artwork, links, audio assets, attributes. MUST include top-level `"category": "Music"`.
- **`LSP33Metadata`** (this standard) — structured: contributors, identifiers, copyright, lyrics, preview, stems, DDEX.

Each key is typically a **separate JSON file**. They MAY be combined into a single JSON document with both keys as top-level properties — in which case both data keys reference the same URI. Combining is an optimization, not a requirement.

#### Release / Album — `LSP4Metadata` Attributes

Set on the LSP8 contract (Path 2).

| Attribute | Required | Description |
| :--- | :---: | :--- |
| `Artist` | ✓ | Primary artist or band name |
| `Release Type` | ✓ | `Single`, `EP`, `Album`, `Compilation` |
| `Release Date` | ✓ | ISO 8601 (`YYYY-MM-DD`) |
| `Primary Genre` | ✓ | Primary genre |
| `Track Count` | ✓ | Total number of tracks |
| `Label` |  | Record label |
| `Secondary Genre` |  | Secondary genre |
| `Language` |  | ISO 639-2 code |

#### Track — `LSP4Metadata` Attributes

Set on a standalone LSP7 (Path 1), on an LSP8 tokenId (Path 2), or on a linked LSP7 (Path 2).

| Attribute | Required | Description |
| :--- | :---: | :--- |
| `Artist` | ✓ | Track artist |
| `Track Number` | ✓ | Position in release (use `1` for standalone) |
| `Primary Genre` | ✓ | Primary genre |
| `Release Date` | ✓ | ISO 8601 |
| `Duration` |  | e.g. `3:35` or `PT3M35S` |
| `Disc Number` |  | For multi-disc releases |
| `Explicit Content` |  | `Explicit`, `NotExplicit`, `Cleaned` |
| `BPM` |  | Beats per minute |
| `Key` |  | Musical key (e.g. `Cm`, `F#`) |
| `Secondary Genre` |  | Secondary genre |
| `Language` |  | ISO 639-2 code |

#### `LSP33Metadata` Fields

| Field | Level | Description |
| :--- | :---: | :--- |
| `contributors` | Release & Track | Ordered list of contributors with roles |
| `identifiers` | Release & Track | Industry identifiers (ISRC, ISWC, UPC, GRid, catalogue) |
| `copyright` | Release & Track | `pLine` (℗) and `cLine` (©) |
| `lyrics` | Track | Full lyrics text, language, synced flag |
| `preview` | Track | `startMs`, `durationMs` |
| `stems` | Track | Array of stem files (name, url, fileType, verification) |
| `ddex` | Release & Track | DDEX ERN XML reference |

##### Contributors

Ordered array. Array order = intended display sequence (maps to DDEX `SequenceNumber`).

```json
{
  "contributors": [
    { "name": "ledfut", "address": "0x1234...abcd", "roles": ["MainArtist", "Producer"] },
    { "name": "ampy", "address": "0x5678...efgh", "roles": ["Composer", "Lyricist"] },
    { "name": "Big Boss", "roles": ["ExecutiveProducer"] }
  ]
}
```

| Property | Type | Required |
| :--- | :---: | :---: |
| `name` | `string` | ✓ |
| `address` | `string` (UP address) |  |
| `email` | `string` |  |
| `roles` | `string[]` | ✓ |

Common roles (extensible — any role MAY be used):

| Role | DDEX Equivalent |
| :--- | :--- |
| `MainArtist` | `MainArtist` |
| `FeaturedArtist` | `FeaturedArtist` |
| `Producer` | `StudioProducer` |
| `ExecutiveProducer` | `ExecutiveProducer` |
| `Composer` | `Composer` |
| `Lyricist` | `Lyricist` |
| `MixingEngineer` | `MixingEngineer` |
| `MasteringEngineer` | `MasteringEngineer` |
| `Remixer` | `Remixer` |

##### Identifiers

```json
{
  "identifiers": {
    "isrc": "USABC2512345",
    "iswc": "T-123.456.789-0",
    "upc": "012345678905",
    "grid": "A12345A67890123456",
    "catalogueNumber": "TMPS005"
  }
}
```

| Property | Level | Description |
| :--- | :---: | :--- |
| `isrc` | Track | International Standard Recording Code |
| `iswc` | Both | International Standard Musical Work Code |
| `upc` | Release | Universal Product Code |
| `grid` | Release | Global Release Identifier |
| `catalogueNumber` | Both | Label catalogue number |

##### Copyright

```json
{
  "copyright": {
    "pLine": { "year": 2026, "text": "TMPS" },
    "cLine": { "year": 2026, "text": "TMPS" }
  }
}
```

`pLine` = sound recording copyright (℗). `cLine` = composition copyright (©).

##### Lyrics

```json
{
  "lyrics": {
    "text": "Verse 1:\nFeel the bass drop...",
    "language": "en",
    "synced": false
  }
}
```

##### Preview

```json
{ "preview": { "startMs": 30000, "durationMs": 30000 } }
```

Maps to DDEX ERN `<PreviewDetails>`.

##### Stems

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

##### DDEX

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

### Full Metadata Examples

LSP33 uses **four separate files** for a release with tracks: one `LSP4Metadata` and one `LSP33Metadata` for the release, and one of each per track. They MAY be combined into fewer files, but are shown separate here.

For **Path 1 (standalone LSP7)**, only the two track files are needed (the LSP7 holds `LSP4Metadata` + `LSP33Metadata` directly).

#### Release — `LSP4Metadata` File

Referenced by the LSP8 contract's `LSP4Metadata` data key.

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
        }
      ]
    ],
    "assets": [],
    "attributes": [
      { "key": "Artist", "value": "ledfut", "type": "string" },
      { "key": "Release Type", "value": "Single", "type": "string" },
      { "key": "Release Date", "value": "2025-11-06", "type": "string" },
      { "key": "Primary Genre", "value": "House", "type": "string" },
      { "key": "Track Count", "value": "2", "type": "string" },
      { "key": "Label", "value": "TMPS", "type": "string" },
      { "key": "Language", "value": "eng", "type": "string" }
    ],
    "category": "Music"
  }
}
```

#### Release — `LSP33Metadata` File

Referenced by the LSP8 contract's `LSP33Metadata` data key.

```json
{
  "LSP33Metadata": {
    "contributors": [
      { "name": "ledfut", "address": "0x1234...abcd", "roles": ["MainArtist", "Producer"] },
      { "name": "Studio Wizard", "address": "0xabcd...1234", "roles": ["MasteringEngineer"] }
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

#### Track — `LSP4Metadata` File

Referenced on a standalone LSP7 (Path 1), or per-tokenId via `setDataForTokenId` / on a linked LSP7 (Path 2).

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

#### Track — `LSP33Metadata` File

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

#### Path 1 — Deploying a Standalone Track

1. Deploy an LSP7 with `LSP4TokenType = 1`, `decimals = 0`, and the artist as `owner()`.
2. Upload `LSP4Metadata` and `LSP33Metadata` JSON files.
3. Set `LSP4Metadata`, `LSP33Metadata`, and `SupportedStandards:LSP33MusicNFT` on the LSP7.
4. The artist mints units to sell to collectors.

#### Path 2 — Creating a Release

1. Deploy an LSP8 with `LSP4TokenType = 2` and the artist as `owner()`.
2. Upload release `LSP4Metadata` and `LSP33Metadata` files.
3. Set `LSP4Metadata`, `LSP33Metadata`, and `SupportedStandards:LSP33MusicNFT` on the LSP8.

#### Path 2 — Adding a Metadata-Only Track

1. Mint a tokenId on the LSP8 (owner = artist).
2. Upload track `LSP4Metadata` and `LSP33Metadata` files.
3. Call `setDataForTokenId(tokenId, LSP4Metadata, ...)` and `setDataForTokenId(tokenId, LSP33Metadata, ...)`.

#### Path 2 — Making a Track Ownable

1. Deploy an LSP7 with:
   - `LSP34OwnershipSource` = `(LSP8 address, tokenId)`
   - `LSP8ReferenceContract` = `(LSP8 address, tokenId)`
   - The parent LSP8 address cached as an `immutable`.
2. Set `LSP4Metadata` and `LSP33Metadata` on the LSP7.
3. On the LSP8, call `setDataForTokenId(tokenId, LSP33OwnableTrackToken, lsp7Address)`. The LSP8 SHOULD verify the bidirectional link first.
4. From now on, the LSP8 routes metadata reads and writes for that tokenId to the LSP7.
5. The artist mints LSP7 units for collectors.

## Rationale

### Two Paths, One Standard

Not every track needs a parent release. A standalone LSP7 is the simplest possible "music NFT" — one track, one contract, units for collectors. The LSP8 collection path exists when tracks belong together in a release, or when the artist wants a single on-chain entity representing the album. Both paths share the same metadata model and the same `SupportedStandards` marker so tooling can recognize them uniformly.

### Composability Over New Contracts

LSP33 composes LSP4, LSP7, LSP8, and LSP34 rather than defining new token types. Existing wallets, indexers, and marketplaces already understand these primitives.

### Separate Metadata Files

`LSP4Metadata` and `LSP33Metadata` are separate data keys so that any LSP4-aware interface works out of the box, concerns stay cleanly separated, and each can evolve independently. They may still be combined into one file when desired.

### Transparent Data Routing

In Path 2, making the LSP8 a transparent router for linked LSP7 metadata gives artists a unified interface, prevents metadata drift between contracts, and degrades gracefully to LSP8 local fallback if the LSP7 becomes unreachable.

### Parent Collection Authorization

The linked LSP7 trusting its parent LSP8 for `setData` is safe because the LSP8 enforces `tokenOwnerOf(tokenId)` before forwarding, both the LSP8 and the LSP7 resolve the same artist address via LSP34, and the LSP7 only trusts the specific LSP8 set immutably at construction.

### Artist vs Collector Ownership

Separating authorship (LSP8 tokenId or standalone LSP7 `owner()`) from collectibility (LSP7 `balanceOf`) ensures the artist always controls their work. LSP7 holders can freely trade units without ever touching metadata or minting — which is what Konstantin's original concern about LSP8 tokenId transfers was about, and why the access model is built around `tokenOwnerOf` rather than generic token holders.

### Metadata-Only Tracks (Path 2)

A track without `LSP33OwnableTrackToken` is simply metadata on an LSP8 tokenId — useful for proving creation or making tracks discoverable without issuing collectibles.

## Implementation

A reference implementation can be found in [lukso-network/lsp-smart-contracts#1088](https://github.com/lukso-network/lsp-smart-contracts/pull/1088).

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
[LSP34]: ./LSP-34-ExternalOwnership.md
