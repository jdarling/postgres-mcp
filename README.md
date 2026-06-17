# PostgreSQL MCP Server Design

A Design doc along with the transcript of how it was created.

## Contents

- [design.md](design.md): Product and implementation design for a PostgreSQL-backed MCP server.
- [transcript.md](transcript.md): Chronological summary of the design conversation and decisions.
- [review.md](review.md): Third-party review notes used to refine the design.
- [schemas/](schemas): Starter JSON Schema files, written as YAML, for config validation.

## Purpose

This repository captures the design for an MCP server that exposes curated PostgreSQL-backed commands as MCP tools. The server discovers approved commands from a filesystem catalog, executes SQL scripts or PostgreSQL functions/procedures with bound parameters, audits every execution, and provides status tools for operational visibility.

The design is implementation-language agnostic. Driver- and framework-specific details are intended to be completed in an implementation profile.

## Current Status

The design covers:

- Single catalog root with multiple PostgreSQL server configs.
- Curated SQL script, function, and procedure execution.
- Multi-statement script support using SQL-aware lexing/tokenization.
- Runtime timeout and optional aggregate row limits.
- Per-server shared PostgreSQL credentials.
- Strict config validation and failed-tool behavior.
- `PostgresMCP.Status` and `PostgresMCP.CatalogStatus`.
- Audit logging and secret redaction rules.
- Base schema files for global, server, command, and implementation-profile config.

