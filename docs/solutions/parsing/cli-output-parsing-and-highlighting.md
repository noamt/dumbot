---
title: "CLI output parsing, per-preset rendering, and inline syntax highlighting"
date: 2026-03-14
category: parsing
tags: [parser, unicode, highlighting, regex, ai-presets, rendering]
component: parse() / highlight() / render() / AI_PRESETS
severity: critical
symptoms:
  - Codex CLI output (using › U+203A prompt char) not recognized as user input
  - Double bullet chars when pasting CLI output that already contains • or ●
  - Multi-paragraph AI responses split into separate turns by blank lines
  - Inline highlight regex corrupts HTML by matching inside generated tags
  - All AI presets render identically regardless of selected bot
  - cont-none continuation lines still indented by 20px
---

# CLI Output Parsing, Per-Preset Rendering, and Inline Syntax Highlighting

## Problem

Dumbot's parser and renderer had six interrelated issues when handling real CLI output pasted from tools like Codex CLI and Claude Code:

1. Only ASCII `>` recognized as user prompt — Codex uses `›` (U+203A)
2. Pasted text already has `•`/`●` bullets — renderer adds another on top
3. Blank lines within a single AI response split it into multiple turns
4. Inline highlight regex matches inside its own generated HTML tags
5. No visual differentiation between AI presets (claude, codex, gemini)
6. `cont-none` CSS class still had `padding-left: 20px`

## Root Causes

### 1. Unicode prompt char

The parser only checked `ln.startsWith('> ')`. Codex CLI outputs `›` (charCode 8250), a completely different Unicode codepoint.

### 2. Double bullets

The renderer unconditionally prepended a styled `<span class="ai-bullet">` to every AI turn. Pasted text like `● Context budget check:` already has a bullet, producing `● ● Context budget check:`.

### 3. Paragraph splitting

The parser treated every blank line as a turn delimiter. In real CLI output, AI responses contain blank lines between paragraphs — these should be spacing within one turn, not separate turns.

### 4. Highlight regex corruption

The `highlight()` function ran in this order: (1) bold `**text**` → `<strong>text</strong>`, (2) path regex. The path regex `/[\w:.-]+` then matched `/strong` inside `</strong>`, corrupting the HTML output.

### 5. Identical presets

All AI label presets only changed the text label, not the visual rendering style.

### 6. CSS padding leak

The `.turn-ai.cont-none .continuation` rule had `padding-left: 20px` despite `cont-none` implying no continuation styling.

## Solution

### 1. Unicode prompt recognition

```javascript
function isUserLine(ln) {
  return ln.startsWith('> ') || ln === '>'
      || ln.startsWith('\u203A ') || ln === '\u203A';
}
function stripUserPrompt(ln) {
  return ln.replace(/^[>\u203A]\s?/, '');
}
```

### 2. Conditional bullet rendering

```javascript
const BULLET_RE = /^[•●◆▸►‣⦿]\s*/;

// In render():
const hasBullet = BULLET_RE.test(firstLine);
if (ps.aiPrefix === 'bullet' && hasBullet) {
  h += `<span class="ai-bullet">&#9679;</span>`;
}
// Strip original bullet from text:
const text = j === 0 && hasBullet ? l.replace(BULLET_RE, '') : l;
```

Only adds a styled bullet when the source text had one. No bullet in source = no bullet in output.

### 3. Look-ahead paragraph merging

```javascript
// In parse(), when encountering a blank line during an AI turn:
else if (cur?.type === 'ai') {
  let j = i + 1;
  while (j < lines.length && !lines[j].trim()) j++;
  const next = lines[j];
  if (!next || isUserLine(next) || BULLET_RE.test(next)) {
    // Next content is a user prompt or new bullet — split the turn
    turns.push(cur); cur = null; turns.push({type:'gap'});
  } else {
    // Blank within AI response — keep as paragraph spacer
    cur.lines.push('');
  }
}
```

Three conditions end an AI turn: end of input, next line is a user prompt, or next line starts with a bullet (new AI thought). Otherwise blank lines become `<div class="ai-para-gap">` for visual spacing.

### 4. Three-phase placeholder tokenization

```javascript
function highlight(text) {
  const tokens = [];
  let s = text;
  // Phase 1: extract backtick/bold as placeholders BEFORE escaping
  s = s.replace(/`([^`]+)`/g, (_, inner) => {
    tokens.push(`<span class="hl-cmd">${esc(inner)}</span>`);
    return `\x00${tokens.length - 1}\x00`;
  });
  s = s.replace(/\*\*([^*]+)\*\*/g, (_, inner) => {
    tokens.push(`<strong>${esc(inner)}</strong>`);
    return `\x00${tokens.length - 1}\x00`;
  });
  // Escape remaining plain text
  s = esc(s);
  // Phase 2: path/number highlighting on safe text (no HTML to corrupt)
  s = s.replace(/(?<![&\w])(\/?(?:[\w@.~-]+\/)+[\w@.~-]*|\/[\w:.-]+)/g,
    '<span class="hl-cmd">$1</span>');
  s = s.replace(/^(\s*\d+\.)\s/gm, '<span class="hl-num">$1</span> ');
  // Phase 3: restore placeholders
  s = s.replace(/\x00(\d+)\x00/g, (_, i) => tokens[+i]);
  return s;
}
```

Null-byte delimiters (`\x00index\x00`) are guaranteed not to appear in user content. Phase 2 regex runs on text containing only plain escaped text and inert placeholders — no HTML tags to accidentally match.

### 5. Per-preset rendering styles

```javascript
const AI_PRESETS = [
  { id: 'claude', promptStyle: 'accent', aiPrefix: 'bullet', continuation: 'none' },
  { id: 'codex',  promptStyle: 'bar',    aiPrefix: 'bullet', continuation: 'none' },
  { id: 'gemini', promptStyle: 'default', aiPrefix: 'label', continuation: 'indent' },
  { id: 'ai',     promptStyle: 'default', aiPrefix: 'label', continuation: 'border' },
];
```

- `promptStyle`: `accent` = left border + subtle bg (Claude), `bar` = full row highlight (Codex), `default` = plain
- `aiPrefix`: `bullet` = ● prefix, `label` = small-caps label text
- `continuation`: `none` = flat, `border` = left border line, `indent` = indented

### 6. CSS fix

```css
.turn-ai.cont-none .continuation { display: block; padding-left: 0; }
```

## Key Principles

1. **Canonicalize early.** Normalize Unicode variants at ingestion, not scattered through `if` checks.
2. **Parser should not know what HTML looks like.** Keep HTML confined to the final render step.
3. **Regex on its own output is always a bug.** Use placeholder tokenization to break the cycle.
4. **Absence is not a signal.** Parse on the presence of structural markers (user prompts, bullets), not on gaps (blank lines).
5. **Names are contracts.** If a CSS class says `none`, the implementation must honor it.

## Testing Checklist

When changing the parser:
- [ ] Paste from Codex CLI (uses `›`), Claude Code (uses `>`), and plain text — all user prompts recognized
- [ ] AI response with multiple blank-line-separated paragraphs stays as one turn
- [ ] Content containing literal `> ` inside AI text doesn't split the response

When changing the highlighter:
- [ ] Text with `**bold**`, `` `backtick code` ``, `/slash-commands`, and `src/paths/file.ts` all highlight correctly
- [ ] No raw HTML tags visible in rendered output
- [ ] Pasted text with existing `•`/`●` bullets doesn't double up

When changing CSS:
- [ ] `1.` and `2.` list items at same indent level in `cont-none` mode
- [ ] Each AI preset (claude/codex/gemini/ai) renders with visually distinct style
