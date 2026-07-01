# UI Reference Screenshots

Reference screenshots of a consumer wallet app used as visual source material when designing the PPI Wallet reference implementation's UI. The screenshots were taken during the initial design phase to establish a familiar wallet interface pattern (wallet strip, sub-wallet cards, passbook list, transaction detail).

## Attribution

These are reference screenshots of a commercial wallet app. They are reproduced here as visual design reference only — not as creative work of this repository.

## Contents

9 `.jpeg` screenshots captured from a mobile wallet app between 2026-04-01 and 2026-04-11. Filenames preserved from WhatsApp export.

## How these map to the implementation

Rather than pixel-match the reference app, the implementation selectively adopts:
- Navy / Cyan / Green palette (see [`../../CLAUDE.md`](../../CLAUDE.md))
- Card-based layout with rounded corners
- Bottom-nav pattern (Home / Scan / Passbook / Profile)
- Wallet strip with expandable sub-wallet list

The implementation **does not copy** the original app's flows, copy, merchant lists, or branded assets beyond the colour tokens.
