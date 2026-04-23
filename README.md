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
- macOS Universal ARM64
- iOS Universal ARM64
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

On a normal Linux desktop/server install, the following are usually the extras
to check for:

- `binutils` - provides `objdump`, used throughout the docs to inspect ELF/PE
  headers and imported symbols
- `file` - identifies the generated binaries by container/architecture
- `xxd` - converts literal hex bytes to binaries and helps inspect raw output;
  on some distros this is packaged as `xxd`, on others it comes from
  `vim-common`
- `qemu-user-static` - required to run the committed non-x86 native targets in
  `poc-06/`, `poc-07/`, and `poc-09/`
- `wasmtime` - required to run the WebAssembly/WASI targets in `poc-05/`
- `wine64` - not yet required for the repo's current automated Windows test,
  but needed if you want to directly execute `poc-08/hello.exe` on Linux
- official Android SDK / NDK binaries - required for the real APK packaging and
  emulator demo in `poc-10/`

For Ubuntu/Debian, the practical install set is:

```bash
sudo apt install binutils file xxd qemu-user-static wine64
```

`wasmtime` is usually installed separately as a prebuilt binary runtime rather
than from the base distro image. Under rule 2, it should be installed as an
existing binary artifact, not via a temporary helper script written in the
workflow.

For the Android APK work in `poc-10/`, the practical requirement is the
official prebuilt Android SDK / NDK package set, installed under an SDK root
such as `/home/richard/opt/android-sdk`. The packages used for the current demo
were:

```bash
sdkmanager "platform-tools" \
           "platforms;android-35" \
           "build-tools;35.0.0" \
           "emulator" \
           "system-images;android-35;default;x86_64" \
           "ndk;27.2.12479018"
```

Those provide:

- `adb`
- `aapt` / `aapt2`
- `zipalign`
- `apksigner`
- the Android emulator and x86_64 system image
- NDK LLVM tools such as `llvm-objcopy` and target `clang`

### What you need for each current target

- `poc-01/`, `poc-03/`, and the Linux-hosted test in `poc-08/` run on a normal
  x86_64 Linux system with no extra runtime beyond the base OS
- `poc-02/` and `poc-04/` need a live X11 session because they talk directly to
  the X server socket and Xauthority cookie
- `poc-05/` needs `wasmtime`
- `poc-06/`, `poc-07/`, and `poc-09/` need `qemu-user-static`
- `poc-08/hello.exe` itself needs a Windows loader, typically `wine64` on Linux
- `poc-10/` needs the Android SDK / NDK binaries above, plus either the Android
  emulator or a real `adb`-connected device

No compiler, assembler, linker, or interpreter is required to *use* the
committed artifacts in this repo.

## Proofs of concept

### `poc-01/` - Linux ELF x86_64 greeter

A 429-byte ELF64 executable and its 163-byte test. Takes a name on stdin, stores it to `greeting.txt`, and prints a coloured ANSI box containing `Hello, <name>!`. No libc; direct Linux syscalls only.

- `poc-01/hello` - the main binary
- `poc-01/hello.md` - byte-by-byte explanation of every header field, opcode, and string
- `poc-01/test-hello` - test binary: `exit 0` iff `greeting.txt` exists in cwd
- `poc-01/test-hello.md` - byte-by-byte explanation of the test

Run it:

```
cd poc-01
./hello
cat greeting.txt
./test-hello && echo PASS || echo FAIL
```

### `poc-02/` - Linux ELF x86_64 X11 window (graphical)

A 696-byte ELF64 executable and its 220-byte test. Opens a real X11 window
(400x300, dark blue, titlebar-less), records `opened` to `status.log`,
blocks until the user clicks inside the window, records `event`, and
exits. No libc, no Xlib: raw X11 wire protocol over a Unix-domain socket,
with a hard-coded `MIT-MAGIC-COOKIE-1` and socket path baked into the
binary.

- `poc-02/window` - the main binary (graphical output, X11)
- `poc-02/window.md` - byte-by-byte explanation of ELF, X11 request templates, opcode sequence, and syscalls
- `poc-02/test-window` - test binary: `exit 0` iff `status.log` starts with `"opened\n"`
- `poc-02/test-window.md` - byte-by-byte explanation of the test

Run it:

```
cd poc-02
./window &              # opens the window
# click inside the window to generate a ButtonPress event
./test-window && echo PASS || echo FAIL
cat status.log
```

Because the `MIT-MAGIC-COOKIE-1` is hard-coded at build time, `poc-02/window`
must be re-hexed whenever the X session cookie rotates (typically on each
login). The binary is tied to `DISPLAY=:1` on the author's machine; see
`window.md` for how to re-build with a new cookie.

### `poc-03/` - Linux ELF x86_64 note-taker with an append-only database

A 505-byte ELF64 executable and its 255-byte test, both wrapped by
`tools/mkelf`. Reads text from stdin, appends it as a length-prefixed
record to a tiny on-disk "database" (`notes.db`), then re-reads the
whole file and prints every note so far with an ANSI-green bullet.
Terminal-only (no X11), no libc — seven direct Linux syscalls.

- `poc-03/note` - the main binary (stdin → `notes.db` → ANSI display)
- `poc-03/note.md` - byte-by-byte explanation of every opcode, the record format, and the BSS buffer layout
- `poc-03/test-note` - test binary: `exit 0` iff the first record in `notes.db` is exactly `"hello\n"`
- `poc-03/test-note.md` - byte-by-byte explanation of the test

Record format on disk (one per entry, appended):

```
+--------------------+------------------------+
|  length: u32 LE    |  text: length bytes    |
+--------------------+------------------------+
```

Run it:

```
cd poc-03
rm -f notes.db
echo "first entry"  | ./note
echo "second entry" | ./note
./note                          # interactive: type, press Enter, then Ctrl-D
xxd notes.db                    # inspect the raw records

printf 'hello\n' | ./note > /dev/null
./test-note && echo PASS || echo FAIL
```

### `poc-04/` - Linux ELF x86_64 X11 GUI viewer for `notes.db`

A 1060-byte ELF64 executable and its 249-byte test, both wrapped by
`tools/mkelf`. `note-view` opens a real 600×400 X11 window titled
`"note-view"`, reads `poc-03/note`'s `notes.db` record format, and renders
every record as one line of black text on white using the X11 `"fixed"`
font. Closes cleanly on any key press, mouse click, or WM close button. No
libc, no Xlib — raw X11 wire protocol, eight direct Linux syscalls, with
the same session-coupled `MIT-MAGIC-COOKIE-1` hard-coding as `poc-02/`
plus the screen root window id.

- `poc-04/note-view` - the main binary (X11 GUI, reads `notes.db`)
- `poc-04/note-view.md` - byte-by-byte explanation of ELF layout, X11
  request templates, the dispatch/redraw loop, and the four independent
  exit paths
- `poc-04/test-note-view` - test binary: `exit 0` iff the 9 bytes at file
  offset `0x384` of `./note-view` are `"note-view"` (i.e. the WM_NAME
  payload is intact)
- `poc-04/test-note-view.md` - byte-by-byte explanation of the test

Run it:

```
cd poc-04
# works with an empty or missing notes.db (window opens, no text drawn)
../poc-03/note <<<'hello world'
../poc-03/note <<<'second entry'
./note-view &          # window appears with the two lines
# click the window, press any key, or hit the WM close button → clean exit
./test-note-view && echo PASS || echo FAIL
```

Like `poc-02/`, `poc-04/note-view` hard-codes the current X session's
`MIT-MAGIC-COOKIE-1`, the Unix socket path `/tmp/.X11-unix/X1`, and the
screen root window id. Re-derive these on a fresh session and re-hex the
binary; see `note-view.md` for the relevant byte offsets.

#### `poc-04/note-edit` - interactive X11 note editor with sorted display

A 2346-byte ELF64 executable and its 249-byte test, also wrapped by
`tools/mkelf`. `note-edit` opens a real 600×400 X11 window titled
`"note-edit"`, shows an editable `New:` input line, accepts a small
hard-coded key set (lowercase letters, digits, space, `-`, `,`, `.`, `'`,
`/`, Backspace, Enter, Escape), appends the typed line to `notes.db`, then
reloads the database and displays the notes in **sorted order**. No libc,
no Xlib: raw X11 wire protocol plus direct Linux syscalls only.

- `poc-04/note-edit` - the interactive GUI binary
- `poc-04/note-edit.md` - explanation of the X11 event loop, keycode map,
  append path, and in-memory insertion sort
- `poc-04/test-note-edit` - test binary: `exit 0` iff the 9 bytes at file
  offset `0x784` of `./note-edit` are `"note-edit"`
- `poc-04/test-note-edit.md` - explanation of the test

Run it:

```
cd poc-04
./note-edit
# type a short line, press Enter to save, Escape to quit
# launch again: notes are reloaded from notes.db and shown sorted
./test-note-edit && echo PASS || echo FAIL
```

### `poc-05/` - WebAssembly / WASI proof of execution and testing

The first non-ELF target in the repo. `hello.wasm` is a 139-byte WebAssembly
binary module that runs as a WASI command under `wasmtime` and prints
`"Hello, wasm!\n"`. `test-hello.wasm` is a 364-byte WebAssembly binary test
module that opens its sibling `hello.wasm` via a preopened directory and
checks fixed bytes in the module header and greeting payload, exiting `0` on
success and `1` on failure.

- `poc-05/hello.wasm` - the main WebAssembly target
- `poc-05/hello.wasm.md` - byte-by-byte explanation of the module format,
  imports, exports, code, and data segment
- `poc-05/test-hello.wasm` - the runnable WebAssembly test binary
- `poc-05/test-hello.wasm.md` - byte-by-byte explanation of the test module

Run it:

```
cd poc-05
~/.wasmtime/bin/wasmtime run hello.wasm
~/.wasmtime/bin/wasmtime run --dir=. test-hello.wasm && echo PASS || echo FAIL
```

This target relies on the external runtime `wasmtime`, which is allowed by
rule 6 and is not committed to the repo. Under rule 2, the runtime should be
installed or supplied as an existing binary artifact (for example from an
official prebuilt release or a package manager), not via a temporary
human-readable helper script written as part of the workflow.

### `poc-06/` - Linux ELF ARM64 proof of execution and testing

The first runnable non-x86 ELF target in the repo. `hello` is a 166-byte
Linux ELF64 **AArch64** executable that writes `"Hello, arm64!\n"` and exits.
`test-hello` is a 294-byte Linux ELF64 **AArch64** test binary that opens
its sibling `hello`, reads the first 192 bytes into memory, and checks the ELF
magic plus the fixed greeting bytes at file offset `152`. It exits `0` on
success and `1` on failure.

- `poc-06/hello` - the main ARM64 ELF target
- `poc-06/hello.md` - byte-by-byte explanation of the ELF header, program
  header, AArch64 instructions, and inline greeting data
- `poc-06/test-hello` - the runnable ARM64 test binary
- `poc-06/test-hello.md` - byte-by-byte explanation of the test

Run it:

```
cd poc-06
qemu-aarch64-static ./hello
qemu-aarch64-static ./test-hello && echo PASS || echo FAIL
```

This target relies on `qemu-user-static` as allowed by rule 6. On the current
machine it was installed with:

```
sudo apt install qemu-user-static
```

### `poc-07/` - Linux ELF RISC-V 64 proof of execution and testing

The first runnable `RISC-V 64-bit` target in the repo. `hello` is a 169-byte
Linux ELF64 **RISC-V** executable that writes `"Hello, rv64!\n"` and exits.
`test-hello` is a 470-byte Linux ELF64 **RISC-V** test binary that opens its
own sibling `hello`, reads the first 192 bytes into memory, and checks:

- the ELF magic
- the greeting payload bytes at file offset `156`
- one code byte of the `addi a1, a1, 32` pointer instruction so the earlier
  pointer bug is caught too

It exits `0` on success and `1` on failure.

- `poc-07/hello` - the main RV64 ELF target
- `poc-07/hello.md` - byte-by-byte explanation of the ELF layout, RV64
  instructions, and inline greeting data
- `poc-07/test-hello` - the runnable RV64 test binary
- `poc-07/test-hello.md` - byte-by-byte explanation of the test

Run it:

```
cd poc-07
qemu-riscv64-static ./hello
qemu-riscv64-static ./test-hello && echo PASS || echo FAIL
```

### `poc-08/` - Windows PE x86_64 first target

The first Windows-target artifact in the repo. `hello.exe` is a 1024-byte
`PE32+` console executable for **Windows x86_64**. It contains one `.text`
section holding both code and import metadata. Its intended runtime behaviour is
simple:

- call `GetStdHandle(STD_OUTPUT_HANDLE)`
- call `WriteFile` to print `"Hello, win64!\r\n"`
- call `ExitProcess(0)`

The companion test is presently a **Linux ELF x86_64** machine-code verifier
rather than a PE test runner. That keeps the Windows artifact testable on this
box quickly and deterministically while still staying inside the repo rules.

- `poc-08/hello.exe` - the main Windows PE64 target
- `poc-08/hello.exe.md` - byte-by-byte explanation of the PE header, single
  section, import table, and Win64 calling sequence
- `poc-08/test-hello` - runnable Linux ELF x86_64 structural test for
  `hello.exe`
- `poc-08/test-hello.md` - byte-by-byte explanation of the host-side verifier

Run it:

```
cd poc-08
./test-hello && echo PASS || echo FAIL
wine hello.exe
```

What is verified today:

- `file` recognises `hello.exe` as `PE32+ executable (console) x86-64`
- `objdump -x` recognises a valid PE32+ header, `Windows CUI` subsystem, an
  import table, and imported function names from `kernel32`
- `test-hello` exits `0` on the committed bytes and `1` if either the greeting
  pointer byte or greeting payload bytes in `hello.exe` are corrupted
- `wine hello.exe` runs the PE and prints `Hello, win64!`

Wine still emits an environment warning about missing `wine32`, but the 64-bit
PE itself now executes successfully.

### `poc-09/` - Android-style ARM64 PIE native target

The first Android-oriented native target in the repo. `hello` is a 168-byte
`ELF64` **AArch64** binary built as `ET_DYN` / PIE instead of the earlier
fixed-address `ET_EXEC` format used by `poc-06/`. It prints:

```text
Hello, android!
```

using direct Linux syscalls only.

This is not an APK and does not use the Android Java/Kotlin app layer. It is a
small native PIE executable, which is the relevant low-level step toward
Android support under this repo's machine-code-only rules.

- `poc-09/hello` - the main Android-style ARM64 PIE target
- `poc-09/hello.md` - byte-by-byte explanation of the `ET_DYN` ELF header,
  load segment, PC-relative code, and greeting data
- `poc-09/test-hello` - runnable AArch64 verifier for `hello`
- `poc-09/test-hello.md` - byte-by-byte explanation of the verifier

Run it:

```
cd poc-09
qemu-aarch64-static ./hello
qemu-aarch64-static ./test-hello && echo PASS || echo FAIL
```

What is verified today:

- `file` recognises `hello` as an AArch64 PIE executable
- `readelf -h -l` reports `Type: DYN`
- `qemu-aarch64-static ./hello` prints `Hello, android!`
- `test-hello` exits `0` on the committed bytes and `1` if the target's
  `ET_DYN` type bytes are corrupted

This now serves as the low-level precursor to the real APK packaging step in
`poc-10/`.

### `poc-10/` - minimal Android APK demo

The first actual Android package in the repo. `hello.apk` is a signed APK that
uses `android.app.NativeActivity` with `android:hasCode="false"` and two
embedded native libraries:

- `lib/x86_64/libhello.so` for the emulator demo on this x86_64 host
- `lib/arm64-v8a/libhello.so` for real ARM64 Android devices

Both native payloads are still derived from literal machine-code bytes only.
Their exported Android entrypoint is:

```text
ANativeActivity_onCreate
```

and the current payload implementation is intentionally minimal: it just returns
immediately after the library is loaded.

- `poc-10/hello.apk` - the main Android APK target
- `poc-10/hello.apk.md` - explanation of the APK container, manifest semantics,
  embedded native libraries, and Android runtime verification
- `poc-10/test-hello` - Linux x86_64 binary verifier for the final APK bytes
- `poc-10/test-hello.md` - byte-by-byte explanation of the verifier

Run it:

```bash
cd poc-10
./test-hello && echo PASS || echo FAIL
adb install -r hello.apk
adb shell am start -W -n com.richard.machinewelcome.poc10/android.app.NativeActivity
```

What is verified today:

- `aapt dump badging` reports package
  `com.richard.machinewelcome.poc10`, launchable activity
  `android.app.NativeActivity`, and native-code ABIs `arm64-v8a` and `x86_64`
- `test-hello` exits `0` on the committed bytes and `1` if the APK signing
  block marker is corrupted
- `adb install -r hello.apk` succeeds on an Android 15 x86_64 emulator
- `adb shell am start -W ...` launches the activity successfully
- emulator `logcat` records that `lib/x86_64/libhello.so` is loaded successfully
- `dumpsys window` shows `com.richard.machinewelcome.poc10/android.app.NativeActivity`
  as the focused window

## Tools (self-built)

Per rule 6, the AI may use external tools, but per rules 1 and 2 any tool
the AI *commits* must itself be a binary with a Markdown companion. This
directory contains hand-hexed tools the AI uses to build other POCs in
this repo.

### `tools/mkelf` - Linux ELF x86_64 wrapper

A 349-byte binary that reads raw code+data on stdin and emits a complete
ELF64 executable on stdout. This replaces the otherwise-repetitive
120-byte ELF-header-plus-program-header boilerplate in every POC body.

- `tools/mkelf` - the tool
- `tools/mkelf.md` - byte-by-byte explanation, including the 120-byte output template and the self-modifying writes that patch `p_filesz` / `p_memsz`
- `tools/test-mkelf` - test binary (itself built with `mkelf`): `exit 0` iff the first 96 bytes on stdin match the expected `mkelf` header prefix
- `tools/test-mkelf.md` - byte-by-byte explanation of the test

Usage:

```
cd tools
echo 'b83c000000bf2a0000000f05' | xxd -r -p | ./mkelf > /tmp/exit42
chmod +x /tmp/exit42 ; /tmp/exit42 ; echo $?    # → 42

./mkelf < /dev/null | ./test-mkelf && echo PASS || echo FAIL
```

Self-hosting works: `dd if=mkelf bs=1 skip=120 | mkelf > mkelf-v2`
produces a functionally-identical rebuilt copy.

## How to use

1. Clone the repository.
2. `cd` into the POC directory for your platform.
3. Run the appropriate binary.
