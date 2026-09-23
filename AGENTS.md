# AGENTS.md

Instructions for AI agents working in this repo. Placeholder; conventions will be added as the first
skills land.

## Context

This is a **public** repo of agent skills for LeanLaw customers. Skills run against a firm's live
billing data through the LeanLaw MCP connector. `README.md` is the external catalog.

## Constraints

- No internal context in anything committed: no customer or employee names, no internal metrics, no
  links to internal tools.
- Refer to LeanLaw MCP tools by their logical name (`list_matters`), never a connector-specific prefix.
- Every write a skill makes (client, matter, fixed fee, time entry) is confirmed by the user first.
