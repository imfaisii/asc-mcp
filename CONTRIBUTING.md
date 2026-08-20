# Contributing

Thanks for considering a contribution to the App Store Connect MCP server.

## Prerequisites

- [Bun](https://bun.sh) 1.1 or later.

## Setup

```bash
bun install
```

## Refreshing the OpenAPI spec and catalog

The tool catalog in `generated/` is produced from Apple's official OpenAPI
spec at `openapi/app-store-connect.openapi.json`. It is **generated, not
hand-edited** — if you need to update it (e.g. Apple published a new spec
version), replace the spec file and regenerate:

```bash
bun run generate
```

Commit both the updated spec and the regenerated `generated/` output
together.

## Verifying your changes

```bash
bun run typecheck
bun run smoke
```

`bun run smoke` runs offline by default (catalog checks, meta-tool
resolution, and a server boot check). It only makes a live App Store Connect
API call if `ASC_ISSUER_ID`, `ASC_KEY_ID`, and `ASC_PRIVATE_KEY_PATH` are set
in your environment.

## Credentials

Never include real App Store Connect credentials, `.p8` files, issuer IDs,
or key IDs in a pull request, commit message, issue, or test fixture. Use
the placeholders in `.env.example`.
