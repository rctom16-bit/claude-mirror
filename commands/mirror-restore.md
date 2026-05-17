---
allowed-tools: Bash(gh --version), Bash(gh auth status), Bash(gh gist view:*), Bash(gh gist list:*), Bash(gh api:*)
description: Restore your Claude Code setup from a previous claude-mirror backup
disable-model-invocation: false
---

Restore the user's Claude Code setup from a private GitHub gist created by `/mirror backup`. Follow these steps precisely — always confirm before overwriting anything.

## Step 1: Verify prerequisites

Run `gh --version` and `gh auth status`. If either fails, give the user the same install/login instructions as the backup command and stop.

## Step 2: Determine which gist to restore from

Check in this order:

1. **If the user passed a gist ID or gist URL as an argument** — use that. Extract the gist ID from URLs of the form `https://gist.github.com/<user>/<id>`.
2. **If `~/.claude/.mirror-gist-id` exists** — read the ID from it and ask the user "Restore from your saved backup gist `<id>`?" before proceeding.
3. **Otherwise** — list the user's recent gists with `gh gist list --limit 20` and look for ones whose description starts with "Claude Code setup backup". Show them and ask which one to use. If none found, tell the user "No backup gist found. Pass a gist ID or URL as the argument: `/mirror restore <gist-id>`."

## Step 3: Fetch the gist and read the manifest

1. Run `gh gist view <gist-id> --files` to get the list of files in the gist.
2. Confirm `manifest.json` is present. If not, stop and tell the user "This gist doesn't look like a claude-mirror backup — no manifest.json found."
3. Run `gh gist view <gist-id> --filename manifest.json` to read the manifest contents. Parse it.
4. Validate the manifest's `schema` field. If it's a higher number than this command knows about (currently 1), warn the user that the backup may have been made by a newer version of claude-mirror.

## Step 4: Show the restore plan and warn about overwrites

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
    (You'll run these manually — claude-mirror will print the /plugin install commands at the end)

  Files that exist locally but not in the backup: 2
    → projects/some-old-project/memory/x.md  [will be left untouched]
```

For each file marked "WILL OVERWRITE": check if the existing local file is byte-identical to the one in the gist. If identical, mark it as "unchanged" instead — no overwrite happening.

## Step 5: Back up the current state before overwriting

Before any overwrite, copy any file that will be overwritten into a safety folder:
`~/.claude/.mirror-backup-before-restore-<UTC-timestamp>/`

Preserve the relative path inside that folder. Tell the user where the safety folder is so they can roll back manually if anything goes wrong.

## Step 6: Confirm

**Wait for explicit user confirmation** ("yes" or "y"). If the user says no, stop cleanly. Do not partially restore.

## Step 7: Download and restore each file

For each entry in the manifest's `files` array:

1. Get the file content from the gist: `gh gist view <gist-id> --filename <gistName>` (or use `gh api gists/<gist-id>` for batch retrieval if many files).
2. Resolve the `originalPath` (expanding `~` to the user's home directory).
3. Create the parent directory if it doesn't exist.
4. Write the content to the original path.
5. Track success/failure per file.

## Step 8: Save the gist ID locally

Write the gist ID to `~/.claude/.mirror-gist-id` so future `/mirror backup` calls update this same gist.

## Step 9: Print the re-install commands for plugins

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

## Errors and edge cases

- If a file in the manifest is missing from the gist: report it, skip that file, keep going.
- If a write fails (permissions, disk full): report the file, stop the restore, and recommend the user check the safety backup folder.
- If the gist is empty or only has `manifest.json`: tell the user "Backup gist appears empty" and stop.
- If the user runs restore on the same machine that made the backup (and nothing has changed locally): say so and offer to skip.

## Important rules

- **Always** create the safety backup folder before any overwrite.
- **Never** restore `credentials.json` (the backup command refuses to include it, so it shouldn't be in the gist — but double-check and skip it if somehow present).
- **Always** confirm with the user before writing files. Restore is destructive.
- Don't silently rewrite the user's settings.json secrets to `<REDACTED>` — show clearly that they need to re-enter them.
