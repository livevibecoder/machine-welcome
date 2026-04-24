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
- `Enter`: save the current editor contents as a note record

## What stayed the same

Large parts of the binary are intentionally unchanged from `poc-04/note-edit`:

- ELF header and `PT_LOAD`
- X11 setup handshake and resource-id patching
- hard-coded X socket / cookie / root window coupling
- note loader and insertion sort
- key handling table and input buffer limits
- helper routines `fill_it8_spaces`, `send_it8`, and `exit_app`
- append-only `notes.db` record format

For those unchanged regions, the canonical byte-level explanation is still
[`../poc-04/note-edit.md`](../poc-04/note-edit.md).

One intentional X11 template change was also made in-place: the window's event
mask now requests `ButtonPress` in addition to the earlier `KeyPress` and
`Expose` delivery, so the product's click-to-load path can actually receive
mouse events from the X server.

## On-disk note format

Like the earlier note tools, the database is:

```text
[4-byte little-endian length][length bytes of text]
```

The current Linux product build remains append-only:

- clicking a note loads it into the editor pane
- editing changes the left-pane text buffer
- pressing `Enter` stores the current left-pane contents as a new record

## Fixed string patches

Three anchored strings were changed in-place:

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

## New code region: `0x539..0x6ca`

The old binary had a long zero padding region between the code and fixed data
base. `notes-linux-x86_64` uses that space for all new behavior without moving
the data section.

### Region overview

| Range | Purpose |
| --- | --- |
| `0x539..0x548` | startup stub |
| `0x549..0x61f` | two-pane draw routine |
| `0x620..0x645` | event dispatcher |
| `0x646..0x6ca` | click hit-testing and note load routine |

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
8. jumps back to the existing event loop

Important absolute stores in this routine patch the `x` field at
`0x4007e0` before calling the existing `send_it8` helper.

The key immediate X coordinates are:

- `0x0010` (`16`) for the left editor pane
- `0x0168` (`360`) for the right list pane

The copied Y coordinates are:

- `30` for pane headers
- `46` for the editor text line and the first list row
- then `+16` per subsequent list row

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
2. rejects clicks above the first list row at `y = 46`
3. walks the note rows in 16-pixel bands
4. stores the selected row index in `0x404804`
5. copies the chosen 64-byte padded note slot into `INPUT_BUF`
6. trims trailing spaces by walking backward from `0x40413f`
7. writes the resulting length to `INPUT_LEN`
8. jumps to `redraw`

That makes mouse selection load the full note into the left editor pane.

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
- same append-only note file semantics
- right pane shows only the first word, but sorting still follows the full
  stored note bytes
- the two panes are logical drawing regions, not toolkit-managed widgets

Those limitations are acceptable for the Linux x86_64 product reference build
because the goal here is to establish the cross-platform behavior contract in a
real native GUI binary.
