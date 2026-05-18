# Architecture

## Rule

Separate database operations, business logic, and presentation.

The exact module structure can vary by feature, but the boundary should stay clear:

- database operations read and write data
- business logic applies domain rules and orchestrates workflows
- presentation turns results into UI or API output

When one module owns all three concerns, changes get coupled. Query changes start breaking pages, UI needs leak into contexts, and tests become broad and brittle.

## Dependency Direction

Prefer a one-way flow:

`presentation -> business logic -> database operations`

Avoid dependencies in the other direction:

- presentation should not call `Repo` directly for feature logic
- database helpers should not know about HTML, LiveView assigns, JSON payloads, or CSS
- business logic should not return presentation-specific strings, labels, or markup unless that is the actual domain concept

## Database Operations

This layer owns persistence concerns:

- queries
- preloads
- inserts, updates, deletes
- transactions
- mapping database constraints into application-level results

Keep this layer focused on data access and data integrity. It should not decide page states, flash messages, button labels, or API response shapes.

## Business Logic

This layer owns feature behaviour:

- invariants and policy checks
- multi-step workflows
- coordination across multiple writes or reads
- decisions that matter regardless of UI

This is usually the right home for code that would still matter if the feature moved from LiveView to an API, a job, or a CLI task.

## Presentation

This layer owns delivery to the caller:

- LiveViews, controllers, components, serializers, presenters
- params and event handling
- UI state, copy, formatting, and display decisions
- mapping domain results into assigns or API responses

Presentation should orchestrate the user interaction, not become the home of the feature's rules or queries.

## Practical Guidance

- It is fine for a context to call `Repo` directly. The important boundary is separating domain behaviour from presentation, not inventing extra layers for their own sake.
- If a query exists only to support one business operation, keeping it private inside the business module is fine at first.
- Split code once a module starts mixing concerns in a way that makes it harder to change, test, or reuse.
- Prefer passing domain values between layers. Avoid passing sockets, conn structs, or raw params deeper than necessary.

## Example

Prefer:

```elixir
defmodule MyApp.Approvals do
  alias MyApp.Approvals.Runs

  def approve_run(run_id, actor) do
    with {:ok, run} <- Runs.get(run_id),
         :ok <- ensure_can_approve(actor, run),
         {:ok, run} <- Runs.mark_approved(run) do
      {:ok, run}
    end
  end
end
```

```elixir
defmodule MyAppWeb.RunLive.Show do
  def handle_event("approve", _, socket) do
    case MyApp.Approvals.approve_run(socket.assigns.run.id, socket.assigns.current_user) do
      {:ok, run} -> {:noreply, assign(socket, :run, run)}
      {:error, reason} -> {:noreply, put_flash(socket, :error, message_for(reason))}
    end
  end
end
```

Avoid:

```elixir
def handle_event("approve", _, socket) do
  run = Repo.get!(Run, socket.assigns.run.id)

  if can_approve?(socket.assigns.current_user, run) do
    run
    |> Run.changeset(%{status: :approved})
    |> Repo.update()
  else
    {:error, "You cannot approve this run"}
  end
end
```

## Review Questions

- Is this code doing database work, domain decision-making, and presentation in the same place?
- Would this logic still matter if the UI changed completely?
- Is the presentation layer orchestrating, or is it quietly becoming the feature implementation?
- Are data-access details leaking into UI code?
- Are UI concerns leaking into contexts or data-access modules?
