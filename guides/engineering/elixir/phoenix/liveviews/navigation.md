# Navigation

## Rule

If a click takes the user to another page, render a real link.

Links are for destinations. Buttons are for actions.

## Use

- `<.link navigate={...}>` for navigation to another LiveView
- `<.link patch={...}>` for state changes within the same LiveView
- `<.link href={...}>` for external URLs, downloads, or full reloads
- `<.link patch={...} replace>` or `<.link navigate={...} replace>` when the URL should change without adding another back-button step

If something should look like a button but still navigates, style the link like a button.

## Avoid

- `<button phx-click={JS.navigate(...)}>`
- row-click-only navigation with no real anchor in the row
- controls that mix mutation and navigation in one click target

## Why

Real links preserve:

- open in new tab
- copy link
- sensible back-button behavior
- browser expectations
- semantic HTML

History behavior is part of the navigation contract.

Do not add history entries for tiny automatic URL corrections or incidental state normalization. Do add them when the user is making a meaningful navigation step they may reasonably expect to revisit with the back button.

## Review Questions

- Can the user open the destination in a new tab?
- Is the primary navigation target a real link?
- Does this navigation create or replace browser history intentionally?
- Are buttons reserved for actions such as save, delete, retry, or toggle?
