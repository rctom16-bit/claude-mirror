# claude-mirror

> Back up your entire Claude Code setup to a private GitHub gist. Restore it on any machine in seconds.

Your global `CLAUDE.md`, your settings, your project memories, your list of installed plugins — all the personalisation you've built up over time — lives only on the laptop you're using. Lose the laptop, switch machines, or accidentally trash a config and that's gone.

`claude-mirror` is a small Claude Code plugin that fixes that.

## What it does

One slash command, `/claude-mirror:mirror`, with four subcommands:

| Command | What it does |
|---|---|
| `/claude-mirror:mirror backup` | Snapshots your Claude setup and uploads it to a private GitHub gist. Updates the same gist on every run, so you build a clean revision history. |
| `/claude-mirror:mirror restore` | Pulls a snapshot back down onto any machine. Always confirms before overwriting anything. Always makes a safety copy first. |
| `/claude-mirror:mirror status` | Shows what's backed up, when, from where, and which local files have drifted since the last backup. |
| `/claude-mirror:mirror list` | Shows every snapshot you've ever made, with timestamps and direct links to each historical version. |

> The `claude-mirror:` prefix is required — Claude Code namespaces every plugin command as `/<plugin>:<command>` to prevent name collisions between plugins.

## Install

You need:

- [Claude Code](https://docs.claude.com/en/docs/agents/claude-code) installed
- [GitHub CLI (`gh`)](https://cli.github.com/) installed and signed in (`gh auth status` should show you logged in)

Then in Claude:

```
/plugin marketplace add rctom16-bit/claude-mirror
/plugin install claude-mirror@claude-mirror
```

The first line registers this repo as a (single-plugin) marketplace; the second installs the plugin from it.

## Quick start

```
/claude-mirror:mirror backup
```

The first run creates a new private gist on your GitHub account and saves its ID locally. Every subsequent `/claude-mirror:mirror backup` updates the same gist, building a revision history.

To restore on a new machine:

```
/claude-mirror:mirror restore <gist-id>
```

Where `<gist-id>` is the ID from the URL of the gist `/claude-mirror:mirror backup` created (e.g. `abc123def456`). You can copy it from `gh gist list` on the original machine.

## What's included in a backup

**Included:**

- `~/.claude/CLAUDE.md` — your global "about me" file
- `~/.claude/settings.json` and `~/.claude/settings.local.json` — your Claude settings, with secrets stripped (see below)
- `~/.claude/keybindings.json` — your keyboard shortcuts
- All per-project memory files: `~/.claude/projects/*/memory/*.md`
- The **list** of installed plugin names — so restore can re-install them via `/plugin install`. The plugin files themselves are not bundled (they're public and re-downloadable).

**Excluded — for safety or size:**

- `~/.claude/credentials.json` — never touched. Contains your Claude login tokens.
- Session transcripts (`projects/*/<session>.jsonl`) — too large, contain every prompt and response.
- Plugin cache, file history, tool-result temp files — local-only data.

## How secrets are handled

`settings.json` can contain API keys (in `env`, in `mcpServers.*.env`, etc.). Before upload, `claude-mirror`:

1. Replaces every value inside any `env` block with the string `"<REDACTED>"`.
2. Scans for strings matching known secret patterns (`sk-…`, `ghp_…`, `xoxb-…`, long hex strings assigned to keys like `*token*`/`*secret*`/`*password*`) and redacts those too.
3. **Shows you exactly what was redacted** before uploading, so you can review.

Your local `settings.json` is never modified. The redaction happens in memory on a copy before upload.

When you restore, redacted secrets are restored as `"<REDACTED>"` placeholders. You re-enter the real values once, then back up again to lock them in (still redacted in the gist).

## Safety guarantees

- **Restore always makes a backup of your existing files first** — into `~/.claude/.mirror-backup-before-restore-<timestamp>/` — before overwriting anything. If a restore goes wrong, your previous state is recoverable.
- **Restore always asks for confirmation** with a clear list of what will be overwritten, before touching anything.
- **`/claude-mirror:mirror backup` shows you the file list and redactions** before uploading, and requires you to confirm.

## How it works (under the hood)

`claude-mirror` is intentionally tiny — about 200 lines of markdown total, no code, no dependencies beyond `gh`. Each slash command is a markdown file in `commands/` that gives Claude an exact recipe to follow. Claude does the work (read files, sanitize JSON, call `gh gist`) on your machine, with you watching.

This means:

- **You can audit it.** Open any `commands/*.md` file and read exactly what `claude-mirror` will do. No hidden code.
- **It's cross-platform for free.** Claude adapts the shell commands to Windows / macOS / Linux as needed.
- **Nothing leaves your machine except what you see.** All file reading, all sanitisation, all decisions happen locally before the single `gh gist` upload.

## FAQ

**Where exactly does my backup live?**
In a private GitHub gist owned by you, on your account. Private gists are not listed publicly, but they are accessible to anyone with the URL — treat the URL as a secret.

**Can I share my backup with someone else?**
You can — just give them the gist URL and have them run `/claude-mirror:mirror restore <gist-id>`. They'll get your full setup. Useful for sharing a curated config with a teammate.

**Does it back up plugin code?**
No — only the **list** of installed plugin names. On restore, `claude-mirror` prints `/plugin install <name>` commands you can run. This keeps the backup tiny and means you always get the latest version of each plugin.

**What if I lose my gist ID?**
Run `gh gist list` on any machine where you're signed into the same GitHub account. Look for the one described "Claude Code setup backup — managed by claude-mirror".

**How big is a typical backup?**
Usually under 100 KB. Even with many projects and large memory files, it's well within GitHub gist limits.

## Roadmap

- Per-snapshot restore (currently restores the latest snapshot only; older snapshots are viewable via `/claude-mirror:mirror list`)
- Optional encryption layer (pass a password, encrypt the gist contents client-side)
- Selective backup (exclude specific projects)

Open issues and PRs welcome.

## License

MIT — see [LICENSE](LICENSE).
