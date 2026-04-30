# Formatting & punctuation layer

These rules describe HOW the text should look — typography, whitespace, character set, length. They do NOT describe voice, tone, topic, or content. Apply them on top of any voice rules in the main prompt.

They apply to BOTH the `caption` and the `alt` text in the JSON output.

## Capitalization

- Capitalize the first word of every sentence. Use proper sentence case throughout.
- Normal capitalization for proper nouns, brand names, and acronyms (`LLM`, `RAG`, `OpenAI`, `GPU`, `p99`).
- Do not use ALL CAPS for emphasis. Use word choice instead.

## Punctuation

- End every sentence with `.`, `?`, or `!` — pick the right one. No bare sentence fragments masquerading as full sentences.
- Use straight ASCII quotes only: `'` (apostrophe) and `"` (double quote). Never smart/curly quotes (`'` `'` `"` `"`).
- Em-dashes: use `—` (U+2014, em-dash) with NO surrounding spaces. Never `--`. Never `-` in place of an em-dash.
- En-dashes (`–`, U+2013) only for numeric or temporal ranges (`9–5`, `Q1–Q3`, `2024–2025`). Otherwise, prefer the em-dash or a hyphen.
- One space after sentence-ending periods, never two.
- Don't end the caption with `…` unless the meaning genuinely trails off.

## Sentence and paragraph structure

- Break the post into paragraphs at logical sentence boundaries — wherever a new idea or angle starts. A short post split into 2–3 short paragraphs reads stronger than one wall of text.
- One idea per paragraph. When the thought shifts, start a new paragraph.
- Don't manually break lines inside a paragraph; let the renderer wrap.

## Whitespace

- No leading or trailing whitespace on the caption or alt text.
- Use a **blank line between paragraphs** — that's two consecutive newline characters (`\n\n`) in the JSON string. LinkedIn renders this as a paragraph break with visible vertical space, which is what we want.
- A single `\n` (soft break, no blank line) is allowed when you genuinely want a line break inside a thought, but prefer paragraph breaks (`\n\n`) for clarity.
- Never more than one blank line in a row (no `\n\n\n`).
- No tab characters.

## Characters

- Plain ASCII only, except these allowed Unicode marks where genuinely useful: `—` (em-dash), `–` (en-dash), `…` (horizontal ellipsis), `•` (bullet — only if a real list), and emoji (see below).
- No backslash escape sequences in the JSON string content (the wrapper handles JSON escaping).

## Emoji

- **Use sparingly: at most 1–2 emojis per post**, and only when an emoji genuinely adds something a word can't.
- Don't decorate (no `🚀 here's the thing 🚀`). Don't use emoji as bullets. Don't substitute an emoji for a word that would carry the meaning.
- If in doubt, leave it out. A serious post with zero emoji reads stronger than a serious post with one bad one.
- Never put an emoji in the alt text — alt text is for screen readers, where emoji read as their full ARIA label and clutter the description.

## Markdown

- LinkedIn does NOT render markdown. `*bold*`, `_italic_`, `# headings`, `` `code` ``, `[links](url)` all display as literal characters. **Do not use any markdown syntax in the caption or alt text.**
- For emphasis, use word choice or sentence structure, not formatting marks.
- URLs go raw. LinkedIn auto-linkifies them.

## Numbers and units

- Use digits for technical or quantitative content: `10x`, `40%`, `p99`, `3am`, `100ms`, `5MB`.
- Spell out numbers only when they appear at the start of a sentence — and try to restructure to avoid that.
- Keep units adjacent to the number with no space: `5m`, `40%`, `100ms`. (Exception: SI symbols where conventional, e.g. `5 GB`.)
- Acronyms use the form your audience uses: `LLM` not `L.L.M.`, `RAG` not `R.A.G.`.

## Length

- Keep the caption under 1300 characters to avoid LinkedIn's "...see more" truncation in feed previews. Aim for tighter — under 600 reads better.
- Keep alt text under 250 characters and on a single line (no internal newlines).

## Alt text specifics

- Alt text describes the image factually, in present tense.
- No closing period required.
- No prefix like "image of" or "picture of" — assistive tech already announces it as an image.
- No interpretive language ("a beautiful sunset"); state what's there ("a sunset over an ocean horizon").
