---
allowed-tools: Bash(gh --version), Bash(gh auth status), Bash(gh gist view:*), Bash(gh api:*)
description: List every snapshot (revision) of your claude-mirror backup gist
disable-model-invocation: false
---

Show the user every revision of their backup gist — every `/mirror backup` they've ever run is a separate revision they can roll back to.

## Step 1: Find the gist ID

Read `~/.claude/.mirror-gist-id`. If it doesn't exist, tell the user:

```
No backup gist found on this machine. Run /mirror backup first.
```

Stop.

## Step 2: Fetch the full revision history

Run `gh api gists/<gist-id>` and parse the response. GitHub's gist API returns a `history` array — each entry is a revision (a snapshot). Each entry has:

- `version` (the commit sha for that revision)
- `committed_at` (ISO timestamp)
- `change_status` ({ total, additions, deletions })
- `url` (API URL — convert to the human-readable web URL: `https://gist.github.com/<user>/<gist-id>/<sha>`)

## Step 3: Print the list

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
  - GitHub doesn't let `gh gist view` target a specific revision directly. To restore an older one,
    visit the snapshot URL, download the manifest.json + files manually, place them in a folder,
    and tell claude-mirror to use that folder (future feature).
  - Or: use the current snapshot with /mirror restore <gist-id>.
```

## Important rules

- Read-only — never write or modify anything.
- If the gist no longer exists, say so cleanly.
- Sort revisions newest-first.
- Show timestamps in UTC for consistency across machines.
