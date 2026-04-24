# `product-notes/test-notes-linux-x86_64`

`test-notes-linux-x86_64` is a **379-byte** statically-linked Linux ELF64
x86_64 binary. It is a headless structural verifier for the Linux product
reference build `./notes-linux-x86_64`.

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
b8 02 00 00 00       mov eax, 2
bf 55 01 40 00       mov edi, 0x400155
31 f6                xor esi, esi
31 d2                xor edx, edx
0f 05                syscall
48 85 c0             test rax, rax
0f 88 b8 00 00 00    js fail
48 89 c3             mov rbx, rax
```

This opens:

```text
notes-linux-x86_64
```

and keeps the file descriptor in `rbx`.

### Block 2 — verify title at `0x784`

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

Linux syscall `17` is `pread64`. This reads the 9-byte title payload into the
BSS buffer at `0x401000`.

The compare block that follows is:

```text
be 00 10 40 00       mov esi, 0x401000
bf 68 01 40 00       mov edi, 0x400168
b9 09 00 00 00       mov ecx, 9
fc                   cld
f3 a6                repe cmpsb
0f 85 79 00 00 00    jne fail
```

So it compares against:

```text
notes-x64
```

### Block 3 — verify `Note:` at `0x824`

The second pread64 block is identical in shape but uses:

- length `5`
- file offset `0x824`

and then compares against the 5-byte literal:

```text
Note:
```

### Block 4 — verify `Words` at `0x870`

The third pread64 block again uses length `5`, now at file offset `0x870`, and
compares against:

```text
Words
```

### Block 5 — success / fail exits

Success:

```text
b8 3c 00 00 00
31 ff
0f 05
```

Fail:

```text
b8 3c 00 00 00
bf 01 00 00 00
0f 05
```

Linux syscall `60` is `exit`.

## Embedded literals

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
