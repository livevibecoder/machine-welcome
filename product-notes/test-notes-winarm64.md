# `product-notes/test-notes-winarm64`

`test-notes-winarm64` is a **363-byte** Linux x86_64 ELF structural verifier for
`./notes-winarm64.exe`.

It checks:

- `0x000` length 2: `4d 5a`
- `0x084` length 2: `64 aa`
- `0x200` length 8: `00 00 80 52 c0 03 5f d6`
- `0x230` length 15: `Notes, win64!\r\n`

## Code shape

The verifier uses `open`, repeated `pread64` anchored reads, `repe cmpsb`, and
`exit`. A mismatch exits `1`; all matches exit `0`.

## Embedded expected bytes

```text
4d 5a
64 aa
00 00 80 52 c0 03 5f d6
4e 6f 74 65 73 2c 20 77 69 6e 36 34 21 0d 0a
```
