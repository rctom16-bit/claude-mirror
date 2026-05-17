---
allowed-tools: Bash(gh --version), Bash(gh auth status), Bash(gh gist create:*), Bash(gh gist edit:*), Bash(gh gist view:*), Bash(gh gist list:*)
description: Back up your Claude Code setup to a private GitHub gist
disable-model-invocation: false
---

Back up the user's Claude Code setup to a private GitHub gist. Follow these steps precisely — do not skip the confirmation prompts.

## Step 1: Verify prerequisites

Run `gh --version` and `gh auth status`. If either fails:
- If `gh` is not installed: tell the user "GitHub CLI is required. Install from https://cli.github.com/, then run `gh auth login` before retrying." Stop.
- If not authenticated: tell the user "Please run `gh auth login` in your terminal, then retry." Stop.

Otherwise note the authenticated GitHub username (you'll need it for the gist URL).

## Step 2: Locate the Claude config directory

The Claude config directory is `~/.claude/` (on Windows: `C:\Users\<username>\.claude\`). Resolve the absolute path and confirm it exists. If it does not, tell the user "No ~/.claude directory found — nothing to back up." Stop.

## Step 3: Check for an existing backup gist

Look for `~/.claude/.mirror-gist-id` — a plain text file containing the gist ID if the user has backed up before.

- If it exists: read the ID. This will be an *update* to an existing gist.
- If not: this will be a *new* gist.

## Step 4: Gather the files to include

Collect these files (use the Read tool on each; skip silently any that don't exist):

**Always include if present:**
- `~/.claude/CLAUDE.md`
- `~/.claude/settings.json`
- `~/.claude/settings.local.json`
- `~/.claude/keybindings.json`

**Per-project memory files:**
- Use Glob to find every `~/.claude/projects/*/memory/*.md` file. Include all of them.
- Use Glob to find every `~/.claude/projects/*/memory/MEMORY.md` index. Include those.

**Never include (skip these even if asked):**
- `~/.claude/credentials.json` — contains login tokens.
- Any `*.jsonl` file under `projects/*/` — those are session transcripts (large, sensitive).
- Anything under `projects/*/subagents/`, `projects/*/tool-results/`, `plugins/cache/`, `file-history/`.

## Step 5: Discover the installed plugin list

Look at the plugins folder structure (`~/.claude/plugins/cache/<source>/<plugin-name>/`). Build a simple list of installed plugin names. Do **not** include the plugin files themselves — just the names so restore can re-install them via `/plugin install`.

## Step 6: Sanitize settings.json (and settings.local.json)

These can contain secrets. Before including either, walk the JSON and redact:

- Any top-level `env` object: replace every value with the string `"<REDACTED>"`.
- Any `mcpServers.*.env` object: same — replace every value with `"<REDACTED>"`.
- Any string value that matches a known secret pattern: replace with `"<REDACTED>"`. Patterns to catch:
  - Starts with `sk-`, `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`, `npm_`, `xoxb-`, `xoxp-`, `xoxa-`
  - Is a 32+ character hex string assigned to a key that contains "key", "token", "secret", "password", or "auth" (case-insensitive)
- Track exactly which keys were redacted and how many — you'll show this to the user in Step 8.

If sanitization changes anything, hold the sanitized version in memory — do NOT modify the user's actual settings.json on disk.

## Step 7: Encode file paths for gist storage

Gists can't have nested folders — every file is flat. To preserve paths:

- For each file, take its path relative to `~/.claude/` and replace every `/` and `\` with `__` (double underscore).
- Example: `projects/C--Projects-foo/memory/user.md` → `projects__C--Projects-foo__memory__user.md`
- Files at the root of `.claude/` (like `CLAUDE.md`, `settings.json`) keep their plain name.

## Step 8: Build the manifest

Create a `manifest.json` with this structure:

```json
{
  "schema": 1,
  "createdAt": "<ISO 8601 timestamp, UTC>",
  "machine": "<hostname>",
  "platform": "<win32 | darwin | linux>",
  "claudeMirrorVersion": "0.1.0",
  "files": [
    { "gistName": "CLAUDE.md", "originalPath": "~/.claude/CLAUDE.md" },
    { "gistName": "settings.json", "originalPath": "~/.claude/settings.json", "sanitized": true, "redactedKeys": ["env.OPENAI_API_KEY"] }
  ],
  "plugins": ["pulse", "code-review", "frontend-design"]
}
```

The manifest is the source of truth for restore — it tells the restore command exactly where each file goes.

## Step 9: Show the user a summary and confirm

Print a summary like:

```
Ready to back up your Claude setup:

  Files to include: 14
    - CLAUDE.md
    - settings.json (3 secrets redacted: env.OPENAI_API_KEY, env.ANTHROPIC_API_KEY, mcpServers.github.env.GH_TOKEN)
    - 11 memory files across 3 projects
    - keybindings.json

  Plugins to record (re-installable on restore): 4
    pulse, code-review, frontend-design, superpowers

  Total size: ~47 KB

  Target: <UPDATE existing gist abc123 | CREATE new private gist>

Proceed? (yes/no)
```

**Wait for explicit user confirmation** before doing anything that touches GitHub. If the user says no, stop cleanly.

## Step 10: Upload to the gist

**For a new gist:**
1. Write all the prepared files (encoded names + sanitized contents + manifest.json) to a temp directory.
2. Run `gh gist create --private --desc "Claude Code setup backup — managed by claude-mirror" <temp-dir>/*` and capture the gist URL from stdout.
3. Extract the gist ID from the URL (last path segment).
4. Save the gist ID to `~/.claude/.mirror-gist-id`.
5. Clean up the temp directory.

**For an existing gist (update):**
1. Write all prepared files to a temp directory.
2. For each file, run `gh gist edit <gist-id> --add <temp-path>` (or `--filename` style depending on what works for adding/updating).
3. Note: `gh gist edit` only updates the gist with the files you pass — files that no longer exist on this machine may remain in the gist. If you detect orphans, list them and ask the user whether to remove them.
4. Clean up the temp directory.

## Step 11: Confirm success

Print the gist URL and a summary:

```
✓ Backup complete.

  Gist: https://gist.github.com/<user>/<gist-id>
  Files: 14
  Snapshot ID (commit sha): <first 7 chars from gh gist view --json>

  To restore on another machine:
    1. Install gh CLI, sign in, install Claude Code
    2. /plugin install rctom16-bit/claude-mirror
    3. /mirror restore <gist-id>
```

## Errors and edge cases

- If `gh gist create` fails (network, auth, etc.): tell the user the exact error and stop. Do not retry silently.
- If the local `.mirror-gist-id` points to a gist that no longer exists: ask the user whether to create a fresh one.
- If there are zero files to back up (empty `~/.claude/`): stop with a friendly message.

## Important rules

- Never read or include `~/.claude/credentials.json`. Never.
- Never include `.jsonl` session transcripts.
- Always show what was redacted before uploading — the user must see what's leaving their machine.
- Never modify the user's real settings.json on disk. Sanitize a copy in memory only.
