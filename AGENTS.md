# Nexus

Nexus turns a supported GitHub codebase into an interactive map of its actual
structure.

Everything shown as structure comes from parsing the code. AI explains and
labels facts the parser found; it never decides what is connected.

The product decisions behind this build live in `docs/project-desc.md`. That
document explains what Nexus is, why it exists, its constraints, and the
reasoning behind the decisions. This file contains the engineering rules that
are always true.

## Stack

Next.js 16 App Router, React 19, TypeScript strict.

Neon Postgres for persistence, PostgreSQL row-level security, and database
branching.

Drizzle ORM for database schema, queries, and migrations.

Better Auth for Nexus authentication and its organization plugin for
organization membership and roles.

GitHub OAuth is a separate connection from Nexus authentication and is used for
repository access.

Use server-sent events (SSE) for live parsing and analysis progress where a
live connection is required. Do not introduce a dedicated realtime or
messaging system until the product actually requires one.

The AI provider must be replaceable. Do not couple the application to the
OpenAI SDK. If the implementation uses OpenAI, Gemini, or another provider,
hide it behind one application-level AI interface and one constructed client
or adapter. The provider can change without changing product code.

Use LangSmith for AI tracing where supported by the chosen provider/client
integration.

ts-morph may be used for TypeScript/JavaScript parsing. Language-specific
parsers must sit behind the common parser interface described in
`docs/project-desc.md`.

React Flow and dagre may be used for graph rendering and layout.

Tailwind CSS v4, shadcn/ui, Biome, pnpm.

All project files use kebab-case naming where the platform permits it.
Follow framework-required filenames exactly when Next.js or another tool
requires a conventional name.

Next.js 16 changed a lot. If you're uncertain about an API, read the docs
inside the installed package rather than going from memory.

Use the installed Matt Pocock skills throughout the project for TypeScript,
architecture, APIs, and implementation decisions where they are relevant.
Do not bypass those skills when they apply.

## How we work

Spec driven. Nothing gets built without a spec.

- `docs/project-desc.md` — the product definition and reasoning behind it.
  Read the relevant part before making a product or architecture decision.
- `docs/specs/phase-NN.md` — one per phase, written immediately before the
  phase starts. It defines behaviour and an acceptance check, never filenames.
- This file — the engineering rules that are always true.

**Starting a phase.** Read this file and that phase's spec. Read the relevant
part of `docs/project-desc.md`. Build what the spec asks and stop.

**Dropped into a fresh context and don't know where we are?** Look at which
files exist in `docs/specs/`, then the git log. The last commit is the last
phase that passed. Tell me what you've worked out before building on it.

**The acceptance check is mine to run, not yours.** It is a list of things I
do by hand in a browser. Do not automate it, install a browser driver, or add a
test runner solely for the acceptance check. Build so those things are true,
then tell me it is ready.

**What you check before saying a phase is done:** types, Biome, build, and any
fast deterministic checks relevant to the phase. Anything verifiable from a
terminal in a few seconds is yours. Anything requiring a browser is mine.

**When a phase comes out wrong, reset rather than patch indefinitely.** Prompting
on top of wrong code produces code nobody understands. If a spec is ambiguous,
say so before building.

## How to talk to me

Short. If a sentence is not telling me something I need, cut it.

**Ask with an answer attached.** Ask one specific question and say which way
you would go and why. Example: "A or B; I'd take B because it keeps the parser
standalone." Never pick a direction silently. Never build both.

**When you need something only I can give you** — a key, token, OAuth setting,
environment value, or product decision — say exactly what is needed, exactly
where it goes, and stop.

**No walls of text.** Do not summarise every file touched or restate the plan.
When a phase is done, say what it does and what the acceptance check should
show, in a few lines.

**Plain English.** If something can be said in one sentence, say it in one
sentence.

**Say when something didn't work.** A failure worked around quietly costs me
an hour later.

## How the code is laid out

Choose the physical file structure based on the current phase. These rules are
about boundaries and behaviour.

- **The parsing code cannot import Next.js, React, Better Auth, Neon, Drizzle, or
  other application infrastructure.** The parser takes repository data in and
  returns normalized data out. It must be runnable from a plain script.
- **No framework checks inside language parsers.** Framework-specific knowledge
  belongs in a framework adapter.
- **Graph calculations are pure functions** over nodes and edges.
- **Database access happens in server-side code**, not inside components.
- **Authentication establishes Nexus identity. GitHub authorization establishes
  repository access.** Do not collapse these into one credential or concept.
- **Organization membership determines Nexus data visibility.** Enforce that
  authorization at the data layer.
- **Who may read a row is decided by a database policy, never by application
  code.** If a query needs a defensive `where` clause to avoid returning rows
  from another organization, the policy is wrong.
- **AI construction has one boundary.** Application code must not instantiate
  provider clients ad hoc.

## Authentication and organizations

Better Auth is the source of truth for Nexus authentication.

Email magic-link authentication establishes the Nexus user identity.

The Better Auth organization plugin establishes organization membership and
default organization roles.

GitHub OAuth is not the Nexus sign-in mechanism. It is a separate connected
credential used to access repositories.

The separation is intentional:

1. User signs into Nexus with email.
2. User connects GitHub.
3. GitHub authorization determines which repositories can be accessed.
4. The user creates or selects a Nexus organization.
5. Graphs and analyses belong to an organization.
6. Organization membership determines who can see those graphs and analyses.

Do not introduce Clerk.

Do not implement organization membership with ad-hoc application tables if
the Better Auth organization plugin already provides the required primitive.
Extend it only where the product actually needs additional domain data.

Do not assume GitHub access from a repository URL. Repository access must be
checked against the current GitHub authorization.

GitHub credentials are security-sensitive. They must be encrypted at rest,
must not be exposed to the client unnecessarily, and must never appear in logs,
AI prompts, graph data, or normal application responses.

Disconnecting GitHub must invalidate the connection and prevent further
repository access until the user reconnects.

## Database and authorization

Neon Postgres is the database, accessed through Drizzle ORM.

Row-level security is mandatory for every application table.

A table with RLS enabled but no policy is preferable to a table where RLS was
forgotten: the former fails visibly, the latter can leak data. RLS must be
enabled for every application table rather than remembered table by table.

Rows belonging to organizations must be protected by database policies based
on the authenticated user's organization membership.

Do not rely on:

- hidden UI controls,
- route guards alone,
- server-side `where` clauses alone,
- client-side organization IDs,
- user-supplied organization IDs.

Those can improve UX or reduce unnecessary work, but they are not the
authorization boundary.

Database access belongs in server-side code through Drizzle ORM.

Name selected columns explicitly. Limit list reads. Do not read from the
database in loops. Subscribe to changes where realtime is actually needed
rather than polling.

## GitHub access

GitHub authorization is external to Nexus authentication.

The supported lifecycle is:

1. Connect GitHub.
2. Receive authorization for the permitted repository scope.
3. List repositories available to the connected account.
4. Select a repository.
5. Fetch and parse the selected repository.
6. Record the analyzed revision.
7. Report access failures explicitly if authorization is revoked or becomes
   insufficient.

The repository picker should feel like importing a repository into a modern
deployment tool:

- search,
- repository name,
- owner/organization,
- public/private indication,
- recently used repositories,
- clear connection state.

Do not make pasting a repository URL the primary workflow.

Never silently treat an inaccessible repository as public.

## Parsing

The parser is the foundation of the product.

**Every box and every line comes from really parsing the code.**

The AI may explain and label. It may never decide that two files are
connected.

Never invent an edge to make the graph look complete.

An edge exists only when a language parser has recovered a real relationship
from source code and resolved it to a real target.

Unresolved relationships are reported with a reason.

A skipped file is reported and counted.

A route that cannot be recovered reliably is absent rather than approximate.

The parser must remain language-agnostic at the product level.

The architecture is:

```text
Repository
    ↓
Language detection
    ↓
Language-specific parser
    ↓
Normalized code representation
    ↓
Dependency / relationship graph
    ↓
Graph UI
```

Every language parser produces the common intermediate representation.

The exact schema can evolve, but the abstraction should preserve concepts
equivalent to normalized nodes and edges.

For example:

```ts
interface LanguageParser {
  detect(files: RepositoryFile[]): boolean
  parse(files: RepositoryFile[]): Promise<ParseResult>
}
```

A normalized relationship may include:

```ts
type CodeEdge = {
  source: string
  target: string
  type:
    | "imports"
    | "exports"
    | "calls"
    | "extends"
    | "implements"
    | "references"
}
```

Do not assume this list is exhaustive. Add relationship types only when a
parser can recover them reliably.

"Supports a language" means the parser can reliably recover the relationships
Nexus claims to display for that language.

Partial support must be visible.

If syntax, dependencies, language constructs, or framework features cannot be
resolved confidently, coverage reports that fact instead of guessing.

A repository with partial coverage must never be presented as fully
understood.

## Framework adapters

Framework-specific knowledge is an adapter, not a branch inside a language
parser.

Knowing that a file in a particular position is a page, route, layout, loader,
or other framework construct belongs in a framework adapter.

An `if (framework === "...")` inside language parsing code is a sign that the
boundary has failed.

Framework adapters may enrich the normalized representation, but they cannot
replace the language parser's responsibility for extracting actual source-code
relationships.

## Graph calculations

Graph calculations are pure functions over normalized nodes and edges.

No data fetching inside them.

No framework dependency inside them.

No database dependency inside them.

No AI dependency inside them.

Blast radius, dependency chains, fan-in, fan-out, and similar structural
calculations must operate deterministically on data already available to the
analysis or browser layer.

If an answer is arithmetic over data already held by the application, it is
immediate. Do not add a network request or spinner to it.

The AI must never walk the graph when deterministic graph traversal can answer
the question.

## AI

The AI layer is deliberately provider-agnostic.

Do not make the product depend directly on the OpenAI SDK. Gemini may be the
provider instead, and the provider can change again.

Use one application-level AI boundary. The rest of the application should ask
for capabilities, not instantiate a vendor client.

For example, the application may depend on concepts equivalent to:

```ts
interface AiClient {
  explainFile(input: ExplainFileInput): Promise<AiResult>
  answerRepositoryQuestion(
    input: RepositoryQuestionInput
  ): Promise<AiResult>
  labelFile(input: LabelFileInput): Promise<AiResult>
}
```

The exact interface is not fixed. Keep it as small as the current phase
requires.

There must be one obvious place where the provider client/adapter is
constructed. Do not create provider clients throughout the application.

Every AI call is traced and cached.

The cache read belongs inside the traced operation so a cache hit appears as a
recorded run containing no model call.

The AI receives factual repository context retrieved from parsed repository
data.

The AI may:

- explain files,
- explain relationships,
- answer repository questions,
- label files that convention could not identify,
- summarize retrieved code.

The AI may not:

- create graph edges,
- invent files,
- invent dependencies,
- invent repository facts,
- perform deterministic graph traversal itself,
- claim unsupported repository facts.

For repository chat, the system should expose what was looked up before the
answer. An answer with no factual lookups behind it is a failure mode, not a
successful shortcut.

Any check that verifies an AI response never names a file absent from the
repository must be based on repository facts, not another AI judgment.

## UI

The UI is a dense developer tool, not a marketing dashboard.

Use small type, tight spacing, and monospace for file paths and code-oriented
identifiers.

Colour means something or it is not there.

Direction and relationship kind may use colour because they encode structural
meaning. Surfaces, borders, and ordinary text remain largely greyscale.

The palette starts with:

- blue for selection and interaction,
- green for incoming relationships,
- amber for outgoing relationships,
- neutral surfaces and text.

Light and dark modes are real palettes. The user's choice wins.

Support:

- system,
- light,
- dark.

The choice survives reload.

Nothing moves unless the user caused it. No ambient animation, pulsing, drifting,
or decorative motion.

The application shell is established once. Later phases add to its panels
rather than repeatedly rearranging the core layout. Specific placement belongs
in the relevant phase spec.

Use shadcn/ui for interface primitives where appropriate. Do not let a component
library dictate the product's information density or visual hierarchy.

Use the installed frontend design skill for UI work when it activates, but the
Nexus UI rules in `docs/project-desc.md` and this file take precedence.

## Conventions

Strict TypeScript. No `any`.

If a type is genuinely awkward, explain the problem instead of solving it with
a blind cast.

Use Biome for formatting and linting. Do not introduce another formatter or
linter without asking.

Use shadcn/ui rather than inventing a duplicate primitive when an appropriate
component already exists.

Use the installed Matt Pocock skills throughout the project where relevant.
Prefer their TypeScript and architecture guidance over ad-hoc patterns.

Comment decisions, constraints, and non-obvious tradeoffs — not syntax.

Keep functions and abstractions obvious. One clear way to do something beats a
configurable framework for doing everything.

Use kebab-case for project files and directories unless a framework, generated
tooling convention, or external contract requires another filename.

## Naming and files

All new project-owned files should use kebab-case:

```text
project-desc.md
phase-01.md
repository-picker.tsx
github-client.ts
graph-calculations.ts
```

Do not create arbitrary variants such as:

```text
projectDesc.md
ProjectDesc.md
repositoryPicker.tsx
GraphCalculations.ts
```

Framework-required files are exceptions. Do not rename a required Next.js,
Better Auth, Neon, or tool convention merely to satisfy this rule.

Do not create speculative directories or files for future phases.

## Things not to do

Breaking one of these is worse than not finishing.

- **Never decide that two files are connected.** An edge exists because the
  parser resolved a real relationship to a real file.
- **Never invent something to fill a gap.** Skipped file? Report and count it.
  Unrecoverable route? Show none. Unsupported syntax? Report it.
- **Do not install a package without asking.** Name it, say what it is for, and
  wait.
- **Do not add a second auth system.** Nexus uses Better Auth. Do not add Clerk.
- **Do not couple the product to one AI vendor.** Keep OpenAI/Gemini/etc. behind
  the AI boundary.
- **Do not build ahead of the current phase.** No scaffolding for future work.
- **Do not grade the code.** No quality scores, severity badges, issue counts,
  or review panels. Nexus explains a codebase; it does not review one.
- **Do not let the AI invent graph structure.**
- **Do not let the AI traverse the graph when deterministic graph calculations
  can do it.**
- **Do not leave the build broken.** Tell me about a failure rather than
  quietly working around it.
- **Do not weaken a check to make it pass.** A check that cannot run has to fail
  loudly.
- **Do not read from the database in a loop.** Name columns, limit list reads,
  and use SSE for live application progress where needed rather than polling.
- **Do not expose GitHub credentials to the client, logs, AI prompts, or graph
  data unnecessarily.**
- **Do not pretend unsupported languages are fully supported.** Coverage must
  remain explicit.
- **Do not introduce queues, containers, or workers prematurely.** The initial
  product is one deployable application. Establish honest repository-size
  limits before adding distributed infrastructure.
- **Do not become a code reviewer.** Security findings, style opinions, quality
  scores, and issue triage are separate products.

## Realtime and progress

Nexus needs live progress while a repository is being parsed because parsing a
real repository is not instant.

Use server-sent events (SSE) for server-to-client progress updates where a live
connection is required. Keep the event stream application-level and scoped to
the authenticated user's authorized organization and graph.

Do not introduce a dedicated realtime database service, message broker, Redis,
queue, worker system, or similar infrastructure merely to provide progress
updates. Add infrastructure only when an actual product requirement justifies
it.

For data that is already available locally, prefer immediate deterministic
calculations over a network request.

## Before declaring a phase done

Run:

1. Type checking.
2. Biome checks.
3. Production build.
4. Any deterministic scripts or parser checks required by the phase.

Do not claim browser acceptance. That is mine to perform.

A phase is ready when the terminal checks pass and the implementation satisfies
the written phase spec.

## Product invariants

These are non-negotiable because they define what Nexus is:

- Every graph edge comes from parsed source code.
- Every graph node represents something recovered from the repository or an
  explicitly documented normalized concept.
- Unsupported or unresolved structure is visible rather than guessed.
- AI explains facts; it does not manufacture structure.
- Deterministic graph calculations remain deterministic.
- Nexus authentication and GitHub repository authorization remain separate.
- Better Auth organization membership is the organization boundary.
- Database authorization is enforced by row-level security.
- GitHub credentials remain protected.
- AI provider choice remains replaceable.
- Parser implementations remain independent of the application framework.
- Adding a language must not require rewriting the graph calculation layer.
- Coverage is explicit.
- The product explains a codebase; it does not score or review it.
