---
allowed-tools: Bash(gh --version), Bash(gh auth status), Bash(gh gist view:*), Bash(gh api:*)
description: Show the current claude-mirror backup status — gist ID, last backup time, what's changed since
disable-model-invocation: false
---

Show the user a quick summary of their claude-mirror backup state. Read-only — never modifies anything.

## Step 1: Look up the saved gist ID

Read `~/.claude/.mirror-gist-id`. If it doesn't exist, tell the user:

```
No claude-mirror backup found on this machine.

To create one: /mirror backup
To restore one from another machine: /mirror restore <gist-id>
```

Stop.

## Step 2: Fetch the gist's metadata and manifest

1. Run `gh api gists/<gist-id>` to get full metadata (URL, created/updated times, file list, revision count).
2. Run `gh gist view <gist-id> --filename manifest.json` to read the manifest.
3. If the gist no longer exists (404), tell the user: "Saved gist ID `<id>` no longer exists on GitHub. Run `/mirror backup` to create a fresh one." Stop.

## Step 3: Compare local state to backup

For each file in the manifest:
- Check if it exists locally at its `originalPath`.
- If it exists, compare contents (byte-equal check) to the gist version.
- Count: unchanged, modified, missing locally, plus any local files that aren't in the backup yet.

Skip the byte-comparison for sanitized settings files — those will always differ because of `<REDACTED>` placeholders. Just note them as "settings present in backup (sanitized)".

## Step 4: Print the summary

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

## Important rules

- Read-only — never write files, never call gh write commands.
- Don't include secrets in any diff output.
- If the manifest is missing or unparseable, say so and stop — don't guess.
