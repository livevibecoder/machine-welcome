# Welcome to the Machine

Machine language only.

## Motivation

In early 2026, Elon Musk [said](https://www.youtube.com/watch?v=HD_SiJDWPcQ&t=683s) that in future, developers would write code in natural language and AI would generate machine code directly, without the need for human-readable code.

I wanted to explore this idea and see how far I could get.

## How it works

The user writes a natural language description of what they want to build.

The AI then generates a machine code implementation of the description.

The AI will also generate a Markdown file with a detailed explanation of the code and how it works.

## Rules for the AI

These rules are binding for all contributions to this repo:

1. **Binaries only.** The AI can produce only *binaries* - directly-executable machine code in the native format of one of the target architectures (ELF, Mach-O, PE, WebAssembly, ...). It may **not** commit source code in any human-readable language (C, Python, JavaScript, assembly, etc.) or hand-written build scripts. Binaries may be produced via tools such as `xxd -r -p` fed with literal hex bytes, but those hex bytes **are** the source of truth - they are not "compiled" from anything else.
2. **Binary tools only.** The AI may use only tools that already exist in binary form, unless it first creates the tool itself as a machine-code binary under these same repo rules. It may **not** write temporary human-readable helper programs or scripts (C, Python, JavaScript, assembly, shell, etc.) even for intermediate build, test, or opcode-derivation steps.
3. **Every binary ships with a Markdown explanation.** For each binary committed, a matching `<binary>.md` must describe its ELF/PE/... layout and every opcode, byte by byte, in enough detail that a reader can mentally single-step through the code.
4. **Tests as binaries.** For each target binary, ship a test binary (also in pure machine code) that exercises it. Tests use the standard Unix exit-code convention: `0` for pass, non-zero for fail.
5. **Cross-architecture.** Produce machine-code test suites for other target architectures as soon as possible, running them under virtual machines or emulators (QEMU, Wasmtime, Apple Silicon VM, ...).
6. **External help is allowed.** The AI may download or ask the user to install tools (assemblers, disassemblers, emulators, linkers) that help produce or verify binaries. These tools are not committed.
7. **Libraries and linking are allowed.** Binaries may link against libraries (libc, X11, Metal, ...), but the Markdown must document the exact interface used (calling convention, struct layouts, symbols).

## Tech Details

We will attempt to build for multiple platforms:

- Linux ELF x86_64 - **current** (see `poc-01/`, `poc-02/`, `poc-03/`, `poc-04/`, `poc-08/test-hello`)
- Linux ELF ARM64
- macOS arm64 Mach-O - host-verified structural POC in `poc-11/`
- iOS arm64 Mach-O - host-verified structural POC in `poc-11/`
- Android ARM64
- WebAssembly
- Windows PE x86_64
- Windows PE ARM64
- RISC-V 64-bit
- Quantum Qubit

## Host Prerequisites

The repo is intentionally light on dependencies, but a few **non-base** tools
are needed if you want to run every current target or inspect the binaries
comfortably.

| Tool / Package | Purpose |
| --- | --- |
| `binutils` | Provides `objdump` and related binary-inspection tools used throughout the docs. |
| `file` | Identifies generated binaries by container and architecture. |
| `xxd` | Converts literal hex bytes to binaries and helps inspect raw output; on some distros it comes from `vim-common`. |
| `wget` | Downloads the Android command-line tools archive in the setup example below. |
| `unzip` | Extracts the Android command-line tools archive in the setup example below. |
| `qemu-user-static` | Runs committed non-x86 native targets in `poc-06/`, `poc-07/`, and `poc-09/`. |
| `wasmtime` | Runs the WebAssembly / WASI targets in `poc-05/`. |
| `wine64` | Lets you directly execute `poc-08/hello.exe` on Linux. |
| Android SDK / NDK binaries | Required for the APK packaging and emulator demo in `poc-10/`. |

For Ubuntu/Debian, the practical install set is:

```bash
sudo apt install binutils file xxd wget unzip qemu-user-static wine64
```

`wasmtime` is usually installed separately as a prebuilt binary runtime rather
than from the base distro image. Under rule 2, it should be installed as an
existing binary artifact, not via a temporary helper script written in the
workflow.

One way to install it is:

```bash
curl https://wasmtime.dev/install.sh -sSf | bash
```

For the Android APK work in `poc-10/`, the practical requirement is the
official prebuilt Android SDK / NDK package set, installed under an SDK root
such as `$HOME/android-sdk`. 

You can do: 
```bash
mkdir -p "$HOME/android-sdk/cmdline-tools"
cd /tmp
cmdline_tools_zip="commandlinetools-linux-14742923_latest.zip"
wget "https://dl.google.com/android/repository/${cmdline_tools_zip}"
unzip "$cmdline_tools_zip"
mv cmdline-tools "$HOME/android-sdk/cmdline-tools/latest"
export ANDROID_SDK_ROOT="$HOME/android-sdk"
export PATH="$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"
```

Then install the packages:

```bash
yes | sdkmanager --licenses
sdkmanager "platform-tools" \
           "platforms;android-35" \
           "build-tools;35.0.0" \
           "emulator" \
           "system-images;android-35;default;x86_64" \
           "ndk;27.2.12479018"
```

Those provide `adb`, `aapt` / `aapt2`, `zipalign`, `apksigner`, the Android
emulator and x86_64 system image, and NDK LLVM tools such as `llvm-objcopy`
and target `clang`.

### What you need for each current target

| Target | What you need |
| --- | --- |
| `poc-01`, `poc-03`, Linux-hosted `poc-08/test-hello`, `poc-11/test-hello-*` | Normal x86_64 Linux system; no extra runtime beyond the base OS. |
| `poc-02`, `poc-04` | Live X11 session with matching X server socket and Xauthority details. |
| `product-notes/notes-linux-x86_64` | Live X11 session with matching X server socket and Xauthority details. |
| `poc-05` | `wasmtime`. |
| `poc-06`, `poc-07`, `poc-09` | `qemu-user-static`. |
| `poc-08/hello.exe` | Windows loader, typically `wine64` on Linux. |
| `poc-10` | Android SDK / NDK binaries above, plus either the Android emulator or a real `adb`-connected device. |
| Apple runtime execution | Not yet demonstrated; current `poc-11` verification is structural on Linux only. |

No compiler, assembler, linker, or interpreter is required to *use* the
committed artifacts in this repo.

## Products

| Product | Status | Platforms | Docs |
| --- | --- | --- | --- |
| `product-notes` | First committed product reference build present. | Linux x86_64 binary now; Linux ARM64, Windows, Android/iOS, macOS, and browser-hosted Wasm implementation notes are in place. | [`product-notes/README.md`](product-notes/README.md), [`product-notes/product-contract.md`](product-notes/product-contract.md) |

## Proofs of concept

This README is intentionally a summary. The detailed byte-by-byte explanations
live beside each binary in its own directory.

| POC | Architecture | Platform | Description | Doc | Test Doc |
| --- | --- | --- | --- | --- | --- |
| `poc-01` | x86_64 | Linux ELF | Greeter that reads stdin, writes `greeting.txt`, and prints an ANSI greeting. | [`hello.md`](poc-01/hello.md) | [`test-hello.md`](poc-01/test-hello.md) |
| `poc-02` | x86_64 | Linux ELF / X11 | Minimal raw-X11 window that records simple status to `status.log`. | [`window.md`](poc-02/window.md) | [`test-window.md`](poc-02/test-window.md) |
| `poc-03` | x86_64 | Linux ELF | Note-taker that appends stdin text to `notes.db` and reprints stored notes. | [`note.md`](poc-03/note.md) | [`test-note.md`](poc-03/test-note.md) |
| `poc-04` | x86_64 | Linux ELF / X11 | Notes GUI with `note-view` and interactive sorted-entry `note-edit`. | [`note-view.md`](poc-04/note-view.md), [`note-edit.md`](poc-04/note-edit.md) | [`test-note-view.md`](poc-04/test-note-view.md), [`test-note-edit.md`](poc-04/test-note-edit.md) |
| `poc-05` | WebAssembly | WASI | First non-ELF target: `hello.wasm` plus a WASM structural verifier. | [`hello.wasm.md`](poc-05/hello.wasm.md) | [`test-hello.wasm.md`](poc-05/test-hello.wasm.md) |
| `poc-06` | ARM64 | Linux ELF | Minimal AArch64 ELF hello program with matching native test binary. | [`hello.md`](poc-06/hello.md) | [`test-hello.md`](poc-06/test-hello.md) |
| `poc-07` | RISC-V 64 | Linux ELF | Minimal RV64 ELF hello program with matching structural verifier. | [`hello.md`](poc-07/hello.md) | [`test-hello.md`](poc-07/test-hello.md) |
| `poc-08` | x86_64 | Windows PE | Minimal `PE32+` console hello program with Linux x86_64 verifier. | [`hello.exe.md`](poc-08/hello.exe.md) | [`test-hello.md`](poc-08/test-hello.md) |
| `poc-09` | ARM64 | Android-style Linux ELF PIE | Native Android-oriented AArch64 PIE executable, before full APK packaging. | [`hello.md`](poc-09/hello.md) | [`test-hello.md`](poc-09/test-hello.md) |
| `poc-10` | x86_64, ARM64 | Android APK | NativeActivity APK with `x86_64` and `arm64-v8a` payloads plus Linux verifier. | [`hello.apk.md`](poc-10/hello.apk.md) | [`test-hello.md`](poc-10/test-hello.md) |
| `poc-11` | ARM64 | macOS Mach-O, iOS Mach-O | Host-verified Apple Mach-O structural POCs for macOS and iOS. | [`hello-macos.md`](poc-11/hello-macos.md), [`hello-ios.md`](poc-11/hello-ios.md) | [`test-hello-macos.md`](poc-11/test-hello-macos.md), [`test-hello-ios.md`](poc-11/test-hello-ios.md) |

Notes:

- `poc-02` and `poc-04` are tied to live X11 session details; see their docs for caveats.
- `poc-11` is host-verified structurally on Linux; Apple runtime execution, signing, and packaging are not yet demonstrated.

## Tools (self-built)

Committed helper tools must also follow the repo rules and ship with their own
binary tests and Markdown explanations.

### `tools/mkelf`

Small Linux x86_64 ELF wrapper used to turn raw code+data bytes into complete
ELF executables.

- docs: [`tools/mkelf.md`](tools/mkelf.md), [`tools/test-mkelf.md`](tools/test-mkelf.md)

## How to use

1. Pick the target directory you want to inspect or run.
2. Read the matching `.md` files in that directory for the exact binary layout,
   verification method, and any platform-specific caveats.
3. Run the binary and its test using the runtime noted in that directory's docs.
