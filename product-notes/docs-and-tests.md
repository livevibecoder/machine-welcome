# Product Documentation and Test Strategy

This file captures the delivery pattern for the cross-platform notes product.

**Vocabulary** (ImageText8, structural tests, syscalls, …):
[`product-notes/glossary.md`](glossary.md).

## Per-artifact rule

For every committed binary artifact, ship:

- the binary itself
- one matching `.md`
- one matching test binary
- one matching `test-*.md`

This follows the repo-wide rules already summarized in
[`../README.md`](../README.md).

## Product families

Expected primary artifacts:

- `notes-linux-x86_64`
- `notes-linux-arm64`
- `notes-win64.exe`
- `notes-winarm64.exe`
- `notes-macos`
- `notes-ios`
- `notes-android.apk`
- `notes-web.wasm`

The non-x86_64 artifacts currently committed are initial GUI scaffold/container
executables with structural verifiers. Their platform plans remain the source of
truth for the full interactive native GUI work.

## Test patterns

Use the repo's existing mixed strategy:

### Native-host behavioral tests

Use when the target runs directly on the current host, or when runtime
interaction is easy to automate.

### Emulated native tests

Use for Linux ARM64 with `qemu-aarch64-static`.

### Host-side structural verifiers

Use for foreign containers or environments where full GUI automation is not yet
practical:

- PE
- APK
- Mach-O
- browser-hosted Wasm

## Minimum acceptance flow per platform

Each platform should eventually have a machine-code acceptance story covering:

1. create a note
2. show first-word list
3. load a note from the list
4. alter it in the editor
5. save it
6. relaunch and confirm persistence
7. enter normal printable ASCII, including uppercase letters and shifted symbols
8. verify pane borders and readable foreground/background colours are present

The Linux x86_64 reference build currently has structural verifiers and the full
product-specific byte-level doc set. It is the baseline for the remaining
targets, including the printable-input, border, and colour anchors now checked
by `test-notes-linux-x86_64`.
