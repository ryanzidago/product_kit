# Elixir Style Guide

This guide defines the default Elixir conventions for this repository. Prefer consistency over personal preference.

These rules apply to new code and newly touched lines. Do not churn unrelated existing code just to satisfy this guide.

## Public Functions

- Public functions that define a real module boundary or reusable API should have `@spec` typespecs.
- Default to `@spec` for contexts, domain modules, integration boundaries, enqueue helpers, and other cross-module entry points.
- Phoenix web-layer callbacks do not need `@spec` by default.
- Page-local web-layer helpers that sit next to callbacks do not need `@spec` by default unless their contract is non-obvious, reused across modules, or meaningfully clarified by a typespec.
- In typespecs, prefer `list(type)` or `list()` over `[...]`.
- In typespecs, use the UUID type that matches your application's primary key module (e.g. `Ecto.UUID.t()`, `UUIDv7.t()`).
- If a public function has a `@spec` and returns `{:ok, value}`, the `@spec` should also declare an explicit error variant.

Prefer:

```elixir
@spec list_users(list(binary())) :: {:ok, list(User.t())} | {:error, term()}
def list_users(user_ids) do
  ...
end
```

Prefer:

```elixir
@impl Phoenix.LiveView
def handle_event("save", params, socket) do
  ...
end
```

Avoid:

```elixir
@impl Phoenix.LiveView
@spec handle_event(String.t(), map(), Phoenix.LiveView.Socket.t()) ::
        {:noreply, Phoenix.LiveView.Socket.t()}
def handle_event("save", params, socket) do
  ...
end
```

Prefer:

```elixir
@spec get_user(Ecto.UUID.t()) :: {:ok, User.t()} | {:error, :not_found}
# or, if the application uses UUIDv7:
@spec get_user(UUIDv7.t()) :: {:ok, User.t()} | {:error, :not_found}
```

Avoid:

```elixir
@spec get_user(binary()) :: {:ok, User.t()} | {:error, :not_found}
```

Prefer:

```elixir
@spec create_user(map()) :: {:ok, User.t()} | {:error, Ecto.Changeset.t()}
```

Avoid:

```elixir
@spec create_user(map()) :: {:ok, User.t()}
```

## Naming

- Predicate functions that return booleans should end in `?`.
- Prefer plural names for collections and singular names for elements.

Prefer:

```elixir
def active?(user), do: user.active

Enum.map(users, fn user -> user.email end)
```

Avoid:

```elixir
def active(user), do: user.active

Enum.map(user_list, fn users -> users.email end)
```

## Function Clauses

- Prefer a single public function head when a function has default arguments.
- Keep multi-clause functions grouped together.
- Order clauses from most specific to least specific.
- Keep clause shapes aligned so the dispatch logic is easy to scan.

Prefer:

```elixir
def fetch_user(user_id, opts \\ [])

def fetch_user(nil, _opts), do: {:error, :missing_user_id}

def fetch_user(user_id, opts) do
  ...
end
```

Avoid:

```elixir
def fetch_user(nil, _opts), do: {:error, :missing_user_id}

def fetch_user(user_id, opts \\ []) do
  ...
end
```

## Aliases

- Do not introduce `alias` statements inside functions.
- Define aliases at module scope.
- Avoid grouped multi-line aliases; prefer one `alias` per line.
- Avoid `defdelegate`; call the target module directly.

Prefer:

```elixir
alias MyApp.Accounts
alias MyApp.Users

def load_user(user_id) do
  Accounts.get_user(user_id)
end
```

Avoid:

```elixir
alias MyApp.{
  Accounts,
  Users
}
```

Avoid:

```elixir
def load_user(user_id) do
  alias MyApp.Accounts
  Accounts.get_user(user_id)
end
```

Avoid:

```elixir
defdelegate load_user(user_id), to: MyApp.Accounts
```

## Pipelines

- Use `|>` for linear data flow through data-first function calls.
- Prefer ordinary function calls when there is only one step.
- Bind intermediate values when you need to branch, pattern match, log, or reuse the result.
- Keep pipelines at one abstraction level.
- Do not pipe directly into `case`, `cond`, or `with`.
- Do not use `then/1` or anonymous functions to smuggle control flow back into a pipeline.
- When control flow starts, bind the intermediate value first, then branch explicitly.
- Avoid side-effect-heavy pipelines; make side effects explicit near the boundary.
- Prefer `Enum.count(enum, fun)` over `Enum.reject(enum, fun) |> Enum.count()`.

Prefer:

```elixir
trimmed_email = String.trim(email)
```

Avoid:

```elixir
trimmed_email =
  email
  |> String.trim()
```

Prefer:

```elixir
normalized_email =
  email
  |> String.trim()
  |> String.downcase()

case normalized_email do
  "" -> {:error, :blank}
  normalized_email -> {:ok, normalized_email}
end
```

Avoid:

```elixir
email
|> String.trim()
|> String.downcase()
|> case do
  "" -> {:error, :blank}
  normalized_email -> {:ok, normalized_email}
end
```

Avoid:

```elixir
email
|> String.trim()
|> String.downcase()
|> then(fn normalized_email ->
  cond do
    normalized_email == "" -> {:error, :blank}
    String.ends_with?(normalized_email, "@example.com") -> {:ok, :internal}
    true -> {:ok, :external}
  end
end)
```

Avoid:

```elixir
user_id
|> with {:ok, user} <- Accounts.fetch_user(user_id) do
  {:ok, user}
end
```

Prefer:

```elixir
normalized_email =
  email
  |> String.trim()
  |> String.downcase()

Logger.info("normalized email")

Accounts.find_user_by_email(normalized_email)
```

Avoid:

```elixir
email
|> String.trim()
|> String.downcase()
|> tap(fn _normalized_email -> Logger.info("normalized email") end)
|> Accounts.find_user_by_email()
```

Prefer:

```elixir
inactive_user_count = Enum.count(users, &inactive?/1)
```

Avoid:

```elixir
users
|> Enum.reject(&active?/1)
|> Enum.count()
```

## Ecto Queries

- Build Ecto queries with `|>` pipelines.
- Always name the root binding with `from(x in X, as: :x)`.
- Prefer named bindings in downstream query clauses.

Prefer:

```elixir
from(user in User, as: :user)
|> where([user: user], user.active)
|> order_by([user: user], asc: user.inserted_at)
```

Avoid:

```elixir
from(u in User,
  where: u.active,
  order_by: [asc: u.inserted_at]
)
```

## Pattern Matching

- Use pattern matching for actual pattern matching and control flow.
- Do not destructure function parameters only to unpack values.
- Use function-head pattern matching only when it changes clause selection.
- Function-head shape checks are fine when they are part of clause selection.
- Prefer extracting required map values in the function body with `Map.fetch/2`.
- When distinct missing-field errors matter, extract explicitly in the body rather than forcing them through one fallback clause.
- See `guides/elixir/parameter-extraction.md` for the full rule.

Prefer:

```elixir
def perform(%Oban.Job{} = job) do
  with {:ok, invoice_id} <- fetch_required_arg(job.args, "invoice_id", :missing_invoice_id),
       {:ok, tenant_id} <- fetch_required_arg(job.args, "tenant_id", :missing_tenant_id) do
    ...
  end
end
```

```elixir
def create_user(attrs) do
  with {:ok, email} <- Map.fetch(attrs, :email) do
    role = Map.get(attrs, :role, :member)

    ...
  end
end
```

Avoid:

```elixir
def perform(%Oban.Job{args: %{"invoice_id" => invoice_id}}) do
  ...
end
```

```elixir
def create_user(%{email: email, role: role} = attrs) do
  ...
end
```

## Map Access

- Prefer `struct.field` for known struct or map fields.
- Prefer `Map.fetch/2` for required dynamic keys so missing-key handling stays explicit.
- Prefer `Map.get/3` for optional keys with defaults.
- Avoid `map[:key]` when the key is required or the distinction between missing and `nil` matters.

Prefer:

```elixir
email = user.email

with {:ok, source} <- Map.fetch(attrs, :source) do
  limit = Map.get(attrs, :limit, 50)
  ...
end
```

Avoid:

```elixir
email = user[:email]
source = Map.fetch!(attrs, :source)
limit = attrs[:limit] || 50
```

## Conditionals

- Prefer `if` over `unless`.
- Prefer `and`, `or`, and `not` for strict boolean logic so non-boolean values fail fast.
- Use `&&`, `||`, and `!` only when you intentionally want weaker truthy/falsy semantics or value propagation.
- Prefer `Enum.count/1` over `length/1` when you actually need a count.
- Do not use `length(list) > 0` or `Enum.count(enum) > 0` to check whether a collection has items.
- Prefer pattern matching for list presence checks.
- Use `Enum.any?/1` or `Enum.empty?/1` where that reads more clearly.
- Prefer boolean expressions directly over comparing to `true` or `false`.
- Use `is_nil/1` when you specifically mean `nil`, rather than generic falsiness.
- Avoid using `nil` as an overloaded public return state when an explicit atom or tagged tuple is clearer.

Why:

- `and`, `or`, and `not` make boolean intent obvious in conditionals.
- They surface mistakes earlier when a value that should have been `true` or `false` is actually `nil`, a string, a map, or some other term.
- `&&`, `||`, and `!` are better when you are deliberately relying on Elixir's truthy/falsy rules, such as fallback selection, optional values, or propagating a non-boolean value.

Prefer:

```elixir
if user.active do
  ...
end
```

Avoid:

```elixir
unless user.active do
  ...
end
```

Prefer:

```elixir
if feature_enabled?(account) and user.admin? do
  ...
end
```

Avoid:

```elixir
if feature_enabled?(account) && user.admin? do
  ...
end
```

Prefer:

```elixir
display_name = user.nickname || user.email
```

`||` is appropriate here because the goal is to pick the first truthy value, not to enforce strict boolean inputs.

Prefer:

```elixir
if user.active do
  ...
end
```

Avoid:

```elixir
if user.active == true do
  ...
end
```

Prefer:

```elixir
item_count = Enum.count(items)
```

Avoid:

```elixir
item_count = length(items)
```

Prefer:

```elixir
defp has_items?([_ | _]), do: true
defp has_items?(_), do: false
```

Or:

```elixir
if Enum.empty?(external_emails_parsed) do
  ...
end
```

Or:

```elixir
if Enum.any?(external_emails_parsed) do
  ...
end
```

Avoid:

```elixir
if length(external_emails_parsed) > 0 do
  ...
end
```

Avoid:

```elixir
if Enum.count(external_emails_parsed) > 0 do
  ...
end
```

- Prefer `if` over a 2-clause `case` when you are only checking whether a value is `nil`.

Prefer:

```elixir
if is_nil(user_id) do
  ...
else
  ...
end
```

Avoid:

```elixir
if !user_id do
  ...
else
  ...
end
```

Avoid:

```elixir
case user_id do
  nil -> ...
  _user_id -> ...
end
```

Prefer:

```elixir
{:error, :not_found}
```

Avoid:

```elixir
nil
```

## Struct and Map Construction

- Avoid building a map or struct directly inside a tuple return.
- Build the value first, then wrap it.

Prefer:

```elixir
record = %User{name: name, email: email}
{:ok, record}
```

Avoid:

```elixir
{:ok, %User{name: name, email: email}}
```

## Atom Safety

- Never create atoms from untrusted input.
- Prefer explicit string-to-atom mapping at boundaries.
- Use `String.to_existing_atom/1` only when the allowed atoms are already defined and a raising failure mode is acceptable.

Prefer:

```elixir
case status do
  "draft" -> {:ok, :draft}
  "published" -> {:ok, :published}
  _ -> {:error, :invalid_status}
end
```

Avoid:

```elixir
{:ok, String.to_atom(status)}
```

## Control Flow

- Prefer `with` over nested `case` statements when chaining fallible operations.
- `with` statements should have an explicit `else` clause when the error handling is local to that function.
- Do not use `{:ok, value} = ...` to unpack fallible calls.
- Prefer explicit control flow that propagates errors without crashing.

Prefer:

```elixir
with {:ok, user} <- Accounts.fetch_user(user_id),
     {:ok, team} <- Teams.fetch_team(team_id) do
  {:ok, {user, team}}
else
  {:error, reason} -> {:error, reason}
end
```

Avoid:

```elixir
case Accounts.fetch_user(user_id) do
  {:ok, user} ->
    case Teams.fetch_team(team_id) do
      {:ok, team} -> {:ok, {user, team}}
      {:error, reason} -> {:error, reason}
    end

  {:error, reason} ->
    {:error, reason}
end
```

Avoid:

```elixir
{:ok, user} = Accounts.fetch_user(user_id)
```

## Time Modeling

- When designing schemas, APIs, or domain models, prefer datetimes over plain dates.
- Prefer timezone-aware datetimes because they preserve the actual moment in time and are easier to reason about across users and systems.
- Use plain `Date` only when the domain is truly date-only.
- For datetime comparisons in Elixir code, prefer `DateTime.compare/2`, `NaiveDateTime.compare/2`, `Date.compare/2`, `before?/2`, or `after?/2` over raw comparison operators.
- Avoid `23:59:59` end-of-day bounds. Prefer the next day at midnight as an exclusive upper bound.

Prefer:

```elixir
field :scheduled_at, :utc_datetime_usec
```

Avoid:

```elixir
field :scheduled_date, :date
```

Good `Date` examples:

```elixir
field :date_of_birth, :date
field :bank_holiday_on, :date
```

Prefer:

```elixir
DateTime.compare(starts_at, ends_at) == :lt
```

Avoid:

```elixir
starts_at < ends_at
```

Prefer:

```elixir
shift_end = ~U[2026-03-21 00:00:00Z]
```

Avoid:

```elixir
shift_end = ~T[23:59:59]
```

## LiveView Return Shape

- Use one `{:noreply, socket}` return per function clause.
- Do all intermediate assignments first, then return `{:noreply, socket}` once at the end of the clause.
- LiveView callback implementations should use `@impl Phoenix.LiveView`.
- Extract `assign(...)` work before the return tuple.
- Only callback functions should return `{:noreply, ...}` or `{:reply, ...}` tuples.
- Helper functions should return the socket or value directly, and let the callback wrap it.

Prefer:

```elixir
@impl Phoenix.LiveView
def handle_event("save", params, socket) do
  socket =
    socket
    |> assign(:form_params, params)
    |> assign(:saved?, true)

  {:noreply, socket}
end
```

Avoid:

```elixir
def handle_event("save", params, socket) do
  if valid?(params) do
    {:noreply, assign(socket, :saved?, true)}
  else
    {:noreply, assign(socket, :error, :invalid_params)}
  end
end
```

Avoid:

```elixir
@impl true
def handle_event("save", params, socket) do
  ...
end
```

Avoid:

```elixir
def handle_event("save", params, socket) do
  {:noreply, assign(socket, :form_params, params)}
end
```

Prefer:

```elixir
def handle_event("save", params, socket) do
  socket = update_form_params(socket, params)
  {:noreply, socket}
end

defp update_form_params(socket, params) do
  assign(socket, :form_params, params)
end
```

Avoid:

```elixir
defp update_form_params(socket, params) do
  {:noreply, assign(socket, :form_params, params)}
end
```

## Error Handling

- Always propagate errors.
- Never hide errors, silently fail, or replace a real failure with a misleading success value.
- Prefer explicit `{:error, reason}` returns.
- Avoid bang functions in user-facing and other recoverable paths.
- Bang functions are acceptable in tests, one-off scripts, setup code, and invariant-enforcing internal paths where crashing is the intended behavior.
- Avoid raising.
- Avoid `try` / `rescue`.
- When matching changeset errors, prefer `{:error, %Ecto.Changeset{} = changeset}` over a bare `{:error, changeset}` pattern.

Prefer:

```elixir
with {:ok, user} <- Accounts.fetch_user(user_id),
     {:ok, updated_user} <- Accounts.update_user(user, attrs) do
  {:ok, updated_user}
end
```

Avoid:

```elixir
def update_user(user_id, attrs) do
  case Accounts.fetch_user(user_id) do
    {:ok, user} ->
      Accounts.update_user(user, attrs)

    {:error, _reason} ->
      {:ok, nil}
  end
end
```

Also avoid:

```elixir
def show(socket, id) do
  record = Repo.one!(from(record in Record, where: record.id == ^id))
  {:ok, assign(socket, :record, record)}
end
```

Prefer:

```elixir
def show(socket, id) do
  case Repo.one(from(record in Record, where: record.id == ^id)) do
    nil -> {:error, :not_found}
    record -> {:ok, assign(socket, :record, record)}
  end
end
```

Prefer:

```elixir
case Accounts.create_user(attrs) do
  {:ok, user} ->
    {:ok, user}

  {:error, %Ecto.Changeset{} = changeset} ->
    {:error, changeset}
end
```

Avoid:

```elixir
case Accounts.create_user(attrs) do
  {:ok, user} ->
    {:ok, user}

  {:error, changeset} ->
    {:error, changeset}
end
```

## Queries And Existence Checks

- Prefer `Repo.exists?/1` for existence checks.
- Do not count rows just to determine whether at least one row exists.

Prefer:

```elixir
Repo.exists?(from(user in User, as: :user, where: user.active))
```

Avoid:

```elixir
Repo.aggregate(from(user in User, as: :user, where: user.active), :count) > 0
```

## Comments

- Use comments to explain invariants, intent, or non-obvious tradeoffs.
- Do not comment obvious mechanics that the code already states clearly.

Prefer:

```elixir
# Keep this order stable because downstream exports depend on it.
fields = [:id, :email, :inserted_at]
```

Avoid:

```elixir
# Assign the fields list.
fields = [:id, :email, :inserted_at]
```

## Tests

- Tests should be self-isolated and readable as standalone units.
- Prefer self-contained tests over heavy shared setup.
- Avoid shared `setup` blocks when the data can be built inline inside each test.
- Prefer explicit test-local fixtures over hidden dependencies through context.
