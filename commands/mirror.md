---
allowed-tools: Bash(gh --version), Bash(gh auth status), Bash(gh gist create:*), Bash(gh gist edit:*), Bash(gh gist view:*), Bash(gh gist list:*), Bash(gh api:*)
description: Back up, restore, or inspect your Claude Code setup (backup | restore | status | list)
disable-model-invocation: false
---

The user invoked `/mirror`. Look at the first word of the user's argument (`$ARGUMENTS`) to decide which operation to run:

- `backup` (or no argument) → follow the **BACKUP** section below
- `restore [gist-id-or-url]` → follow the **RESTORE** section
- `status` → follow the **STATUS** section
- `list` → follow the **LIST** section
- Anything else → tell the user the four valid subcommands (`backup`, `restore`, `status`, `list`) and stop.

The subcommand is positional. Any further arguments (like a gist ID for restore) follow it.

---

## BACKUP

Back up the user's Claude Code setup to a private GitHub gist. Follow these steps precisely — do not skip the confirmation prompt.

### Step 1: Verify prerequisites

Run `gh --version` and `gh auth status`. If either fails:
- If `gh` is not installed: tell the user "GitHub CLI is required. Install from https://cli.github.com/, then run `gh auth login` before retrying." Stop.
- If not authenticated: tell the user "Please run `gh auth login` in your terminal, then retry." Stop.

Otherwise note the authenticated GitHub username (needed for the gist URL).

### Step 2: Locate the Claude config directory

The Claude config directory is `~/.claude/` (on Windows: `C:\Users\<username>\.claude\`). Resolve the absolute path and confirm it exists. If it does not, tell the user "No ~/.claude directory found — nothing to back up." Stop.

### Step 3: Check for an existing backup gist

Look for `~/.claude/.mirror-gist-id` — a plain text file containing the gist ID if the user has backed up before.

- If it exists: read the ID. This will be an *update* to an existing gist.
- If not: this will be a *new* gist.

### Step 4: Gather the files to include

Collect these files (use the Read tool on each; skip silently any that don't exist):

**Always include if present:**
- `~/.claude/CLAUDE.md`
- `~/.claude/settings.json`
- `~/.claude/settings.local.json`
- `~/.claude/keybindings.json`

**Per-project memory files:**
- Use Glob to find every `~/.claude/projects/*/memory/*.md` file. Include all of them (including each `MEMORY.md` index).

**Never include (skip these even if asked):**
- `~/.claude/credentials.json` — contains login tokens.
- Any `*.jsonl` file under `projects/*/` — session transcripts (large, sensitive).
- Anything under `projects/*/subagents/`, `projects/*/tool-results/`, `plugins/cache/`, `file-history/`.

### Step 5: Discover the installed plugin list

Read `~/.claude/plugins/installed_plugins.json` and extract the list of installed plugin names. Do **not** include the plugin files themselves — just the names so restore can re-install them via `/plugin install`.

### Step 6: Sanitize settings.json (and settings.local.json)

These can contain secrets. Before including either, walk the JSON and redact:

- Any top-level `env` object: replace every value with the string `"<REDACTED>"`.
- Any `mcpServers.*.env` object: same — replace every value with `"<REDACTED>"`.
- Any string value that matches a known secret pattern: replace with `"<REDACTED>"`. Patterns to catch:
  - Starts with `sk-`, `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`, `npm_`, `xoxb-`, `xoxp-`, `xoxa-`
  - Is a 32+ character hex string assigned to a key that contains `key`, `token`, `secret`, `password`, or `auth` (case-insensitive)
- Track exactly which keys were redacted and how many — you'll show this to the user in Step 8.

If sanitization changes anything, hold the sanitized version in memory — do NOT modify the user's actual settings.json on disk.

### Step 7: Encode file paths for gist storage

Gists can't have nested folders — every file is flat. To preserve paths:

- For each file, take its path relative to `~/.claude/` and replace every `/` and `\` with `__` (double underscore).
- Example: `projects/C--Projects-foo/memory/user.md` → `projects__C--Projects-foo__memory__user.md`
- Files at the root of `.claude/` (like `CLAUDE.md`, `settings.json`) keep their plain name.

### Step 8: Build the manifest

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

### Step 9: Show the user a summary and confirm

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

### Step 10: Upload to the gist

**For a new gist:**
1. Write all the prepared files (encoded names + sanitized contents + manifest.json) to a temp directory.
2. Run `gh gist create --private --desc "Claude Code setup backup — managed by claude-mirror" <temp-dir>/*` and capture the gist URL from stdout.
3. Extract the gist ID from the URL (last path segment).
4. Save the gist ID to `~/.claude/.mirror-gist-id`.
5. Clean up the temp directory.

**For an existing gist (update):**
1. Write all prepared files to a temp directory.
2. For each file, use `gh gist edit <gist-id> --add <temp-path>` (or `--filename` style depending on what works for adding/updating).
3. If you detect orphans (files in the gist that no longer exist locally), list them and ask the user whether to remove them.
4. Clean up the temp directory.

### Step 11: Confirm success

Print the gist URL and a summary:

```
✓ Backup complete.

  Gist: https://gist.github.com/<user>/<gist-id>
  Files: 14
  Snapshot ID (commit sha): <first 7 chars from gh gist view --json>

  To restore on another machine:
    1. Install gh CLI, sign in, install Claude Code
    2. /plugin install claude-mirror (after adding the marketplace)
    3. /mirror restore <gist-id>
```

### Backup errors and edge cases

- If `gh gist create` fails (network, auth, etc.): tell the user the exact error and stop. Do not retry silently.
- If the local `.mirror-gist-id` points to a gist that no longer exists: ask the user whether to create a fresh one.
- If there are zero files to back up (empty `~/.claude/`): stop with a friendly message.

### Backup safety rules

- Never read or include `~/.claude/credentials.json`. Never.
- Never include `.jsonl` session transcripts.
- Always show what was redacted before uploading — the user must see what's leaving their machine.
- Never modify the user's real settings.json on disk. Sanitize a copy in memory only.

---

## RESTORE

Restore the user's Claude Code setup from a previous claude-mirror backup. Always confirm before overwriting anything.

### Step 1: Verify prerequisites

Run `gh --version` and `gh auth status`. If either fails, give the user the install/login instructions and stop.

### Step 2: Determine which gist to restore from

Check in this order:

1. **If the user passed a gist ID or gist URL as an argument** (after `restore`) — use that. Extract the gist ID from URLs of the form `https://gist.github.com/<user>/<id>`.
2. **If `~/.claude/.mirror-gist-id` exists** — read the ID from it and ask the user "Restore from your saved backup gist `<id>`?" before proceeding.
3. **Otherwise** — list the user's recent gists with `gh gist list --limit 20` and look for ones whose description starts with "Claude Code setup backup". Show them and ask which one to use. If none found, tell the user "No backup gist found. Pass a gist ID or URL: `/mirror restore <gist-id>`."

### Step 3: Fetch the gist and read the manifest

1. Run `gh gist view <gist-id> --files` to get the list of files in the gist.
2. Confirm `manifest.json` is present. If not, stop and tell the user "This gist doesn't look like a claude-mirror backup — no manifest.json found."
3. Run `gh gist view <gist-id> --filename manifest.json` to read the manifest contents. Parse it.
4. Validate the manifest's `schema` field. If higher than what this command knows (currently 1), warn the user that the backup may have been made by a newer version of claude-mirror.

### Step 4: Show the restore plan and warn about overwrites

Print a summary of what's about to happen:

```
Restore plan from gist <gist-id>:

  Snapshot taken: 2026-05-17 14:32 UTC
  From machine: ROBIN-LAPTOP (win32)
  By claude-mirror: v0.1.0

  Files to restore: 14
    → CLAUDE.md                          [WILL OVERWRITE existing]
    → settings.json                      [WILL OVERWRITE existing — secrets were redacted, you'll need to re-enter them]
    → projects/.../memory/user.md        [NEW — no conflict]
    → keybindings.json                   [NEW — no conflict]
    ...

  Plugins to re-install after restore: 4
    pulse, code-review, frontend-design, superpowers
    (claude-mirror will print the /plugin install commands at the end)

  Files that exist locally but not in the backup: 2
    → projects/some-old-project/memory/x.md  [will be left untouched]
```

For each file marked "WILL OVERWRITE": check if the existing local file is byte-identical to the one in the gist. If identical, mark it as "unchanged" — no overwrite happening.

### Step 5: Back up the current state before overwriting

Before any overwrite, copy any file that will be overwritten into a safety folder:
`~/.claude/.mirror-backup-before-restore-<UTC-timestamp>/`

Preserve the relative path inside that folder. Tell the user where the safety folder is so they can roll back manually if anything goes wrong.

### Step 6: Confirm

**Wait for explicit user confirmation** ("yes" or "y"). If the user says no, stop cleanly. Do not partially restore.

### Step 7: Download and restore each file

For each entry in the manifest's `files` array:

1. Get the file content from the gist: `gh gist view <gist-id> --filename <gistName>` (or `gh api gists/<gist-id>` for batch).
2. Resolve the `originalPath` (expanding `~` to the user's home directory).
3. Create the parent directory if it doesn't exist.
4. Write the content to the original path.
5. Track success/failure per file.

### Step 8: Save the gist ID locally

Write the gist ID to `~/.claude/.mirror-gist-id` so future `/mirror backup` calls update this same gist.

### Step 9: Print the re-install commands for plugins

For each plugin name in the manifest's `plugins` array, print a `/plugin install <name>` line the user can copy. Example:

```
Restore complete.

  Files restored: 14
  Safety backup: ~/.claude/.mirror-backup-before-restore-20260517T143200Z/

Next steps:
  1. Re-install your plugins by running each of these in Claude:
       /plugin install pulse
       /plugin install code-review
       /plugin install frontend-design
       /plugin install superpowers

  2. Re-enter any secrets that were redacted in settings.json:
       env.OPENAI_API_KEY
       env.ANTHROPIC_API_KEY
       mcpServers.github.env.GH_TOKEN

  3. Restart Claude Code to load the new settings.
```

### Restore errors and edge cases

- If a file in the manifest is missing from the gist: report it, skip that file, keep going.
- If a write fails (permissions, disk full): report the file, stop the restore, and recommend the user check the safety backup folder.
- If the gist is empty or only has `manifest.json`: tell the user "Backup gist appears empty" and stop.
- If the user runs restore on the same machine that made the backup and nothing has changed locally: say so and offer to skip.

### Restore safety rules

- **Always** create the safety backup folder before any overwrite.
- **Never** restore `credentials.json` (the backup command refuses to include it, so it shouldn't be in the gist — but double-check and skip it if somehow present).
- **Always** confirm with the user before writing files. Restore is destructive.
- Don't silently rewrite the user's settings.json secrets to `<REDACTED>` — show clearly that they need to re-enter them.

---

## STATUS

Show the user a quick summary of their claude-mirror backup state. Read-only — never modifies anything.

### Step 1: Look up the saved gist ID

Read `~/.claude/.mirror-gist-id`. If it doesn't exist, tell the user:

```
No claude-mirror backup found on this machine.

To create one: /mirror backup
To restore one from another machine: /mirror restore <gist-id>
```

Stop.

### Step 2: Fetch the gist's metadata and manifest

1. Run `gh api gists/<gist-id>` to get full metadata (URL, created/updated times, file list, revision count).
2. Run `gh gist view <gist-id> --filename manifest.json` to read the manifest.
3. If the gist no longer exists (404), tell the user: "Saved gist ID `<id>` no longer exists on GitHub. Run `/mirror backup` to create a fresh one." Stop.

### Step 3: Compare local state to backup

For each file in the manifest:
- Check if it exists locally at its `originalPath`.
- If it exists, compare contents (byte-equal check) to the gist version.
- Count: unchanged, modified, missing locally, plus any local files not in the backup.

Skip the byte-comparison for sanitized settings files — those will always differ because of `<REDACTED>` placeholders. Note them as "settings present in backup (sanitized)".

### Step 4: Print the summary

```
claude-mirror status

  Backup gist:    https://gist.github.com/<user>/<gist-id>
  Last backup:    2026-05-17 14:32 UTC (3 hours ago)
  Last backup from: ROBIN-LAPTOP (win32)
  Total snapshots: 7  (use /mirror list to see all)

Local vs backup:
  ✓ 11 files unchanged
  ✎  2 files modified locally since last backup
       - CLAUDE.md
       - projects/foo/memory/feedback_new.md
  +  1 new local file not yet backed up
       - projects/bar/memory/user.md
  -  0 files in backup but missing locally

Plugins recorded in backup: pulse, code-review, frontend-design, superpowers

Run /mirror backup to push the 3 changes above.
```

If everything is in sync, say so cleanly:

```
✓ Local state matches backup. Nothing to back up.
```

### Status safety rules

- Read-only — never write files, never call gh write commands.
- Don't include secrets in any diff output.
- If the manifest is missing or unparseable, say so and stop — don't guess.

---

## LIST

Show every revision (snapshot) of the user's backup gist — every `/mirror backup` they've ever run is a separate revision they can roll back to.

### Step 1: Find the gist ID

Read `~/.claude/.mirror-gist-id`. If it doesn't exist, tell the user:

```
No backup gist found on this machine. Run /mirror backup first.
```

Stop.

### Step 2: Fetch the full revision history

Run `gh api gists/<gist-id>` and parse the response. GitHub's gist API returns a `history` array — each entry is a revision. Each has:

- `version` (commit sha for that revision)
- `committed_at` (ISO timestamp)
- `change_status` (`{ total, additions, deletions }`)
- `url` (API URL — convert to web URL: `https://gist.github.com/<user>/<gist-id>/<sha>`)

### Step 3: Print the list

```
Snapshots of your claude-mirror backup gist

  Gist: https://gist.github.com/<user>/<gist-id>
  Total snapshots: 7

  #  When                    Changes                   Snapshot URL
  -  ----                    -------                   ------------
  7  2026-05-17 14:32 UTC    +2 / -0  (most recent)    https://gist.github.com/<user>/<id>/<sha7>
  6  2026-05-15 09:18 UTC    +1 / -1                   https://gist.github.com/<user>/<id>/<sha6>
  5  2026-05-12 22:04 UTC    +5 / -0                   https://gist.github.com/<user>/<id>/<sha5>
  ...
  1  2026-05-01 10:00 UTC    +14 / -0  (initial)       https://gist.github.com/<user>/<id>/<sha1>

To restore from a specific snapshot:
  - Open the snapshot URL in your browser to see exactly what was in it.
  - To roll back, download the manifest.json + files from that snapshot URL,
    place them in a folder, and a future version of claude-mirror will support
    restoring from that folder. For now, /mirror restore <gist-id> always uses
    the most recent snapshot.
```

### List safety rules

- Read-only — never write or modify anything.
- If the gist no longer exists, say so cleanly.
- Sort revisions newest-first.
- Show timestamps in UTC for consistency across machines.
