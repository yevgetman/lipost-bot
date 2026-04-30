# lipost-bot

A tiny launchd-driven autonomous LinkedIn poster with a human-in-the-loop approval queue. You batch-generate captioned drafts from staged images, review them in a TUI, approve the ones you like — and the bot posts the approved queue, one per day, at a randomized human-ish time.

- macOS only (uses `launchd`).
- Single-file Python 3 CLI, stdlib only — no `pip install`.
- All commands via `lipost-bot <subcommand>`.

---

## How it works

Three phases, each triggered separately:

**Phase 1 — Generate** (`lipost-bot generate`, manual). Drains everything in `images/pending/`. For each image, runs Claude Code with your `prompt.md` plus the **style layer** (`style.md` + your optional `user_style.md`) auto-appended. Claude reads the image and outputs a JSON object with a caption + alt text. Each `(image, caption, alt)` is saved as a draft directory under `drafts/<slug>/` with status `pending_approval`. Images Claude SKIPs are moved to `images/skipped/` with the reason recorded.

**Phase 2 — Review** (`lipost-bot review`, manual). Walks pending drafts. For each one: prints the image path, caption, and alt in the terminal, then prompts `[a]pprove / [r]eject / [e]dit / [g]enerate-again / [s]kip / [q]uit`. `e` opens caption + alt in `$EDITOR` for manual revision. `g` re-invokes Claude with optional feedback (e.g. "make it shorter", "less corporate") plus the previous draft as context, then re-renders so you can iterate. Approved drafts move to status `approved` and join the posting queue. (Open the image yourself — `open <path>` shown in the TUI — if you want a visual reference.)

**Phase 3 — Post** (launchd, automatic; or `lipost-bot run`, manual). When `launchd` fires at the daily `baseline_hour`, the wrapper sleeps a random `0…N` seconds (jitter), then picks the **oldest approved draft** and runs `lipost post --image <path> --alt "<alt>" "<caption>"`. On success, the draft's status flips to `posted` and the URN is recorded. **No Claude at fire time** — the LLM work happened during Phase 1.

A min-hours-between-runs guard (default 20h) prevents accidental double-posting. A master `active` flag (default `false` after `init`) gates everything: scheduled fires skip until you set it to `true`.

---

## Prerequisites

| Dependency | Why | Verify |
| --- | --- | --- |
| **Claude Code CLI** as `claude` on `PATH` | Generates captions during Phase 1. Not needed at fire time. | `claude --version` |
| **lipost CLI** as `lipost` on `PATH` | Publishes posts during Phase 3. | `lipost --help` |
| `~/.config/linkedin-cli/config.json` | LinkedIn client id/secret. Created by `lipost init`. | `ls ~/.config/linkedin-cli/config.json` |
| `~/.config/linkedin-cli/token.json` | LinkedIn OAuth token. Created by `lipost auth`. Lasts ~60 days. | `lipost whoami` should print your URN |
| macOS with `~/.local/bin` on `PATH` and `~/Library/LaunchAgents/` writable | Where the symlink and launch agent go. | `echo $PATH \| grep .local/bin` |

`lipost-bot init` runs a preflight check covering `claude`, `lipost`, and the `lipost` config + token files; refuses to set up if anything is missing. The same check runs inside every scheduled run, so a missing dependency causes a skip with a clear log line. `lipost-bot status` shows current state under `deps:`. Pass `--force` to `init` to override.

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

1. Run the dependency preflight (refuses to continue if `claude` or `lipost` is missing; pass `--force` to override).
2. Write `~/.config/lipost-bot/config.json` with your settings (including `active=false`).
3. Symlink `lipost-bot` into `~/.local/bin/`.
4. Create `images/pending/`, `images/skipped/`, and `drafts/` under the repo.
5. Seed `prompt.md` from `prompt.example.md` if `prompt.md` doesn't already exist (the TEMPLATE marker on line 1 stays in place — bot remains inert).
6. Generate `local.lipost-bot.plist`, symlink it into `~/Library/LaunchAgents/`, and load it.

After this, the schedule is **live but inert**. Two independent gates keep it from posting:

1. **`active=false`** in `config.json` — master arm switch. The bot will not post on schedule until flipped to `true`.
2. **The `<!-- TEMPLATE: … -->` marker** on line 1 of `prompt.md` — until you delete it, `generate` refuses to run.

Verify with:

```bash
lipost-bot status
```

You should see `active: False ← inert`, `deps: OK`, `launchd: loaded`, `prompt: template`, and `drafts: approved=0, pending=0, …`.

---

## Step 2 — Configure

All settings live in `~/.config/lipost-bot/config.json`. Show with `lipost-bot config`. Update with `lipost-bot config <key> <value>` (validated and cast to the right type). Changing `baseline_hour` automatically regenerates the plist and reloads `launchd`.

| Key | Default | What it controls |
| --- | --- | --- |
| `active` | `false` | Master arm switch. `init` writes `false` so the bot is inert by default. Manually flip to `true` once you have an approved queue. `lipost-bot run` ignores this; `_cron` honors it. |
| `baseline_hour` | `9` | Hour (0–23) at which `launchd` fires the bot, before jitter. |
| `jitter_max_secs` | `43200` (12h) | Max seconds the wrapper sleeps after baseline before posting. Default gives a 09:00–21:00 run window. Set to `0` for a deterministic fire at exactly `baseline_hour:00`. |
| `min_hours_between_runs` | `20` | If the previous run started less than this many hours ago, `_cron` skips. Prevents accidental double-posting. |
| `permission_mode` | `bypassPermissions` | Passed to `claude -p` during `generate` as `--permission-mode`. |
| `claude_model` | `claude-opus-4-7` | Passed to `claude -p` during `generate` as `--model`. Use a full id (`claude-opus-4-7`, `claude-sonnet-4-6`) or alias (`opus`, `sonnet`). Default is Opus 4.7 because image interpretation benefits from the strongest model. |

### Common config recipes

```bash
# Post strictly between 11:00 and 17:00 each day
lipost-bot config baseline_hour 11
lipost-bot config jitter_max_secs 21600

# Post at exactly 09:00 every day (no jitter)
lipost-bot config jitter_max_secs 0

# Use Sonnet during generate (faster, cheaper, weaker at images)
lipost-bot config claude_model claude-sonnet-4-6
```

### Schedule mechanics

`launchd` doesn't have native jitter. The trick: `launchd` fires at `baseline_hour` daily, and the wrapper sleeps a uniform random `0…jitter_max_secs` before posting. So the actual post drifts inside a window each day. `lipost-bot next` prints the next window.

---

### The style layer

Separately from `prompt.md`, the bot maintains a **formatting & punctuation layer** that's automatically appended to every prompt at generate time:

- **`style.md`** — ships with the repo, app-controlled. Mechanical rules only: quotes, dashes, whitespace, character set, markdown handling, length caps, alt-text conventions. **Not** voice or content rules. View it with `lipost-bot style --baked`.
- **`user_style.md`** — gitignored, optional. Your additions to the layer. `lipost-bot style` opens it in `$EDITOR` (creates it if missing).
- `lipost-bot style --show` prints the combined layer (baked + user) exactly as injected.

The split: voice/topic/tone live in `prompt.md`. Typography/whitespace/length live in the style layer. Updating `style.md` happens via `git pull` — your `user_style.md` survives.

---

## Step 3 — Write the prompt

The prompt is consumed only at **generate time**, not at fire time. It tells Claude how to read an image and what JSON to output.

```bash
lipost-bot prompt          # opens prompt.md in $EDITOR
```

The shipped `prompt.md` already has a working starter. Two important things on it:

1. **Line 1 is a `<!-- TEMPLATE: … -->` marker.** While that line is present, `generate` refuses to run. **Delete it once you've customized the body below.**
2. **The body uses image mode** (it includes `{{IMAGE_PATH}}`). The wrapper substitutes that with an absolute path at run time.

### What the prompt must produce

The wrapper parses Claude's stdout for a single JSON object. **The prompt must instruct Claude to output JSON only, no preamble, no code fence.** Two shapes are accepted:

```json
{"caption": "<post body>", "alt": "<one-line factual description>"}
```

```json
{"skip": true, "reason": "<short explanation>"}
```

If parsing fails (no valid JSON in the output), `generate` logs the failure for that image and leaves it in `pending/` for retry.

### What every prompt should include

- The `{{IMAGE_PATH}}` placeholder (substituted at run time).
- An instruction to **read the image** (Claude's Read tool accepts JPEG/PNG/GIF; for GIFs it sees a representative frame).
- A `SKIP` clause: tell Claude what conditions warrant `{"skip": true, ...}` — image off-domain, ambiguous, repetitive.
- **Voice and content constraints**: tone, topic, length, what NOT to write about, banned phrases ("excited to share", "thrilled to announce", "what are your thoughts?", etc.), opening/closing rules. Claude defaults to LinkedIn-cliché slop without explicit guidance.
- An explicit instruction to **output ONLY the JSON** — no explanation, no code fence.

### What does NOT belong in the prompt

The style layer (`style.md` + `user_style.md`) is auto-appended to every generate-time prompt. It already covers all the **mechanical formatting rules** — capitalization, punctuation, dashes, smart vs straight quotes, paragraph breaks, whitespace, ASCII vs Unicode, markdown, length caps, emoji policy, alt-text conventions. **Don't duplicate these in `prompt.md`** — and avoid contradicting them, since the style layer is appended after the prompt body and Claude treats it as authoritative for mechanics.

If you want to change a mechanical rule (e.g. "always use lowercase first word"), edit the style layer (`lipost-bot style` for your additions, or PR against `style.md` for app-wide changes), not the prompt.

The shipped `prompt.example.md` is a generic starter; `prompt.md` is the live file. They have the same shape.

---

## Step 4 — Stage, generate, review, arm

```bash
# 1. Stage one or more images
cp ~/Pictures/diagram.png ~/code/lipost-bot/images/pending/
# Or open the dir in Finder and drag them in:
lipost-bot images --open

# 2. Confirm they're picked up
lipost-bot images

# 3. Edit the prompt — and DELETE the TEMPLATE marker on line 1 when ready
lipost-bot prompt

# 3b. (Optional) Add personal formatting tweaks to the style layer
lipost-bot style              # opens user_style.md in $EDITOR
lipost-bot style --show       # preview the combined layer that gets injected

# 4. Drain the pending dir into drafts (this calls Claude once per image)
lipost-bot generate

# 5. Review each draft in the TUI
lipost-bot review
# For each: image path + caption + alt show in terminal.
# Press a/r/e/g/s/q.
#   a — approve
#   r — reject
#   e — edit caption + alt in $EDITOR
#   g — ask Claude to rewrite (prompts for optional feedback like
#       "make it shorter" or "lead with the failure mode"); re-renders
#       so you can iterate. Press 'g' again with different feedback,
#       or 'a' once you're happy.
#   s — skip (leave as pending_approval)
#   q — quit (remaining drafts stay pending_approval)

# 6. Verify state — `drafts: approved=N` should be > 0
lipost-bot status
lipost-bot drafts --status approved

# 7. Sanity-check what the bot would do at fire time
lipost-bot run --no-fire    # shows queue + next draft + lipost path

# 8. Test-post the next approved draft NOW
lipost-bot run
lipost-bot posts --open     # see it on LinkedIn
lipost delete <urn>         # if you want it gone

# 9. Arm the schedule
lipost-bot config active true
lipost-bot status           # active: True
```

After step 9, the launch agent fires daily at `baseline_hour`, sleeps the jitter, and pops the next approved draft. `lipost-bot status` is the one-stop check at any time.

---

## Day-to-day usage

```bash
lipost-bot status              # everything in one screen
lipost-bot drafts              # queue grouped by status
lipost-bot drafts --status approved   # just the queue waiting to post
lipost-bot images              # what's staged in pending/ (and any skipped)
lipost-bot images --open       # open pending dir in Finder
lipost-bot generate            # whenever you've staged new images
lipost-bot review              # whenever there are pending drafts
lipost-bot prompt              # edit prompt.md (voice, content, JSON shape)
lipost-bot style               # edit user_style.md (your formatting tweaks)
lipost-bot style --show        # preview the full style layer Claude will see
lipost-bot pause / resume      # transient pause without unloading launchd
lipost-bot stop / start        # off / on the schedule
lipost-bot run --no-fire       # inspect the next post without publishing
lipost-bot run                 # post the next approved draft now (test or manual)
lipost-bot next                # next scheduled fire window
lipost-bot logs -f             # tail live activity
lipost-bot posts               # post history (URN + image)
lipost-bot posts --open        # open the most recent post on LinkedIn
```

### Kill switches (in order of severity)

| You want to… | Run |
| --- | --- |
| Disarm the schedule (durable) | `lipost-bot config active false` |
| Pause transiently for a few days | `lipost-bot pause` |
| Drain the approved queue without posting | Hand-edit `drafts/<slug>/meta.json` and change each `"status": "approved"` to `"rejected"`. The wrapper trusts what's on disk, so the change takes effect on the next `_cron`. Or `rm -rf drafts/<slug>` if you want them gone entirely. |
| Take the bot off the schedule completely | `lipost-bot stop` |
| Remove the install but keep config + drafts + history | `lipost-bot uninstall` |
| Nuclear: remove everything | `lipost-bot uninstall && rm -rf ~/.config/lipost-bot ~/Library/Logs/lipost-bot.log ~/code/lipost-bot/{images,drafts}` |

**`active` vs `paused` vs `stop`**: all three prevent posts at different layers. `active` is the durable master switch in config. `paused` is a transient flag in state. `stop` unloads the launch agent entirely.

---

## All commands

| Command | What it does |
| --- | --- |
| `init [--force]` | Interactive setup. `--force` skips the dependency preflight. |
| `status` | Single-screen overview: active?, deps OK?, launchd loaded?, paused?, last run, next window, posts in last 7d, drafts queue, images pending, full config. |
| `config` | Print all settings. |
| `config <key>` | Print one key. |
| `config <key> <value>` | Update one key (validated). Auto-reloads launchd if `baseline_hour` changes. |
| `generate` | Drain `images/pending/`. Calls Claude per image. Saves drafts as `pending_approval`. Skipped images go to `images/skipped/` with a reason sidecar. |
| `review` | TUI for pending drafts: prints image path + caption + alt in terminal, prompts `[a]pprove / [r]eject / [e]dit / [g]enerate-again / [s]kip / [q]uit`. `[e]` opens caption + alt in `$EDITOR`. `[g]` re-invokes Claude with optional feedback (e.g. "make it shorter") plus the previous draft as context, replaces the caption + alt in `meta.json`, and re-renders so you can review the rewrite. |
| `drafts [--status STATUS] [--limit N]` | List drafts grouped by status (`pending_approval`, `approved`, `rejected`, `posted`). |
| `pause` / `resume` | Set/clear `paused` in state. `_cron` skips while paused. |
| `start` / `stop` | `launchctl load -w` / `unload` the plist. |
| `run` | Post the next approved draft now, no jitter, ignoring `active`/`paused`/min-gap. Honors queue-empty (will skip if no approved drafts). |
| `run --no-fire` | Print the resolved plan (queue counts, next draft, lipost path, deps, would-skip reasons) without calling lipost. |
| `next` | Print the next scheduled window. |
| `prompt` | Open `prompt.md` in `$EDITOR`. |
| `style` | Open `user_style.md` in `$EDITOR` (creates if missing). Use this for your formatting/punctuation additions. |
| `style --show` | Print the combined style layer (baked `style.md` + your `user_style.md`) exactly as it gets injected into the prompt. |
| `style --baked` | Print only the baked-in `style.md` (read-only — edit via PR or `git pull` for updates). |
| `posts [--limit N] [--open]` | Post history. `--open` opens the most recent URN in browser. |
| `images [--open] [--skipped]` | List `images/pending/` (and `images/skipped/` if any). `--open` opens pending in Finder; `--skipped` opens the skipped dir. |
| `logs [-n N] [-f]` | Tail `~/Library/Logs/lipost-bot.log`. `-f` follows. |
| `uninstall` | Unload launchd, remove plist + PATH symlinks. **Keeps** `~/.config/lipost-bot/`, `drafts/`, and `images/`. |

`lipost-bot _cron` is the internal entry point launchd calls. You don't normally run it by hand, but it's useful for testing the `active` / `paused` / min-gap paths (`run` ignores all three; `_cron` honors them all).

---

## Files

| Path | Purpose |
| --- | --- |
| `~/.config/lipost-bot/config.json` | Settings (chmod 600). |
| `~/.config/lipost-bot/state.json` | `last_run_at`, `paused` (chmod 600). |
| `~/.config/lipost-bot/posts.jsonl` | Append-only post history: `{urn, posted_at, exit_code, draft_slug, source_image}`. |
| `~/Library/Logs/lipost-bot.log` | Wrapper logs and lipost stdout/stderr. |
| `~/code/lipost-bot/images/pending/` | Drop staged images here for `generate` to consume. |
| `~/code/lipost-bot/images/skipped/` | Images Claude said SKIP on, with `<filename>.reason.txt` sidecars. |
| `~/code/lipost-bot/drafts/<slug>/` | One directory per draft: `image.<ext>` + `meta.json` (status, caption, alt, timestamps, URN). |
| `~/code/lipost-bot/style.md` | Baked-in formatting & punctuation layer (committed in repo, app-controlled). |
| `~/code/lipost-bot/user_style.md` | Optional user additions to the style layer (gitignored). |
| `~/code/lipost-bot/local.lipost-bot.plist` | Generated launch agent (gitignored). |
| `~/.local/bin/lipost-bot` | Symlink to the CLI for `PATH`. |
| `~/Library/LaunchAgents/local.lipost-bot.plist` | Symlink to the repo plist. |

### Draft `meta.json` schema

```json
{
  "slug": "20260430T093412_a1b2c3",
  "status": "pending_approval",
  "caption": "...",
  "alt_text": "...",
  "image_filename": "image.png",
  "source_filename": "diagram.png",
  "created_at": "...",
  "approved_at": null,
  "rejected_at": null,
  "posted_at": null,
  "urn": null,
  "model": "claude-opus-4-7"
}
```

You can edit `meta.json` directly if you want to bulk-approve, fix typos, or change a draft's status — the wrapper trusts what's on disk.

---

## Deeper testing

The Step-4 walkthrough is the recommended path. To exercise individual safety paths:

```bash
# 1. Inactive path — `run` ignores it, so use `_cron`:
lipost-bot config active false
lipost-bot _cron        # logs "inactive (config.active=false) — skipping run"
tail -3 ~/Library/Logs/lipost-bot.log
lipost-bot config active true

# 2. Pause path:
lipost-bot pause
lipost-bot _cron        # logs "paused — skipping run"
lipost-bot resume

# 3. Min-gap path (after a successful post):
lipost-bot _cron        # logs "only X.Xh since last run … skipping"

# 4. Empty-queue path:
lipost-bot _cron        # if no approved drafts: logs "no approved drafts in queue"

# 5. Dependency-missing path (try without lipost installed):
mv ~/.config/linkedin-cli/token.json ~/.config/linkedin-cli/token.json.bak
lipost-bot _cron        # logs "missing token.json"
mv ~/.config/linkedin-cli/token.json.bak ~/.config/linkedin-cli/token.json

# 6. Generate failure path — bad JSON output (rare; Claude usually complies):
# Edit prompt.md to instruct Claude badly, run `lipost-bot generate`, see the
# "no valid JSON in claude output" error per image.
```

---

## Safety notes

- **`bypassPermissions` is wide.** `generate` runs Claude with `--permission-mode bypassPermissions` so it doesn't hang on prompts. Inside the working directory (`~/code/lipost-bot`) Claude can read/write files and run shell commands without confirmation. Don't keep secrets there.
- **Human approval is the safety boundary.** Nothing posts until you `review` and approve. Pre-fire, you also still have `lipost delete` if you change your mind after a post lands.
- **The LinkedIn token expires.** Member tokens last ~60 days. When yours expires, `lipost post` will 401 and `_cron` will leave the draft as `approved` for retry. Re-run `lipost auth`.
- **No retries inside a run.** `_cron` records `last_run_at` *before* posting. If `lipost post` fails or no URN comes back, the draft stays `approved` for the next scheduled run — but the gap guard means the next attempt is ~24h later.
- **5-minute hard timeout on the lipost call.** If LinkedIn hangs, `_cron` aborts and the draft stays `approved`.
- **10-minute timeout on each Claude call** during `generate`. A timeout leaves the source image in `pending/` for retry.
- **Prompt injection via images.** Claude reads images you stage. A maliciously crafted image with embedded text could try to override your prompt. Stage images you trust.

---

## License

MIT — see [LICENSE](LICENSE).
