---
name: update-project
description: Update previously ingested codebase memories when files have changed. Replaces stale navigation maps, patterns, gotchas, or config memories with fresh ones and links them via supersedes edges.
---

# Update Ingested Codebase Memories

You are a codebase analyst performing a targeted memory refresh. Your job is to identify which ingested memories are stale due to code changes and replace them with accurate, up-to-date versions.

## Trigger

This command runs when the user says "update ingestion", "refresh memory", "update nav map", "re-ingest", or mentions that specific files or modules have changed.

## Prerequisites

1. Confirm `memory.*` MCP tools are available. If not, stop and tell the user.
2. Confirm prior ingestion exists by searching memory:
   ```json
   {
     "query": "codebase ingestion navigation patterns",
     "namespace": { "scope": "workspace", "workspace_id": "<absolute-path>" }
   }
   ```
   If no prior ingestion is found, tell the user to run `/ingest` first.

## Phase U1: Identify what changed

1. Run `git diff --name-only HEAD~1` to get modified files. If that yields nothing useful, ask the user what changed.
2. Map changed files to ingestion memory categories:

| Changed files | Affected memory |
|---|---|
| `src/core/**` | core navigation map |
| `src/storage/**` | storage navigation map |
| `supabase/functions/**` | edge function navigation map |
| `src/utils/**`, `tests/**` | utils + tests navigation map |
| `package.json`, `Cargo.toml`, etc. | project overview |
| Broad structural changes | project overview, dependency map, patterns |
| New non-obvious constraints | gotchas |
| New env vars or config | config decoder |

Only process categories with actual changes. Skip unchanged areas entirely.

## Phase U2: Retrieve affected memories

For each affected category, search for the existing memory:
```json
{
  "query": "<category name> navigation ingestion",
  "namespace": { "scope": "workspace", "workspace_id": "<path>", "topic": "ingestion" },
  "k": 3
}
```
Record the old memory ID for each hit.

## Phase U3: Write replacement memories

For each stale memory:
1. Read the changed source files.
2. Write a **new** memory with updated content — same `kind`, `tags`, and `importance` as the original.
3. Add `"supersedes": "<old_id>"` to the new memory's `metadata`.
4. Link new → old: `memory.link(from=new_id, to=old_id, edge_type="supersedes")`.

Do **not** delete or archive the old memory — the supersedes edge signals staleness to future retrievals.

If a new major module was added with no existing memory, create it fresh and link `belongs_to` the project overview.

## Phase U4: Write an update summary

Write a `kind: "summary"` memory noting:
- Date of update
- Which memories were replaced and why (file changes that triggered each)
- Old ID → new ID mapping

Link this summary to the original ingestion summary using `edge_type: "related_to"`.

## Quality rules

- Only rewrite memories whose content actually changed — don't touch unaffected modules.
- If a file was renamed or deleted, note it explicitly in the replacement memory.
- Keep the same density and style as the original ingested memories.
- Do not re-ingest the entire codebase — targeted updates only.

## Output

When done, report:
- Which memories were updated (category + old ID → new ID)
- Which areas were skipped (no changes detected)
- The namespace used
