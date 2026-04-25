# `product-notes/test-notes-web`

`test-notes-web` is a **329-byte** Linux x86_64 ELF structural verifier for
`./notes-web.wasm`.

It checks:

- `0x000` length 4: `00 61 73 6d`
- `0x07a` length 13: `Notes, wasm!\n`

The verifier uses `open`, `pread64`, `repe cmpsb`, and `exit`.

## Embedded expected bytes

```text
00 61 73 6d
4e 6f 74 65 73 2c 20 77 61 73 6d 21 0a
```
