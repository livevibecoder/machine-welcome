# Mobile Product Plan

This file covers the mobile note product targets:

- Android ARM64
- iOS ARM64

## Goal

Keep the same core note-taking behavior as the desktop reference build while
allowing a mobile-appropriate presentation:

- editable note pane
- note list derived from first words
- tap to load note
- save on the platform's equivalent primary action, with `Enter` behavior where
  physical or software keyboards make that natural

## Android ARM64

Starting point:

- packaging/runtime proof: [`../poc-10/hello.apk.md`](../poc-10/hello.apk.md)

Required implementation work:

1. replace the current no-op `ANativeActivity_onCreate`
2. create a real native render/input path
3. provide list selection and note editing
4. store notes in app-private storage using the same record framing
5. preserve the same add/load/edit behavior as the Linux reference build

## iOS ARM64

Starting point:

- structural Mach-O precedent: [`../poc-11/hello-ios.md`](../poc-11/hello-ios.md)

Required implementation work:

1. real UIKit-based runtime path
2. touch selection in the note list
3. note editing surface
4. same note storage format
5. Apple-host validation and signing during execution

## Mobile layout policy

If the device cannot comfortably show two fixed panes side-by-side, the product
should still preserve the same operations using the best native mobile layout
available:

- list visible
- editor visible
- tap-to-load
- edit
- save

The behavior matters more than a strict desktop geometry clone.
