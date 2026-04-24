# macOS Desktop Product Plan

This file isolates the macOS-specific desktop work for the cross-platform notes
product.

## Goal

Ship:

- `notes-macos`

as a real native macOS note editor with the same visible behavior as the Linux
reference build:

- left editor pane
- right first-word list
- click list item to load note
- save current note with the product's shared semantics

## Starting point

The current Apple artifact in the repo is only a structural Mach-O precedent:

- [`../poc-11/hello-macos.md`](../poc-11/hello-macos.md)

That file is useful for container shape only. It does not yet prove a runnable
AppKit product path.

## Required macOS work

1. real Mach-O desktop executable using native macOS GUI APIs
2. file I/O using the shared `notes.db` format
3. list selection and editor rendering
4. click handling and save behavior
5. Apple-host runtime validation
6. code-signing workflow suitable for local testing and eventual distribution

## Validation policy

Unlike the current Apple POCs, the product target is not considered complete on
container shape alone. It needs:

- a runnable binary on Apple-host tooling or hardware
- a matching test artifact
- detailed interface documentation for any linked Apple frameworks
