# `product-notes/notes-linux-x86_64`

`notes-linux-x86_64` is a **2346-byte** statically-linked Linux ELF64 x86_64
binary. It is the first product-oriented note tool in the repo and is derived
directly from the earlier raw-X11 editor documented in
[`../poc-04/note-edit.md`](../poc-04/note-edit.md).

The file keeps the same overall container size and most of the same machine
code, but patches the title, labels, draw path, and event dispatch so the UI
behaves like a two-pane note tool:

- left side: full note editor
- right side: alphabetical first-word list
- mouse click on the right list: load note into the editor
- click the per-row `Del` affordance at the far right: remove that row and
  rewrite `notes.db` so the deletion persists
- `Enter`: save the current editor contents as a note record

## What stayed the same

Large parts of the binary are intentionally unchanged from `poc-04/note-edit`:

- ELF header and `PT_LOAD`
- X11 setup handshake and resource-id patching
- hard-coded X socket / cookie / root window coupling
- note loader and insertion sort
- key handling table and input buffer limits
- helper routines `fill_it8_spaces`, `send_it8`, and `exit_app`
- the old note loader and insertion sort

For those unchanged regions, the canonical byte-level explanation is still
[`../poc-04/note-edit.md`](../poc-04/note-edit.md).

One intentional X11 template change was also made in-place: the window's event
mask now requests `ButtonPress` in addition to the earlier `KeyPress` and
`Expose` delivery, so the product's click-to-load and click-to-delete paths can
actually receive mouse events from the X server.

## On-disk note format

Like the earlier note tools, the database is:

```text
[4-byte little-endian length][length bytes of text]
```

The current Linux product build now has split persistence behavior:

- clicking a note loads it into the editor pane
- clicking `Del` rewrites `notes.db` from the current in-memory note slots
- editing changes the left-pane text buffer
- pressing `Enter` stores the current left-pane contents as a new record

The delete rewrite helper keeps the same outer framing:

```text
[4-byte little-endian length][length bytes of text]
```

but currently emits each surviving note as a 64-byte padded payload record. The
loader already accepts those fixed-width records because `64` is still within
its existing accepted length range.

## Fixed string patches

Four anchored strings are now changed or populated in-place:

### Window title at file offset `0x784`

```text
6e 6f 74 65 73 2d 78 36 34
```

ASCII:

```text
notes-x64
```

This replaces the older `note-edit` title while keeping the same 9-byte payload
size inside the `ChangeProperty(WM_NAME=...)` template.

### Left-pane label at `0x824`

```text
4e 6f 74 65 3a
```

ASCII:

```text
Note:
```

This replaces the old `"New: "` label.

### Right-pane header at `0x870`

```text
57 6f 72 64 73
```

ASCII:

```text
Words
```

This is written into previously unused zero padding near the end of the old
keymap area and used by the new draw routine.

### Delete label at `0x878`

```text
44 65 6c
```

ASCII:

```text
Del
```

The draw helper copies these three bytes into the `ImageText8` payload when it
renders the per-row delete affordance near the right edge of the list pane.

## Control-flow patches

Three old control-flow sites were redirected into new code stored in the
zero-filled gap between the original code and data sections.

### Startup redirect at `0x1d9`

The original jump into `redraw`:

```text
e9 1b 01 00 00
```

became:

```text
e9 5b 03 00 00
```

which jumps to the new startup stub in the padding area.

### Draw redirect at `0x417`

The first call in the old draw sequence was replaced with:

```text
e9 2d 01 00 00
```

which transfers control to the new pane-oriented draw routine.

### Event dispatch redirect at `0x4d7`

The old `Expose` / `KeyPress` compare chain was replaced with:

```text
e9 44 01 00 00
```

which jumps to the new event dispatcher.

## New code regions: `0x539..0x6e6` and `0x893..0x90d`

The old binary had a long zero padding region between the code and fixed data
base, plus another unused zero pocket near the later data area.
`notes-linux-x86_64` uses both spaces for new behavior without moving the data
section.

### Region overview

| Range | Purpose |
| --- | --- |
| `0x539..0x548` | startup stub |
| `0x549..0x61f` | two-pane draw routine |
| `0x620..0x645` | event dispatcher |
| `0x646..0x6c9` | row draw helper plus click hit-testing / row action logic |
| `0x6ca..0x6e6` | tail clear helper for unused right-pane rows |
| `0x893..0x90d` | persistent delete rewrite helper |

### Startup stub

The first bytes in the new region are:

```text
c7 04 25 04 48 40 00 ff ff ff ff
e9 b0 fd ff ff
```

Meaning:

- `mov dword [0x404804], 0xffffffff` — initialise the selected-row slot to
  `-1`
- `jmp redraw` — rejoin the old load/sort path after initialisation

### Two-pane draw routine

The new draw code:

1. clears the fixed-width `ImageText8` payload buffer
2. patches the X position field in the text template
3. draws `"Note:"` at the left pane
4. draws the current editor buffer beneath it
5. draws `"Words"` at the right pane
6. loops over the sorted slot array and copies only the first word of each note
   into the right-pane text buffer
7. sends one `ImageText8` request per visible row
8. draws a small `Del` affordance to the right of each visible row
9. blanks any now-unused lower right-pane rows left behind after deletions
10. jumps back to the existing event loop

Important absolute stores in this routine patch the `x` field at
`0x4007e0` before calling the existing `send_it8` helper.

The key immediate X coordinates are:

- `0x0010` (`16`) for the left editor pane
- `0x0168` (`360`) for the right list pane
- `0x0220` (`544`) for the per-row delete affordance

The copied Y coordinates are:

- `30` for pane headers
- `46` for the editor text line and the first list row
- then `+16` per subsequent list row

After the visible rows are drawn, a small tail helper now emits blank
right-pane rows from the current `NOTE_COUNT` up to the 20-row visual limit.
That prevents deleted rows from leaving stale text on screen that no longer
corresponds to any live note slot.

### Event dispatcher

The new dispatcher reads:

```text
EVENT_BUF[0] & 0x7f
```

and branches as follows:

- `Expose (12)` -> jump to existing `redraw`
- `KeyPress (2)` -> jump to the existing key handler
- `ButtonPress (4)` -> jump to the new click handler
- anything else -> jump back to the event loop

This is the only behavioral difference in event dispatch. The old binary simply
ignored button presses.

### Click handler

The new click handler reads:

- `event_x` from `EVENT_BUF + 24` (`0x404018`)
- `event_y` from `EVENT_BUF + 26` (`0x40401a`)

Then it:

1. rejects clicks left of `x = 360`
2. rejects clicks above the top of the first list row at `y = 30`
3. walks the note rows in 16-pixel bands
4. uses 16-pixel row bands starting at `y = 30`, which matches the visible text
   bands better than using the `ImageText8` baseline directly
5. treats clicks at or beyond `x = 544` as delete-button hits
6. otherwise jumps into the existing note-load path
7. for delete hits, shifts later 64-byte note slots down over the chosen row
8. decrements the in-memory note count
9. jumps into a later helper that truncates and rewrites `notes.db`

That makes mouse selection load the full note into the left editor pane, while
far-right clicks on the same row act as a persistent delete request.

### Persistent delete helper

The extra helper stored at `0x893..0x90d` is reached from the delete path after
the in-memory row compaction is complete.

Its logic is:

1. `open("notes.db", O_WRONLY | O_CREAT | O_TRUNC, 0644)`
2. store constant length `64` into the scratch word at `0x404184`
3. loop over the current in-memory slot array from row `0` to `NOTE_COUNT - 1`
4. `write(fd, &scratch_len, 4)`
5. `write(fd, slot, 64)`
6. `close(fd)`
7. jump back to the old reload path so the live view is rebuilt from disk

This helper is intentionally compact, so it serializes each surviving note as a
fixed-width 64-byte payload. That is larger than the minimal append path used by
`Enter`, but still compatible with the loader.

## Runtime memory usage

The unchanged BSS layout from `poc-04/note-edit` still applies, with one extra
product-specific slot now used:

- `0x404804` — selected row index, initialised to `-1`

## Known limitations

This first product reference binary is intentionally still constrained:

- same hard-coded X11 session coupling as `poc-04`
- same fixed keyboard map
- same 63-character editor limit
- same 20-note in-memory display cap
- `Enter` still appends new records rather than rewriting the file
- right pane shows only the first word, but sorting still follows the full
  stored note bytes
- delete rewrites surviving notes as 64-byte fixed-width payload records rather
  than minimal-length records
- the two panes are logical drawing regions, not toolkit-managed widgets

Those limitations are acceptable for the Linux x86_64 product reference build
because the goal here is to establish the cross-platform behavior contract in a
real native GUI binary.
