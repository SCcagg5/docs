---
description: Run queries, DDL, and multi-statement scripts on ClickHouse over its HTTP interface with the built-in clickhouse.query action, using server-side parameters and settings.
---

# ClickHouse

Run SQL on ClickHouse with the built-in `clickhouse.query` action. The action talks to the ClickHouse HTTP interface, so no ClickHouse client or driver is needed on the host.

## Basic Usage

```yaml
secrets:
  - name: CLICKHOUSE_PASSWORD
    provider: env
    key: CLICKHOUSE_PASSWORD

steps:
  - id: rollup
    action: clickhouse.query
    with:
      url: https://clickhouse.example.com:8443
      user: etl
      password: ${env.CLICKHOUSE_PASSWORD}
      database: analytics
      query: |
        INSERT INTO daily SELECT * FROM raw WHERE day = {day:Date};
        SELECT count() AS rows FROM daily WHERE day = {day:Date}
      params:
        day: "2024-01-02"
      settings:
        max_execution_time: 300
```

Results are written to **stdout** as they arrive. Use `output: VAR_NAME` to capture a small result for later steps.

## Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `url` | string | required | `http://` or `https://` endpoint, such as `http://localhost:8123`. It may have a path but no credentials, query string, or fragment. |
| `user` | string | - | User name, sent in the `X-ClickHouse-User` header |
| `password` | string | - | Password, sent in the `X-ClickHouse-Key` header and masked as `***` in errors |
| `database` | string | - | Default database |
| `query` | string | - | Inline SQL. Set `query` or `file`, not both. |
| `file` | string | - | Path of a `.sql` file, relative to the step working directory |
| `params` | map | - | Server-side query parameters |
| `settings` | map | - | ClickHouse settings applied to every statement |
| `output_format` | string | `jsonl` | `jsonl`, `json`, or `csv` |
| `headers` | bool | false | Add a header row to `csv` output |
| `timeout` | int | 60 | Step timeout in seconds; 0 means no limit |
| `tls_ca` | string | - | PEM bundle of extra root CAs for an `https://` endpoint |
| `tls_skip_verify` | bool | false | Skip TLS certificate verification |

Credentials never go in the URL. `dagu validate` rejects a URL with a user or password in it. The password travels in a request header, so use `https://` unless the server is on a trusted network.

`tls_ca` is read when the step starts, not by `dagu validate`. Redirects are not followed.

## SQL From a File

```yaml
steps:
  - id: rollup
    action: clickhouse.query
    with:
      url: http://localhost:8123
      file: sql/rollup.sql
```

## Multiple Statements

`query` or `file` may hold several statements separated by semicolons. They run in order in one ClickHouse session, so `SET`, `USE`, and temporary tables carry over to the statements that follow. The step stops at the first failing statement.

```yaml
steps:
  - id: stage
    action: clickhouse.query
    with:
      url: http://localhost:8123
      database: analytics
      query: |
        SET max_threads = 4;
        CREATE TEMPORARY TABLE ids AS SELECT id FROM events WHERE day = today();
        SELECT count() AS n FROM ids;
```

- A semicolon inside a string, a quoted identifier, a heredoc, or a comment, including nested `/* */` comments, does not end a statement.
- `database` is selected once with `USE` at the start of the session, so a `USE` in the script changes the database for later statements.
- When a setting is also listed in `settings`, the `settings` value wins over a `SET` in the script.
- A single statement runs without a session.
- A cluster behind a load balancer needs session stickiness for multi-statement scripts, because every statement must reach the same replica.

## Query Parameters

Bind values with ClickHouse `{name:Type}` placeholders and `params`. Values are sent as server-side parameters and are never interpolated into the SQL.

```yaml
steps:
  - id: orders
    action: clickhouse.query
    with:
      url: http://localhost:8123
      query: |
        SELECT id, amount
        FROM orders
        WHERE status = {status:String}
          AND region IN {regions:Array(String)}
          AND amount >= {min_amount:UInt32}
      params:
        status: paid
        regions: [eu, us]
        min_amount: 100
```

A string is passed as written, a number or boolean as its literal, a list as an `Array` literal, and a map as a `Map` literal. The server may echo parameter values in its error messages.

## Settings

`settings` passes ClickHouse settings, such as `max_execution_time` or `readonly`, to every statement:

```yaml
steps:
  - id: report
    action: clickhouse.query
    with:
      url: http://localhost:8123
      query: SELECT * FROM daily ORDER BY day DESC LIMIT 7
      settings:
        readonly: 1
        max_execution_time: 30
```

The action manages a few settings itself, and `dagu validate` rejects them in `settings`: `query`, `database`, `user`, `password`, `default_format`, `query_id`, `session_id`, `session_check`, `session_timeout`, `compress`, `decompress`, `http_write_exception_in_output_format`, and any name starting with `param_`.

### Read-Only Users

Steps are not read-only by default. Set `readonly: 1` in `settings` for a step that must not write.

A user whose ClickHouse profile sets `readonly = 1` cannot change any setting, including the ones the action manages:

| ClickHouse version | `jsonl`, `csv` | `json` |
|--------------------|----------------|--------|
| 25.11 and later | Works | Fails |
| Before 25.11 | Fails | Fails |

Give such users a profile with `readonly = 2` instead. It still blocks writes, and it works on all versions.

## Output Formats

`output_format` applies to statements without their own `FORMAT` clause. A `FORMAT` clause in the SQL wins.

| Value | ClickHouse format | Output |
|-------|-------------------|--------|
| `jsonl` | `JSONEachRow` | One JSON object per row |
| `json` | `JSONEachRow` as an array | A JSON array of row objects |
| `csv` | `CSV`, or `CSVWithNames` with `headers: true` | CSV rows |

```yaml
steps:
  - id: export
    action: clickhouse.query
    with:
      url: http://localhost:8123
      query: SELECT day, rows FROM daily ORDER BY day
      output_format: csv
      headers: true
```

## Statement Log

Each statement writes one JSON line to stderr:

```json
{"statement":1,"query_id":"...","started_at":"...","finished_at":"...","duration_ms":12,"bytes_returned":48,"status":"success"}
```

`statement` is the statement number; 0 is the `USE` that selects `database`. A failed statement also has an `error` field.

## Errors and Cancellation

- A non-200 response fails the step with the HTTP status and the server's error text. In a script, the error names the failing statement.
- An error the server reports after it started streaming results also fails the step. When the server marks the report with the `X-ClickHouse-Exception-Tag` header, the report is removed from stdout and becomes the step error.
- A response that ends early fails the step as interrupted.
- When the step times out, is stopped, or is cancelled, the action closes the request and asks the server to stop the statement with `KILL QUERY`. That request is best effort; if it fails, the step still fails with the original cause.

Retries use the normal step fields:

```yaml
steps:
  - id: rollup
    action: clickhouse.query
    with:
      url: http://localhost:8123
      file: sql/rollup.sql
      timeout: 600
    retry_policy:
      limit: 3
      interval_sec: 30
```

## Limitations

- Only the HTTP interface is supported, not the native TCP protocol.
- The action does not import data. Inline data after `INSERT ... FORMAT` is not supported; use `INSERT ... VALUES` or `INSERT ... SELECT`.
- The action does not close a session; the server expires it after its session timeout.

## See Also

- [ETL & SQL Overview](/step-types/sql/)
- [PostgreSQL](/step-types/sql/postgresql)
- [Secrets](/writing-workflows/secrets)
