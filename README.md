# Problem-to-Product Agents for Claude Code

A set of Claude Code subagents and one skill that take you from **"what problem should I solve?"** to **a working, verified web app (version 1)**, with evidence at every step.

The pipeline has three phases:

1. **Research**: find real complaints, check the receipts, score the problems.
2. **Build**: generate the backend and frontend from a written spec.
3. **Verify**: join the pieces, run the build, and check the running app like a user would.

A separate **context agent** keeps project notes up to date along the way.

---

## The pipeline at a glance

```
 RESEARCH
 pain-scout  ->  receipt-checker  ->  score-problems (skill)
   (finds)         (verifies)            (ranks)
                                            |
                          you pick a problem and write
                          PROBLEM.md, TODO.md, API.md
                                            |
 BUILD                                      v
 backend-builder  +  frontend-builder  ->  integrator
   (API + data)        (pages + UI)         (joins, builds)
                                                |
 VERIFY                                         v
                                           ui-verifier  ->  VERIFY.md
                                                |
                          FAIL items? -> back to integrator

 ANYTIME
 update-context  (keeps CLAUDE.md and /docs current)
```

---

## What's in the box

| Name | Type | Phase | One-line job |
|---|---|---|---|
| [`pain-scout`](#pain-scout) | Subagent | Research | Finds real complaints about one domain, with receipts |
| [`receipt-checker`](#receipt-checker) | Subagent | Research | Opens every link and stamps whether the quote is really there |
| [`score-problems`](#score-problems-skill) | Skill | Research | Groups checked pains into problems and scores them |
| [`backend-builder`](#backend-builder) | Subagent | Build | Builds API routes and JSON data storage |
| [`frontend-builder`](#frontend-builder) | Subagent | Build | Builds the pages and components |
| [`integrator`](#integrator) | Subagent | Build | Fixes mismatches and gets `npm run build` passing |
| [`ui-verifier`](#ui-verifier) | Subagent | Verify | Screenshots every page and marks each item PASS / FAIL / UNSURE |
| [`update-context`](#update-context) | Subagent | Anytime | Keeps `CLAUDE.md` and `/docs` notes up to date |

---

## Installation

Place the files in your project like this:

```
your-project/
├── .claude/
│   ├── agents/
│   │   ├── pain-scout.md
│   │   ├── receipt-checker.md
│   │   ├── backend-builder.md
│   │   ├── frontend-builder.md
│   │   ├── integrator.md
│   │   ├── ui-verifier.md
│   │   └── update-context.md
│   └── skills/
│       └── score-problems/
│           └── SKILL.md
├── research/        # created by pain-scout
├── web/             # the Next.js app (created by you / the builders)
├── PROBLEM.md       # you write this
├── TODO.md          # you write this
├── API.md           # you write this
└── PRODUCT.md       # optional
```

Agents live in `.claude/agents/`. The skill lives in its own folder under `.claude/skills/`. To use them across all projects, put them in `~/.claude/agents/` and `~/.claude/skills/` instead.

Each agent has a `description` in its frontmatter, so Claude Code can pick it automatically. You can also call one by name, e.g. *"Use the pain-scout agent to scout tuition centres in Chennai."*

---

## Phase 1: Research

### pain-scout

Finds real complaints about **one domain** on the public web and saves them with receipts.

- **Tools:** WebSearch, WebFetch, Write
- **Output:** `research/<domain-in-lowercase-with-dashes>.md`
- **Limits:** max 15 searches, max 12 pages opened, 8 to 12 pains

Key rules:

- A pain only counts if a real person said it. Quotes are copied exactly and never rewritten.
- Every pain needs a receipt: exact quote, the link of a page it actually opened, the person's **role** (never name), and the date if shown.
- It never invents links.
- Good sources: app store reviews, Reddit, Quora, forums, Google reviews, comments under news and YouTube videos. Company blogs and ads are skipped.
- It records what people do **today** instead (workarounds, spreadsheets, WhatsApp groups).
- Anything it believes but didn't find evidence for goes under *My guesses (not evidence)*.

Final reply is 3 lines: pain count, strongest pain, file path.

### receipt-checker

Audits a research file produced by `pain-scout`. It never adds pains, deletes rows, or changes quotes.

- **Tools:** Read, Edit, WebFetch
- **Input:** one research file
- **Output:** the same file with a new `Check` column and a `## Receipt check` summary

Stamps:

| Stamp | Meaning |
|---|---|
| `FOUND` | The quote is on the page, word for word (spacing and punctuation may differ) |
| `CLOSE` | The page says something similar, but not these words |
| `NOT FOUND` | The page opened and nothing like the quote is there |
| `COULD NOT OPEN` | Page didn't load, needed a login, or blocked access |

When unsure, it writes `COULD NOT OPEN` rather than guessing. If a **Who** cell contains a person's name, it replaces it with their role only.

### score-problems (skill)

Groups the checked pains in `research/` into problems and ranks them. Triggered when you ask to score, rank, or compare problems.

- **Reads:** every file in `research/` except `scores.md`
- **Output:** `research/scores.md`

How it works:

1. Uses only rows stamped `FOUND` or `CLOSE` (plus rows you checked by hand). Ignores `NOT FOUND`, `COULD NOT OPEN`, and all "My guesses".
2. Groups pains from the same need into one problem, written as *"`<who>` struggle to `<what>` because `<why>`"*. It names the pain, never a solution.
3. Scores each problem 1 to 5 on:

| Criterion | Question |
|---|---|
| Pain | How much does it cost them (money, hours, stress)? |
| Frequency | How often does it happen? |
| Paying today | Are they already spending money or time on a workaround? |
| Buildable | Could a beginner build a first web version in 3 Claude Code sessions? (no hardware, no licence, no two-sided marketplace on day one) |
| AI edge | Would AI make the fix clearly better, not just add a chatbot? |

   *Reachable* and *I care* are left as `?`. Only you can score those.

4. Every score cites row numbers, e.g. `(tuition #3, #7)`.
5. Flags **kill signals**: needs hardware; needs a licence or medical/money records; needs both marketplace sides on day one; fewer than 3 receipts.

Final reply: the top 3 problems, one line each.

---

## Phase 2: Build

Before building, you choose a problem and write three files in the project root:

| File | Purpose |
|---|---|
| `PROBLEM.md` | Who the user is |
| `TODO.md` | What they need to do |
| `API.md` | The pages, endpoints, JSON fields, and the version 1 items |

`PRODUCT.md` is optional extra context. All three builders also read `web/AGENTS.md`, because the Next.js version in use may differ from what the model knows.

### backend-builder

Builds the data side of version 1.

- **Tools:** Read, Write, Edit, Glob, Grep
- **Owns:** `web/src/app/api/`, `web/src/lib/`, `web/data/`
- **Never touches:** pages, layouts, components

Rules:

- Implements every endpoint in `API.md` exactly: same paths, methods, JSON field names.
- Stores data in `web/data/db.json` using Node's `fs`. No database, no new npm packages.
- Seeds 5 to 8 realistic example records for the user in `PROBLEM.md` (Indian names, rupees, local details, never real people's data).
- Validates input: `400` with `{"error": "..."}` for missing or wrong fields, `404` for an unknown id.

### frontend-builder

Builds the screens of version 1.

- **Tools:** Read, Write, Edit, Glob, Grep
- **Owns:** `web/src/app/` (excluding `api/`) and `web/src/components/`
- **Never touches:** `web/src/app/api/`, `web/src/lib/`, `web/data/`

Rules:

- Builds every page listed in `API.md`, fetching data from the endpoints exactly as documented. The backend may not exist yet.
- Tailwind classes only. No other UI libraries, no new packages.
- Phone-first design for the user in `PROBLEM.md`: large tap targets, their own words, rupees as `Rs`, no lorem ipsum.
- Every version 1 item must be doable from the screens, with clear empty states and clear fetch-failure messages.
- Replaces the Next.js starter page completely.

### integrator

Makes the app work end to end. Run it after both builders, or to fix failures listed in `VERIFY.md`.

- **Tools:** Read, Write, Edit, Glob, Grep, Bash

What it does:

1. Compares every `fetch` in the frontend against the backend routes and `API.md` (paths, methods, field names). **`API.md` always wins.**
2. Runs `npm run build` in `web/` and fixes errors, for up to 5 rounds. If it still fails, it explains what's left.
3. Fixes any `FAIL` items in `VERIFY.md` with the smallest change that works.

Guardrails: no new npm packages, no redesigns, never edits `API.md`, `PROBLEM.md`, or `TODO.md`, and never starts or stops the app.

Final reply: build `PASS` or `FAIL`, plus each fix in one line.

---

## Phase 3: Verify

### ui-verifier

Checks the **running** app the way a user would. It never changes code.

- **Tools:** Read, Glob, Bash, Write
- **Requires:** the app already running at `http://localhost:3000` (start it yourself with `npm run dev` in `web/`)
- **Requires:** Playwright available via `npx playwright`
- **Output:** `VERIFY.md` and screenshots in `shots/`

What it does:

1. Confirms the app is up with `curl`. If not, it stops and tells you to run `npm run dev`.
2. Screenshots every page in `API.md` at laptop (1280x800) and phone (390x844) sizes.
3. Looks at every screenshot.
4. Calls every `GET` endpoint in `API.md` and checks the JSON fields.
5. Marks every version 1 item:

| Verdict | Meaning |
|---|---|
| `PASS` | The screens show a user could do it |
| `FAIL` | They can't, with the reason |
| `UNSURE` | A still screenshot can't prove it (e.g. a save button). It tells you exactly what to click |

6. Also flags starter-page leftovers, lorem ipsum, tiny or cut-off text on phone, and visible error messages.

Shell commands are run one at a time with no pipes, so pre-approved permissions cover them without extra prompts. If `VERIFY.md` already exists, rows marked "tested by hand" are preserved and only the rest are rechecked.

Final reply: PASS / FAIL / UNSURE counts.

---

## Anytime

### update-context

Keeps the project's written context in sync with the codebase.

- **Tools:** WebSearch, WebFetch, Write
- **Writes to:** `CLAUDE.md` and markdown files in `/docs`

Rules:

- Updates `CLAUDE.md` only for changes that affect the whole project.
- Domain-specific notes go in `/docs`, one markdown file per domain, e.g. `ui_feedback.md`, `customer_feedback.md`, `what_is_working.md`, `what_broke.md`, `market_ideas.md`.
- No unnecessary commentary.

> **Note:** Subagents don't run on their own. To make this agent run "automatically", wire it up with a Claude Code [hook](https://docs.claude.com/en/docs/claude-code/hooks) (for example on `Stop`) or invoke it manually. Since it only has the `Write` tool, it overwrites files rather than editing them in place; add `Read` and `Edit` to its `tools` line if you want it to update existing notes safely.

---

## Typical workflow

```text
# 1. Research
Use pain-scout to scout "<domain>".
Use receipt-checker on research/<domain>.md.
Run the score-problems skill.

# 2. Decide
Pick a problem. Write PROBLEM.md, TODO.md and API.md.

# 3. Build
Use backend-builder.
Use frontend-builder.
Use integrator.

# 4. Verify (app running via `npm run dev` in web/)
Use ui-verifier.
Use integrator to fix the FAIL items in VERIFY.md.
Use ui-verifier again.
```

Repeat the last two steps until nothing is `FAIL`, then manually test the `UNSURE` items and mark them "tested by hand" in `VERIFY.md`.

---

## Design principles

- **Evidence over opinion.** Every pain has a quote, a link, and a stamp, and guesses are kept separate.
- **Clear ownership.** The backend and frontend builders have non-overlapping folders, and `API.md` is the single source of truth.
- **Small, safe agents.** Each agent gets only the tools it needs, and the verifier can never edit code.
- **Beginner-friendly scope.** No new packages, no databases, and version 1 should be buildable in a few sessions.

## Models

All agents use `model: sonnet`. Change the `model` field in any agent's frontmatter to switch.
