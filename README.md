# Warehouse Connectors

Community gallery of connector specs for [Warehouse](https://github.com/paulmeller/warehouse-cli).

Each connector is a single JSON file that tells Warehouse how to sync data from a REST or GraphQL API into your local SQLite database. No code required.

## Available Connectors

| Connector | Description | Auth | Install |
|-----------|-------------|------|---------|
| **[PocketSmith](connectors/pocketsmith.json)** | Accounts, categories, transactions | API key | `warehouse connector add https://raw.githubusercontent.com/paulmeller/warehouse-connectors/main/connectors/pocketsmith.json` |
| **[Monarch Money](connectors/monarch.json)** | Accounts, transactions, recurring, budgets | Token chain (env / Safari) | `warehouse connector add https://raw.githubusercontent.com/paulmeller/warehouse-connectors/main/connectors/monarch.json` |
| **[Twitter/X](connectors/twitter.json)** | Bookmarks and likes | Browser cookies | `warehouse connector add https://raw.githubusercontent.com/paulmeller/warehouse-connectors/main/connectors/twitter.json` |
| **[Notion](connectors/notion.json)** | Pages and databases | API key | `warehouse connector add https://raw.githubusercontent.com/paulmeller/warehouse-connectors/main/connectors/notion.json` |
| **[GitHub](connectors/github.json)** | Starred repositories | Personal access token | `warehouse connector add https://raw.githubusercontent.com/paulmeller/warehouse-connectors/main/connectors/github.json` |
| **[Hacker News](connectors/hackernews.json)** | Front page stories | None (public API) | `warehouse connector add https://raw.githubusercontent.com/paulmeller/warehouse-connectors/main/connectors/hackernews.json` |

## Quick Start

```bash
# Install a connector
warehouse connector add https://raw.githubusercontent.com/paulmeller/warehouse-connectors/main/connectors/github.json

# Configure auth (if needed) in ~/.warehouse/config.toml
# [github]
# token = "ghp_..."

# Sync data
warehouse sync github

# Search
warehouse search "rust cli tool"
```

Or use `warehouse setup` which will interactively offer to install popular connectors.

## Managing Connectors

```bash
warehouse connector list              # see all installed connectors
warehouse connector info pocketsmith  # show connector details
warehouse connector remove github     # uninstall a connector
```

## Writing Your Own

See [AUTHORING.md](AUTHORING.md) for the full connector spec reference.

The short version: create a JSON file with `version`, `name`, `description`, `tables` (with columns, endpoint, response mappings), and optionally `auth` and `fts` sections. Test it locally, then open a PR to add it here.

## Contributing

1. Create your connector JSON file in `connectors/`
2. Test it: `warehouse connector add ./connectors/my-connector.json && warehouse sync my-connector`
3. Open a PR

## License

MIT
