# lipost-bot

A tiny launchd-driven autonomous LinkedIn poster. Fires Claude Code once a day at a randomized human-ish time, lets it write and publish a post via [`lipost`](https://github.com/yevgetman/lipost), and gets out of the way.

- macOS only (uses `launchd`).
- Single-file Python 3 CLI, stdlib only — no `pip install`.
- All scheduling, pause/resume, history, and config via `lipost-bot <command>`.

## What it actually does

`launchd` fires the bot at a fixed "baseline" hour every day (default `09:00`). The bot then sleeps for a random `0…N` seconds (default `0…12h`), so the actual post lands somewhere inside a window — e.g. between 09:00 and 21:00 — at a different time each day. Then it runs `claude -p "$(cat prompt.md)" --permission-mode bypassPermissions` inside the repo directory. Whatever Claude does next is up to your `prompt.md`. If the prompt instructs Claude to call `lipost post …`, the bot scrapes the resulting `urn:li:share:…` from Claude's output and records it to a local history file.

There's a min-hours-between-runs guard (default 20h) so accidental double-fires can't double-post, and a `pause` flag so you can take it offline without unloading the launch agent.

## Prerequisites

1. **Claude Code** on your `PATH` as `claude`. <https://docs.claude.com/en/docs/claude-code>
2. **lipost** installed and authenticated. <https://github.com/yevgetman/lipost>
   - `lipost init` (writes LinkedIn client credentials to `~/.config/linkedin-cli/`)
   - `lipost auth` (one-time browser OAuth)
   - `lipost whoami` should print your member URN
3. macOS with a writable `~/.local/bin` and `~/Library/LaunchAgents`.

`lipost-bot init` runs a preflight check covering all of the above — `claude` on `PATH`, `lipost` on `PATH`, plus the `lipost` config + token files — and refuses to set up if anything is missing. The same check runs inside `_cron`, so a missing dependency causes a run to skip with a clear log line instead of failing inside Claude. `lipost-bot status` shows the current state under the `deps:` row. Pass `--force` to `init` to override the check.

## Install

```bash
git clone https://github.com/yevgetman/lipost-bot.git ~/code/lipost-bot
cd ~/code/lipost-bot
chmod +x lipost-bot
./lipost-bot init
```

`init` is interactive (every prompt has a default — press enter to accept). It will:

- Write `~/.config/lipost-bot/config.json` with your settings.
- Symlink `lipost-bot` into `~/.local/bin/` so the command is on your `PATH`.
- Generate `local.lipost-bot.plist` in the repo, symlink it into `~/Library/LaunchAgents/`, and `launchctl load -w` it.

After `init`, write your instructions:

```bash
cp prompt.example.md prompt.md
$EDITOR prompt.md
```

`prompt.example.md` is a working starter template. **It includes a `<!-- TEMPLATE: … -->` marker on line 1 that the bot recognizes — while that line is present, `_cron` will refuse to run and log `prompt is still the template (remove the TEMPLATE marker line)`.** Edit the prompt to your liking and **delete the marker line** when you're ready to go live. The bot also skips when `prompt.md` is missing or empty.

## Usage

```bash
lipost-bot status              # everything in one screen
lipost-bot run                 # fire claude immediately (no jitter, ignores pause + gap)
lipost-bot pause               # skip future runs without unloading launchd
lipost-bot resume              # undo pause
```

### All commands

| Command | What it does |
| --- | --- |
| `init [--force]` | Interactive setup. `--force` skips the `lipost` dependency check. |
| `status` | Single-screen overview: dependencies OK?, launchd loaded?, paused?, last run, next window, posts in last 7d, all settings. |
| `config` | Print current settings. |
| `config <key>` | Print one key. |
| `config <key> <value>` | Update one key (validated). Auto-reloads launchd if `baseline_hour` changes. |
| `pause` | Set `paused: true`. `_cron` skips runs while paused. Survives reboot. |
| `resume` | Clear `paused`. |
| `start` | `launchctl load -w` the plist. |
| `stop` | `launchctl unload` the plist. The bot is fully off the schedule. |
| `run` | Fire `_cron` immediately, no jitter, ignoring pause + min-gap guard. For testing. |
| `next` | Print the next scheduled window (e.g. `2026-05-01 09:00 – 21:00`). |
| `prompt` | Open `prompt.md` in `$EDITOR` (defaults to `vi`). |
| `posts [--limit N]` | List recorded URNs, newest first. |
| `posts --open` | Open the most recent recorded post in your browser. |
| `logs [-n N] [-f]` | Tail the log. `-f` follows. |
| `uninstall` | Unload launchd, remove plist + PATH symlinks. **Keeps** `~/.config/lipost-bot/` (config + history). |

`lipost-bot _cron` is the internal entry point launchd calls; you don't normally run it by hand, but it's useful for testing the pause / min-gap paths (see *Testing* below).

## Configuration

`~/.config/lipost-bot/config.json` (chmod 600):

| Key | Default | Meaning |
| --- | --- | --- |
| `baseline_hour` | `9` | The hour (0–23) at which `launchd` fires the bot. Editing this re-writes the plist and reloads `launchd`. |
| `jitter_max_secs` | `43200` | Max seconds the wrapper sleeps after baseline before invoking Claude. Default `43200` = 12h, giving a 09:00–21:00 run window. Set to `0` for a deterministic fire at exactly `baseline_hour:00`. |
| `min_hours_between_runs` | `20` | If the previous run started less than this many hours ago, `_cron` skips. Prevents accidental double-posting. |
| `permission_mode` | `bypassPermissions` | Passed to `claude -p` as `--permission-mode`. The bot is unattended, so it must not stop on permission prompts. |

Update via `lipost-bot config <key> <value>`. Values are cast and validated.

## Files this CLI writes

| Path | Purpose |
| --- | --- |
| `~/.config/lipost-bot/config.json` | Settings (chmod 600). |
| `~/.config/lipost-bot/state.json` | `last_run_at` and `paused` (chmod 600). |
| `~/.config/lipost-bot/posts.jsonl` | Append-only record: `{urn, posted_at, exit_code}` per published post. |
| `~/Library/Logs/lipost-bot.log` | Wrapper logs and Claude stdout/stderr. |
| `~/.local/bin/lipost-bot` | Symlink to the CLI for `PATH` access. |
| `~/Library/LaunchAgents/local.lipost-bot.plist` | Symlink to the repo plist; the launch agent. |

The repo itself stays clean — nothing is written into the repo at runtime except the generated `local.lipost-bot.plist` (which is `.gitignore`d).

## Writing the prompt

The bot is dumb on purpose. All policy lives in `prompt.md`. See `prompt.example.md` for a starter with constraints that produce a single short post per run. Things you almost certainly want in your prompt:

- **Tell Claude to actually post** with `lipost post "<text>"`. If you forget, the bot will run, write a draft, and not publish anything.
- **Tell Claude to output only the URN** (or "SKIP"). Anything else just clutters the log.
- **Tell Claude not to use `--dry-run`** — useful for testing, but you'll never publish if it's stuck on dry-run.
- **Constraints on tone, topics, length, hashtags, emojis** — Claude will default to LinkedIn-cliché slop without explicit instruction.
- **A "skip if you'd be repeating yourself" clause.** The bot can't see prior posts on its own; the URN list in `posts.jsonl` is recorded, but the prompt doesn't see it unless you choose to inject it.

## Testing

Recommended test sequence — safe → real:

1. **Plumbing only.** No Claude, no post.
   ```bash
   lipost-bot status
   lipost-bot config
   lipost-bot next
   ```
2. **Pause path.** `run` ignores pause by design, so use `_cron`:
   ```bash
   lipost-bot pause
   lipost-bot _cron        # logs "paused — skipping run"
   tail -3 ~/Library/Logs/lipost-bot.log
   lipost-bot resume
   ```
3. **Claude runs but doesn't post.** Put a dry-run-only prompt in `prompt.md`:
   ```
   Run `lipost post --dry-run "test"` and report the JSON. Do not publish.
   ```
   Then `lipost-bot run` and `tail -f ~/Library/Logs/lipost-bot.log`.
4. **End-to-end.** Replace the prompt with real instructions, then:
   ```bash
   lipost-bot run
   lipost-bot posts        # the URN should appear
   lipost-bot posts --open # opens it in browser
   lipost delete <urn>     # nuke the test post
   ```

## How the schedule works

`launchd` doesn't have native jitter. The trick: `launchd` fires at a fixed `baseline_hour` daily, and the wrapper script sleeps a uniform random `0…jitter_max_secs` before invoking Claude. So the actual post time drifts inside a window each day.

To fire at *exactly* `baseline_hour:00` with no randomness:

```bash
lipost-bot config jitter_max_secs 0
```

To change the window:

```bash
lipost-bot config baseline_hour 11      # window now starts at 11:00
lipost-bot config jitter_max_secs 21600 # 6h jitter → window is 11:00–17:00
```

`baseline_hour` changes regenerate the plist and reload `launchd` automatically.

## Stopping the bot

| You want to… | Run |
| --- | --- |
| Pause for a few days, easy resume | `lipost-bot pause` |
| Take it off the schedule completely | `lipost-bot stop` |
| Kill switch *right now*, no commands | `: > prompt.md` (truncate prompt to zero bytes — `_cron` skips) |
| Remove the install but keep history | `lipost-bot uninstall` |
| Nuclear: remove everything | `lipost-bot uninstall && rm -rf ~/.config/lipost-bot ~/Library/Logs/lipost-bot.log` |

## Safety notes

- **`bypassPermissions` is wide.** The bot runs Claude with `--permission-mode bypassPermissions` so it doesn't hang on prompts. Inside that working directory (`~/code/lipost-bot`) Claude can read/write files and run shell commands without confirmation. Don't keep secrets in this directory and don't broaden the prompt to invite shell mischief.
- **The token expires.** LinkedIn member tokens last ~60 days. When yours expires, `lipost post` will 401 and the bot will fail at runtime. Re-run `lipost auth`. The dependency check verifies the token *file* exists, not that it's still valid.
- **No retries.** `_cron` updates `last_run_at` *before* invoking Claude. So a crash mid-run still counts as "today's run" and won't retry. This prevents double-posting; the tradeoff is that a Claude failure means no post that day.
- **No queue.** Claude decides what to write at run time. There's no "draft 7 posts and queue them." If you want that, build it into your prompt or extend the wrapper.

## License

MIT — see [LICENSE](LICENSE).
