# Welcome to the Machine

Machine language only.

## Motivation

In early 2026, Elon Musk said that in future, developers would write code in natural language and AI would generate machine code directly, without the need for human-readable code.

I wanted to explore this idea and see how far I could get.

## How it works

The user writes a natural language description of what they want to build.

The AI then generates a machine code implementation of the description.

The AI will also generate a Markdown file with a detailed explanation of the code and how it works.

## Rules for the AI

These rules are binding for all contributions to this repo:

1. **Binaries only.** The AI can produce only *binaries* - directly-executable machine code in the native format of one of the target architectures (ELF, Mach-O, PE, WebAssembly, ...). It may **not** commit source code in any human-readable language (C, Python, JavaScript, assembly, etc.) or hand-written build scripts. Binaries may be produced via tools such as `xxd -r -p` fed with literal hex bytes, but those hex bytes **are** the source of truth - they are not "compiled" from anything else.
2. **Every binary ships with a Markdown explanation.** For each binary committed, a matching `<binary>.md` must describe its ELF/PE/... layout and every opcode, byte by byte, in enough detail that a reader can mentally single-step through the code.
3. **Tests as binaries.** For each target binary, ship a test binary (also in pure machine code) that exercises it. Tests use the standard Unix exit-code convention: `0` for pass, non-zero for fail.
4. **Cross-architecture.** Produce machine-code test suites for other target architectures as soon as possible, running them under virtual machines or emulators (QEMU, Wasmtime, Apple Silicon VM, ...).
5. **External help is allowed.** The AI may download or ask the user to install tools (assemblers, disassemblers, emulators, linkers) that help produce or verify binaries. These tools are not committed.
6. **Libraries and linking are allowed.** Binaries may link against libraries (libc, X11, Metal, ...), but the Markdown must document the exact interface used (calling convention, struct layouts, symbols).

## Tech Details

We will attempt to build for multiple platforms:

- Linux ELF x86_64 - **current** (see `poc-01/`, `poc-02/`)
- Linux ELF ARM64
- macOS Universal ARM64
- iOS Universal ARM64
- Android ARM64
- WebAssembly
- Windows PE x86_64
- Windows PE ARM64
- RISC-V 64-bit
- Quantum Qubit

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

## Tools (self-built)

Per rule 5, the AI may use external tools, but per rules 1 and 2 any tool
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
