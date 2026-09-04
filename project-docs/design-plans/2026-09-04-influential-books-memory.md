# Influential Books Memory Design

## Summary

A standalone Python command-line tool, outside the Mnemion worker, pulls the owner's reading library out of Hardcover over its GraphQL API, then scans the owner's own writing (the lu.is blog and the Bluesky archive in content-archive) for sentences that mention those books or their authors. Hits are scored, grouped by author, and written to a review file the owner reads before anything reaches Mnemion. Bare surnames and shorthand such as "MSCD" count as evidence, because the most influential authors are cited without links.

Two new Mnemion patterns, `authors` and `books`, hold the result. Loading is an interview, not a batch: an agent shows the owner the evidence for one candidate at a time, asks what they took from it, and writes that answer into the entry in the owner's own words. A separate refresh step keeps rating, status, and read dates in sync with Hardcover and never touches the influence or evidence text. Wider sources (Mastodon, Twitter) are added only after the owner judges the first blog-plus-Bluesky pass useful.

## Definition of Done

- The owner's reading library lives in Hardcover (the StoryGraph import is already complete; this design does not repeat it).
- A Python CLI, standard library only, can:
  - pull the owner's whole Hardcover library over the GraphQL API into a local JSON snapshot;
  - mine the owner's blog (`lu.is`) and social archive (`content-archive`) for mentions of library books and their authors, aggregated by author, with an aliases file for shorthand such as "MSCD" or "Lessig", and write a scored candidate review file with evidence sentences and links;
  - refresh rating, status, and read dates on existing Mnemion `books` entries from Hardcover without touching `influence` or `evidence`.
- Mnemion has two user patterns, `authors` and `books`, linked by `author_of`, with `hardcover_id` as the exclusive facet on both.
- An interview protocol exists that an agent follows to load approved candidates one at a time: show the evidence, ask the owner what they took from the book or author, write one entry with the owner's own words, mark the candidate done. Skips are recorded, never re-asked. Any session can resume from the candidate file.
- The first real run of the candidate miner uses blog + Bluesky only, and the owner has reviewed its output before Mastodon or Twitter parsers are built.

## Glossary

- **Hardcover**: A book-tracking web service that hosts the owner's reading library and exposes it over a GraphQL API.
- **StoryGraph**: The book-tracking service the owner migrated from. Already imported into Hardcover; this design does not touch it.
- **GraphQL**: An API style where the client names exactly the fields it wants in one structured query.
- **user_book**: Hardcover's record joining a user's library entry to a book (status, rating, review, read dates).
- **Pattern**: A user-defined entry type in Mnemion, analogous to a table. Here, `authors` and `books`.
- **Facet**: A named field on a Mnemion entry, such as `title` or `hardcover_id`.
- **Memory policy**: Per-pattern Mnemion configuration for decay (`half_life_days`) and duplicate detection (`conflict_check`, `exclusive_facets`).
- **Exclusive facet**: A facet declared in memory policy as a natural dedupe key; Mnemion advises on a likely duplicate when its value repeats.
- **`_links`**: Mnemion's table of labeled, directed relationships between entries. Used here for `author_of` from a book to its author.
- **`propose_change` / `apply_change`**: Mnemion's two-step schema-evolution tools; the first previews, the second commits.
- **`create_pattern` / `set_memory_policy`**: The two change types this design applies through those tools.
- **Ingress (`POST /i/{path}`)**: Mnemion's untrusted, create-only HTTP endpoint for pushing data in. Not used here because loading is an interview.
- **Doctrine text**: The free-text purpose statement on a Mnemion pattern that agents read to learn what it is for.
- **Corpus**: The text the miner searches: blog posts and the Bluesky archive at first.
- **`aliases.toml`**: A hand-maintained file mapping shorthand ("MSCD", "Lessig") to a canonical book or author.
- **Candidate**: A book or author the miner surfaced, tracked as `pending`, `done`, or `skipped` in `candidates.json`.
- **content-archive**: The separate repository holding the owner's exported social archives.
- **JSONL**: One JSON object per line; the format of the normalized Bluesky archive.
- **Frontmatter**: The YAML metadata block at the top of a blog post's Markdown file.
- **Bearer token**: An API credential sent in an HTTP header; used for both Hardcover and Mnemion.
- **Version-checked update**: A Mnemion update that carries the last-seen `version` and is rejected if the entry changed since.

## Architecture

Three pieces, each with one job.

**1. `book-tools/` — a standalone Python CLI.** Not part of the worker; it lives in its own directory beside the Mnemion repo (final location decided at implementation, candidate: `Self/book-tools/`). Standard library only (`csv`, `json`, `urllib`, `re`, `argparse`). Package path is `src/book_tools/` (src layout, as in `content-archive`); `hardcover_tools/` in the phase lists below means that package. Config is environment variables: `HARDCOVER_TOKEN`, and a bearer token for the Mnemion instance (`*` scope, or a scoped token once one exists) for `refresh`. Subcommands:

| Subcommand | Reads | Writes |
|---|---|---|
| `pull` | Hardcover GraphQL `me { user_books … }`, paged 50 at a time, ≤1 req/s | `data/hardcover-library-YYYY-MM-DD.json` |
| `candidates` | library JSON, blog markdown under `lu.is/src/content/blog/`, normalized Bluesky JSONL under `content-archive/social-data/normalized/bluesky/`, `aliases.toml` | `candidates.md` (human review file) + `candidates.json` (machine cursor) |
| `refresh` | latest library JSON, Mnemion `books` entries via the `/api` query surface | Mnemion `books` updates (rating, status, first_read, last_read only) |

Rate limits (Hardcover free plan, read 2026-09-03 at docs.hardcover.app/api/getting-started): 60 requests/minute, 5,000/day. A 1,000-book library is ~20 `pull` requests. The CLI stops on HTTP 429 and on 401 (beta tokens may be reset; the fix is to regenerate, not retry).

**2. Two Mnemion patterns.** Created via `propose_change` / `apply_change` (`create_pattern`, then `set_memory_policy`).

`authors`:

| Facet | Type | Source |
|---|---|---|
| name | text, required | Hardcover contributor |
| hardcover_id | text | Hardcover author id; exclusive facet |
| influence | text | the owner's words, from the interview |
| evidence | text | Markdown list of blog/social URLs with dates |
| topics | text | comma-separated |

`books`:

| Facet | Type | Source |
|---|---|---|
| title | text, required | Hardcover |
| author | text | Hardcover, display only; the real link is `author_of` |
| hardcover_id | text | Hardcover user_book/book id; exclusive facet |
| hardcover_url | text | derived from slug |
| isbn | text | Hardcover edition |
| status | select: read, reading, want, dnf | Hardcover status_id 3/2/1/5 |
| rating | number | Hardcover |
| first_read | datetime | earliest finished read |
| last_read | datetime | latest finished read |
| influence | text, optional | the owner's words; only when a specific book matters beyond its author |
| evidence | text | Markdown list of URLs with dates |
| topics | text | comma-separated |

Memory policy on both: `exclusive_facets: ["hardcover_id"]`, `conflict_check: "annotate"`, no half-life (influential books must not decay). Link label `author_of` from book to author in `_links`. Doctrine text states that these patterns hold books and authors that shaped the owner's thinking, with evidence, and that the full reading log stays in Hardcover.

**3. The interview protocol.** A short document an agent follows in a session with the Mnemion MCP tools. Lives as a system doc (`mnemion-js/src/system-docs/influential-books.md`) so any client session can load it, or as a Claude Code skill in the CLI repo; implementation picks one. Steps:

1. Read `candidates.json`; take the highest-scored candidate not marked `done` or `skipped`.
2. Show the owner: author or book, Hardcover rating and read dates, and every evidence sentence with date and link.
3. Ask one open question: what did you take from this?
4. On an answer: `mutate` one `authors` or `books` entry with the answer as `influence`, the evidence list, and topics; then one `_links` row if it is a book. One mutate per entry, no batches.
5. On "skip": mark `skipped`. Either way, mark the candidate and continue. Stopping at any point is fine; the file is the cursor.

**Data flow:** Hardcover → `pull` → library JSON → `candidates` (+ blog, Bluesky, aliases) → `candidates.md`/`.json` → owner review → interview session → Mnemion entries. Later: `refresh` keeps rating/status/dates current; never touches influence or evidence.

**Candidate scoring.** For each library book the miner builds keys: exact title, title with subtitle stripped, author full name, and author surname only when the surname is rare in the corpus (frequency check, threshold set at implementation). Every hit is evidence: `{source, date, url, sentence}`. Signals are reported separately, not collapsed:

- blog mention (highest weight);
- social mention with a tracker/bookseller link or quoted title;
- bare social mention of author, title, or alias.

Bare mentions carry real weight because the most influential authors (Lessig, Adams) are cited by surname or abbreviation and never linked. Scores aggregate by author; the review file groups each author's library books beneath them. Books with a Hardcover review by the owner but no corpus hits are listed and flagged. The miner also emits alias suggestions: frequent capitalized tokens near "book", "read", "chapter" that matched nothing.

## Existing Patterns

Investigation (2026-09-03) of `mnemion-js/` found:

- No script pushes external data into a hive today; the `pull-hive`/`import-hive` commands named in `CLAUDE.md` are not present in `package.json`. This design does not add one to the worker; the CLI is external.
- `mutate` has no upsert. Dedupe is advisory via `exclusive_facets` in memory policy (`entities/Hive/data.ts:781–792`, `entities/Hive/prime.ts`). `refresh` therefore queries by `hardcover_id` first and updates by id, which is the same pattern an agent uses by hand.
- Ingress (`POST /i/{path}`, `entities/Hive/hive.ts:1389–1434`) is create-only and untrusted; it is not used here because loading is an interview, not a feed.
- No scheduled mechanism exists in the worker (`wrangler.toml` has no triggers). `refresh` is run by hand or by an external cron.
- Pattern creation and memory policy follow `CHANGE_TYPES` in `entities/Hive/evolution.ts:130–189` (`create_pattern`) and `:354–394` (`set_memory_policy`).
- `shared/core/dev-seed.ts` has no books pattern; these are the first.

The CLI follows the `content-archive` repo's convention of a Python project with a small CLI (`src/social_archiver/cli.py`) rather than the Node stack of the worker, because the work is text manipulation.

## Implementation Phases

### Phase 1: Hardcover pull
**Goal:** A dated local snapshot of the owner's whole Hardcover library.

**Components:**
- `book-tools/hardcover_tools/api.py` — GraphQL client: bearer auth, paging, 1 req/s throttle, stop on 429/401.
- `book-tools/hardcover_tools/pull.py` — the `pull` subcommand; writes `data/hardcover-library-YYYY-MM-DD.json` with, per user_book: ids, title, slug, contributors, edition ISBNs, status_id, rating, review_raw, and reads (started/finished).
- `book-tools/hardcover_tools/cli.py` — argparse entry.

**Dependencies:** `HARDCOVER_TOKEN` in `book-tools/.env` (gitignored). Field names confirmed by a live query on 2026-09-04 (library: 1,254 books):

```graphql
{ me { id username books_count
  user_books(limit: 50, offset: 0, order_by: {date_added: desc}) {
    id status_id rating date_added reviewed_at review_raw
    book { id title slug cached_contributors }   # [{author:{id,slug,name}, contribution}]
    edition { id isbn_10 isbn_13 }
    user_book_reads { id started_at finished_at } # dates as YYYY-MM-DD; both null for an in-progress read
} } }
```

`me` returns a one-element list. The response carried no `RateLimit-Policy` header on this call.

**Done when:** `pull` writes a JSON with a book count matching the Hardcover UI; paging and throttle covered by unit tests with a fake transport.

### Phase 2: Corpus loaders and matching core
**Goal:** Pure functions that turn blog and Bluesky into searchable text and match library books against it.

**Components:**
- `hardcover_tools/corpus.py` — loaders for `lu.is/src/content/blog/*.md` (frontmatter + body → paragraphs with date/url) and `content-archive/social-data/normalized/bluesky/*.jsonl` (post text/date/url). One `Doc` record shape shared by both.
- `hardcover_tools/match.py` — key builders (title, stripped title, author, rare-surname), alias loading from `aliases.toml`, surname-frequency check, and the matcher returning `Evidence` records.
- Fixtures: ~10 blog paragraphs and ~20 posts drawn from real data covering a link mention, a bare surname, an alias, and a common-surname non-match.

**Dependencies:** Phase 1 (library JSON shape).

**Done when:** Unit tests pass for both loaders and the matcher on the fixtures; a common surname does not match; an alias does.

### Phase 3: Candidate report
**Goal:** The human review file and the machine cursor.

**Components:**
- `hardcover_tools/candidates.py` — scoring by author aggregate with separate signal counts, grouping of books under authors, flagged reviewed-but-unmentioned books, alias suggestions.
- `candidates.md` renderer and `candidates.json` writer (`{id, kind: author|book, hardcover_id, score, signals, evidence[], state: pending|done|skipped}`).

**Dependencies:** Phase 2.

**Done when:** `candidates` runs against the real blog + Bluesky corpus and the owner has looked at the output (the inspection window before any further parsers). Renderer and scorer have unit tests.

### Phase 4: Mnemion patterns
**Goal:** `authors` and `books` exist in the owner's hive with policy and link label.

**Components:**
- Two `create_pattern` changes and two `set_memory_policy` changes, applied via the MCP tools; facet table as in Architecture.
- Doctrine text for each pattern.

**Dependencies:** None on the CLI; can run in parallel with Phases 1–3.

**Done when:** `mnemion://schema/authors` and `mnemion://schema/books` resolve with the listed facets; index shows the memory policy; a test create returns an `exclusive_facet` overlap advisory on a duplicate `hardcover_id`.

### Phase 5: Interview protocol
**Goal:** A repeatable one-entry-at-a-time loading conversation.

**Components:**
- The protocol document (system doc under `mnemion-js/src/system-docs/` or a skill in `book-tools/.claude/skills/`), specifying the five steps, the exact mutate shapes for `authors`, `books`, and the `author_of` link, and how `candidates.json` state is updated.

**Dependencies:** Phases 3 and 4.

**Done when:** One real session loads at least five candidates end to end; a second session resumes from the file without re-asking the first five.

### Phase 6: Refresh
**Goal:** Keep rating/status/dates current on existing entries without touching the owner's words.

**Components:**
- `hardcover_tools/refresh.py` — reads latest library JSON, queries Mnemion `books` by `hardcover_id` (batched `|=` filter), updates only the four bibliographic facets when changed, using version-checked updates.

**Dependencies:** Phases 1 and 5 (there must be entries to refresh).

**Done when:** A run against the real hive changes only rows whose Hardcover data changed, and `influence`/`evidence` are byte-identical before and after (covered by a test against a fake Mnemion client).

### Phase 7: Mastodon and Twitter loaders
**Goal:** Extend the corpus to the raw archives.

**Components:**
- Loaders in `corpus.py` for `content-archive/archives/social/archives/mastodon-download/outbox.json` and `twitter-download/data/cleaned_tweet.js`, emitting the same `Doc` shape.

**Dependencies:** Phase 3 output reviewed by the owner and judged useful enough to widen.

**Done when:** `candidates` runs over all four sources; loaders have fixture tests.

## Additional Considerations

**Nothing destructive, anywhere.** The CLI never writes to Hardcover. `refresh` never archives or creates Mnemion entries and never edits `influence`/`evidence`. The interview creates entries only on an explicit owner answer.

**Proposed, not decided (2026-09-04): review write-back.** When the interview (Phase 5) records an influence note, offer a per-book opt-in: "also post this note as your Hardcover review?" Per book, never global, because some influence notes are private in a way a public review is not. Adopting this revises the "never writes to Hardcover" rule above to "never destructive": the write must only create a review where none exists (or show the existing one and ask before replacing), and the token would need write scope. Before committing to it, confirm the Hardcover GraphQL mutation for user book reviews and whether it overwrites.

**Optional: post-import audit.** The importer's fidelity for read dates and reviews is reported inconsistent. If the `pull` snapshot shows gaps against the original StoryGraph CSV, an `audit` subcommand diffing the two is a natural addition; it is not in scope until the snapshot shows a need.

**Beta API.** Hardcover's API may reset tokens or change field names. Field names are confirmed against the live account in Phase 1 before any code depends on them; the client isolates the query text in one place.

**Corpus counts (2026-09-03).** Blog: 290 posts, ~16 about books. Bluesky: 13.7k posts (2024-02 to 2026-06), 531 containing "book", 76 with bookseller/tracker links. Mastodon: 13.2k records raw. These are the denominators for judging Phase 3's output.
