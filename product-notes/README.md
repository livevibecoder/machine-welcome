# Cross-Platform Notes Product

This directory holds the first product-oriented note-taking deliverables for the
repo.

The current committed product artifacts are:

- `notes-linux-x86_64` — Linux x86_64 raw-X11 reference build
- `test-notes-linux-x86_64` — Linux x86_64 structural verifier
- `test-notes-linux-x86_64-clicks` — Linux x86_64 click-path structural verifier
- `notes-linux-arm64` / `test-notes-linux-arm64` — Linux ARM64 marker and verifier
- `notes-win64.exe` / `test-notes-win64` — Windows x86_64 PE marker and verifier
- `notes-winarm64.exe` / `test-notes-winarm64` — Windows ARM64 PE marker and verifier
- `notes-macos` / `test-notes-macos` — macOS arm64 Mach-O marker and verifier
- `notes-ios` / `test-notes-ios` — iOS arm64 Mach-O marker and verifier
- `notes-android.apk` / `test-notes-android` — Android NativeActivity APK container and verifier
- `notes-web.wasm` / `test-notes-web` — WASI Wasm marker and verifier

The rest of the files in this directory define the shared product contract and
the implementation notes for the other target platforms in the approved plan.
Those platform plans inherit the current Linux behavior, including printable
ASCII input, shifted uppercase/symbols, pane borders, and simple colours.

| Artifact | Type | Purpose |
| --- | --- | --- |
| [`product-contract.md`](product-contract.md) | contract | Shared behavior, storage model, and UI rules for the product |
| [`glossary.md`](glossary.md) | reference | [ImageText8](glossary.md#imagetext8), *IT8*, *slot*, *NOTE_COUNT*, *BSS* — repo-wide terms; [PoC-04 index](../poc-04/glossary.md) for `note-edit` / `note-view` pointers |
| [`notes-linux-x86_64.md`](notes-linux-x86_64.md) | binary doc | Byte-level explanation of the Linux x86_64 reference build |
| [`test-notes-linux-x86_64.md`](test-notes-linux-x86_64.md) | test doc | Explanation of the Linux x86_64 structural verifier |
| [`test-notes-linux-x86_64-clicks.md`](test-notes-linux-x86_64-clicks.md) | test doc | Explanation of the click-path structural verifier |
| [`notes-linux-arm64.md`](notes-linux-arm64.md) | binary doc | Linux ARM64 Notes marker executable |
| [`test-notes-linux-arm64.md`](test-notes-linux-arm64.md) | test doc | Linux ARM64 structural verifier |
| [`notes-win64.exe.md`](notes-win64.exe.md) | binary doc | Windows x86_64 PE Notes marker executable |
| [`test-notes-win64.md`](test-notes-win64.md) | test doc | Windows x86_64 structural verifier |
| [`notes-winarm64.exe.md`](notes-winarm64.exe.md) | binary doc | Windows ARM64 PE Notes marker executable |
| [`test-notes-winarm64.md`](test-notes-winarm64.md) | test doc | Windows ARM64 structural verifier |
| [`notes-macos.md`](notes-macos.md) | binary doc | macOS arm64 Mach-O Notes marker executable |
| [`test-notes-macos.md`](test-notes-macos.md) | test doc | macOS structural verifier |
| [`notes-ios.md`](notes-ios.md) | binary doc | iOS arm64 Mach-O Notes marker executable |
| [`test-notes-ios.md`](test-notes-ios.md) | test doc | iOS structural verifier |
| [`notes-android.apk.md`](notes-android.apk.md) | binary doc | Android APK Notes package container |
| [`test-notes-android.md`](test-notes-android.md) | test doc | Android APK structural verifier |
| [`notes-web.wasm.md`](notes-web.wasm.md) | binary doc | WebAssembly Notes marker module |
| [`test-notes-web.md`](test-notes-web.md) | test doc | WebAssembly structural verifier |
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
- normal printable ASCII input, including shifted uppercase letters and symbols
- a dark window background with light text and simple pane borders

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
./test-notes-linux-arm64 && echo PASS || echo FAIL
./test-notes-win64 && echo PASS || echo FAIL
./test-notes-winarm64 && echo PASS || echo FAIL
./test-notes-macos && echo PASS || echo FAIL
./test-notes-ios && echo PASS || echo FAIL
./test-notes-android && echo PASS || echo FAIL
./test-notes-web && echo PASS || echo FAIL
```
