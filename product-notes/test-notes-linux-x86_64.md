# `product-notes/test-notes-linux-x86_64`

`test-notes-linux-x86_64` is a **379-byte** statically-linked Linux ELF64
x86_64 binary. It is a headless structural verifier for the Linux product
reference build `./notes-linux-x86_64`.

Terminology for [BSS](glossary.md#bss), syscalls, and similar implementation
phrases: [product notes glossary](glossary.md).

It checks three anchored byte ranges:

1. window title `notes-x64` at file offset `0x784`
2. left-pane label `Note:` at file offset `0x824`
3. right-pane header `Words` at file offset `0x870`

Exit status:

- `0` = pass
- `1` = fail

Verified behavior:

- it exits `0` on the committed binary
- corrupting byte `0x784` makes it exit `1`
- restoring the original byte returns it to `0`

## Why this test is structural

The target is a GUI program tied to the live X11 session cookie and socket, so a
small structural verifier is the safest always-runnable regression check on the
Linux host. It proves that the patched product-specific strings and therefore
the redirected build are present in the expected file layout.

## File layout

`mkelf` wraps a `259`-byte body:

```text
0x000..0x077   120   ELF header + PT_LOAD program header
0x078..0x0dc   101   open + three pread64 blocks + compares
0x0dd..0x0e8    12   success / fail exits
0x0e9..0x0fb    19   path string "notes-linux-x86_64\0"
0x0fc..0x104     9   expected title
0x105..0x109     5   expected left-pane label
0x10a..0x10e     5   expected right-pane header
```

Total file size: `379` bytes.

## Overall logic

Pseudocode:

```text
fd = open("notes-linux-x86_64", O_RDONLY)

pread64(fd, buf, 9, 0x784)
memcmp(buf, "notes-x64", 9)

pread64(fd, buf, 5, 0x824)
memcmp(buf, "Note:", 5)

pread64(fd, buf, 5, 0x870)
memcmp(buf, "Words", 5)

exit(0) on success, else exit(1)
```

## Code walk-through

### Block 1 — open sibling binary

```text
; --- Open ./notes-linux-x86_64 read-only; keep fd in RBX for pread64 ---
; Glossary: [mkelf](glossary.md#mkelf), [anchored structural test](glossary.md#anchored-structural-test), [syscalls](glossary.md#linux-syscalls-x86-64-syscall).
400078: b8 02 00 00 00                mov     eax, 0x2
40007d: bf 55 01 40 00                mov     edi, 0x400155
400082: 31 f6                         xor     esi, esi
400084: 31 d2                         xor     edx, edx
400086: 0f 05                         syscall
400088: 48 85 c0                      test    rax, rax
40008b: 0f 88 b8 00 00 00             js      fail
400091: 48 89 c3                      mov     rbx, rax
```

(Equivalent `xxd` / commented form in older revisions:)

```text
b8 02 00 00 00       mov eax, 2
bf 55 01 40 00       mov edi, 0x400155
31 f6                xor esi, esi
31 d2                xor edx, edx
0f 05                syscall
48 85 c0             test rax, rax
0f 88 b8 00 00 00    js fail
48 89 c3             mov rbx, rax
```

**Execution logic:** this is the standard static `open` setup: `eax=2` selects
`__NR_open`, `edi` points at the first byte of the embedded path, `esi=0` and
`edx=0` request read-only open with no extra flags, and the syscall is invoked
with `syscall`. A negative return is treated as failure via `js` to the
non-zero exit. On success, the fd in `rax` is copied to `rbx` so later syscalls
can keep the same fd value while `pread64` repurpose `rax` for syscall numbers.

This opens:

```text
notes-linux-x86_64
```

and keeps the file descriptor in `rbx`.

### Block 2 — verify title at `0x784`

```text
; --- Pread 9 bytes at 0x784; compare to embedded "notes-x64" (ChangeProperty title) ---
; [pread64](glossary.md#pread64), [ImageText8 / WM_NAME](glossary.md#changeproperty-and-wm_name) context in glossary.
400094: b8 11 00 00 00                mov     eax, 0x11
400099: 48 89 df                      mov     rdi, rbx
40009c: be 00 10 40 00                mov     esi, 0x401000
4000a1: ba 09 00 00 00                mov     edx, 0x9
4000a6: 41 ba 84 07 00 00             mov     r10d, 0x784
4000ac: 0f 05                         syscall
4000ae: 48 83 f8 09                   cmp     rax, 0x9
4000b2: 0f 85 91 00 00 00             jne     fail
```

(Commented mnemonic-only view:)

```text
b8 11 00 00 00       mov eax, 17
48 89 df             mov rdi, rbx
be 00 10 40 00       mov esi, 0x401000
ba 09 00 00 00       mov edx, 9
41 ba 84 07 00 00    mov r10d, 0x784
0f 05                syscall
48 83 f8 09          cmp rax, 9
0f 85 91 00 00 00    jne fail
```

**Execution logic:** this is a fixed-offset `pread64`: set `__NR_pread64` in
`eax`, set `rdi` to the open fd, set `esi` to a scratch buffer, `edx` to the
read length, and set `r10d` to the in-file offset (here `0x784`). The syscall
returns the number of bytes read in `rax`; compare that to the requested
length, and if they differ, jump to the failure exit so a truncated or error
read cannot pass as a match.

Linux syscall `17` is `pread64`. This reads the 9-byte title payload into the
BSS buffer at `0x401000`.

The compare block that follows is:

```text
4000b8: be 00 10 40 00                mov     esi, 0x401000
4000bd: bf 68 01 40 00                mov     edi, 0x400168
4000c2: b9 09 00 00 00                mov     ecx, 0x9
4000c7: fc                            cld
4000c8: f3 a6                         repz cmpsb
4000ca: 0f 85 79 00 00 00             jne     fail
```

```text
be 00 10 40 00       mov esi, 0x401000
bf 68 01 40 00       mov edi, 0x400168
b9 09 00 00 00       mov ecx, 9
fc                   cld
f3 a6                repe cmpsb
0f 85 79 00 00 00    jne fail
```

**Execution logic:** `esi` is the just-read data in BSS, `edi` is the
immutable expected string embedded in the test binary, `ecx` is the compare
count, and `cld` ensures string instructions move upward in memory. `repe
cmpsb` compares the buffers while equal and decrements `ecx` each step; it stops
on the first mismatch, leaving a mismatching pair or exhausting `ecx`. A
mismatch (or the flags state after the repeat) is followed by a branch on `jne`
to the failure path.

So it compares against:

```text
notes-x64
```

### Block 3 — verify `Note:` at `0x824`

**Execution logic:** this repeats the `pread64` + `repe cmpsb` pattern with a new
`r10` offset and new expected string pointer, but the control flow is the same
as in Block 2: read exactly the requested number of bytes, then compare
memory-to-memory with `DF=0`.

**Disassembly (`0x4000d0`–`0x400106`, fall-through of prior blocks; failure
targets the shared `fail` entry at `0x400149`):**

```text
4000d0: b8 11 00 00 00                mov     eax, 0x11
4000d5: 48 89 df                      mov     rdi, rbx
4000d8: be 00 10 40 00                mov     esi, 0x401000
4000dd: ba 05 00 00 00                mov     edx, 0x5
4000e2: 41 ba 24 08 00 00             mov     r10d, 0x824
4000e8: 0f 05                         syscall
4000ea: 48 83 f8 05                   cmp     rax, 0x5
4000ee: 0f 85 55 00 00 00             jne     0x400149
4000f4: be 00 10 40 00                mov     esi, 0x401000
4000f9: bf 71 01 40 00                mov     edi, 0x400171
4000fe: b9 05 00 00 00                mov     ecx, 0x5
400103: fc                            cld
400104: f3 a6                         repz cmpsb
400106: 0f 85 3d 00 00 00             jne     0x400149
```

The second pread64 block is identical in shape but uses:

- length `5`
- file offset `0x824`

and then compares against the 5-byte literal:

```text
Note:
```

### Block 4 — verify `Words` at `0x870`

**Execution logic:** same as Block 3, only the offset and the embedded expected
string differ.

**Disassembly (`0x40010c`–`0x40013e`, shared `fail` at `0x400149`):**

```text
40010c: b8 11 00 00 00                mov     eax, 0x11
400111: 48 89 df                      mov     rdi, rbx
400114: be 00 10 40 00                mov     esi, 0x401000
400119: ba 05 00 00 00                mov     edx, 0x5
40011e: 41 ba 70 08 00 00             mov     r10d, 0x870
400124: 0f 05                         syscall
400126: 48 83 f8 05                   cmp     rax, 0x5
40012a: 75 1d                         jne     0x400149
40012c: be 00 10 40 00                mov     esi, 0x401000
400131: bf 76 01 40 00                mov     edi, 0x400176
400136: b9 05 00 00 00                mov     ecx, 0x5
40013b: fc                            cld
40013c: f3 a6                         repz cmpsb
40013e: 75 09                         jne     0x400149
```

The third pread64 block again uses length `5`, now at file offset `0x870`, and
compares against:

```text
Words
```

### Block 5 — success / fail exits

This ELF has no section table; the listing below is from `objdump` on the
code slice starting at the entry `0x400078` (same raw bytes as the product
binary layout).

Success:

```text
400140: b8 3c 00 00 00                mov     eax, 0x3c
400145: 31 ff                         xor     edi, edi
400147: 0f 05                         syscall
```

(Short form:)

```text
b8 3c 00 00 00
31 ff
0f 05
```

**Execution logic:** set `__NR_exit`, pass exit status `0` in `edi` via
`xor edi, edi`, and trap into the kernel.

Fail (single shared label; all failure branches land here first):

```text
400149: b8 3c 00 00 00                mov     eax, 0x3c
40014e: bf 01 00 00 00                mov     edi, 0x1
400153: 0f 05                         syscall
```

(Short form:)

```text
b8 3c 00 00 00
bf 01 00 00 00
0f 05
```

**Execution logic:** set `__NR_exit`, pass exit status `1` in `edi`, and invoke
`syscall` so the shell can distinguish failure with `$?`.

Linux syscall `60` is `exit`.

## Embedded literals

The hex blocks in this section are **read-only data** that happen to be in the
same segment as the code; the test program never jumps into these bytes.

### Path at `0x400155`

```text
6e 6f 74 65 73 2d 6c 69 6e 75 78 2d 78 38 36 5f 36 34 00
```

ASCII:

```text
notes-linux-x86_64
```

### Expected strings

At:

- `0x400168` — `notes-x64`
- `0x400171` — `Note:`
- `0x400176` — `Words`

## Syscalls used

Exactly three Linux x86_64 syscalls:

- `2` = `open`
- `17` = `pread64`
- `60` = `exit`

## What this test proves

It proves that the committed Linux product reference binary still contains the
product-specific title and pane-label patches at the expected anchored file
offsets.

It does **not** prove full GUI runtime behavior. That behavior depends on a live
X11 session whose cookie and socket details match the binary's baked data.
