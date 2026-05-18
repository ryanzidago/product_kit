# Copy-Pasteable IEx Examples

Use these rules when writing IEx examples in docs, specs, PR descriptions, and agent output.

They are also the default format for ad hoc smoke checks when you want to paste realistic inputs into IEx and verify that a business-logic or algorithm change behaves plausibly.

The default goal is simple: a reader should be able to copy one block, paste it into a fresh IEx prompt, and have it run successfully.

## Core Rules

- Prefer fenced `elixir` code blocks over raw `iex>` transcripts when the example is meant to be runnable.
- Make the example a single valid IEx expression.
- If the example includes setup statements plus a pipeline, wrap the whole thing in parentheses.
- Keep all required `alias`, `import`, `require`, and setup code inside the same block unless the surrounding text explicitly says those steps were already run.
- Avoid grouped multi-line aliases; prefer one `alias` per line in examples.
- End the block with the value the reader should inspect.
- Do not include transcript noise such as `iex(12)>`, lists of aliased modules, or stray `nil` outputs inside the runnable example.

## Why Parentheses Matter

IEx does not allow a pipeline to start on the next line immediately after a match expression unless the surrounding expression is parenthesized.

This fails in IEx:

```elixir
query =
  %QueryModel{}
  |> QueryModel.changeset(%{source: %{name: "orders", type: :table}})
```

Use this instead:

```elixir
(
  query =
    %QueryModel{}
    |> QueryModel.changeset(%{source: %{name: "orders", type: :table}})

  query
)
```

Wrapping the full example in parentheses also removes noisy intermediate outputs from `alias` and assignment lines, so the reader sees one final result instead of a transcript full of `nil`.

## Output Convention

If the example should be self-sufficient when read in a markdown file, show expected output as comments instead of transcript lines.

- Use `#=>` for the main returned value.
- Abbreviate long output when needed, but keep the important shape or key fields.
- If exact output is unstable, comment the stable parts only.

Prefer:

```elixir
(
  value = String.upcase("hello")
  value
)
#=> "HELLO"
```

Avoid:

```elixir
iex(1)> value = String.upcase("hello")
"HELLO"
iex(2)> value
"HELLO"
```

## Recommended Pattern

For anything beyond a one-liner, prefer this structure:

```elixir
(
  alias MyApp.SomeContext

  input = ...

  result =
    input
    |> SomeContext.run(...)
    |> SomeOtherModule.transform()

  result
)
#=> ...
```

This keeps the example copy-pasteable while still reading top-to-bottom.

## Concrete Example

Prefer a single parenthesized block with commented output:

```elixir
(
  alias AgenticOs.Dashboards.QueryBuilders.ClickhouseCompiler
  alias AgenticOs.Dashboards.QueryBuilders.ColumnMeta
  alias AgenticOs.Dashboards.QueryBuilders.FieldClassifier
  alias AgenticOs.Dashboards.QueryBuilders.QueryModel
  alias AgenticOs.Dashboards.QueryBuilders.SchemaContext
  alias AgenticOs.Dashboards.QueryBuilders.TableMeta

  table = %TableMeta{
    database: "analytics",
    name: "orders",
    type: :table,
    engine: "MergeTree"
  }

  context = %SchemaContext{
    table_meta: table,
    field_metas: [
      FieldClassifier.classify(%ColumnMeta{name: "status", physical_type: "String"}, table),
      FieldClassifier.classify(%ColumnMeta{name: "revenue", physical_type: "Decimal(12,2)"}, table)
    ],
    default_timezone: "UTC"
  }

  query =
    %QueryModel{}
    |> QueryModel.changeset(%{
      source: %{name: "orders", type: :table},
      dimensions: [%{source_name: "orders", column: "status"}],
      measures: [%{source_name: "orders", column: "revenue", aggregation: :sum}]
    })
    |> Ecto.Changeset.apply_changes()

  {:ok, result} = ClickhouseCompiler.compile(query, context)
  result.sql
)
#=> "SELECT `status` AS `status`, sum(`revenue`) AS `sum_revenue` ..."
```

If the returned struct is the important part, make that the final expression instead:

```elixir
(
  ...
  {:ok, result} = ClickhouseCompiler.compile(query, context)
  result
)
#=> %AgenticOs.Dashboards.QueryBuilders.CompileResult{sql: ..., warnings: [...], field_map: ...}
```

## When A Transcript Is Acceptable

Use raw `iex>` transcripts only when the point is to show an interactive debugging session or a sequence of separate commands. If you do that, add a separate copy-pasteable block first when the example is also meant to be runnable by readers.
