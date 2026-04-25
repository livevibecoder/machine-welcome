# `product-notes/test-notes-win64`

`test-notes-win64` is a **343-byte** Linux x86_64 ELF structural verifier for
`./notes-win64.exe`.

It checks:

- `0x000` length 2: `4d 5a`
- `0x084` length 2: `64 86`
- `0x230` length 15: `Notes, win64!\r\n`

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
4e 6f 74 65 73 2c 20 77 69 6e 36 34 21 0d 0a
```
