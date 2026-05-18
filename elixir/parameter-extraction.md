# Parameter Extraction

## Rule

Do not destructure function parameters just to pull values out of them.

Prefer ordinary parameters in the function head and extract what you need inside the function body.

Use pattern matching in function heads only when it is doing real dispatch work, such as:

- selecting between distinct clauses
- constraining the accepted input shape for a clause
- expressing a meaningful base case or special case
- checking that a required data shape is present for that clause

## Why

Destructuring parameters to extract fields makes function heads do hidden work.

That usually makes the code harder to scan because:

- required keys are mixed into clause selection
- ordinary data extraction looks like control flow
- function heads become noisy and brittle
- adding one more extracted field means rewriting the clause shape

Keep the function head about clause selection. Keep data extraction in the body.

It is fine for the function head to assert that a key or nested shape exists when that is how the clause decides whether it applies. The rule is about extraction, not about structural checks.

If you need distinct error handling for multiple required fields, prefer explicit extraction in the body over head matches that force everything into one fallback clause.

## Prefer

```elixir
def perform(%Oban.Job{args: %{"invoice_id" => _, "tenant_id" => _}} = job) do
  %{"invoice_id" => invoice_id, "tenant_id" => tenant_id} = job.args

  case InvoiceSync.run(tenant_id, invoice_id) do
    :ok ->
      :ok

    {:error, :rate_limited} ->
      {:error, :rate_limited}

    {:error, :invoice_missing} ->
      {:cancel, :invoice_missing}
  end
end

def perform(%Oban.Job{}), do: {:discard, :missing_required_args}
```

```elixir
def create_user(attrs) do
  with {:ok, email} <- Map.fetch(attrs, :email) do
    role = Map.get(attrs, :role, :member)

    ...
  end
end
```

## Avoid

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

## Good Uses Of Function-Head Pattern Matching

Use function-head pattern matching when the clause itself is different because the input shape is different.

Prefer:

```elixir
def fetch_user(nil), do: {:error, :missing_user_id}
def fetch_user(user_id), do: Accounts.get_user(user_id)
```

```elixir
def enqueue([]), do: :ok
def enqueue(job_ids), do: enqueue_all(job_ids)
```

```elixir
def handle_result({:ok, value}), do: {:ok, value}
def handle_result({:error, reason}), do: {:error, reason}
```

```elixir
def perform(%Oban.Job{args: %{"invoice_id" => _, "tenant_id" => _}} = job) do
  %{"invoice_id" => invoice_id, "tenant_id" => tenant_id} = job.args
  ...
end

def perform(%Oban.Job{}), do: {:discard, :missing_required_args}
```

In each of these, the pattern match is controlling which clause runs. That is the right use.

## Map And Struct Access

Inside the function body:

- use `Map.fetch/2` for required dynamic keys
- use `Map.get/3` for optional dynamic keys with defaults
- use `struct.field` for known struct fields
- use `Map.fetch!/2` when the function head has already established the key is present

Prefer:

```elixir
def publish(attrs) do
  with {:ok, topic} <- Map.fetch(attrs, :topic) do
    retries = Map.get(attrs, :retries, 3)

    ...
  end
end
```

Avoid:

```elixir
def publish(%{topic: topic, retries: retries}) do
  ...
end
```

## Review Questions

- Is the function head doing clause selection, or is it just unpacking data?
- Would the function read more clearly with ordinary parameters and explicit extraction in the body?
- Are required dynamic keys fetched explicitly with `Map.fetch/2`?
- If multiple required args exist, can they fail independently with distinct reasons?
- Is pattern matching being used because the clause is genuinely different?
