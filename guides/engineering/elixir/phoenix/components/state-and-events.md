# State And Events

## Rule

Components should express UI intent clearly and keep side effects in the right layer.

Do not let reusable UI silently become a second controller or context.

## Function Components

Function components should usually be:

- stateless
- deterministic from assigns
- free of persistence and business decisions

They are ideal for rendering and composition.

## LiveComponents

LiveComponents may own local UI state when that keeps the page simpler, but the state should stay genuinely local.

Good candidates:

- open or closed UI state
- local selection state
- focused interactive widgets
- event handling that benefits from `@myself`

Bad candidates:

- duplicated page state
- URL-backed workflow state
- business workflows that matter outside the component

## Events

Name events around user intent, not low-level DOM trivia.

Prefer event boundaries where:

- the component captures the local interaction
- the parent LiveView decides page consequences
- contexts perform writes and business logic

Do not hide database calls, authorization, or cross-feature side effects inside reusable components.

## Review Questions

- Is this state truly local to the component?
- Would the page become clearer if this state moved up?
- Is the event named around user intent?
- Are persistence and business rules still owned by contexts?
