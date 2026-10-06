<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo-dark.svg">
    <img alt="Abstraction" src="assets/logo-light.svg" height="44">
  </picture>
</p>

# Abstraction for Claude Code

This plugin connects Claude Code to [Abstraction](https://abstraction.dev) and teaches it to write AQL and code checks.

- **Abstraction MCP server.** Ask Astrid about your codebase, run AQL, manage checks and read pull request reviews. You sign in with your Abstraction account in the browser the first time a tool is used.
- **`abstraction-aql` skill.** Write and debug AQL, Abstraction's query language for code structure: select code by name, follow call graphs and data flow, and assert architectural rules.
- **`abstraction-checks` skill.** Turn an architectural rule into a check that runs on every pull request, validate it against your main branch, and commit it as an `.abstraction/checks/<topic>.abstrcheck.yaml` file.

## Install

```
/plugin marketplace add abstraction-dev/claude-plugin
/plugin install abstraction@abstraction
```

Then run `/mcp`, pick `abstraction` and sign in.

## Requirements

An Abstraction account with at least one analysed workspace.
