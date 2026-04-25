# `product-notes/notes-win64.exe`

`notes-win64.exe` is a **1024-byte** `PE32+` Windows x86_64 console executable.
It is derived from [`../poc-08/hello.exe`](../poc-08/hello.exe.md), with the
console payload changed to:

```text
Notes, win64!
```

The PE imports and x86_64 code path are unchanged from the POC: it calls
`GetStdHandle`, `WriteFile`, and `ExitProcess`.

## File layout

```text
0x000..0x03f   64    DOS header
0x040..0x07f   64    DOS stub / padding
0x080..0x187  264    PE signature, COFF header, optional header
0x188..0x1af   40    `.text` section header
0x1b0..0x1ff   80    header padding
0x200..0x3ff  512    code, product string, imports, names
```

## Key bytes

At `0x000`:

```text
4d 5a
```

At `0x084`:

```text
64 86
```

This is the AMD64 PE machine type.

At `0x230`:

```text
4e 6f 74 65 73 2c 20 77 69 6e 36 34 21 0d 0a
```

ASCII:

```text
Notes, win64!
```

## Code

The x86_64 opcodes are byte-for-byte the same as `poc-08/hello.exe`; see
[`../poc-08/hello.exe.md`](../poc-08/hello.exe.md#code-at-0x200) for the full
instruction walk-through. The only product-specific change is the 15-byte string
consumed by the existing `WriteFile` call.

## Verification

`test-notes-win64` checks the `MZ` signature, AMD64 machine type, and product
string bytes.
