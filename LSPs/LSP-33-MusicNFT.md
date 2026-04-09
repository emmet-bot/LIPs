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

A composable standard for music on LUKSO. Two building blocks — an [LSP8] release and an [LSP7] track — that plug together in any order. Start with just a track, just a release, or both. Link them whenever you want.

## Abstract

LSP33 represents music as digital assets by composing existing LUKSO primitives ([LSP4], [LSP7], [LSP8], [LSP34]) instead of defining new token contracts. It has two building blocks:

- An **[LSP8] collection** represents a release or album. Each `tokenId` is a track. Tracks can be metadata-only.
- An **[LSP7] token** represents ownable units (collectible editions) of a single track.

The two can be used independently or linked. Linking is done with a single data key, `LSP33OwnableTrackToken`, that points from an LSP8 tokenId to an LSP7. Once linked:

- The LSP8 acts as a **transparent router** for track metadata — reads proxy to the LSP7, writes forward to it.
- The LSP7 uses [LSP34] to derive its `owner()` from `LSP8.tokenOwnerOf(tokenId)`, so the artist controls both from a single identity.

This makes four workflows all work out of the box — start with a release, start with a track, add the other later, or do both from day one.

### Artist vs Collector Ownership

LSP33 separates authorship from collectibility:

| | **Artist** | **Collector** |
|---|---|---|
| **What they hold** | LSP8 `tokenId` and/or LSP7 `owner()` | LSP7 units (`balanceOf`) |
| **Can do** | Set metadata, mint, link LSP7 ↔ LSP8 | Transfer / trade units |
| **Tradeable?** | No — represents authorship | Yes — fungible units |

Holding LSP7 units never grants control over metadata or minting. That always stays with the artist.

## Motivation

Music on-chain needs a way to represent releases, tracks, and collectibles in one composable framework — with **provenance** (prove creation without selling), **artist control** (metadata + minting stay with the artist regardless of who holds units), **flexibility** (start anywhere, extend later), and **industry compatibility** (DDEX, ISRC, ISWC, GRid for DSP interoperability). LSP33 achieves this without introducing a new token type.

## Specification

### Building Blocks

#### LSP8 Release Collection

An LSP8 contract represents a release or album. Each `tokenId` is a track.

- MUST set `LSP4TokenType` = `2` (Collection).
- MUST set `SupportedStandards:LSP33MusicNFT`.
- MUST set `LSP4Metadata` and `LSP33Metadata` on the contract for release-level data.
- SHOULD set `LSP8TokenIdFormat` = `0` (uint256). TokenIds SHOULD be sequential starting from `1`.
- Per-track metadata is set via `setDataForTokenId(tokenId, LSP4Metadata|LSP33Metadata, value)`.
- If any tokenId has `LSP33OwnableTrackToken` set, the LSP8 MUST implement the [routing behavior](#data-routing) below.

A tokenId with no `LSP33OwnableTrackToken` is a **metadata-only track** — useful for proving creation or cataloguing without issuing collectibles.

#### LSP7 Ownable Track

An LSP7 contract represents ownable units of a single track.

- MUST set `SupportedStandards:LSP33MusicNFT`.
- MUST set `LSP4Metadata` and `LSP33Metadata` on the contract.
- SHOULD set `LSP4TokenType` = `1` (NFT/NDT) and `decimals()` = `0`.
- MUST implement [LSP34]. When `LSP34OwnershipSource` is **unset**, `owner()` falls back to standard [ERC173] (the LSP7 is standalone). When **set** to `(lsp8Address, tokenId)`, `owner()` resolves to `LSP8.tokenOwnerOf(tokenId)` (the LSP7 is linked).
- MUST restrict minting and `setData` to the resolved `owner()`, plus the [parent LSP8 authorization](#parent-authorization) below.

An LSP7 is deployed as a plain ERC173-owned contract — its constructor takes a name, symbol, initial owner (the artist), and divisibility flag. It does **not** need to know about any parent LSP8 at deployment time. Linking is done entirely via data-key writes (see below), which means the same LSP7 bytecode supports every workflow — standalone, release-first, track-first, or both-at-once — without redeployment. An LSP7 intended to join a specific collection SHOULD additionally set `LSP8ReferenceContract` = `(lsp8Address, tokenId)` when it is linked, so the LSP8 can perform bidirectional verification.

### Linking (Plug and Play)

Linking is a single write per side:

1. On the LSP7: set `LSP34OwnershipSource` = `(lsp8Address, tokenId)`.
2. On the LSP8: call `setDataForTokenId(tokenId, LSP33OwnableTrackToken, lsp7Address)`.

Both writes MUST be authorized by the artist (the current `tokenOwnerOf(tokenId)` on the LSP8 side, and the current `owner()` on the LSP7 side — which becomes the same address after linking). The LSP8 SHOULD verify the [bidirectional link](#bidirectional-link-verification) before committing.

Because linking is just two data-key writes, any of these workflows is valid:

1. **Release-first.** Deploy an LSP8, add metadata-only tracks. Later, deploy an LSP7 for any track and link it — that track becomes ownable without touching the others.
2. **Track-first.** Deploy a standalone LSP7 (LSP34 source unset → ERC173 owner = artist). Later, deploy or reuse an LSP8, mint a tokenId for this track, and link. The LSP7 now derives its owner from the LSP8 tokenId.
3. **Both at once.** Deploy LSP8 + LSP7 in one transaction and link immediately.
4. **Pure standalone.** Deploy only an LSP7. It never has to join a collection.

### Data Routing

When `LSP33OwnableTrackToken` is set for a tokenId, the LSP8 MUST act as a transparent router for the data keys `LSP4Metadata` and `LSP33Metadata`. All other data keys are read/written locally on the LSP8's tokenId storage.

**Reads — `getDataForTokenId(tokenId, dataKey)`:**

```
LSP33OwnableTrackToken unset OR dataKey not LSP4/LSP33 Metadata
  → return LSP8 local tokenId storage

otherwise
  → call LSP7.getData(dataKey)
     success → return LSP7 value
     revert  → fall back to LSP8 local tokenId storage
```

The fallback keeps metadata available if the linked LSP7 becomes unreachable.

**Writes — `setDataForTokenId(tokenId, dataKey, value)`:**

```
require: msg.sender == tokenOwnerOf(tokenId)   // NOT the contract owner()

LSP33OwnableTrackToken unset OR dataKey not LSP4/LSP33 Metadata
  → write to LSP8 local tokenId storage, emit TokenIdDataChanged

otherwise
  → call LSP7.setData(dataKey, value)
     success → LSP7 emits DataChanged; LSP8 emits no event
     revert  → whole call reverts
```

**Access control.** `setDataForTokenId` is gated by `tokenOwnerOf(tokenId)`, not the contract-level `owner()`. Because a linked LSP7 resolves `owner()` via LSP34 to the same address, both write paths converge on the artist.

**Events.** The LSP8 does **not** emit `TokenIdDataChanged` for forwarded writes — the data lives on the LSP7, which emits its own `DataChanged`. Indexers MUST follow the `LSP33OwnableTrackToken` link and subscribe to `DataChanged` on the LSP7. `TokenIdDataChanged` on the LSP8 only reflects locally stored data.

#### Parent Authorization

A linked LSP7 MUST accept `setData` / `setDataBatch` if **either**:

1. `msg.sender == owner()` (the artist, via LSP34), **or**
2. `msg.sender` equals the parent LSP8 address decoded from the LSP7's current `LSP34OwnershipSource` data key.

This is safe because the LSP8 already enforces `tokenOwnerOf(tokenId)` before forwarding, both paths resolve to the same artist, and the LSP7 only trusts whichever LSP8 the artist has (via their own `setData` call) pointed `LSP34OwnershipSource` at. Writing `LSP34OwnershipSource` itself is gated by the same `onlyOwnerOrParentCollection` check, so only the current resolved owner can change which parent the LSP7 trusts.

#### Bidirectional Link Verification

Before setting `LSP33OwnableTrackToken` for a tokenId, the LSP8 SHOULD verify that the referenced LSP7's `LSP8ReferenceContract` points back to `(address(this), tokenId)`. If verification fails, the call SHOULD revert.

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

MUST be set on any LSP33 contract — LSP8 release or LSP7 track.

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

Set per-tokenId on the LSP8 via `setDataForTokenId`, pointing to a linked LSP7.

- **Unset** → the tokenId is metadata-only.
- **Set** → the referenced LSP7 must meet the [LSP7 building-block requirements](#lsp7-ownable-track).

Requirements:

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

References the extended music metadata JSON file (see below). Can be set at the contract level on an LSP7 or LSP8, or per-tokenId on an LSP8.

### Metadata Model

LSP33 uses two data keys:

- **`LSP4Metadata`** (from [LSP4]) — human-readable: name, description, artwork, links, audio assets, attributes. MUST include top-level `"category": "Music"`.
- **`LSP33Metadata`** (this standard) — structured music data: contributors, identifiers, copyright, lyrics, preview, stems, DDEX.

Each key is typically a **separate JSON file**. They MAY be combined into a single JSON document with both keys as top-level properties (in which case both data keys reference the same URI) — an optimization, not a requirement.

#### LSP4Metadata Attributes

**Release level** (set on the LSP8 contract):

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

**Track level** (set on an LSP7 contract, or per-tokenId on an LSP8):

| Attribute | Required | Description |
| :--- | :---: | :--- |
| `Artist` | ✓ | Track artist |
| `Track Number` | ✓ | Position in release (use `1` when standalone) |
| `Primary Genre` | ✓ | Primary genre |
| `Release Date` | ✓ | ISO 8601 |
| `Duration` |  | e.g. `3:35` or `PT3M35S` |
| `Disc Number` |  | For multi-disc releases |
| `Explicit Content` |  | `Explicit`, `NotExplicit`, `Cleaned` |
| `BPM` |  | Beats per minute |
| `Key` |  | Musical key (e.g. `Cm`, `F#`) |
| `Secondary Genre` |  | Secondary genre |
| `Language` |  | ISO 639-2 code |

#### LSP33Metadata Fields

| Field | Level | Description |
| :--- | :---: | :--- |
| `contributors` | Release & Track | Ordered list of contributors with roles |
| `identifiers` | Release & Track | Industry identifiers (ISRC, ISWC, UPC, GRid, catalogue) |
| `copyright` | Release & Track | `pLine` (℗) and `cLine` (©) |
| `lyrics` | Track | Full lyrics text, language, synced flag |
| `preview` | Track | `startMs`, `durationMs` |
| `stems` | Track | Stem/multitrack files |
| `ddex` | Release & Track | DDEX ERN XML reference |
| `audio_ai_usage` | Track (required) | Extent of generative AI used in the sound recording. One of `none`, `some`, `material`, `all` |
| `composition_ai_usage` | Track (required) | Extent of generative AI used in the underlying composition. One of `none`, `some`, `material`, `all` |
| `commercial_samples` | Track (required) | Whether the track contains third-party commercial material. One of `none`, `interpolation`, `sample` |

##### AI Classification (DDEX ERN 4.3.2)

LSP33 aligns with DDEX ERN 4.3.2's AI-disclosure model (as used by LabelGrid, Spotify and other DSPs). Every track MUST declare three top-level fields on its `LSP33Metadata`:

| Field | Required Values | Meaning |
| :--- | :--- | :--- |
| `audio_ai_usage` | `none`, `some`, `material`, `all` | How much of the **sound recording** was generated by AI. `none` = purely human, `some` = minor AI assistance, `material` = substantial AI generation combined with human work, `all` = entirely AI-generated. |
| `composition_ai_usage` | `none`, `some`, `material`, `all` | How much of the **underlying composition** (melody, lyrics, arrangement) was generated by AI. Same scale as above. |
| `commercial_samples` | `none`, `interpolation`, `sample` | Whether the track reuses third-party commercial material. `none` = original, `interpolation` = re-recorded/replayed reference to another work, `sample` = directly sampled recording. |

Additionally, every contributor object on a track MUST declare an `ai_contribution` field describing that contributor's individual involvement. At least **one** contributor per track MUST have `ai_contribution` != `"all"` — i.e. every track MUST credit at least one human contributor. Indexers and DSPs SHOULD reject a track whose contributors are all marked `"all"`.

| Value | Meaning |
| :--- | :--- |
| `none` | The contributor's work is fully human. |
| `partly` | The contributor used AI tools as part of their work. |
| `all` | The contributor's work is entirely AI-generated (e.g. a credited AI model). |

##### Contributors

Ordered array. Order = display sequence (maps to DDEX `SequenceNumber`).

```json
{
  "contributors": [
    { "name": "ledfut", "address": "0x1234...abcd", "roles": ["MainArtist", "Producer"], "ai_contribution": "none" },
    { "name": "ampy", "address": "0x5678...efgh", "roles": ["Composer", "Lyricist"], "ai_contribution": "partly" },
    { "name": "Big Boss", "roles": ["ExecutiveProducer"], "ai_contribution": "none" }
  ]
}
```

| Property | Type | Required |
| :--- | :---: | :---: |
| `name` | `string` | ✓ |
| `address` | `string` (UP address) |  |
| `email` | `string` |  |
| `roles` | `string[]` | ✓ |
| `ai_contribution` | `string` (`none` \| `partly` \| `all`) | ✓ (track level) |

Common roles (extensible):

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

A release with tracks naturally produces **four separate files** — one `LSP4Metadata` and one `LSP33Metadata` for the release, and one of each per track. A standalone LSP7 only needs the two track files. Files MAY be combined into fewer.

#### Release — `LSP4Metadata`

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

#### Release — `LSP33Metadata`

```json
{
  "LSP33Metadata": {
    "contributors": [
      { "name": "ledfut", "address": "0x1234...abcd", "roles": ["MainArtist", "Producer"], "ai_contribution": "none" },
      { "name": "Studio Wizard", "address": "0xabcd...1234", "roles": ["MasteringEngineer"], "ai_contribution": "none" }
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

#### Track — `LSP4Metadata`

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

#### Track — `LSP33Metadata`

```json
{
  "LSP33Metadata": {
    "contributors": [
      { "name": "ledfut", "address": "0x1234...abcd", "roles": ["MainArtist", "Producer", "Composer"], "ai_contribution": "none" },
      { "name": "ampy", "address": "0x5678...efgh", "roles": ["Lyricist"], "ai_contribution": "partly" },
      { "name": "Studio Wizard", "address": "0xabcd...1234", "roles": ["MixingEngineer", "MasteringEngineer"], "ai_contribution": "none" }
    ],
    "audio_ai_usage": "none",
    "composition_ai_usage": "some",
    "commercial_samples": "none",
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

## Rationale

**Plug-and-play composition.** The LSP8 release and LSP7 track are independent building blocks joined by a single data key. Any workflow — release-first, track-first, both-at-once, or pure standalone — works from the same primitives. No migration contracts, no special "upgrade" paths.

**Composition over new contracts.** LSP33 reuses LSP4, LSP7, LSP8, and LSP34. Existing wallets, indexers, and marketplaces already understand these primitives.

**Transparent data routing.** Making the LSP8 a router for linked LSP7 metadata gives artists one interface, prevents metadata drift between contracts, and degrades gracefully to an LSP8 local fallback if the LSP7 is unreachable.

**`tokenOwnerOf` access control.** Gating `setDataForTokenId` on `tokenOwnerOf(tokenId)` — not the contract `owner()` — and mirroring the same resolution on the LSP7 via LSP34 ensures there is exactly one concept of "the artist" for each track, and that transferring the tokenId transfers control of both sides together.

**Artist vs collector separation.** Authorship lives in the LSP8 tokenId (or the LSP7 `owner()` when standalone); collectibility lives in LSP7 `balanceOf`. Units can change hands freely without ever touching metadata or minting.

**Separate metadata files.** `LSP4Metadata` and `LSP33Metadata` are separate data keys so any LSP4-aware interface works out of the box and each can evolve independently. They may still be combined into a single file when desired.

## Implementation

Reference implementation: [lukso-network/lsp-smart-contracts#1088](https://github.com/lukso-network/lsp-smart-contracts/pull/1088).

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

[ERC173]: https://eips.ethereum.org/EIPS/eip-173
[ERC725Y]: https://github.com/ERC725Alliance/ERC725/blob/develop/docs/ERC-725.md#erc725y
[LSP2]: ./LSP-2-ERC725YJSONSchema.md
[LSP4]: ./LSP-4-DigitalAsset-Metadata.md
[LSP7]: ./LSP-7-DigitalAsset.md
[LSP8]: ./LSP-8-IdentifiableDigitalAsset.md
[LSP34]: ./LSP-34-ExternalOwnership.md
