# Nexus

The product decisions behind this build, written before it starts, with the reasoning attached. The reasoning matters more than the decision — you'll hit forks this doesn't cover, and you'll need to know which way the thinking was leaning.

Read the part you need before you build something. Don't read it all at once, and don't paste it into a prompt.

## What it is

Nexus turns a supported GitHub codebase into an interactive map of its actual structure.

Sign in with email using a magic link, connect a GitHub account, choose a repository, and get a map of it. Folders become navigable groups, files become nodes, and real relationships between them become edges.

Open a folder into its files. Select a file and see what it imports, what imports it, and what breaks two levels out if it changes. Ask for an explanation and get a paragraph written from that file and its real neighbours.

Then a panel where you can ask the repository a question and watch it look the answer up.

Nexus is language-agnostic at the product level. The parser is built around a common interface so different programming languages can be supported independently without changing the graph engine or application.

## The problem

Codebases now routinely contain code nobody on the team has read. An agent wrote it, someone confirmed it worked, it shipped. That is a reasonable way to work and it is not going to stop — but it accumulates structure nobody chose. Files that import through three layers of re-exports. A utility forty files depend on. A module nothing has referenced in months.

The questions that follow are structural. What is this file, what depends on it, what breaks if it changes. Answering them today means reading imports one file at a time and holding the results in working memory, which stops working at around thirty files and stops being attempted at all somewhere past a hundred.

Nexus makes that structure visible without asking an AI model to invent it.

## Who it's for

A developer opening a codebase they didn't write — most often one they built themselves with an agent, across many sessions, and can no longer hold in their head.

They can read code fine. What they can't see is its shape.

And increasingly that isn't one person's problem. A team of four all shipping this way inherits four sets of code nobody read, and the person who joins next inherits all of it at once. So an analysis belongs to an organization rather than to whoever happened to run it. A repository is mapped once and everyone after that starts from the map.

This description is the test every feature gets measured against. When something tempting comes up — a quality score, a rating, a review panel — the question is whether the person described above wants it. Usually they don't. They want to know what depends on this file.

## Why this hasn't stuck before

Codebase visualisation is not a new idea and the track record is bad. CodeSee built close to this with real funding behind it and is gone. Sourcetrail ran for years as a desktop tool and was archived. Any plan here has to account for that rather than assume it away.

The most likely explanation: both were selling a map of a codebase the user had already read. A map you could have drawn yourself is a nice-to-have, and nice-to-haves lose. What changed is that people now ship code they have never read, which turns the same artifact into something needed on a specific afternoon for a specific reason.

The current generation of AI tools mostly work the other way round — hand the repository to a model and let it describe the structure. That is much faster to build, which is why there are many of them, and it is the approach this product is deliberately not taking. The next section is why.

That's the bet. It is a bet, and it should be stated as one.

## The rule everything rests on

**Every box and every line comes from really parsing the code.**

The AI may explain and label. It may never decide that two files are connected, and it may never walk the graph when arithmetic can do it instead.

The reason, bluntly: a dependency graph that is ninety percent right is worse than no graph, because there's no way to tell which ten percent is wrong. The moment a model is allowed to guess at an edge, every edge becomes a maybe, and the product is a nice-looking picture.

This rule applies across every supported language.

The parser for each language is responsible for extracting relationships from actual source code. Those relationships are converted into a common internal representation. The graph engine operates only on that normalized representation.

Pressure to break this will come from two directions. "AI code analysis" pulls toward letting the model reason about structure. Anything calling itself a code tool drifts toward becoming a code reviewer. Both are refused.

## Authentication and GitHub access

### Email authentication

Sign-in is handled through email magic links.

The email identity establishes the Nexus user account. It does not provide repository access.

There is no requirement for users to sign in with GitHub.

### Connect GitHub

After signing in, the user connects their GitHub account through GitHub OAuth.

GitHub access is separate from Nexus authentication.

The user controls which repositories Nexus can access through the GitHub authorization flow. Nexus should support the GitHub permission model needed for:

- Public repositories
- Selected repositories
- All repositories available to the user, including private repositories

Nexus must never assume that a repository is public simply because it is being analyzed.

Repository access must be checked against the user's current GitHub authorization rather than inferred from repository URLs.

### Repository access lifecycle

GitHub authorization should be treated as an external credential with an explicit lifecycle:

1. User connects GitHub.
2. Nexus receives the authorization required to access permitted repositories.
3. Nexus lists repositories available to that user.
4. User chooses a repository for a graph.
5. Nexus fetches the repository and parses it.
6. If GitHub authorization is revoked or becomes insufficient, Nexus must report the access problem rather than silently failing.

The exact mechanism for storing, refreshing, encrypting, and revoking GitHub credentials must be treated as a security concern and implemented explicitly. Credentials must never be exposed to the client unnecessarily.

## Scope

- Email magic-link authentication for Nexus identity.
- GitHub OAuth as a separate connection for repository access.
- Repository access based on the user's GitHub authorization and selected scope.
- A repository picker similar to the repository connection flow used by tools such as Vercel.
- Graphs. A user can create a graph and connect it to a repository.
- Organizations. Every analysis belongs to one, people are invited into one, and what you can see follows from which one you're in. Default roles only.
- Parsing codebases across multiple programming languages through pluggable language parsers.
- A normalized intermediate representation shared by all language parsers.
- Imports, re-exports, dynamic imports, requires, references, calls, inheritance and other relationships where the language parser can recover them reliably.
- A framework adapter layer so framework-specific knowledge does not leak into language parsing.
- Fan-in and fan-out per file.
- The map: folders folding into boxes, expanding into files, selection, highlighting, and a detail panel.
- Blast radius and dependency chain, as arithmetic over the edge list.
- Explaining a file, with a cache that isn't optional.
- Labelling files that convention couldn't identify.
- Coverage reporting — how much of the repository was parsed, which languages were detected, and why anything was skipped.
- Live progress during parsing, because parsing a real repository is not instant.
- A chat panel answered by an agent that looks facts up rather than recalling them.
- A check on whether the AI ever names a file that doesn't exist.
- Clear reporting when a language, syntax construct, dependency, or framework feature cannot be resolved confidently.

## Language support

Nexus must not be architecturally tied to TypeScript or JavaScript.

The product should support any programming language for which a reliable parser can be implemented.

The important abstraction is not "one parser that understands every language."

It is:

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

Each language parser converts source code into the same internal model.

For example, different source languages may express imports differently:

```ts
import { foo } from "./foo"
```

```py
from foo import bar
```

```java
import com.example.foo.Bar;
```

The graph engine does not need to understand those syntaxes. It receives normalized relationships from the appropriate parser.

### Parser interface

The exact implementation may change, but the architecture should preserve a contract equivalent to:

```ts
interface LanguageParser {
  detect(files: RepositoryFile[]): boolean

  parse(files: RepositoryFile[]): Promise<ParseResult>
}
```

The parser should produce normalized nodes and edges.

A normalized model may look roughly like:

```ts
type CodeNode = {
  id: string
  type: "file" | "module" | "class" | "function"
  path: string
  name: string
  language: string
}

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

This is illustrative rather than a fixed schema. The implementation should evolve as language support expands.

### Language coverage

"Supports a language" means the parser can reliably recover the relationships that Nexus claims to display for that language.

Partial support must be visible.

If a repository contains unsupported files or syntax that cannot be reliably parsed, Nexus reports that in coverage rather than guessing.

A repository with 80% parse coverage is not presented as if it were 100% understood.

## Graphs and repositories

A graph is the user's persistent analysis of a repository.

### Creating a graph

The basic flow is:

```text
Dashboard
    ↓
New Graph
    ↓
Name Graph
    ↓
Connect Repository
    ↓
Repository Picker
    ↓
Select GitHub Repository
    ↓
Parse Repository
    ↓
Graph Ready
```

When creating a graph, the user should see a searchable list of repositories available through their connected GitHub account.

The experience should feel similar to importing a repository into Vercel:

```text
Connect a repository

Search repositories...

mycampusadmin
snipai
portfolio
some-private-project

[Import]
```

The exact interface can evolve, but repository selection should be obvious, fast, and familiar.

### Repository relationship

A graph is connected to a repository and represents a particular analysis of that repository.

The implementation should preserve enough repository metadata to identify:

- GitHub repository
- Owner / organization
- Repository name
- Default branch
- Commit or revision analyzed
- Analysis status
- Parser coverage
- Last successful analysis

A future version may support re-analysis when the repository changes, but the initial implementation should make the analyzed revision explicit.

## Out of scope, and why

Each of these is something a model will propose, because each of them sounds like an improvement. The reason matters more than the refusal.

- **Scores, grades, severity badges, "three issues found".** This product explains a codebase; it does not review one. Once there's a number, people argue with the number instead of reading the map.
- **Letting the agent traverse the graph itself.** It picks a starting point and a direction; the same arithmetic that draws the canvas does the walk. If the model does the walking, invented structure comes straight back in.
- **AI-generated graph edges.** The AI may explain existing facts, but it cannot invent relationships that the parser did not establish.
- **Approximate routes.** If a route's method and full path can't both be recovered from the syntax or framework adapter, show nothing. A wrong route is the same failure as an invented edge.
- **Pretending unsupported languages are fully supported.** Coverage must remain explicit. A parser that cannot reliably understand part of a repository must say so.
- **Queues, containers, workers.** One deployable application initially. If a repository is too large to parse in a request, that is a limit to state honestly rather than an architecture to build around prematurely.
- **Becoming a code reviewer.** Security findings, style opinions, quality scores, and issue triage are separate products. They are not the core problem Nexus is solving.

## How it works

No folder names here — the structure is yours to choose. These are constraints on behaviour.

### The parser is standalone

The parser takes a repository directory and returns files, normalized nodes, edges and a coverage report.

It does not know the web framework exists, does not know the database exists, and must run from a plain script with nothing else running.

If it can't run standalone, the layering is wrong.

### Language parsers are adapters

Each programming language gets its own parser implementation.

The parser knows the syntax and semantics required to recover relationships from that language. It produces the common intermediate representation.

Adding support for another language must not require rewriting the graph engine.

### Framework knowledge is an adapter, not a branch

Knowing that a file in a particular position is a page is framework knowledge and belongs in a framework adapter.

An `if (framework === ...)` inside language parsing code means the interface built to prevent exactly that has failed.

Framework adapters may enrich the normalized representation, but they cannot replace the language parser's responsibility for extracting actual code relationships.

### Graph calculations are pure functions

Graph calculations are pure functions over a list of nodes and a list of edges.

No data fetching inside them, nothing to mock in order to check them.

Blast radius, dependency chains, fan-in, fan-out and similar structural calculations should operate on data the browser or analysis layer already holds.

### Identity and authorization

Authentication and GitHub authorization are separate concerns.

The user's Nexus identity comes from email magic-link authentication.

Repository access comes from the user's connected GitHub account.

Organizations determine which Nexus graphs and analyses the user can see.

Authorization must be enforced at the data layer rather than relying on interface checks.

### Database

The database is Postgres on Supabase.

Rows belong to an organization, and authorization is enforced through database policies.

Row-level security is on by default, not per table. An agent is writing these migrations, and the two failure modes are not symmetrical. A table with security enabled and no policy yet returns nothing, which is visible. A table where someone forgot to enable it returns everything, silently. The second one must be impossible rather than unlikely.

### GitHub credentials

GitHub credentials are security-sensitive.

Tokens must be encrypted at rest and must never be unnecessarily exposed to the browser, logs, AI prompts, graph data, or normal application responses.

The system should request the minimum GitHub access required for the selected repository-access mode.

Disconnecting GitHub must invalidate the connection and prevent further repository access until the user reconnects.

### AI

Every AI call is traced and cached, with the cache read inside the traced function, so a cache hit appears as a recorded run containing no model call.

That's what makes the cache verifiable rather than assumed.

The AI receives factual repository context retrieved from the parsed repository.

The AI can:

- Explain files
- Explain relationships
- Answer repository questions
- Label files that convention could not identify
- Summarize retrieved code

The AI cannot:

- Create graph edges
- Invent files
- Invent dependencies
- Walk the graph instead of using deterministic graph calculations

## The interface

A dense developer tool, not a dashboard. Small type, tight spacing, monospace for anything that is a file path. Someone using this is scanning a lot of names at once, and the layout should assume that rather than fighting it.

### Authentication

The initial authentication experience should be simple:

```text
Welcome to Nexus

Enter your email

[email@example.com]

[Continue]

We'll send you a magic link.
```

After authentication, the primary next step is connecting GitHub.

```text
Connect GitHub

Nexus needs access to your GitHub repositories
to analyze the codebases you choose.

[Connect GitHub]
```

Once GitHub is connected, the user reaches the dashboard.

### Repository picker

Repository selection should be familiar and efficient.

It should support:

- Search
- Repository name
- Owner / organization
- Public/private indication
- Recently used repositories
- Clear connection state

Do not make the user paste repository URLs as the primary workflow.

### Graph

Colour carries meaning or it isn't there. Two things earn colour. Direction: what flows into a file and what flows out of it are different colours, because that distinction is the entire reason to look at an edge. And kind: a file gets a colour for what sort of thing it is, with enough hues to separate a handful of categories and no more. Everything else — surfaces, borders, body text — is greyscale.

The palette starts at four decisions. One blue as the accent, for selection and anything interactive. Green for incoming, amber for outgoing. Surfaces near white in light mode, near black in dark. Everything else derives from those — pick sensible values, keep them consistent, and don't ask about each one.

Dark and light, and the user's choice wins. Three states: follow the system, force light, force dark. The choice survives a reload. Both palettes are real palettes rather than one of them inverted.

Nothing moves on its own. Motion happens because something was clicked. No ambient animation, no pulsing, no drifting graph.

Anything derivable from the edge list is instant. If an answer is arithmetic over data the browser already holds, it appears immediately — no spinner, no network request. Putting a spinner in front of a calculation teaches people to distrust the fast paths, which are most of them.

The shell gets built once. Where the panels sit, and what each one is for, is settled in an early phase. Later phases add to those panels; they don't rearrange them. Specific placement lives in the phase specs.

## Where this is likely to go wrong

### Folding folders into readable boxes

A large repository has to collapse into something legible, and the threshold has to be solved for rather than guessed. Expect at least one wrong answer here before a right one.

### Import and dependency resolution

Path aliases, index files, re-exports and language-specific resolution rules are difficult.

The risk isn't failure — it's silent failure.

A repository built on barrel files can lose most of its edges and still render a clean, confident, wrong picture.

This is the one place loud reporting wins over a tidy interface.

### Multi-language parsing

Supporting many languages superficially is easy.

Supporting them reliably is not.

Each language parser needs tests against real code patterns and must report what it could not understand.

A language should not be considered supported merely because its files can be detected.

### GitHub access

Repository permissions, private repositories, organizations, revoked access, selected-repository permissions, token expiration and rate limits all introduce failure states.

The interface must explain access failures rather than turning them into generic "analysis failed" messages.

### Repository size

Some repositories will be too large to process quickly.

The initial product should establish honest limits and communicate progress and coverage clearly rather than immediately introducing a distributed processing architecture.

### Measuring whether the AI output is any good

Reading a few outputs and judging them fine is not measurement.

The requirement is that answer quality becomes a number; the method for producing that number is genuinely open at the time of writing, and it is the least settled part of this plan.

## What has to be true before this ships

Not "it looks right". Specifics:

- A user can create an account using an email magic link.
- A user can connect and disconnect GitHub independently of Nexus authentication.
- The repository picker only shows repositories the connected GitHub account is authorized to access.
- A private repository can be analyzed when the user's GitHub authorization permits it.
- A graph can be created and connected to a selected GitHub repository.
- The analyzed repository revision is recorded.
- Pointed at a known repository, five files checked by hand match what the parser claims they import or otherwise relate to.
- Coverage states what was skipped and why, and the number is believable.
- At least the initially supported languages have parser tests covering their important relationship syntax.
- Adding a language does not require changing the graph calculation layer.
- Signed in from a different organization, the first organization's rows do not come back from the query at all.
- Every table in the schema has row-level security enabled. Not most of them.
- GitHub credentials are not exposed to the client, logs, or AI context unnecessarily.
- Disconnecting GitHub prevents further repository access through that connection.
- An explanation never names a file absent from the repository, and that is demonstrated by a check rather than asserted.
- The chat panel shows what it looked up before it answers. An answer arriving with no lookups behind it is the exact failure this product exists to prevent.
