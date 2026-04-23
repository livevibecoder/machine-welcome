# `poc-04/test-note-edit` — structural smoke test for `note-edit`

`test-note-edit` is a **249-byte** statically-linked ELF64 binary that checks
one fixed invariant of `./note-edit`:

> the 9 bytes at file offset `0x784` must be exactly `"note-edit"`

That byte range is the `WM_NAME` payload inside `note-edit`'s anchored
`ChangeProperty` request template.

The binary exits:

- `0` if the bytes match
- `1` otherwise

It does not open an X connection and does not depend on the current X session.

## Usage

```bash
cd poc-04
./test-note-edit && echo PASS || echo FAIL
```

## Why offset `0x784`?

`note-edit` fixes its data section at file offset `0x700`.

Inside that data section:

- `ChangeProperty` starts at data offset `0x6c`
- the string payload begins 24 bytes into that template

So:

```text
0x700 + 0x6c + 0x18 = 0x784
```

That is where the 9-byte `"note-edit"` title begins.

## Body layout

Like `test-note-view`, the body is tiny:

```text
body offset   size   content
-----------   ----   ----------------------------------
0x000..0x068  105    code
0x069..0x074   12    PATH     "note-edit\0" padded
0x075..0x080   12    EXPECTED "note-edit\0" padded
```

`mkelf` then prepends the standard 120-byte ELF header + PT_LOAD header.

## Code walkthrough

### Block 1 — open sibling binary

```text
mov eax, 2                ; sys_open
mov edi, 0x4000e1         ; "note-edit"
xor esi, esi              ; O_RDONLY
xor edx, edx
syscall
test rax, rax
js fail
mov rbx, rax
```

`rbx` holds the file descriptor for the remainder of the program.

### Block 2 — pread the 9-byte title at offset `0x784`

```text
mov eax, 17               ; sys_pread64
mov rdi, rbx
mov esi, 0x401000         ; BUF in BSS
mov edx, 9
mov r10d, 0x784
syscall
cmp rax, 9
jne fail
```

Using `pread64` avoids a separate `lseek`.

### Block 3 — compare against expected bytes

```text
mov esi, 0x401000         ; BUF
mov edi, 0x4000ed         ; EXPECTED
mov ecx, 9
cld
repe cmpsb
jne fail
```

If all 9 bytes match, `ZF=1` after `repe cmpsb` and the program falls through
to the success exit.

### Block 4 — success

```text
mov eax, 60
xor edi, edi
syscall
```

### Block 5 — fail

```text
mov eax, 60
mov edi, 1
syscall
```

## Syscalls used

Only three:

| nr | name      | purpose |
| -- | --------- | ------- |
| 2  | `open`    | open `./note-edit` |
| 17 | `pread64` | read the anchored title bytes |
| 60 | `exit`    | return pass/fail |

## Verified behaviour

Confirmed by corruption test:

1. overwrite byte `0x784` in `note-edit`
2. `test-note-edit` returns `1`
3. restore the original byte
4. `test-note-edit` returns `0`

So the test is a real guard, not a vacuous smoke test.

## What this test does not prove

- that the X11 handshake succeeds
- that text entry works
- that notes are sorted correctly
- that the current cookie/root window match the live X session

Those behaviours were verified manually and are described in `note-edit.md`.
This binary test is intentionally headless and structural so it can run in a
non-graphical environment.
