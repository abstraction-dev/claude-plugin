---
name: abstraction-checks
description: Turn architectural intent into code contracts enforced on every pull request — "only billing calls Stripe", "the UI never touches the database", "every webhook verifies its signature". Use to add, write, fix or test an Abstraction check or AQL query, or whenever someone wants a boundary, invariant or design decision to stay true.
---

# Abstraction checks

A check is a **code contract**: a rule about the codebase's architecture — who
may call what, what must always happen, what must never meet — evaluated on
every pull request. Each check is a named group of **steps**, and each step
reports its own verdict. An **AQL step** is a query whose assertions state the
rule; the elements that break an assertion are its findings. The query runs over
the whole analysed codebase, so it proves a rule holds rather than spotting a
violation in a diff.

The job has three parts, and each one matters:

1. **Write** the rule as an AQL query.
2. **Validate** it against what exists on `main` with `run_check`, until its
   verdict is the one you expect and you understand every finding.
3. **Commit** it as a check file under `.abstraction/checks/` in the pull
   request, so the rule is versioned with the code and keeps being checked on
   every pull request after it.

**Load the `abstraction-aql` skill before writing a query.** It covers the
language, running queries with `run_check`, reading results, and worked queries
for the common rule shapes. This skill covers turning a query into a contract.

## Tools

On the `abstraction` MCP server. Each takes a `workspace`: the `Slug` field
from `list_workspaces` (a UUID), found by the workspace's `Name`.

| Tool | Use |
|---|---|
| `run_check` | Evaluate an unsaved check against the workspace's analysis of `main`. Saves nothing. Your REPL. |
| `ask` | Have Astrid locate code or explain an area when you don't know the names. |
| `list_checks` | The workspace's stored checks, to avoid duplicating a rule. Its output is large and usually lands in a file — grep that for your topic. |
| `create_check` | Store a workspace check instead of a file — only when asked (see [Workspace checks](#workspace-checks)). |

## Workflow

1. **State the rule as a set relation.** Most rules are one of: *these must be
   empty*, *these must be a subset of those*, *nothing in scope X may call Y*,
   *everything in scope X must call Y*. Name the sets in plain words first. If
   it isn't one, see [Choosing the tool](#choosing-the-tool).
2. **Look for an existing rule.** Read any `.abstraction/checks/` files in the
   repository, and grep `list_checks` for the area's names even when there are
   none — workspace checks live there, not in the repository. Its output usually
   lands in a file. Extend a check that already covers the rule.
   Take the rule as the user stated it. If the literal reading fails on main
   because of a deliberate design — "in the same function", but the code
   delegates one call down — don't quietly switch to another reading: show both
   and their verdicts, recommend one, and let the user choose.
3. **Pin down each set** — exact package, module, file and function names
   (`abstraction-aql` → *Selecting code by name*). Use `ask` when you don't know
   them, and read the code that wires the area (a router, a registry) so the set
   is complete: a pattern's operands are examples, not an inventory. Prove each
   operand is non-empty with `assertNotEmpty` — a filter that matches nothing is
   the most common way a check silently proves nothing.
4. **Write the query and run it** with `run_check` (`abstraction-aql` →
   *Running a query*, *Choosing the assertion*).
5. **Understand every finding on main** ([below](#validating-against-main)),
   iterating at most about five rounds. Aim for a check that passes; when it
   won't, warn the user before opening the pull request.
6. **Prove it can fail** ([below](#proving-it-can-fail)).
7. **Write the check file** ([below](#committing-the-check-file)) with the exact
   query you last ran, and add it to the pull request.
8. **Report** ([below](#reporting-back)).

## Choosing the tool

| The rule is about… | Use |
|---|---|
| Who calls what, what must or must never be reached, what may depend on what | An AQL step |
| Judgement a reviewer applies by reading the change ("error messages are actionable") | An [agent step](#agent-steps) |
| Text inside code — SQL statements, URLs, literals | An agent step, or a linter or grep in CI |
| Imports, or calls to runtime globals the analysis doesn't trace | A linter rule (e.g. ESLint `no-restricted-imports` / `no-restricted-globals`) |
| Repository layout — file naming, "every X file needs a matching Y file" | A script in CI. Non-code files are indexed, but pairing one file with another needs path logic AQL doesn't have |
| An obligation fulfilled across a service boundary (a worker returns usage, the backend bills it) | AQL can't follow data over RPC. Offer an allow-list check that fails when a new call site appears, so a person confirms it — see *Allow-list check* in the patterns — an agent step, or a runtime guard in code |

`abstraction-aql` → *What AQL can't see* explains the last two rows. When a
check is not the right tool, say so and recommend the alternative — a check
that proves nothing, or floods reviews with false positives, is worse than
none. A rule can also split: AQL for the part it sees (component use, calls)
plus a linter rule for the rest (type and constant imports); recommend both.

## Validating against main

`run_check` evaluates against the workspace's analysis of the **main branch**,
not your working tree. That is exactly the question to answer before
committing: *what will this rule say about the code as it is?* For what each
verdict means, see `abstraction-aql` → *Reading results*.

**Classify every finding.** Each one is:

- **a real violation** — the rule is doing its job. Fixing it in the same pull
  request is welcome but not required;
- **a legitimate exception** — exempt it *explicitly and narrowly* in the query
  (a named file, module or function) with a comment saying why. Dev-time tools
  inside a package — code generators, `cmd/` binaries, scripts — are the common
  case: `.filterNotIn(functions().filterInPackageWithName("core").filterInFileWithName("internal/codegen/*"))`;
- **a wrong operand** — fix the filter.

Ask the user when you can't tell a violation from an exception. Don't commit:

- a step that reports "did not check anything" — it proves nothing;
- a step whose findings you can't attribute or classify (a data-flow rule that
  names sinks but not sources, say) — report it and what it would take instead.

And say plainly when a passing rule is weak: non-empty operands and a pass don't
make a rule strong if it can hardly fail by construction (no logger takes a
`password` parameter, because loggers take `msg, args...`).

**Prefer committing a check that passes**, but a check may land with known
violations. What matters is that nobody is surprised: once the file lands, the
step reports on every pull request it runs on — this one included — until the
violations are gone. So when any step will not pass on the pull request, at
`fail` *or* `warning` severity, **warn the user before opening it**: name the
step, its severity and its current findings, and offer the choices — fix them
here, exempt them, lower the severity (`warning` keeps them visible without
failing the review, `info` only records them), or land it as is.

**When the pull request itself adds the code the rule talks about**, main
doesn't have it yet: an operand that is empty on main is expected. Validate the
rest on main and let the pull request's own review be the proof.

**Rules about changed code** (`filterChanged()`) and agent steps have no diff on
main, so `run_check` passes them vacuously. Validate their operands without
`filterChanged()`, and tell the user their first real test is a pull request.

## Proving it can fail

A passing query has shown nothing until you've seen it catch something. Once,
in a throwaway step:

- **Set rules:** narrow the allowed set to part of itself (one file) — it should
  fail naming real code.
- **Layering rules:** run the reverse direction, which usually does share code.
- **Obligations:** require a function you've confirmed is not reachable.
- **Prohibitions with exemptions:** run it once without the exemptions — it
  should fail on exactly what you exempted.

When nothing can violate the rule yet (two services that share no code, so the
reverse direction passes too), swap the forbidden set for a package you know is
reachable: that proves the traversal, not the rule — say so.

A deliberate failure that passes means its filter matched nothing; fix the
filter. If nothing anywhere calls the forbidden function, the rule can't be
shown to fail — report it as unproven.

## Committing the check file

A check file is discovered by the review pipeline on every pull request. It
lives at `.abstraction/checks/<topic>.abstrcheck.yaml` directly under the root
of a main package — a directory with its own manifest (`go.mod`, `package.json`,
`pyproject.toml`, …) that is part of the workspace's own code rather than a
dependency. That package decides which pull requests run it:

- **Cross-cutting rule** — a violation could be introduced from anywhere ("only
  common/logger may use zerolog", "nothing outside billing calls Stripe"): the
  **repository root**, `.abstraction/checks/<topic>.abstrcheck.yaml`.
- **Local rule** — every place a violation could appear is inside one package
  ("billing's pricing module stays pure"): that **package's root**, e.g.
  `billing/.abstraction/checks/pricing.abstrcheck.yaml`.

A rule *about* one package isn't necessarily local. "The worker never reaches
the database" can be broken by a change in a shared package the worker calls,
so it belongs at the repository root.

Group related checks in one file per topic; add to an existing file rather than
creating a near-duplicate. A local rule usually needs no `filters`: the
package's own gate is the right one.

```yaml
# .abstraction/checks/billing.abstrcheck.yaml (repository root: a violation can come from any package)
version: 1
checks:
  - name: stripe-sdk-only-from-billing
    steps:
      - aql:
          name: direct-callers
          comment: Only billing's Stripe adapter may call the Stripe SDK.
          query: |
            let sdk     = functions().filterInPackageWithName("github.com/stripe/stripe-go*")
            let adapter = functions().filterInPackageWithName("billing").filterInFileWithName("internal/stripe/*")
            assertNotEmpty(adapter)
            // Only billing's Stripe adapter talks to the Stripe SDK; everything else goes through billing.Service.
            assertSubset(sdk.callers().filterPkgCategory(PkgCategory.Main).setTestVisibility(TestVisibility.Exclude), adapter)
```

The step's `comment` is the one-line summary shown next to its verdict on every
run; the `//` comment above the assertion is shown with a failure. Keep both
short. Write the `//` comment into the query before your final `run_check`, so
the file holds exactly what you ran. Both are part of the rule's text, so a
repository's "no comments" convention doesn't apply to them.

How the pipeline treats the file:

- Each check is reported as `<package>: <name>`, e.g. `billing: pricing-stays-pure`.
- It runs only when the pull request changes a file in the package that holds
  it — at the repository root, any file in the repository. A root-level rule can
  list the packages whose changes could break it in `filters.paths` to run less
  often. Narrow it with
  `filters.paths`, relative to that root (`["billing/**"]` at the repository
  root, `["internal/stripe/**"]` in `billing`). The query itself always sees the
  whole codebase.
- It runs from the pull request's head, so the pull request that adds the file
  is the first one checked by it.
- Parsing is strict. An unknown field, a misspelled key or a bad value skips
  the whole file, and the review shows no checks from it. Check every key
  against the schema below, and copy the query text verbatim from your last
  `run_check`.

Schema:

| Level | Fields |
|---|---|
| file | `version: 1`, `checks` |
| check | `name` (unique in the file), `enabled` (default true), `filters`, `steps` (at least one) |
| `filters` | `paths`; `file_types` (`go`, `typescript`, `javascript`, `python`, `java`, `svelte`, `vue`, …); `entity_tags` (e.g. `auth_token`, `email`); `function_roles` (the snake_case names, e.g. `http_handler`) |
| step | exactly one of `aql:` or `agent:` |
| `aql` | `name`, `comment`, `severity` (`fail` default, `warning`, `info`), `enabled`, `query` |
| `agent` | the same common fields, plus `model`, `thinking_effort`, `message` |

Step names must be unique within a check and can't start with `#`.

`run_check` and the file spell the same step differently:

| `run_check` | Check file |
|---|---|
| `check_steps: [{check_type: "aql", settings_json: {query}}]` | `steps: [{aql: {query}}]` |
| `settings_json: {userMessage, model, thinkingEffort}` | `agent: {message, model, thinking_effort}` |
| `filter_paths`, `filter_file_types`, … | `filters: {paths, file_types, …}` |

Commit the file with the rest of the change. If a step still reports findings,
the warning above comes first. Once the pull request's review completes,
confirm each check reported: `get_pr_review` with the PR URL lists every check's
verdict and summary, file checks under `<package>: <name>`. A check missing
from that list means the file did not parse.

## Workspace checks

When the user wants the rule in the workspace rather than the repository —
managed from the app, applying to every repository the workspace tracks — the
workflow is the same up to step 6; then, instead of a file and a pull request,
call `create_check` with the definition you validated with `run_check`:

```json
{"workspace": "<slug>",
 "name": "API handlers are documented",
 "filter_paths": ["api/**"],
 "filter_function_roles": ["http_handler"],
 "check_steps": [{
   "name": "changed-handlers-have-doc-comment",
   "check_type": "aql",
   "severity": "warning",
   "runs_on": "diff_review",
   "comment": "HTTP handlers a pull request adds or changes carry a doc comment.",
   "settings_json": {"query": "let handlers = files().filterInPackage(packages().filterName(\"api\")).functions().filterRole(Role.HTTPHandler)\nassertNotEmpty(handlers)\nassertSubset(handlers.filterChanged(), handlers.filterHasDocComment())"}}]}
```

- Writing workspace checks (`create_check`, `update_check`, `delete_check`)
  needs workspace admin.
- The `filter_*` fields gate the check in reviews: it runs only when the diff
  touches a match. `filter_paths` match a file's package-relative path or the
  same path prefixed with its package (`api/**`).
- `runs_on` is `workspace_update`, `diff_review` or `both`; a rule about changed
  code belongs on `diff_review`.
- `update_check` with `check_steps` replaces every step and needs the
  `expected_revision` that `list_checks` reports.

## Agent steps

Use an `agent` step when the rule needs reading and judgement rather than set
relations — naming quality, whether an error message is helpful, whether a
change matches a design. It costs an LLM run on every pull request it applies
to, so gate it with `filters`.

```yaml
      - agent:
          name: handler-review
          model: claude-sonnet-5-5
          thinking_effort: low
          message: Review the changed HTTP handlers for fail-open auth paths and unvalidated input.
```

Models: `claude-fable-5`, `claude-opus-5-5`, `claude-sonnet-5-5`, `gpt-6-sol`,
`gpt-6-luna`; efforts `low`, `medium`, `high`, `max`.

Write the message as a reviewer's brief: what to flag, what not to flag, and
whether test code is in scope — the agent has no `setTestVisibility`.

An agent step can't be validated before the pull request. `run_check` runs it
— a real, billed model call — but with no diff, so a step about changed code
passes vacuously; that SUCCESS proves nothing, and you can't make it fail on
purpose. Check the current code yourself instead (read or grep for what the
step would flag), and tell the user the step gets its first real test on the
pull request.

## Reporting back

"No check" is a valid outcome. When you conclude the rule needs another tool,
report that, why, and what you recommend instead — and question the premise if
the tooling can't honour it.

Otherwise, tell the user, briefly: the check file's path and each check's rule in one
sentence; the verdict on main and each current finding (file and function); any
exemptions in the query and why; and anything that needs their decision. If any
step will not pass on the pull request, lead with that: which step, its
severity, and what it will report.
