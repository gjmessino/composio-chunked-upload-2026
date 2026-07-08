# Composio tool dependency graph — GitHub & Google Super

## TL;DR

Open [`graph.html`](./graph.html) in a browser (double-click it, no server needed — the
data is inlined). It's an interactive node/edge graph: search a tool, click a node to see
its required/optional params, and see which tools are its precursors ("needs first") and
which tools it feeds into ("unlocks").

Run `npm install && npm run build` to regenerate everything from scratch.

## What this models

An edge `A → B` means: *tool A is a plausible precursor for tool B* — either because A
produces an identifier/value that B requires as input, or because a natural-language
argument to B (a name, not an ID) needs to be resolved to a canonical value first (the
"send email to a name → look up the contact" pattern from the task brief).

Each edge carries:
- `param` — which input parameter of the consumer this precursor satisfies
- `confidence` — `high` / `medium` / `low`
- `source` — how the edge was found (see below)
- `reason` — a plain-English justification

## Data source

`src/fetch-tools.mjs` calls `composio.tools.getRawComposioTools(...)` via the SDK, as the
task suggested, and writes the raw response to `raw/{toolkit}_tools_raw.json`.
`src/normalize-sdk-tools.mjs` then flattens each tool's JSON-Schema-shaped
`inputParameters`/`outputParameters` (`{type, properties, required: [...]}`) into a flatter
per-parameter dict (`{name: {type, description, required}}`) that the rest of the pipeline
consumes — `github_tools.json` / `googlesuper_tools.json`. `npm run build` runs this path
end to end (893 GitHub tools, 467 Google Super tools).

The first key provided for this task was rejected by the live API (401 `Invalid API key`),
so a fallback was built first and is still included: `src/fetch-docs.mjs` pulls
`https://docs.composio.dev/toolkits/github` and `.../googlesuper` — public, unauthenticated
pages that happen to server-render the **entire tool catalog** (including full
`input_parameters`/`output_parameters` schemas) into the page's RSC payload.
`src/extract-tools.mjs` parses that embedded JSON out of the raw HTML into the same flat
shape. `npm run build-from-docs` runs this path instead, no API key required. Both paths
were diffed against each other once a working key arrived — same 893/467 tool counts, and
the resulting graph differs by only 2 edges (a couple of doc-text mentions present in the
SDK's copy but not yet published to the docs site at scrape time), confirming the
docs-scraping fallback was a faithful stand-in.

## Building the edges — two methods

**1. Mining explicit references from Composio's own text (`src/build-graph.mjs`)**

Google Super's parameter descriptions frequently spell out the precursor directly, e.g.
`GOOGLESUPER_REPLY_TO_THREAD`'s `thread_id` param says: *"Use GMAIL_LIST_THREADS or
GMAIL_FETCH_EMAILS to retrieve valid thread IDs."* These references still use the
toolkit's pre-merge names (`GMAIL_`, `GOOGLEDRIVE_`, `GOOGLESHEETS_`, `GOOGLECALENDAR_`,
etc.) since Google Super absorbed several previously-separate Composio toolkits. The
script regexes for `UPPER_SNAKE_CASE` tokens in every tool/param description, strips the
legacy prefix, re-adds `GOOGLESUPER_`, and checks it against the real slug list — this
gives ~196 high/medium-confidence edges essentially "for free," straight from Composio's
own documentation of its own tools. The same pass on GitHub descriptions yields far fewer
(21) because GitHub's tool descriptions describe REST semantics rather than pointing at
sibling tools by name — hence method 2.

**2. Curated rules over parameter names (`src/curate-github.mjs`, `src/curate-googlesuper.mjs`)**

For GitHub, ~40 recurring "entity ID" parameters (`issue_number`, `pull_number`, `sha`,
`branch`, `run_id`, `hook_id`, `secret_name`, `team_slug`, `release_id`, `comment_id`, …)
were mapped by hand to the `LIST_*`/`SEARCH_*`/`GET_*` tool that actually produces that
value, reading each candidate's real description to confirm the mapping (not guessed from
naming alone). Where the same parameter name means different things depending on scope
(e.g. `secret_name` on a repo vs. an org vs. an environment secret, `hook_id` on a repo vs.
org webhook), the rule branches on the consumer tool's own slug to pick the right
producer. This is the bulk of the graph: 1008 edges.

For Google Super, `curate-googlesuper.mjs` adds the pattern the task explicitly calls out
as underspecified in the tool descriptions themselves: if a user gives a *name* instead of
an email for `recipient_email`/`cc`/`bcc`/`attendee_email`/etc., `GOOGLESUPER_GET_CONTACTS`
is the precursor that resolves it. Composio's own docs don't say this anywhere (they just
describe the param as "an email address") — this is inferred, not mined, so it's tagged
`semantic_resolution` and kept at medium confidence.

## Known gaps / honest limitations

- `owner`/`repo` on GitHub tools (the two most common required params, ~445 tools each)
  are deliberately **not** wired to a producer tool. In practice they're either known
  up front (user says "in my `foo/bar` repo") or resolved via `GITHUB_SEARCH_REPOSITORIES`
  — but that would make search a precursor for nearly half the graph, which felt more like
  noise than signal, so it was left out rather than force a low-value edge onto every node.
- `unresolved_refs.json` lists the `UPPER_SNAKE_CASE` tokens the miner found but couldn't
  resolve to a real slug (mostly enum constants like `CATEGORY_UPDATES`, `FORMATTED_VALUE`,
  or two genuinely-typo'd tool names in Composio's own docs) — kept for transparency
  rather than silently discarded.
- Coverage is intentionally uneven: dense on the ~40 parameter names that recur constantly
  (issues, PRs, secrets, webhooks, workflows, releases, threads, files, sheets, contacts…)
  and thin on long-tail one-off tools. 756 of the 1360 tools have at least one edge.

## Files

| File | What it is |
|---|---|
| `raw/*_tools_raw.json` | unmodified SDK response from `getRawComposioTools` |
| `docs/*_toolkit_page.html` | raw docs pages used by the fallback extraction path |
| `github_tools.json`, `googlesuper_tools.json` | normalized tool catalog (slug, description, flat input/output schema) — output of either data-source path |
| `mined_edges.json` | edges from method 1 (explicit doc-text references) |
| `curated_github_edges.json`, `curated_googlesuper_edges.json` | edges from method 2 (curated param rules + semantic resolution) |
| `unresolved_refs.json` | tokens the miner couldn't resolve, for transparency |
| `graph.json` | final merged nodes + edges + stats |
| `graph.html` | the interactive visualization (self-contained) |
