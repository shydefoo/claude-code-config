---
name: mr-walkthrough
description: >
  Walk through a phase (or the whole list) from an `mr-reading-order` output one point at a time —
  actual diff hunk plus explanation — pausing for discussion/questions after each point before
  advancing. Use when the user says "walk me through phase N", "go through this point by point",
  "show me the diff for each of these", or references a reading-order list already produced and
  wants to step through it interactively rather than getting the summary in bulk. Follow-up to
  `mr-reading-order` — assumes that skill (or equivalent file-by-file list) has already run in this
  conversation.
---

# MR Walkthrough

Turns an `mr-reading-order` list (or any explicit ordered set of files/points) into an interactive,
one-point-at-a-time session: real diff, explanation, then stop and wait. This is a pacing control,
not a new analysis — don't re-derive the reading order, use the one already produced.

## Preconditions

- A reading order already exists in the conversation (from `mr-reading-order` or stated explicitly by
  the user, e.g. "phase 2" or "these 4 files"). If none exists, run `mr-reading-order` first — don't
  fabricate a point-by-point list from scratch.
- Resolve which phase/points are in scope. If the user says "phase 2", pull exactly the file list
  under that phase header from the prior output, in the same order.

## Per-point loop

For each point, in order:

1. **Get the diff for just that file/hunk and open it in nvim, not in chat.**
   - Local checkout: `git diff <base>...<head> -- <path>` (or `<base>...HEAD` if head isn't a named
     ref). GitLab/GitHub without local checkout: `mcp__gitlab__get_merge_request_file_diff` /
     `gh pr diff <n> -- <path>`.
   - If a point pairs a production file with its test (per reading-order's own pairing rule), combine
     both into the same diff dump — don't split a pair across two turns or two files.
   - Write the diff to a scratch file with a `.diff` extension (e.g. `/tmp/mr-walkthrough-<n>.diff`)
     so nvim gets diff syntax highlighting for free, then open it:
     - Inside tmux (`$TMUX` set) with `nvim` on `PATH`: `tmux split-window -h "nvim <file>"` (fall back
       to `-v` if the user prefers a stacked layout, or if they've said so earlier in the session).
       Reuse the same scratch path each turn (overwrite it) unless the user asks to keep history —
       one pane, updated per point, beats accumulating panes.
     - No tmux / no nvim available: fall back to printing the diff in a code block in chat as before
       — don't silently skip showing the diff.
   - Do not paste the diff into the chat response when the nvim path succeeds — the point/explanation
     text stands alone in the reply; the code lives in the editor pane.

2. **Explain what's going on**, tied to the "why it matters" already established for this point in
   the reading-order output — don't repeat that summary verbatim, ground it in the actual lines now
   visible: which line does the thing the summary promised, what would break or look different
   without this hunk, how it connects to the point before/after it in the same phase.

3. **Stop. Wait for the user.** Do not advance to the next point in the same turn. The user may:
   - Ask questions / discuss this point further — stay on it, answer, and only move on when they
     signal readiness (e.g. "next", "ok continue", "go on").
   - Say to move on — proceed to the next point in the loop.
   - Ask to skip/jump — re-resolve scope from their instruction rather than assuming.

## Output format per point

Lead with a short header identifying the point (e.g. `2. GrantIdGenerator.kt`), then a short
explanation (a few sentences, not a re-review) — the diff itself lives in the nvim pane, not in the
reply. No numbered sub-lists unless the diff genuinely has multiple independent hunks worth
separating. End the turn there — no "shall I continue?" filler, the pause is implicit in the format.

## What this is not

- Not a code review — don't hunt for bugs or leave severity-tagged findings (`code-review` /
  `caveman-review` skills do that). Stay descriptive: what changed and why, not what's wrong with it.
- Not a re-run of `mr-reading-order` — don't recompute phases, layers, or ordering; consume what
  already exists.
- Not a bulk dump — never show more than one point's diff in a single turn, even if the user seems
  impatient; if they want bulk, that's a different request ("just show me the whole diff").
