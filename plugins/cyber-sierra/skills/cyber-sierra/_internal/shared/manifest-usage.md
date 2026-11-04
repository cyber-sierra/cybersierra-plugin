# Manifest Usage

The CLI manifest is the single source of truth for available commands. No component may hardcode CLI knowledge. This document defines how to query and interpret the manifest.

## CLI Environment

The `cybersierra` CLI is distributed as the npm package `@cybersierra/cybersierra-cli`.

Before running any `cybersierra` command, verify that version 1.1.0 or newer is installed:

```bash
cybersierra --version
```

If it is not found or older than `1.1.0`, install the latest release before proceeding (this also removes the deprecated package, which provides the same binary):

```bash
npm uninstall -g @cybersierra-cli/cybersierra; npm install -g @cybersierra/cybersierra-cli@latest
```

After installation it will be available on PATH.

## Querying the Manifest

### Full catalog (all modules)

```bash
cybersierra manifest
```

Returns:

```json
{
  "data": [ CatalogEntry, ... ],
  "meta": { "version": "string", "count": number }
}
```

### Raw tree (module-keyed)

```bash
cybersierra manifest --raw
```

Returns:

```json
{
  "version": "string",
  "tree": { "module": [ Operation, ... ], ... }
}
```

### Filtering to specific modules

The full manifest is too large to load whole. Sign-in saves it, pretty-printed, to `.cybersierra/agent-api.json` in the user's home folder, with each module's key indented four spaces inside `"tree"`. Slice it with the read-only Grep and Read tools rather than a script:

1. **List modules:** Grep the file for `^    "[a-z0-9-]+": \[` with line numbers. Each match is a module name and its start line.
2. **Load a module:** Read the file from that module's start line up to the line before the next module's start line (or to the end for the last one).

If the file does not exist, the user has not signed in yet: run the Authentication Protocol first. Before sign-in, `cybersierra manifest` returns no modules.

The Router determines which modules are relevant. The Planner receives only those modules.

## CatalogEntry Schema

```typescript
CatalogEntry {
  command: string        // "module resource action"
  module: string
  resource: string
  action: string
  method: string         // GET | POST | PUT | PATCH | DELETE
  path: string           // OpenAPI path
  description?: string
  safe: boolean          // true = read-only, false = write operation
  params: Param[]
}

Param {
  name: string
  in: "path" | "query" | "body" | "header" | "file"
  required: boolean
  default?: any
  enum?: any[]
  examples?: any[]
}
```

## Constructing CLI Commands

```
cybersierra <module> <resource> <action> [flags]
```

| Param location | CLI flag                             |
| -------------- | ------------------------------------ |
| `path`, `query`| `--paramName value`                  |
| `body`         | `--data '{"key": "value"}'`          |
| `file`         | `--file /path/to/file`               |

## Safety

- `safe: true` commands are read-only. Execute freely.
- `safe: false` commands modify state. Require user confirmation before execution.

## Module Discovery

Do not hardcode module lists. List the `tree` keys from the manifest file (see Filtering to specific modules) to discover available modules dynamically.
