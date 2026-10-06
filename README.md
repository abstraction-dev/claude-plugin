# <picture><source media="(prefers-color-scheme: dark)" srcset="assets/banner-dark.svg"><img alt="Abstraction for Claude Code" src="assets/banner-light.svg" height="40"></picture>

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

## License

[Apache-2.0](LICENSE). Abstraction and the Abstraction logo are registered trademarks of Abstr AB; the license does not grant permission to use them.
