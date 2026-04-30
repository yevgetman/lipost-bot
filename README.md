# lipost-bot

A tiny launchd-driven autonomous LinkedIn poster. Once a day, at a randomized human-ish time, it fires up Claude Code, lets Claude look at one of your staged images (optional), write a caption, and publish to your LinkedIn feed via [`lipost`](https://github.com/yevgetman/lipost).

- macOS only (uses `launchd`).
- Single-file Python 3 CLI, stdlib only — no `pip install`.
- All scheduling, pause/resume, history, and config via `lipost-bot <command>`.

---

## How it works

1. `launchd` fires at a fixed **baseline hour** every day (default `09:00`).
2. The bot sleeps a random `0…N` seconds (default `0…12h`) — so the actual post lands somewhere inside a window (e.g. 09:00–21:00) at a different time each day.
3. The bot reads `prompt.md`. If it contains `{{IMAGE_PATH}}`, the bot picks a random image from `images/pending/` and substitutes the absolute path.
4. The bot runs `claude -p "<prompt>" --permission-mode bypassPermissions --model claude-opus-4-7` inside the repo dir.
5. Claude follows your prompt — typically: read the image, write a caption, call `lipost post --image <path> --alt "<alt>" "<caption>"`.
6. The bot scrapes the `urn:li:share:…` from Claude's output, appends it to `posts.jsonl`, and (if image mode) moves the image to `images/used/<timestamp>_<filename>` so it's never reused.
7. Min-hours-between-runs guard (default 20h) prevents accidental double-posting on the same day.

A master `active` switch (default `false` after install) gates everything: until you set it to `true`, every scheduled fire skips. `lipost-bot run` for manual testing ignores this gate so you can verify end-to-end before arming.

---

## Prerequisites

| Dependency | Why | Verify |
| --- | --- | --- |
| **Claude Code CLI** as `claude` on `PATH` | Generates the post text. | `claude --version` |
| **lipost CLI** as `lipost` on `PATH` | Publishes the post to LinkedIn. | `lipost --help` |
| `~/.config/linkedin-cli/config.json` | LinkedIn client id/secret. Created by `lipost init`. | `ls ~/.config/linkedin-cli/config.json` |
| `~/.config/linkedin-cli/token.json` | LinkedIn OAuth token. Created by `lipost auth`. Lasts ~60 days. | `lipost whoami` should print your URN |
| macOS with `~/.local/bin` on `PATH` and `~/Library/LaunchAgents/` writable | Where the symlink and launch agent go. | `echo $PATH \| grep .local/bin` |

If any of these are missing, `lipost-bot init` will refuse to set up and tell you which one. (Pass `--force` to override.) The same check runs inside every scheduled run, so a missing dependency causes a skip with a clear log line — never a confusing failure deep inside Claude.

Install Claude Code: <https://docs.claude.com/en/docs/claude-code>
Install lipost: <https://github.com/yevgetman/lipost>

---

## Step 1 — Install

```bash
git clone https://github.com/yevgetman/lipost-bot.git ~/code/lipost-bot
cd ~/code/lipost-bot
chmod +x lipost-bot
./lipost-bot init
```

`init` is interactive. Every prompt has a sensible default — press enter to accept. It will:

1. Run the dependency preflight (above).
2. Write `~/.config/lipost-bot/config.json` with your settings.
3. Symlink `lipost-bot` into `~/.local/bin/` so the command is on your `PATH`.
4. Create `images/pending/` and `images/used/` under the repo for image staging.
5. Generate `local.lipost-bot.plist`, symlink it into `~/Library/LaunchAgents/`, and `launchctl load -w` it.

After this, the schedule is **live but inert**. The bot is loaded into `launchd`, but two independent gates keep it from posting:

1. **`active=false`** in `config.json` — the master arm switch. `init` always writes `false`. The bot will not run until you explicitly flip it to `true`. (`lipost-bot run` for manual testing ignores this; `_cron` honors it.)
2. **The `<!-- TEMPLATE: … -->` marker** on line 1 of `prompt.md` — until you delete it, every run skips with `prompt is still the template`.

Both must be cleared before the bot will publish anything autonomously.

Verify with:

```bash
lipost-bot status
```

You should see `active: False ← inert`, `deps: OK`, `launchd: loaded`, and `prompt: template — edit before going live (will skip)`.

---

## Step 2 — Configure

All settings live in `~/.config/lipost-bot/config.json`. View them with:

```bash
lipost-bot config
```

Update one with `lipost-bot config <key> <value>` (validated and cast to the right type). Changing `baseline_hour` automatically regenerates the plist and reloads `launchd`.

| Key | Default | What it controls |
| --- | --- | --- |
| `active` | `false` | Master arm switch. `init` writes `false` so the bot is **inert by default after install** — every scheduled fire skips with `inactive (config.active=false) — skipping run`. Manually flip to `true` only after you've reviewed the prompt and staged any images. `lipost-bot run` ignores this (so testing is unaffected); `_cron` honors it. |
| `baseline_hour` | `9` | Hour (0–23) at which `launchd` fires the bot, before jitter. |
| `jitter_max_secs` | `43200` (12h) | Max seconds the wrapper sleeps after baseline before invoking Claude. Default gives a 09:00–21:00 run window. Set to `0` for a deterministic fire at exactly `baseline_hour:00`. |
| `min_hours_between_runs` | `20` | If the previous run started less than this many hours ago, `_cron` skips. Prevents accidental double-posting. |
| `permission_mode` | `bypassPermissions` | Passed to `claude -p` as `--permission-mode`. The bot is unattended, so it must not stop on permission prompts. |
| `claude_model` | `claude-opus-4-7` | Passed to `claude -p` as `--model`. Use a full model id (`claude-opus-4-7`, `claude-sonnet-4-6`) or an alias (`opus`, `sonnet`). Default is Opus 4.7 because image interpretation benefits from the strongest model. |

### Common config recipes

```bash
# Post strictly between 11:00 and 17:00 each day
lipost-bot config baseline_hour 11
lipost-bot config jitter_max_secs 21600

# Post at exactly 09:00 every day (no jitter)
lipost-bot config jitter_max_secs 0

# Use Sonnet instead of Opus (faster, cheaper, weaker at image reading)
lipost-bot config claude_model claude-sonnet-4-6

# Allow two posts per day if you're feeling chatty
lipost-bot config min_hours_between_runs 10
```

### Schedule mechanics

`launchd` doesn't have native jitter. The trick: it fires at a fixed `baseline_hour` daily, and the wrapper sleeps a uniform random `0…jitter_max_secs` before invoking Claude. So the actual post drifts inside `[baseline, baseline + jitter_max_secs]` each day.

`lipost-bot next` prints the next scheduled window (e.g. `2026-05-01 09:00 – 21:00 (window 12h)`).

---

## Step 3 — Write the prompt

The bot is intentionally dumb. *All policy lives in `prompt.md`.* Open it:

```bash
lipost-bot prompt          # opens prompt.md in $EDITOR
```

You'll see a working starter template (copied from `prompt.example.md`) with two important things on it:

1. **Line 1 is a `<!-- TEMPLATE: … -->` marker.** While that line is present, the bot refuses to run. It's an explicit "I'm not ready yet" gate. **Delete the line when you're ready to go live.**
2. **The body uses image mode** (it includes `{{IMAGE_PATH}}`). Read the next subsection for what that means; if you want text-only posts, just remove the placeholder and the image-related instructions.

### Image-mode prompts (default)

When `prompt.md` contains the placeholder `{{IMAGE_PATH}}`:
- The bot picks a random file from `~/code/lipost-bot/images/pending/` and substitutes its absolute path into the prompt at run time.
- Claude reads the image (the Read tool accepts image files), writes a caption from it, and runs `lipost post --image "<path>" --alt "<alt>" "<caption>"`.
- After a successful post, the image is moved to `images/used/<YYYYMMDDTHHMMSS>_<filename>` so it's never reused.
- If the placeholder is present but `images/pending/` is empty, the run is skipped with `prompt requires {{IMAGE_PATH}} but ... is empty`.

Drop your staged JPEG/PNG files into `~/code/lipost-bot/images/pending/`. List with `lipost-bot images`.

### Text-only prompts

If you don't want image mode, just don't use the placeholder. The bot ignores `images/` entirely. A minimal text-only prompt:

```
You are the autonomous voice of <name> on LinkedIn. On each run, publish exactly one post (1–3 sentences) about something you find interesting in software/AI today. Use `lipost post "<text>"`. Output only the URN it prints. No hashtags, no "excited to share", no LinkedIn cliches.
```

### Things every prompt should include

- **Tell Claude to actually post.** `lipost post "<text>"` (or `lipost post --image …` for image mode). Without an explicit call, Claude will write a draft and exit.
- **Tell Claude to output ONLY the URN** (or the literal string `SKIP`). Anything else clutters the log.
- **Tell Claude not to use `--dry-run`.** It's useful for testing but you'll never publish if Claude defaults to it.
- **Tone, topic, length, hashtags, emojis** — all explicit. Claude defaults to LinkedIn-cliché slop without instruction.
- **A `SKIP` clause.** "If you'd be repeating yourself or have nothing meaningful, output `SKIP`." Gives the bot a non-posting fallback.

The bot can't see prior posts on its own. The URN list in `~/.config/lipost-bot/posts.jsonl` is recorded but isn't injected into the prompt unless you explicitly write logic to do so.

---

## Step 4 — Verify and go live

```bash
# 1. Inspect what would happen — no LinkedIn call
lipost-bot run --no-fire
```

Prints cwd, resolved `claude` and `lipost` paths, the launchd-context PATH, prompt body, image mode (on/off + which image would be picked), the exact command that would run, and any unmet dependencies. Useful for catching shell-PATH vs launchd-PATH mismatches.

```bash
# 2. Real run, no jitter, ignores active/paused/min-gap guards
lipost-bot run
```

Watch the log:

```bash
lipost-bot logs -f
```

When Claude exits, check that the URN was recorded:

```bash
lipost-bot posts                # URN + which image was used
lipost-bot posts --open         # opens it on LinkedIn in your browser
lipost-bot images               # confirm the image moved to used/
```

If you don't want the test post live, delete it:

```bash
lipost delete <urn>
```

When you're confident, **arm the bot** so the daily schedule actually publishes:

```bash
lipost-bot config active true
lipost-bot status                 # confirm: active: True
```

Until you flip `active` to `true`, the launch agent will fire on schedule but every run will skip. After arming, the launch agent fires daily at `baseline_hour` and Claude does the rest. `lipost-bot status` is the one-stop check.

---

## Day-to-day usage

```bash
lipost-bot status              # everything in one screen
lipost-bot pause               # skip future runs without unloading launchd
lipost-bot resume              # undo pause
lipost-bot stop                # off the schedule entirely
lipost-bot start               # back on the schedule
lipost-bot images              # what's staged, what's been posted
lipost-bot posts               # post history
lipost-bot logs -f             # tail what's happening
```

### Kill switches (in order of severity)

| You want to… | Run |
| --- | --- |
| Disarm — bot stays loaded but won't publish on schedule | `lipost-bot config active false` (durable, persists across reboots) |
| Pause transiently for a few days, easy resume | `lipost-bot pause` |
| Stop posting *right now* without commands | `: > prompt.md` (truncate to zero bytes — `_cron` skips) |
| Take it off the schedule completely | `lipost-bot stop` |
| Remove the install but keep config + history | `lipost-bot uninstall` |
| Nuclear: remove everything | `lipost-bot uninstall && rm -rf ~/.config/lipost-bot ~/Library/Logs/lipost-bot.log ~/code/lipost-bot/images` |

**`active` vs `paused` vs `stop`:** all three prevent posts, but they're at different layers and feel different.
- `active=false` is the master switch in `config.json`. Use it for "I'm not ready for this to be running yet" or "take it offline indefinitely." Doesn't touch launchd.
- `paused=true` is in `state.json`. Use it for short tactical pauses (vacation, conference week). Same effect, but a different verb makes the intent clearer in your shell history.
- `stop` unloads the launch agent entirely. Use it when you want the bot off the system, not just dormant.

---

## All commands

| Command | What it does |
| --- | --- |
| `init [--force]` | Interactive setup. `--force` skips the dependency preflight. |
| `status` | Single-screen overview: active?, deps OK?, launchd loaded?, paused?, last run, next window, posts in last 7d, images pending, full config. |
| `config` | Print all settings. |
| `config <key>` | Print one key. |
| `config <key> <value>` | Update one key (validated). Auto-reloads launchd if `baseline_hour` changes. |
| `pause` | Set `paused: true`. `_cron` skips runs while paused. Survives reboot. |
| `resume` | Clear `paused`. |
| `start` | `launchctl load -w` the plist. |
| `stop` | `launchctl unload` the plist. The bot is fully off the schedule. |
| `run` | Fire `_cron` immediately, no jitter, ignoring `active` + `paused` + min-gap guards. For testing — works even when the bot is disarmed. |
| `run --no-fire` | Print the resolved plan (cwd, claude/lipost paths, PATH, prompt, image mode, full command, deps) without invoking Claude. |
| `next` | Print the next scheduled window. |
| `prompt` | Open `prompt.md` in `$EDITOR` (defaults to `vi`). |
| `posts [--limit N]` | List recorded URNs, newest first. |
| `posts --open` | Open the most recent recorded post in your browser. |
| `images` | List `images/pending/` and recent `images/used/` contents. |
| `images --open` | Open `images/pending/` in Finder (so you can drag images in). Add `--used` to open the archive instead. |
| `logs [-n N] [-f]` | Tail the log. `-f` follows. |
| `uninstall` | Unload launchd, remove plist + PATH symlinks. **Keeps** `~/.config/lipost-bot/` (config + history). |

`lipost-bot _cron` is the internal entry point launchd calls. You don't normally run it by hand, but it's useful for testing the `active` / `paused` / min-gap paths (`run` ignores all three; `_cron` honors them all).

---

## Files this CLI writes

| Path | Purpose |
| --- | --- |
| `~/.config/lipost-bot/config.json` | Settings (chmod 600). |
| `~/.config/lipost-bot/state.json` | `last_run_at` and `paused` (chmod 600). |
| `~/.config/lipost-bot/posts.jsonl` | Append-only record: `{urn, posted_at, exit_code, image}` per published post. |
| `~/Library/Logs/lipost-bot.log` | Wrapper logs and Claude stdout/stderr. |
| `~/code/lipost-bot/images/pending/` | Drop staged images here for the bot to consume. |
| `~/code/lipost-bot/images/used/` | Successfully-posted images, prefixed with a timestamp. |
| `~/code/lipost-bot/local.lipost-bot.plist` | Generated launch agent (gitignored — regenerated by `init` and on `baseline_hour` config changes). |
| `~/.local/bin/lipost-bot` | Symlink to the CLI for `PATH` access. |
| `~/Library/LaunchAgents/local.lipost-bot.plist` | Symlink to the repo plist; the launch agent. |

---

## Deeper testing

The Step-4 walkthrough is the recommended path. If you want to exercise the safety paths explicitly:

```bash
# 1. Inactive path — `run` ignores active, so use `_cron`:
lipost-bot config active false
lipost-bot _cron        # logs "inactive (config.active=false) — skipping run"
tail -3 ~/Library/Logs/lipost-bot.log
lipost-bot config active true

# 2. Pause path — `run` ignores pause, so use `_cron`:
lipost-bot pause
lipost-bot _cron        # logs "paused — skipping run"
tail -3 ~/Library/Logs/lipost-bot.log
lipost-bot resume

# 3. Min-gap path — fire once, then `_cron` again:
lipost-bot run          # records last_run_at
lipost-bot _cron        # logs "only X.Xh since last run … skipping"

# 4. Empty-prompt path:
mv prompt.md prompt.md.bak
lipost-bot _cron        # logs "prompt file missing"
mv prompt.md.bak prompt.md

# 5. Template-marker path:
# (default state right after init — `_cron` logs "still the template")

# 6. Missing-image path (with image-mode prompt + empty pending dir):
mv images/pending/* /tmp/   # if any
lipost-bot _cron            # logs "prompt requires {{IMAGE_PATH}} but ... is empty"
mv /tmp/<file> images/pending/

# 7. Dry-run-only Claude run (Claude executes, doesn't actually post):
echo 'Run `lipost post --dry-run "test"` and report the JSON. Do not publish.' > prompt.md
lipost-bot run
tail -n 50 ~/Library/Logs/lipost-bot.log
```

---

## Safety notes

- **`bypassPermissions` is wide.** The bot runs Claude with `--permission-mode bypassPermissions` so it doesn't hang on prompts. Inside the working directory (`~/code/lipost-bot`) Claude can read/write files and run shell commands without confirmation. Don't keep secrets there and don't broaden the prompt to invite shell mischief.
- **The LinkedIn token expires.** Member tokens last ~60 days. When yours expires, `lipost post` will 401 and the bot will fail at runtime. Re-run `lipost auth`. The dependency check verifies the token *file* exists, not that it's still valid.
- **No retries.** `_cron` records `last_run_at` *before* invoking Claude. So a crash mid-run still counts as "today's run" and won't retry. This prevents double-posting; the tradeoff is that a Claude failure means no post that day.
- **30-minute hard timeout.** Claude is run with a 30-minute timeout. If it hangs (e.g. an LLM API outage), the wrapper kills it and logs a partial-output snippet.
- **No queue.** Claude decides what to write at run time. There's no "draft 7 posts and queue them" feature. If you want that, build it into your prompt or extend the wrapper.
- **Prompt injection via images.** Claude reads images you stage. A maliciously crafted image with embedded text could try to override your prompt. Stage images you trust.

---

## License

MIT — see [LICENSE](LICENSE).
