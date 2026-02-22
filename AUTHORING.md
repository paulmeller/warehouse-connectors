# Connector Authoring Guide

This guide explains how to write a Warehouse connector spec — a single JSON file that tells Warehouse how to sync data from any REST or GraphQL API.

## Minimal Example

The simplest possible connector (no auth, single table, no FTS):

```json
{
  "version": 1,
  "name": "hackernews",
  "description": "Hacker News top stories via Algolia API.",
  "api_type": "rest",
  "tables": [
    {
      "name": "hackernews_stories",
      "columns": [
        { "name": "id", "type": "TEXT", "primary_key": true },
        { "name": "title", "type": "TEXT" },
        { "name": "url", "type": "TEXT" },
        { "name": "author", "type": "TEXT" },
        { "name": "points", "type": "INTEGER" },
        { "name": "_extracted_at", "type": "TIMESTAMP", "default": "CURRENT_TIMESTAMP" }
      ],
      "endpoint": {
        "url": "https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=100",
        "method": "GET"
      },
      "response": {
        "results_path": "hits",
        "field_mappings": [
          { "column": "id", "path": "objectID" },
          { "column": "title", "path": "title" },
          { "column": "url", "path": "url" },
          { "column": "author", "path": "author" },
          { "column": "points", "path": "points" }
        ]
      }
    }
  ]
}
```

## Full Spec Reference

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | integer | yes | Always `1` |
| `name` | string | yes | Unique identifier (lowercase, no spaces). Used as sync source name. |
| `description` | string | yes | Short description shown in `warehouse connector list` |
| `api_type` | string | yes | `"rest"` or `"graphql"` |
| `governance_fields` | string[] | no | Fields subject to governance redaction |
| `auth` | object | no | Authentication configuration |
| `client` | object | no | HTTP client configuration |
| `discover` | object[] | no | Pre-sync discovery steps (fetch user ID, tokens, etc.) |
| `tables` | object[] | yes | One or more tables to sync |
| `fts` | object[] | no | Full-text search index definitions |

### Auth Types

#### `config_key` — API key from config file

```json
{
  "type": "config_key",
  "key": "pocketsmith.api_key",
  "header_name": "X-Developer-Key"
}
```

The key is read from `~/.warehouse/config.toml`. For Bearer tokens, add `"header_prefix": "Bearer"`.

#### `env_var` — Environment variable

```json
{
  "type": "env",
  "var": "MY_API_TOKEN",
  "header_name": "Authorization",
  "header_prefix": "Bearer"
}
```

#### `browser_cookies` — Extract cookies from browser

```json
{
  "type": "browser_cookies",
  "domains": [".example.com", "example.com"],
  "cookies": ["session_token", "csrf_token"],
  "headers": {
    "Cookie": "session={{cookies.session_token}}",
    "X-CSRF-Token": "{{cookies.csrf_token}}"
  }
}
```

#### `token_chain` — Try multiple strategies in order

```json
{
  "type": "token_chain",
  "cache_file": "myservice_token",
  "strategies": [
    { "type": "env", "var": "MY_TOKEN" },
    { "type": "safari_localstorage", "origin_marker": "...", "localstorage_key": "...", "token_path": "..." }
  ],
  "header_name": "Authorization",
  "header_prefix": "Token",
  "validate_url": "https://api.example.com/me",
  "validate_query": "query { me { id } }"
}
```

### Tables

Each table maps to a SQLite table created during sync.

```json
{
  "name": "my_table",
  "soft_delete": true,
  "columns": [...],
  "endpoint": {...},
  "response": {...}
}
```

#### Columns

```json
{ "name": "id", "type": "TEXT", "primary_key": true }
{ "name": "title", "type": "TEXT" }
{ "name": "amount", "type": "REAL" }
{ "name": "count", "type": "INTEGER" }
{ "name": "_extracted_at", "type": "TIMESTAMP", "default": "CURRENT_TIMESTAMP" }
```

Always include an `_extracted_at` timestamp column. Always have a `primary_key` column.

#### Endpoint (REST)

```json
{
  "url": "https://api.example.com/v1/items",
  "method": "GET",
  "pagination": {
    "type": "page_number",
    "page_size": 100,
    "max_pages": 50
  },
  "rate_limit": { "delay_seconds": 0.5 }
}
```

#### Endpoint (GraphQL)

```json
{
  "url": "https://api.example.com/graphql",
  "query": "query GetItems { items { id name } }",
  "operation_name": "GetItems",
  "variables": { "limit": 100 }
}
```

#### Pagination Types

- **`page_number`** — Adds `?page=N` parameter. Fields: `page_size`, `max_pages`.
- **`offset`** — Uses `$offset` and `$limit` GraphQL variables. Fields: `page_size`, `max_pages`.
- **`cursor`** — Follows cursor tokens. Fields: `cursor_path`, `has_next_path`, `cursor_variable`, `max_pages`. For GraphQL timeline-style cursors: `cursor_from_results`, `cursor_entry_id_path`, `cursor_entry_prefix`, `cursor_value_path`.

#### Response Mapping

```json
{
  "results_path": "data.items",
  "field_mappings": [
    { "column": "id", "path": "id" },
    { "column": "name", "path": "nested.field.name" },
    { "column": "tags", "path": "tags", "transform": "join_array" },
    { "column": "is_active", "path": "active", "transform": "to_bool" },
    { "column": "count", "path": "count", "transform": "to_string" }
  ],
  "filter": { "path": "type", "starts_with": "item-" }
}
```

**Transforms:** `to_string`, `to_bool`, `join_array`

**Alt paths:** For fields that may appear at different locations:

```json
{
  "column": "title",
  "path": "properties.Name.title[0].plain_text",
  "alt_paths": ["properties.Title.title[0].plain_text"]
}
```

#### Soft Delete

When `"soft_delete": true`, Warehouse marks rows that disappear from full-result-set endpoints with `_deleted_at` instead of removing them.

#### Incremental Sync

```json
"incremental": {
  "stop_date_path": "last_edited_time"
}
```

Stops paginating when results are older than the last sync timestamp.

### Discovery Steps

Pre-sync steps to resolve dynamic values (user IDs, query IDs, etc.):

```json
"discover": [
  {
    "id": "user",
    "action": "fetch_json",
    "url": "https://api.example.com/me"
  }
]
```

Reference discovered values in URLs with `{{user.id}}`.

Available actions: `fetch_json`, `fetch`, `regex_all`, `fetch_regex_map`.

### FTS (Full-Text Search)

Add FTS indexes so the data appears in `warehouse search` results:

```json
{
  "table_name": "my_fts",
  "source_table": "my_table",
  "columns": ["title", "description"],
  "tokenizer": "porter unicode61",
  "id_column": "id",
  "map_id_column": "item_id",
  "search_type": "mytype",
  "title_column": "title",
  "date_column": "created_at",
  "snippet_column": 1,
  "metadata_columns": {
    "url": "t.url",
    "date": "t.created_at"
  }
}
```

| Field | Description |
|-------|-------------|
| `table_name` | FTS5 virtual table name |
| `source_table` | SQLite table to index |
| `columns` | Columns to include in the FTS index |
| `tokenizer` | FTS5 tokenizer (`"porter unicode61"` recommended) |
| `id_column` | Primary key column in source table |
| `map_id_column` | Column name in the FTS map table |
| `search_type` | Type name used in `warehouse search --type <name>` |
| `title_column` | Column to use as result title |
| `title_expr` | SQL expression for title (alternative to `title_column`) |
| `title_fallback` | Fallback title when column is NULL |
| `date_column` | Column for date filtering and display |
| `snippet_column` | Which FTS column (0-indexed) to use for snippets |
| `snippet_template` | Special snippet format (`"amount_category"` for transactions) |
| `source_tag` | Tag for shared FTS tables (e.g., multiple sources into one FTS table) |
| `column_expressions` | SQL expressions for computed FTS columns |
| `metadata_columns` | Additional columns to include in search results |

### Template Variables

Use `{{...}}` in URLs and values:

| Variable | Description |
|----------|-------------|
| `{{user.id}}` | From discovery step `user` |
| `{{last_sync.date}}` | Last successful sync date |
| `{{date.today}}` | Today's date (YYYY-MM-DD) |
| `{{date.month_start}}` | First day of current month |
| `{{date.month_end}}` | Last day of current month |
| `{{cookies.name}}` | Cookie value from browser_cookies auth |
| `{{query_ids.Name}}` | From `fetch_regex_map` discovery |

## Testing

```bash
# Install locally from file
warehouse connector add ./connectors/my-connector.json

# Check it loaded
warehouse connector list
warehouse connector info my-connector

# Sync
warehouse sync my-connector

# Verify data
warehouse status

# If FTS is configured, rebuild indexes and search
warehouse index
warehouse search "test query" --type mytype

# Clean up
warehouse connector remove my-connector
```

## Tips

- Start simple: one table, no pagination, no FTS. Add complexity after the basic sync works.
- Use `"results_path": "$"` when the API returns a plain array.
- Always include `_extracted_at` as the last column.
- For GraphQL APIs, set `"api_type": "graphql"` and use `query`/`operation_name` in the endpoint.
- The `governance_fields` array controls which fields can be redacted by Warehouse's permission system.
- Test pagination by setting `"max_pages": 2` first, then increase once it works.
