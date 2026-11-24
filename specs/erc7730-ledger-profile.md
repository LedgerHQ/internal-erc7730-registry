# ERC-7730 Ledger profile

**Status:** normative for this registry. **Profile version:** 1.

This document defines how Ledger adds its own data to an ERC-7730 descriptor without changing the descriptor. It defines no display format and no new ERC-7730 key.

**This profile MUST NOT redefine, extend or contradict ERC-7730.** The descriptor stays a plain ERC-7730 file, valid against `erc7730-v2.schema.json`, and this profile never requires changing it: a descriptor that comes from the Ethereum Foundation registry stays identical to it. Where the two documents disagree, ERC-7730 wins and this document is wrong.

RFC 2119 / RFC 8174 keywords apply.

## 1. Why an overlay

ERC-7730 is owned by the Ethereum Foundation, and its registry is shared by every wallet. Some data Ledger needs is specific to Ledger: an icon identifier that points into a Ledger repository and is converted to device-specific sizes is meaningless to another wallet. Putting it in the descriptor would either need a change to the standard for a single vendor, or make the descriptor invalid.

So Ledger data lives in a separate file, one per registry folder, next to the descriptors. This follows the pattern of the Solana registry (`LedgerHQ/clear-signing-solana-registry`): the reference file stays untouched, and a Ledger file carries only the Ledger extensions.

## 2. Placement and binding

```text
registry/<entity>/
  ledger.json                       Ledger overlay (this profile)
  calldata-<Name>.json              descriptors, plain ERC-7730
  eip712-<Name>.json
  tests/calldata-<Name>.tests.json  device tests (existing)
```

- The overlay `registry/<entity>/ledger.json` applies to the `calldata-*.json` and `eip712-*.json` descriptors of `registry/<entity>/`, and to nothing else. A file only used through `includes` (`common-*.json`) is never served on its own: it gets nothing, and the descriptor that includes it gets the extension.
- A folder has at most one overlay. A folder without an overlay has no Ledger extension, and its descriptors are handled exactly as today.
- The overlay file name does not match `calldata-*` or `eip712-*`, so `erc7730 lint` and the device tests do not see it.

## 3. Content

The icon can be set for the whole folder, per descriptor, or both.

**Whole folder.** Every descriptor of `registry/1inch/` is a 1inch contract:

```json
{
  "$schema": "../../specs/erc7730-ledger-overlay.schema.json",
  "ledgerhq": {
    "version": 1,
    "icon": "1inch"
  }
}
```

**Per descriptor.** `registry/tally/` holds the governance contracts of several DAOs, so there is no folder icon:

```json
{
  "$schema": "../../specs/erc7730-ledger-overlay.schema.json",
  "ledgerhq": {
    "version": 1,
    "descriptors": {
      "eip712-tally-ethereum-ens-governor.json": { "icon": "ens" },
      "eip712-tally-ethereum-uni-token.json": { "icon": "uniswap" }
    }
  }
}
```

- Every Ledger extension sits in one object, `ledgerhq`, with one `version`. Adding a key is a profile change, made in `erc7730-ledger-overlay.schema.json` and in this document.
- A key at the top of `ledgerhq` (`icon`) is the **default** for every descriptor of the folder.
- `descriptors` holds the **exceptions**, keyed by descriptor file name. A value replaces the default for that descriptor; `null` removes it (`"icon": null`: network icon, as today). A descriptor that is neither covered by a default nor listed gets nothing. Every name in `descriptors` MUST be an existing descriptor of the folder.
- A registry folder is the entity that **submitted** the descriptors, not the owner of the contracts (registry README), so a folder can hold contracts of other dApps. Use a default only when every descriptor of the folder belongs to the same dApp, and exceptions for the others. Folders that group many dApps (`permit`, `tally`) SHOULD have no default.
- **A default also applies to descriptors added to the folder later.** Reviewing a new descriptor therefore includes checking that the folder default fits it, or adding an exception.
- The overlay holds **extensions only**. It MUST NOT contain any ERC-7730 key (`context`, `metadata`, `display`, `includes`): it never overrides what a descriptor binds to, decodes or displays as text. The schema enforces it (`additionalProperties: false`).
- An overlay without any extension is useless and is rejected (`minProperties: 2` on `ledgerhq`).

## 4. Consumer rules (CAL)

1. The CAL reads the descriptor as today, then the overlay of its folder, and attaches the resolved extensions (default, then exception) to the resolved descriptor. Nothing else in the descriptor changes. The CAL ingestion MUST NOT take `ledger.json` for a descriptor.
2. An overlay that fails validation, has an unknown `version` or references a missing resource MUST be ignored, and the descriptor MUST be served without it. A Ledger extension can degrade the presentation, never turn a clear-signed transaction into a blind-signed one.
3. The absence of an overlay, or of a key in it, MUST NOT be read as a statement about the transaction.

## 5. Extensions

### 5.1 `icon`

Identifier of a dApp icon in `LedgerHQ/ledger-network-icons`, stored as `icons/dapps/dapp_<icon>_<size>px.gif` in the sizes the devices need. The device shows it in place of the network icon on the review screens of every function of the descriptor.

- **Format:** lowercase letters, digits and single hyphens, 32 characters at most (`^[a-z0-9]+(-[a-z0-9]+)*(?![\s\S])`). The schema patterns end with `(?![\s\S])` rather than `$`: some regex engines (Python) let `$` match before a final newline.
- **One line covers a dApp:** the 10 descriptors of `registry/1inch/` get their icon from `"icon": "1inch"` in `registry/1inch/ledger.json`. An icon is identified by the dApp, not by the folder (section 3).
- **The icon is presentation, not identity.** It never replaces the intent, the contract name or any field. An icon showing another dApp's brand is a phishing vector: the icon MUST represent the dApp the user interacts with through the contract (Morpho for a Morpho vault, even when `metadata.owner` names its curator), and this is a review rule on both repositories.
- **Trust:** the device shows the icon only when its hash matches the one signed by the CAL for this contract. The registry only names the icon; the image itself is reviewed and owned in `ledger-network-icons`.

## 6. Checks

- Every `registry/*/ledger.json` is validated against `erc7730-ledger-overlay.schema.json`, whatever its `$schema` says, on every pull request. The schema also pins `$schema` to `../../specs/erc7730-ledger-overlay.schema.json`.
- Every name in `descriptors` is an existing descriptor of the folder, compared literally.
- Every icon, default or exception, of an overlay changed by the pull request exists in `LedgerHQ/ledger-network-icons` (branch `main`) in the three sizes: `icons/dapps/dapp_<icon>_{14,48,64}px.gif`. The CI fetches only that folder. The icon is therefore merged in `ledger-network-icons` before the overlay that uses it. Overlays the pull request does not change are not checked again, so that a change in `ledger-network-icons` never blocks an unrelated pull request; an icon removed there later is handled by consumer rule 2.
