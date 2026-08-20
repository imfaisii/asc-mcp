# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [1.1.0] - 2026-08-21

### Changed

- Extracted the MCP server into its own standalone repository
  (`imfaisii/asc-mcp`), separate from its previous home inside a shared
  skills monorepo.

### Summary

- 4 token-efficient meta-tools (`asc_search`, `asc_schema`, `asc_call`,
  `asc_tags`) exposed to the MCP client, covering 1263 operations generated
  from Apple's official App Store Connect API OpenAPI spec, version 4.4.1.
