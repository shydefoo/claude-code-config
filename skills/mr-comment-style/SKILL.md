---
name: mr-comment-style
description: "Write and post GitLab/GitHub MR/PR review comments in the user's own plain-text voice - no emojis, no markdown headers/bold labels, no severity-tag decoration. Always show the drafted comment(s) and get explicit confirmation before posting. Trigger: any time comments/threads are about to be posted to a merge request or pull request on the user's behalf."
---

# MR Comment Style

When posting review comments to a GitLab MR (or GitHub PR) on the user's behalf — via `mcp__gitlab__create_merge_request_thread`, `create_merge_request_note`, `create_merge_request_discussion_note`, `gh pr comment`, etc. — write them the way the user actually writes, not the way an internal review report looks.

## Rules for comment text

- No emojis.
- No markdown headers (`#`, `##`).
- No bold severity labels like `**[CRITICAL]**` or `**Fix:**` — say it in plain sentences instead.
- No bullet-point lists of "Issue / Fix" unless the comment genuinely has multiple independent points; even then prefer flowing sentences over a formatted template.
- Short and direct. State the problem, then the fix, in 1-3 sentences. Skip preamble ("I noticed that...", "It looks like...").
- Code identifiers get backticks (`` `variable` ``). That's the only markdown to keep.
- Match the tone of the user's own past PR/MR comments if any are visible in the conversation or repo history — terse, technical, no hedging.
- Give the author the benefit of the doubt: don't tell them what to do or hand down a fix. State what you observed and its consequence, then ask whether it's intended — e.g. "Is `NODE_ENV` trusted as safe for this here, or should it be a separate flag?" not "Gate on an explicit `ALLOW_LOCAL_AUTH` flag instead." The author may have context you don't; the comment should invite that context, not assume the finding is a bug.

Example of the wrong style (internal-report style — do not post this):
```
**[CRITICAL] Path traversal in proxy target URL**

`apiPath` segments are individually encoded...

**Fix:** reject/strip any segment equal to `.` or `..`...
```

Example of the right style (what actually gets posted):
```
apiPath segments get encodeURIComponent'd individually but `..` survives that unchanged, and new URL() collapses it — looks like it'd let a caller path-traverse past /api/v1/ on DOCS_QUALITY_API_URL. Intended, or should segments be rejected if they're `.`/`..`/contain a slash?
```

## Workflow

1. Draft every finding fully, in two parts:
   - the original issue as found: severity (CRITICAL/HIGH/MEDIUM/LOW) and a plain description of the problem, unreframed — this is your internal analysis, not what gets posted.
   - the reframed comment: the plain-voice, intent-check version from the rules above — this is what actually gets posted.
2. Show the user both parts for every finding before calling any post/create tool, with the exact file/line each inline comment will attach to. Format the preview as a markdown table, one row per finding, columns: `#`, `Location`, `Severity`, `Issue (raw)`, `Comment (to post)`. e.g.:
   ```
   | # | Location | Severity | Issue (raw) | Comment (to post) |
   |---|----------|----------|-------------|--------------------|
   | 1 | path/to/file.ts:43 | CRITICAL | <original finding, unreframed — what's wrong and why it matters> | <reframed text that will be posted> |
   | 2 | path/to/file.ts:8 | HIGH | <original finding, unreframed> | <reframed text that will be posted> |
   | 3 | (general note) | MEDIUM | <original finding, unreframed> | <reframed text that will be posted> |
   ```
   Keep cell text tight (1-2 sentences) so the table stays scannable — trim the raw issue description if needed, the full reasoning already lives in the comment column.
3. Ask for confirmation (or changes) before posting. Do not call `create_merge_request_thread` / `create_merge_request_note` / `gh pr comment` / equivalent until the user has explicitly approved — either the whole batch or after requested edits.
4. If the user requests edits to specific comments, revise and re-show only the changed ones, then confirm again before posting.
5. Once approved, post only the `Comment:` text for each finding — never the `Issue:` line. The severity-tagged `Issue:` line is for the user's review in-chat only and never reaches the MR/PR.

This applies regardless of what internal review format (severity tags, headers, etc.) was used earlier in the conversation to analyze or summarize findings — that internal format is fine for talking to the user, it just never goes into the posted comment body.
