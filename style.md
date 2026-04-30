# Formatting & punctuation layer

These rules describe HOW the text should look — typography, whitespace, character set, length. They do NOT describe voice, tone, topic, or content. Apply them on top of any voice rules in the main prompt.

They apply to BOTH the `caption` and the `alt` text in the JSON output.

## Punctuation

- Use straight ASCII quotes only: `'` (apostrophe) and `"` (double quote). Never smart/curly quotes (`'` `'` `"` `"`).
- Em-dashes: use `—` (U+2014, em-dash) with NO surrounding spaces. Never `--`. Never `-` in place of an em-dash.
- En-dashes (`–`, U+2013) only for numeric or temporal ranges (`9–5`, `Q1–Q3`, `2024–2025`). Otherwise, prefer the em-dash or a hyphen.
- One space after sentence-ending periods, never two.
- No trailing punctuation on lines that don't need it. Don't end the caption with `…` unless the meaning genuinely trails off.

## Whitespace

- No leading or trailing whitespace on the caption or alt text.
- Use a single `\n` between paragraphs. LinkedIn renders single newlines as soft breaks.
- Avoid more than one consecutive blank line.
- No tab characters.

## Characters

- Plain ASCII only, except these allowed Unicode marks where genuinely useful: `—` (em-dash), `–` (en-dash), `…` (horizontal ellipsis), `•` (bullet — only if a real list).
- No emoji unless the main prompt explicitly authorizes them. They read as a tell with technical audiences.
- No backslash escape sequences in the JSON string content (the wrapper handles JSON escaping).

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
