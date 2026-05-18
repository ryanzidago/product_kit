# Authorization

## Rule

Authorization is a real boundary, not just a UI concern.

LiveViews may hide or disable controls, but the actual permission enforcement must survive direct events, crafted params, and alternate entrypoints.

## Enforce At The Right Layers

- contexts enforce the real rule
- LiveViews enforce page access and workflow gating
- components and templates reflect permissions in the UI

All three matter, but only the context boundary is final.

## Check Early

- gate page access on mount or params handling
- reject unauthorized writes in the event path
- scope reads correctly before rendering data

## Avoid

- relying on hidden buttons as protection
- assuming a user cannot trigger an event because the control is absent
- burying permission rules only inside templates

## Review Questions

- What happens if an unauthorized user hits the route directly?
- What happens if an unauthorized event is fired manually?
- Is the context still safe if the LiveView UI is bypassed?
