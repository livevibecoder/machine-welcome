# `product-notes/test-notes-win64`

`test-notes-win64` is a **357-byte** Linux x86_64 ELF structural verifier for
`./notes-win64.exe`.

It checks:

- `0x000` length 2: `4d 5a`
- `0x084` length 2: `64 86`
- `0x0dc` length 2: `02 00`
- `0x230` length 15: `Notes GUI x64\r\n`

## Code shape

The verifier opens `notes-win64.exe`, reads each anchored range with `pread64`,
compares with `repe cmpsb`, and exits `0` only if all ranges match.

Syscalls used:

- `2` = `open`
- `17` = `pread64`
- `60` = `exit`

## Embedded expected bytes

```text
4d 5a
64 86
02 00
4e 6f 74 65 73 20 47 55 49 20 78 36 34 0d 0a
```
