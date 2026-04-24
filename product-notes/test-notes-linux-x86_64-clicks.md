# `product-notes/test-notes-linux-x86_64-clicks`

`test-notes-linux-x86_64-clicks` is an **878-byte** statically-linked Linux
ELF64 x86_64 binary. It is a second structural verifier for
`./notes-linux-x86_64`, focused specifically on the click-routing and redraw
logic that was changed while adding per-row delete buttons.

Unlike `test-notes-linux-x86_64`, which only checks the visible product strings,
this verifier checks the byte ranges that anchor the current click behavior.

## Checked ranges

It validates nine machine-code or literal ranges in `notes-linux-x86_64`:

1. `0x435` — `83 fa 1e 72 33`
   - right-pane row-band Y compare
   - also proves the `jb` opcode after the compare is still intact
2. `0x443` — `b8 1e 00 00 00`
   - row-band origin immediate, aligning the first hit band with `y = 30`
3. `0x460` — `81 fe 20 02 00 00 73 0a e9 19 02 00 00`
   - split between normal row clicks and far-right `Del` clicks
4. `0x4a8` — `e9 e6 03 00 00`
   - delete path jumps into the persistent delete rewrite helper
5. `0x61b` — `e9 aa 00 00 00`
   - draw loop tail jumps into the stale-row clear helper
6. `0x686` — `89 2c 25 04 48 40 00 89 e8 c1 e0 06 05 00 43 40 00`
   - normal click path still stores the selected row and computes the slot base
7. `0x6ca` — `83 fd 14 7d 1d e8 21 fe ff ff 66 c7 04 25 e0 07 40 00 68 01 e8 22 fe ff ff 66 41 83 c6 10 ff c5 eb de e9 c3 fd ff ff`
   - tail-clear helper body for blanking unused right-pane rows after deletions
8. `0x878` — `44 65 6c`
   - `Del` label literal
9. `0x893` — `b8 02 00 00 00 bf c8 07 40 00 ... e9 3b fc ff ff`
   - persistent delete helper that truncates and rewrites `notes.db`

Exit status:

- `0` = pass
- `1` = fail

## Why this test exists

The recent regressions were not in the stable ELF header or the obvious title
strings. They were in short, easy-to-break click-handler bytes:

- a row-band compare immediate
- the row-band origin immediate itself
- a branch opcode beside that compare
- the split between load clicks and delete clicks
- the jump target used after a delete
- the redraw helper that clears stale rows
- the helper that persists deletions to disk

This verifier gives those paths a stable, always-runnable regression check on
the Linux host.

## File layout

`mkelf` wraps an `823`-byte body:

```text
0x000..0x077   120   ELF header + PT_LOAD program header
0x078..0x24c   469   open + nine pread64 / compare blocks + exit paths
0x24d..0x3ae   354   path string and expected byte literals
```

Total file size: `943` bytes.

## Overall logic

Pseudocode:

```text
fd = open("notes-linux-x86_64", O_RDONLY)

for each (offset, expected_bytes):
    pread64(fd, buf, len(expected_bytes), offset)
    compare buf against expected_bytes

exit(0) on success, else exit(1)
```

## Code walk-through

### Block 1 — open sibling binary

The opening sequence is the same shape as the earlier structural verifier:

```text
b8 02 00 00 00       mov eax, 2
bf 4d 02 40 00       mov edi, 0x40024d
31 f6                xor esi, esi
31 d2                xor edx, edx
0f 05                syscall
48 85 c0             test rax, rax
0f 88 ...            js fail
48 89 c3             mov rbx, rax
```

This opens `notes-linux-x86_64` read-only and keeps the fd in `rbx`. In the
current rebuilt verifier the embedded path begins at `0x4002c5`.

### Block 2 — repeated `pread64` checks

Each check uses the same template:

```text
b8 11 00 00 00       mov eax, 17
48 89 df             mov rdi, rbx
be 00 10 40 00       mov esi, 0x401000
ba <len>             mov edx, len
41 ba <offset>       mov r10d, file_offset
0f 05                syscall
48 83 f8 <len>       cmp rax, len
0f 85 ...            jne fail

be 00 10 40 00       mov esi, 0x401000
bf <expected>        mov edi, expected_literal
b9 <len>             mov ecx, len
fc                   cld
f3 a6                repe cmpsb
0f 85 ...            jne fail
```

Linux x86_64 syscall `17` is `pread64`, so the verifier can read anchored byte
ranges without needing to seek.

### Block 3 — success and failure exits

Success:

```text
b8 3c 00 00 00
31 ff
0f 05
```

Failure:

```text
b8 3c 00 00 00
bf 01 00 00 00
0f 05
```

Linux syscall `60` is `exit`.

## Embedded literals

### Path at `0x40024d`

```text
6e 6f 74 65 73 2d 6c 69 6e 75 78 2d 78 38 36 5f 36 34 00
```

ASCII:

```text
notes-linux-x86_64
```

### Expected byte blocks

Embedded expected ranges begin at `0x4002d8` and are laid out in the same order
as the checks listed above.

## Syscalls used

Exactly three Linux x86_64 syscalls:

- `2` = `open`
- `17` = `pread64`
- `60` = `exit`

## What this test proves

It proves that the committed Linux product reference binary still contains the
current click-routing, delete-routing, normal-load, stale-row-clear, and
persistent-delete rewrite byte sequences at the expected file offsets.

It does **not** prove full runtime X11 behavior, and it does not synthesize
real repeated pointer clicks. It is still structural. Its value is that it
catches the exact class of byte-level regressions that recently broke repeated
click handling.
