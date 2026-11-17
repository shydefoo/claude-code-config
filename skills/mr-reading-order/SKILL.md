---
name: mr-reading-order
description: >
  Produce a suggested file-by-file reading order for understanding a merge/pull request, ordered by
  data flow rather than diff order or alphabetical path. Use when the user wants to *understand* an
  MR/PR before reviewing it, is onboarding to someone else's change, or asks "what should I read first",
  "how do I make sense of this MR", "walk me through this PR", or gives an MR/PR URL and asks for a
  reading/review order. Works across languages and repos — hexagonal/layered, MVC, or flat.
---

# MR Reading Order

Given an MR/PR, produce an ordered list of files to read to build understanding fastest — not a
code review (see `code-review` / `review` skills for that), and not a diff dump.

## Steps

0. **Gather context beyond the diff before writing anything.** The diff alone is not context — it's
   what changed, not why the surrounding code exists or how it fit before this MR. For each layer
   identified in step 2, spend a little effort on:
   - **Why this area exists**: `git log --oneline -- <file>` / `git log -p -1 <file>` on the
     *pre-change* version, or `mcp__gitlab__get_file_blame` / `git blame`, to see if the code being
     touched was itself recent, part of a prior related ticket, or long-stable (stable code being
     changed is a bigger signal than churny code being changed).
   - **What called/used this before the MR**: grep for callers of a changed function/class outside
     the diff. If a port/interface is being extended, find its other implementations to see the
     existing pattern being followed or broken.
   - **Related tickets/MRs**: the description usually links prior/follow-up MRs, RFCs, or tickets —
     skim them enough to state the one-sentence "why" in your own words, don't just link them.
   - **Domain vocabulary**: if the MR introduces new terms (an enum, a config flag, a schema version),
     find where else that vocabulary is used or expected to be used, so the reader isn't learning a
     term in a vacuum.
   Keep this proportional — a 3-file MR needs a sentence of context, not an investigation. Skip
   ceremony; the goal is one or two sentences of "why does this exist" per phase, not a research report.

1. **Get the changed files + full diff**, not just names:
   - GitLab: `mcp__gitlab__get_merge_request` + `mcp__gitlab__list_merge_request_changed_files`, then
     `git diff <base_sha>...<head_sha> --stat` and full diff locally if the branch is checked out
     (cheaper than paginating diff via API for anything but tiny MRs).
   - GitHub: `gh pr view <n> --json files,body` + `gh pr diff <n>`.
   - No platform access / local-only: `git diff <base>...<head>` or `git diff <base>...HEAD`.
   - Read the MR/PR description too — it usually states intent and often enumerates "Part 1 / Part 2"
     or module boundaries that map directly to reading phases.

2. **Classify every changed file into a layer.** Don't assume hexagonal — infer from what's actually
   in the diff. Common buckets, roughly innermost-first:
   - **Pure logic / domain types** — enums, value objects, pure functions, interfaces/ports with no
     framework annotations. Zero or near-zero dependencies on the rest of the diff.
   - **Contracts / ports** — interfaces that other changed files implement or call.
   - **Orchestration / use-case / service layer** — where the new logic gets *invoked* and wired into
     existing control flow.
   - **Adapters / infra** — HTTP clients, DB repos, RPC adapters implementing the ports above.
   - **Wiring / config / DI** — bean definitions, dependency injection graphs, route registration.
   - **Build/deps/migrations** — version bumps, `build.gradle`/`package.json`/`go.mod`, SQL migrations.
     Almost always last, unless a migration is *the point* of the MR.

   Skip test files entirely — this skill orders production code only.

   If the MR is small/flat (e.g. a single-file frontend component change), skip formal layering —
   just order by "state/types → logic → rendering/wiring", or say the MR is small enough to read
   in diff order and stop.

3. **Order phases by data flow, not by layer purity.** The layer buckets above are a starting
   heuristic, not a rule — reorder when the diff's actual dependency chain says otherwise. Ask: what
   triggers this behavior, what decision/data gets produced, where does it get consumed, how does it
   reach the outside world? That chain is the reading order. If the MR has distinct parts (e.g. "Part
   1: compute a decision" / "Part 2: act on it"), preserve that as top-level grouping.

4. **Within each file, point at the specific lines/functions worth reading**, not "read the whole
   file" — especially for large files where only one function changed. Say *why* it matters in one
   clause tied to the data flow (e.g. "where the frozen decision is read back at exchange time"), not
   what the code does structurally.

5. **Omit test files from the list.** They don't need their own phase and don't get paired with the
   production files they cover — this skill is a map of the production-code data flow only.

6. **Put wiring/config/DI, build/dependency, and migration files last** unless the MR *is* a wiring or
   migration change, in which case that's the whole point and goes first.

## Output format

Start with a **2-4 sentence overview**: what problem/feature this MR addresses, and the shape of the
solution (e.g. "resolve-then-freeze a decision at exchange time, then branch token issuance on it") —
in your own words, not copy-pasted from the description.

Then a numbered list, grouped under short phase headers matching the MR's own structure (reuse the MR
description's part/module names if it has them). For each phase, lead with **one sentence of "why this
phase exists"** pulled from step 0's context-gathering — not a repeat of what the files do, but why
that piece of the system needed to change. Then list files: `path` — why it matters / what to look at,
folding in relevant background (e.g. "extends the `AuthApi` port — compare against its other
implementation in `LocalAuthApiAdapter` to see the pattern being followed") instead of only diff-level
description.

End with a one-line note on anything explicitly out of scope or blocking merge (pinned SNAPSHOT deps,
"do not merge yet" flags, etc.) if the description mentions any — that context matters for reading but
isn't itself a file to read.

Do not restate the whole diff or duplicate a full code review — this is a map with terrain notes, not
the territory itself, and not a research report either.
