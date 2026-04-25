# `product-notes/notes-web.wasm`

`notes-web.wasm` is a **139-byte** WebAssembly MVP module using WASI Preview 1.
It is derived from [`../poc-05/hello.wasm`](../poc-05/hello.wasm.md), with the
linear-memory string changed to:

```text
Notes, wasm!
```

The module still exports `memory` and `_start`, imports `fd_write`, writes one
iovec to stdout, and returns normally.

## Key sections

The module begins:

```text
00 61 73 6d 01 00 00 00
```

The code section is unchanged from the WASI POC:

```text
41 01 41 00 41 01 41 18 10 00 1a 0b
```

This pushes stdout fd `1`, iovec pointer `0`, iovec count `1`, byte-count
scratch pointer `0x18`, calls imported `fd_write`, drops the return code, and
ends.

## Product string

At file offset `0x7a`:

```text
4e 6f 74 65 73 2c 20 77 61 73 6d 21 0a
```

ASCII:

```text
Notes, wasm!
```

## Verification

`test-notes-web` is a Linux x86_64 structural verifier. It checks the Wasm magic
and product string bytes.
