# `product-notes/notes-winarm64.exe`

`notes-winarm64.exe` is a **1024-byte** `PE32+` Windows ARM64 console executable
container. It reuses the PE layout from `notes-win64.exe`, changes the COFF
machine type to ARM64, and replaces the entrypoint with a tiny ARM64 return
stub.

This is an initial structural Windows ARM64 product artifact. It does not yet
implement the Win32 Notes GUI described in [`windows-plan.md`](windows-plan.md).

## Key bytes

At `0x000`:

```text
4d 5a
```

At `0x084`:

```text
64 aa
```

This is the Windows ARM64 machine type.

At `0x200`, the entry stub is:

```text
00 00 80 52   mov   w0, #0
c0 03 5f d6   ret
```

At `0x230`, the product marker string inherited from the x86_64 sibling is:

```text
4e 6f 74 65 73 2c 20 77 69 6e 36 34 21 0d 0a
```

ASCII:

```text
Notes, win64!
```

The retained import-table bytes are inert for the current return stub but keep
the PE section layout identical to the x86_64 sibling while a real ARM64 Win32
GUI path is still pending.

## Verification

`test-notes-winarm64` checks the `MZ` signature, ARM64 machine type, ARM64 entry
stub, and product marker string.
