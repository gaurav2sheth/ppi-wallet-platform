# UI Reference Screenshots

Reference screenshots of the Paytm consumer wallet app used as visual source material when designing the PPI Wallet reference implementation's UI. The screenshots were taken during the initial design phase to align the demo's look-and-feel with familiar Paytm patterns (wallet strip, sub-wallet cards, passbook list, transaction detail).

## Attribution

These are screenshots of the **Paytm consumer app**, owned by Paytm Payment Services Limited (PPSL). They are reproduced here as visual reference only — not as creative work of this repository.

## Contents

9 `.jpeg` screenshots captured from the Paytm Android app between 2026-04-01 and 2026-04-11. Filenames preserved from WhatsApp export.

## How these map to the implementation

Rather than pixel-match the Paytm app, the reference implementation selectively adopts:
- Navy / Cyan / Green PODS palette (see [`../../CLAUDE.md`](../../CLAUDE.md))
- Card-based layout with rounded corners
- Bottom-nav pattern (Home / Scan / Passbook / Profile)
- Wallet strip with expandable sub-wallet list

The implementation **does not copy** Paytm-specific flows, copy, merchant lists, or branded assets beyond the colour tokens.
