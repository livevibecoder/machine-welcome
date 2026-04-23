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

- Linux ELF x86_64 - **current** (see `poc-01/`)
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

## How to use

1. Clone the repository.
2. `cd` into the POC directory for your platform.
3. Run the appropriate binary.
