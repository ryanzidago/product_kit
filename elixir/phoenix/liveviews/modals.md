# Modals

## Rule

Choose patch-driven modals when the modal meaningfully changes page state. Use local boolean state only for ephemeral UI.

## Prefer Patch-Driven Modals When

- the modal has its own meaningful URL
- refresh should reopen it
- back and forward should close or reopen it
- the modal represents a real workflow step
- the modal needs deep-linking or sharing

Examples:

- edit record
- new item flow
- inspect details

## Prefer Local Boolean State When

- the modal is short-lived and purely local
- it is a small confirm dialog
- it is not meaningful to share or restore on refresh

Examples:

- delete confirmation
- lightweight picker
- local help panel

## Avoid

- patching for tiny ephemeral dialogs that do not matter outside the moment
- boolean-only state for modals that are really route-level workflows

## Review Questions

- Should refresh restore this modal?
- Should back and forward control it?
- Is this a workflow state or just a local UI convenience?
