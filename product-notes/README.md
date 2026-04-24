# Cross-Platform Notes Product

This directory holds the first product-oriented note-taking deliverables for the
repo.

The current committed binary is:

- `notes-linux-x86_64` — Linux x86_64 raw-X11 reference build
- `test-notes-linux-x86_64` — Linux x86_64 structural verifier
- `test-notes-linux-x86_64-clicks` — Linux x86_64 click-path structural verifier

The rest of the files in this directory define the shared product contract and
the implementation notes for the other target platforms in the approved plan.

| Artifact | Type | Purpose |
| --- | --- | --- |
| [`product-contract.md`](product-contract.md) | contract | Shared behavior, storage model, and UI rules for the product |
| [`notes-linux-x86_64.md`](notes-linux-x86_64.md) | binary doc | Byte-level explanation of the Linux x86_64 reference build |
| [`test-notes-linux-x86_64.md`](test-notes-linux-x86_64.md) | test doc | Explanation of the Linux x86_64 structural verifier |
| [`test-notes-linux-x86_64-clicks.md`](test-notes-linux-x86_64-clicks.md) | test doc | Explanation of the click-path structural verifier |
| [`linux-arm64-plan.md`](linux-arm64-plan.md) | platform plan | Linux ARM64 port plan |
| [`windows-plan.md`](windows-plan.md) | platform plan | Windows x86_64 and ARM64 GUI plan |
| [`mobile-plan.md`](mobile-plan.md) | platform plan | Android ARM64 and iOS ARM64 mobile GUI plan |
| [`apple-desktop-plan.md`](apple-desktop-plan.md) | platform plan | macOS arm64 desktop GUI and Apple-host validation plan |
| [`web-plan.md`](web-plan.md) | platform plan | Browser-hosted WebAssembly plan |
| [`docs-and-tests.md`](docs-and-tests.md) | delivery plan | Per-artifact documentation and test strategy |

## Current Linux x86_64 behavior

The reference build already provides the requested core interaction pattern on
Linux x86_64:

- a left editor pane for the full note text
- `Enter` saves the current left-pane note
- a right-side alphabetical list derived from the first words of stored notes
- mouse clicks on the right-side list load a note into the left pane
- each visible right-pane row also shows a `Del` affordance on its far right

The current Linux build keeps the existing tiny record format inherited from
the repo's earlier note tools. Clicking a note loads it for amendment,
clicking `Del` rewrites `notes.db` from the current in-memory list so the
deletion persists, and pressing `Enter` stores the current editor contents as a
new note record in the shared database.

## Verification

Run the Linux x86_64 verifiers:

```bash
cd product-notes
./test-notes-linux-x86_64 && echo PASS || echo FAIL
./test-notes-linux-x86_64-clicks && echo PASS || echo FAIL
```
