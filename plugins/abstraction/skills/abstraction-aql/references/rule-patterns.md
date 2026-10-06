# Rule patterns

Worked queries for the rule shapes that come up most, each a complete step.
They use an example codebase: a Go monorepo whose packages are `api` (the main
service), `admin` (a separate admin service), `billing`, `common` (shared code,
with a `logger` module) and `core`, with dependencies such as the Stripe SDK and
zerolog. Swap in your own names and run the query — a pattern is a starting
point, not a proof for your codebase.

## Gateway: only one place may call a dependency

*"Only billing's Stripe adapter may call the Stripe SDK."* A direct-caller rule:
everything else is expected to go through the adapter.

```typescript
let sdk     = functions().filterInPackageWithName("github.com/stripe/stripe-go*")
let adapter = functions().filterInPackageWithName("billing").filterInFileWithName("internal/stripe/*")
assertNotEmpty(adapter)
// Only billing's Stripe adapter talks to the Stripe SDK; everything else goes through billing.Service.
assertSubset(sdk.callers().filterPkgCategory(PkgCategory.Main).setTestVisibility(TestVisibility.Exclude), adapter)
```

`filterPkgCategory(PkgCategory.Main)` is what makes this work: a dependency's
`callers()` includes its own internals.

Expect findings in one-off tools — a setup script, a migration binary — that
call the SDK directly. They are usually exemptions, named narrowly:
`.filterNotIn(functions().filterInPackageWithName("api").filterInFileWithName("tools/stripe-setup/*"))`.

## Wrapper: a library is used only through our wrapper

*"Only common/logger may use zerolog."* Same shape, with a module as the
allowed set.

```typescript
let zerolog = functions().filterInPackageWithName("github.com/rs/zerolog")
let logger  = functions().filterInPackageWithName("common").filterInModuleWithName("logger")
assertNotEmpty(logger)
// Services log through common/logger so level, format and context fields stay uniform.
assertSubset(zerolog.callers().filterPkgCategory(PkgCategory.Main), logger)
```

## Isolation: one area never reaches another

*"admin never ends up in api code."* A scope rule: a call anywhere in the call
tree below an `admin` function counts.

```typescript
// admin is deployed apart from api and must not depend on its code.
duringCallsTo(functions().filterInPackageWithName("admin"))
    .assertNeverCalls(functions().filterInPackageWithName("api"))
```

The set form says the same thing and names the shared functions when it fails:

```typescript
assertNoOverlap(functions().filterInPackageWithName("admin").callChainDown(),
                functions().filterInPackageWithName("api"))
```

Exempt sanctioned bridges with `.exempt(...)` on the scope; this removes them
and everything below them. To prove the rule can fail, check the reverse
direction, which usually does share code.

## Obligation: every entry point performs a step

*"Every webhook verifies the request signature."* `assertCalls` holds when each
root's call tree reaches a required function, at any depth and across packages.

```typescript
let handlers = functions().filterInPackageWithName("api")
    .filterInFile(files().filterNameAny("*_webhook_controller.go", "slack_events_controller.go"))
    .filterNameAny("HandleWebhook", "HandleEvent")
let verify = functions().filterPkgCategory(PkgCategory.Main)
    .filterNameAny("verifySignature", "ParseEvent", "VerifyRequest")
assertNotEmpty(handlers)
// An inbound webhook body is untrusted until its signature has been checked.
duringCallsTo(handlers).assertCalls(verify)
```

Take the roots from the code that registers them — the router — not from a
naming convention: endpoints that don't follow it (here the Slack one) are
exactly the ones a convention-based selector misses. The required function may
sit several calls down, in another package; that still counts.

Two limits to report with a rule like this:

- `assertCalls` proves reachability, not order or every path. A handler that
  verifies only `if secret != ""` passes either way.
- A generic required name (`ParseEvent`) is satisfied by any function of that
  name. Pin it with a package or scoped name when the name is common.

The role-based variant over-reports: `filterRole(Role.HTTPHandler)` also tags
a controller's internal per-event handlers, which run only after the entry
point has verified, so each of them fails `assertCalls(verify)`. Use roles to
discover candidates, not as the roots.

## Registration: every route sits behind a middleware

*"Every admin endpoint requires an authenticated admin."* Handlers are called by
the router framework, not by your code, so `duringCallsTo(handlers)` has nothing
to look below. Assert where they are **registered** instead: the function that
installs the middleware also registers the protected routes, so everything it
reaches is behind the gate.

```typescript
let handlers = types().filterInPackageWithName("admin").filterInModuleWithName("httpapi")
    .filterName("*Controller").methods(Is.Exported).setTestVisibility(TestVisibility.Exclude)
let gate   = functions().filterInPackageWithName("admin").filterInFileWithName("internal/httpapi/authenticate.go").filterNameExact("Authenticate")
let group  = gate.callers().setTestVisibility(TestVisibility.Exclude)
let publicRoutes = types().filterInPackageWithName("admin").filterNameExact("AuthController")
    .methods(Is.Exported).filterNameAny("Login", "Callback", "Logout")
assertNotEmpty(handlers)
assertNotEmpty(group)
assertNotEmpty(publicRoutes)
// Every controller handler is registered in the route group guarded by Authenticate; only the login flow is public.
assertSubset(handlers.filterNotIn(publicRoutes), group.callChainDown())
```

`group` is the closure that calls `Authenticate` — closures have no name, so it
is reached as the gate's caller. New public endpoints must be added to
`publicRoutes` on purpose. Report the limits: this proves registration below the
gate, not that the middleware runs first or what it checks.

## Layering: one part of a package never calls another

*"Inside core, `query` never depends on `rules`."* Select each side as a
directory subtree, since module names repeat:

```typescript
let query = functions().filterInPackageWithName("core").filterInFileWithName("pkg/query/**").setTestVisibility(TestVisibility.Exclude)
let rules = functions().filterInPackageWithName("core").filterInFileWithName("pkg/rules/**")
assertNotEmpty(query)
assertNotEmpty(rules)
// rules builds on query, never the other way round.
duringCallsTo(query).assertNeverCalls(rules)
```

Prove it can fail with the reverse direction —
`assertEmpty(query.callers().filterInFileWithName("pkg/rules/**"))` should fail
with real calls. This sees calls, not imports: a type-only dependency passes.

## Same-directory rules: each folder is private to its owner

*"A route's `_components` folder is only for that route."* AQL has no path
variables, so "callers must sit next to the callee" takes one step per folder,
plus a guard that fails when a new folder appears without a step.

```typescript
// One step per folder:
let inner = functions().filterInPackageWithName("app").filterInFileWithName("src/routes/(private)/settings/_components/**")
let route = functions().filterInPackageWithName("app").filterInFileWithName("src/routes/(private)/settings/**")
assertNotEmpty(inner)
// A route's _components folder is private to that route; shared components belong in src/lib/components.
assertEmpty(inner.callers().filterPkgCategory(PkgCategory.Main).setTestVisibility(TestVisibility.Exclude).filterNotIn(route))
```

```typescript
// The guard, listing every folder that has a step:
let all     = functions().filterInPackageWithName("app").filterInFileWithName("src/routes/**/_components/**")
let covered = functions().filterInPackageWithName("app").filterInFile(files().filterNameAny(
    "src/routes/(private)/settings/_components/**", "src/routes/(public)/login/_components/**"))
assertNotEmpty(all)
// A new _components folder needs its own step in this check.
assertEmpty(all.filterNotIn(covered))
```

Bracketed route segments (`[slug]`) are character classes in a glob — match
them with `*`. Using a component counts as a call; importing a plain `.ts`
constant from another route's folder does not, so pair this with a linter rule
(`import/no-restricted-paths`) if constants matter.

## Changed code: what a pull request adds must meet a bar

*"Handlers a pull request adds or changes carry a doc comment."*
`filterChanged()` keeps the definitions the diff added or modified.

```typescript
let handlers = files().filterInPackage(packages().filterName("api")).functions()
    .filterRole(Role.HTTPHandler).setTestVisibility(TestVisibility.Exclude)
assertNotEmpty(handlers)
// Handlers a pull request adds or changes carry a doc comment.
assertSubset(handlers.filterChanged(), handlers.filterHasDocComment())
```

Starting from `files()….functions()` drops closures, which a role often tags
and which can't carry a doc comment. Without a diff — on main, in `run_check` —
`filterChanged()` is empty and the rule passes vacuously; validate the operand
without it (`assertSubset(handlers, handlers.filterHasDocComment())` shows the
existing debt) and use `severity: warning` so edits to old code nudge rather
than block.

## Data-store gateway: only one module touches a table

*"Only billing touches the billing tables."* Tables accessed through an ORM or
table objects are written with method calls, so `mutatedBy()` on the table
global returns nothing. `referencedBy()` sees every use:

```typescript
let table   = files().filterInPackage(packages().filterName("common")).filterName("database/tables.go")
    .variables(Is.Global).filterScopedName("OrganizationSubscriptions")
let billing = functions().filterInPackageWithName("billing")
assertNotEmpty(table)
// Only billing touches organization_subscriptions; everything else goes through billing.Service.
assertSubset(table.referencedBy().setTestVisibility(TestVisibility.Exclude), billing)
```

This forbids reads outside billing too — say so. Raw SQL strings and
migrations are invisible to it.

## Allow-list check: new call sites need a human

When the property itself can't be proven — usage is recorded on the far side of
a service boundary, say — pin the set of places that may do the thing, so a
*new* one fails the review and someone confirms it:

```typescript
let modelCalls = functions().filterInPackageWithName("github.com/acme/llmkit").filterInModuleWithName("agent")
    .filterNameAny("Run", "RunStream")
let known = functions().filterPkgCategory(PkgCategory.Main).filterNameAny("execute", "verifyRound", "RunCheck")
// Every model call site has been confirmed to record usage. A new one must be reviewed, then added here.
assertSubset(modelCalls.callers().filterPkgCategory(PkgCategory.Main).setTestVisibility(TestVisibility.Exclude), known)
```

It proves nothing about the property, so say that in the step's comment and to
the user, and expect to edit `known` whenever a call site is legitimately added.

## Allow-list: a scope may only call these

*"The pricing module only calls pricing code."*

```typescript
let pricing = functions().filterInPackageWithName("billing").filterInModuleWithName("pricing")
    .setTestVisibility(TestVisibility.Exclude)
duringCallsTo(pricing).assertOnlyCalls(pricing)
```

Expect this to fail, instructively. Universality counts **every** function
reached, transitively, standard library included: one logging call pulls in the
logger, its library, `fmt`, `reflect` and `runtime`, and each failing root lists
all of them — a result can run to hundreds of kilobytes. Build the allow-list
from what the area legitimately uses (`pricing`, the logger module,
`functions().filterInPackageWithName("Go standard library")`), try it on one
file of roots first (`pricing.filterInFileWithName("tiers.go")` — `limit` would
not narrow it), and reserve it for small, deliberately closed areas. It is rejected on `codebase()`.

When "pure" is the real intent — no network, no writes — a prohibition over
direct callees is usually the better rule:

```typescript
let std     = functions().filterInPackageWithName("Go standard library")
let network = std.filterInModule(modules().filterNameAny("net", "http", "sql", "smtp", "rpc"))
assertNotEmpty(network)
// pricing computes; it never talks to the network.
assertNoOverlap(pricing.callees(), network)
```

## Prohibition: nothing calls this

*"Production code never uses the standard library logger."* The set form lists
offenders by caller:

```typescript
let stdlog = functions().filterInPackageWithName("Go standard library").filterInModuleWithName("log")
assertNotEmpty(stdlog)
// Services log through common/logger; the stdlib logger bypasses levels and structured fields.
assertEmpty(stdlog.callers().filterPkgCategory(PkgCategory.Main).setTestVisibility(TestVisibility.Exclude))
```

A package that cannot depend on your logger (a zero-dependency config package,
say) is an exemption, not a violation. Exempt the file, not the whole package,
so a new call elsewhere in it is still caught:
`.filterNotIn(functions().filterInPackageWithName("config").filterInFileWithName("loader.go"))`.

The scope form is `codebase().assertNeverCalls(f)`, but **`codebase()` includes
dependency code**: the stdlib `log` package calling its own helpers fails the
rule. Exempt everything that isn't the workspace's production code:

```typescript
let ours = functions().filterPkgCategory(PkgCategory.Main).setTestVisibility(TestVisibility.Exclude)
codebase().exempt(functions().filterNotIn(ours)).assertNeverCalls(stdlog)
```

`codebase().assertCalls(f)` is the dual: every function in `f` is called by
something — a guard for an API that must stay wired.

## Ordering: after X, before Y

*"Between opening a transaction and committing it, nothing sends HTTP."*
`afterCallTo` opens an activation at each call to `open` and closes it when the
enclosing function returns, or at `untilCallTo(close)`.

```typescript
afterCallTo(functions().filterScopedName("Database.BeginTx"))
    .untilCallTo(functions().filterNameAny("Commit", "Rollback"))
    .assertNeverCalls(functions().filterRole(Role.HTTPClient))
```

The bracket never crosses the enclosing function's return, so a transaction
opened in one function and committed in another is not covered. Confirm each
operand is non-empty first; this shape depends heavily on your names.

## Existence and naming

```typescript
// Every Go source file name is lower case (snake_case).
assertEmpty(files().filterInPackage(packages().filterName("api")).filterName("*[A-Z]*.go"))
```

`FileStream` has no `filterInPackageWithName`; go through
`filterInPackage(packages()...)`.

Counting rules use the boolean `assert`, which can't name offenders — prefer a
set assertion when one fits:

```typescript
assert(functions().filterInPackageWithName("api").filterInFileWithName("*_webhook_controller.go")
    .filterNameExact("HandleWebhook").count() == 2)
```

## Sensitive data

Tags are classifier output on variables. Cheap structural form — no logger
takes a secret as a parameter:

```typescript
assertEmpty(functions().filterPkgCategory(PkgCategory.Main).filterRole(Role.Logger)
    .filterHasParam(parameters().filterTag(Tag.AuthToken, Tag.Password, Tag.PrivateKey)))
```

Data-flow forms (`valueSinks`, `valueDownstream`) follow values through calls
and are far more expensive: `valueSinks()` over every tagged variable in a large
workspace can outrun the tool's timeout. Narrow the start set to one package
and bound the search (`valueDownstream(context, maxHops)`) before using one in a
check.
