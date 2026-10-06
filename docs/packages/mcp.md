---
title: marko/mcp
description: MCP server exposing Marko codebase introspection to AI coding agents.
---

Model Context Protocol (MCP) server that exposes Marko codebase introspection to AI coding agents (Claude Code, Cursor, Codex, and any MCP client). It speaks JSON-RPC over stdio (`marko mcp:serve`) and answers structured questions about the project — modules, routes, observers, plugins, config, templates — so an agent can reason about a Marko app without grepping.

Like [`marko/lsp`](/docs/packages/lsp/), it builds on [`marko/codeindexer`](/docs/packages/codeindexer/): tools read the cached symbol index rather than re-parsing. (Installing `marko/mcp` pulls codeindexer in automatically.)

## Installation

```bash
composer require marko/mcp
```

## Usage

```bash
marko mcp:serve
```

Register it with your agent, e.g. for Claude Code:

```bash
claude mcp add marko-mcp -- marko mcp:serve
```

`mcp:serve` refuses to start when the app environment is production, and an unset `APP_ENV`/`MARKO_ENV` counts as production. Set `APP_ENV=local` (or `development`) on development machines. To serve a production environment on purpose, set `MCP_ALLOW_PRODUCTION=true`.

## Tools

Always available (13 total):

Index-backed tools (10):

| Tool | Purpose |
|------|---------|
| `list_modules`, `list_commands`, `list_routes` | Enumerate the module graph |
| `find_event_observers` | Observers listening to an event |
| `find_plugins_targeting` | Plugins intercepting a class |
| `resolve_preference` | Resolve an interface/class to its bound implementation or `#[Preference]` |
| `resolve_template` | Resolve a `module::template` to an absolute path |
| `get_config_schema`, `check_config_key` | Inspect config keys |
| `validate_module` | Validate a module's structure |

Runtime tools (3):

| Tool | Notes |
|------|-------|
| `app_info` | Application name and installed package versions |
| `read_log_entries` | Returns the last `count` lines (default 50, clamped to 1–500) of the most recent log in `storage/logs`. The file is read backwards from the end, so a large log is never loaded into memory |
| `run_console_command` | Runs an allowlisted `marko` CLI command and captures output (see below) |

Conditional tools (registered only when their dependency is present):

| Tool | Requires | Notes |
|------|----------|-------|
| `query_database` | a `marko/database` driver | Read-only unless the server config enables writes (see below); registered only when a DB connection is available |
| `search_docs` | a docs driver (`marko/docs-fts`) | Registered only when a `DocsSearchInterface` is bound |

> There is intentionally **no `last_error` tool** and no global error-capture plugin. Read the most recent errors with `read_log_entries`, with no production-time side effects.

## `run_console_command` Safety

An agent can be steered by content it reads, and it controls the command name and every argument, including `--force`. So the server operator, not the agent, decides what `run_console_command` may run:

1. **Allowlist.** Only commands named in `mcp.console.allowed_commands` run (an alias of an allowed command also runs). Every other command is refused before it is dispatched. The default list is read-only: `list`, `module:list`, `route:list`, `db:status`, `db:diff`, `cache:status`, `page-cache:status`, `queue:status` and `queue:failed`.
2. **Destructive commands.** A command marked `#[Command(destructive: true)]` is refused even when it is allowlisted, unless `mcp.console.allow_destructive` is `true` (`MCP_ALLOW_DESTRUCTIVE=true`). Passing `--force` cannot enable it. Every shipped command that changes or deletes stored state carries the marker: `db:migrate`, `db:reset`, `db:rollback`, `db:rebuild`, `db:seed`, `cache:clear`, `page-cache:clear`, `page-cache:purge`, `queue:clear`, `log:clear`, `session:gc`, `auth:clear-tokens`, `discovery:clear`, `indexer:rebuild`, `admin-auth:permissions:sync` and `devai:install`. The tool reads the marker from the command's definition; it does not guess from option names, so a destructive command without a `--force` flag (such as `cache:clear`) is refused too. Mark your own state-changing commands the same way (see [marko/cli](/docs/packages/cli/#destructive-commands)).

To allow more commands, override the list in your app's `config/mcp.php`:

```php title="config/mcp.php"
return [
    'console' => [
        'allowed_commands' => ['list', 'module:list', 'route:list', 'db:status', 'cache:clear'],
        'allow_destructive' => true,
    ],
];
```

`cache:clear` is destructive, so it needs both settings: the allowlist entry and `allow_destructive`. Turning on `allow_destructive` enables only the destructive commands that are also in the allowlist.

## `query_database` Safety

An agent can be steered by content it reads (database rows, log lines, repository files), so `query_database` does not trust the agent to stay read-only. Every read passes two guards:

1. **SQL check.** The statement must be a single `SELECT`, `WITH`, `SHOW`, `EXPLAIN` or `DESCRIBE` statement. Comments (`--`, `#`, `/* */`) and string literals are stripped first, and the statement is checked under the lexical rules of MySQL, PostgreSQL and SQLite, so a quote or comment that one database reads differently cannot hide a second statement. The tool rejects:
    - any `;` outside a string literal, except one trailing `;`
    - `INSERT`, `UPDATE`, `DELETE`, `MERGE` and `INTO` anywhere in the statement, which covers data-modifying CTEs (`WITH d AS (DELETE ...) SELECT ...`), `EXPLAIN ANALYZE DELETE ...` and `SELECT ... INTO OUTFILE`
    - functions with side effects, such as `dblink_exec()`, `pg_read_file()`, `setval()` and `load_file()`
    - MySQL executable comments (`/*! ... */`), and unterminated strings or comments
2. **Read-only transaction.** The statement runs inside a read-only transaction that is always rolled back: `START TRANSACTION READ ONLY` on MySQL and MariaDB, `BEGIN READ ONLY` on PostgreSQL, and `PRAGMA query_only = ON` plus a transaction on SQLite. The database refuses any write the SQL check missed. On any other driver the tool refuses to run the query.

Because the check is conservative across dialects, a few valid reads are rejected too, such as a PostgreSQL dollar-quoted string that contains a `;`. Rewrite the query with a plain string literal.

### Enabling writes

Writes are off by default. The `allowWrite` and `confirm` tool arguments come from the agent, so on their own they cannot enable writes. To let the agent run writes, set the server-side flag:

```bash
MCP_ALLOW_WRITES=true
```

or set it in your app's `config/mcp.php`:

```php title="config/mcp.php"
return [
    'database' => [
        'allow_writes' => true,
    ],
];
```

With writes enabled, the tool advertises the `allowWrite` and `confirm` arguments. A call with both set to `true` runs the statement outside the read-only transaction, with no SQL check, and prefixes the result with `WARNING: WRITE OPERATION executed.` Calls without `allowWrite` still go through both read guards.

For the strongest guarantee, also point the MCP server at a database user that only has read privileges. The read-only transaction blocks writes but not reads, so a privileged user can still read anything the connection can see.

## Configuration

| Key | Env | Default | Purpose |
|-----|-----|---------|---------|
| `mcp.allow_production` | `MCP_ALLOW_PRODUCTION` | `false` | Let `mcp:serve` start when the app environment is production |
| `mcp.console.allowed_commands` | — | read-only commands (see above) | Commands `run_console_command` may run, by name or alias |
| `mcp.console.allow_destructive` | `MCP_ALLOW_DESTRUCTIVE` | `false` | Let `run_console_command` run allowlisted commands marked `#[Command(destructive: true)]` |
| `mcp.database.allow_writes` | `MCP_ALLOW_WRITES` | `false` | Let `query_database` run writes when the agent passes `allowWrite` and `confirm` |

## Related Packages

- [`marko/codeindexer`](/docs/packages/codeindexer/) — the cached index the tools read
- [`marko/lsp`](/docs/packages/lsp/) — the editor-facing peer that reads the same index
- [`marko/docs-fts`](/docs/packages/docs-fts/) — enables `search_docs`
