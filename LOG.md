# Math Rendering Fixes: Jekyll to Hugo Migration

## Context

Site migrated from Jekyll (Kramdown + MathJax) to Hugo (Goldmark + KaTeX).
Hugo's Goldmark markdown processor does NOT have a math passthrough extension enabled,
so it processes content inside `$$...$$` and `$...$` as regular markdown before KaTeX sees it.

## Issues and Fixes

### 1. Inline math: `$$...$$` -> `$...$`

**Problem**: Jekyll/Kramdown uses `$$...$$` for both inline and display math. Hugo/KaTeX uses `$...$` for inline and `$$...$$` for display.

**Fix**: Replace `$$...$$ ` with `$...$` when the math appears inline (within paragraph text). Keep `$$...$$` for display math (on its own line or as sole content of a line).

**How to detect**: Grep for `\$\$` in `.md` files. If a `$$...$$` pair appears on a line with other non-math text, it's inline and needs to become `$...$`.

**Files fixed**: tsgctf2021, pbctf2021, pbctf2020, ethernaut-ctf-writeups-1, ethernaut-ctf-writeups-2

### 2. Kramdown syntax: `{:target="_blank"}`

**Problem**: `{:target="_blank"}` is Kramdown (Jekyll) link attribute syntax, not supported in Hugo/Goldmark. It renders as raw text.

**Fix**: Remove all `{:target="_blank"}` occurrences.

**How to detect**: Grep for `{:target=` in `.md` files.

**Files fixed**: pwn2winctf-2020, tsgctf-2020, XMASCTF-2019, inctfi-2020, zer0ptsctf-2020, m0leconctf-2020, inctfi-2019-huygens-gebouw

### 3. Matrix row separators: `\\` -> `\\\\`

**Problem**: In LaTeX, `\\` is a row separator inside `\begin{bmatrix}` / `\begin{pmatrix}`. Goldmark interprets `\\` as an escaped backslash and outputs just `\`, so KaTeX never sees the row break. The matrix renders as a single line.

**Fix**: Use `\\\\` in the markdown source. Goldmark reduces `\\\\` to `\\`, which KaTeX interprets as a row break.

**How to detect**: Look for `\begin{bmatrix}` or `\begin{pmatrix}` environments that are NOT inside `<center>` HTML blocks (HTML blocks are not processed by Goldmark). Check if `\\` is used for row breaks.

**Exception**: Matrices inside `<center>...</center>` tags are treated as raw HTML by Goldmark and are NOT processed, so their `\\` passes through correctly.

**Files fixed**: zkhack-I-lets-hash-it-out

### 4. Underscore in variable names: `\_` inside display math

**Problem**: When a variable name contains a literal underscore (e.g., `known_constant`) inside display math that is NOT wrapped in an HTML block (`<center>`), Goldmark's emphasis rules can match the `_` with later `_` subscripts in the equation, wrapping part of the equation in `<em>` tags and completely breaking KaTeX rendering.

Specifically:
- `\_` in markdown: Goldmark treats `\_` as escaped underscore, outputs bare `_`. KaTeX sees `_` as subscript (wrong, but renders).
- `\\_` in markdown: Goldmark processes `\\` -> `\`, then bare `_` remains. This `_` can trigger emphasis matching with later `_{...}` subscripts, corrupting the equation with `<em>` tags (completely breaks rendering).
- `\\\_` in markdown: Goldmark processes `\\` -> `\`, then `\_` -> `_` (escaped, no emphasis). KaTeX sees `\_` -> literal underscore (correct).

**Fix**: Use `\\\_` (three backslashes) before underscores in variable names inside display math blocks that are NOT wrapped in HTML tags.

**Exception**: Inside `<center>...</center>` HTML blocks, use single `\_` since Goldmark doesn't process HTML block content.

**Files fixed**: tsgctf2021 (`known_constant` equation)

### 5. Asterisk as multiplication: `*` in display math

**Problem**: `*` characters inside display math can trigger Goldmark's emphasis matching. For example, `2^{8*i}` contains `*i` where `*` is both left-flanking and right-flanking. If another `*` appears later in the same equation (e.g., `8*i} ... 8*i}`), Goldmark matches them as emphasis delimiters and wraps the content in `<em>` tags.

**Fix**: Replace `*` with `\cdot` for multiplication in display math. This is also better LaTeX practice.

**Exception**: Inside `<center>...</center>` HTML blocks or code blocks, `*` passes through safely.

**Files fixed**: tsgctf2021

### 6. Escaped underscores/braces in subscripts (zkhack soundness-of-music)

**Problem**: LaTeX subscripts like `L_j(\tau)` where the subscript is at a word boundary can sometimes interact with Goldmark's emphasis rules. Also `\{...\}` for set notation - Goldmark processes `\{` as escaped brace -> `{`.

**Fix**: Use `L\_j` to prevent Goldmark emphasis, and `\\{...\\}` for literal braces that KaTeX needs to see as `\{...\}`. Added blank lines between consecutive `$$` display math blocks to prevent them from merging.

**Files fixed**: zkhack-I-soundness-of-music

## Root Cause

All issues stem from Goldmark processing markdown syntax inside math delimiters. The proper long-term fix is to enable Hugo's passthrough extension (requires Hugo 0.122+):

```yaml
markup:
  goldmark:
    extensions:
      passthrough:
        enable: true
        delimiters:
          block:
            - - "$$"
              - "$$"
          inline:
            - - "$"
              - "$"
```

This tells Goldmark to skip content between math delimiters entirely, passing it through to KaTeX unchanged. This would eliminate issues 3, 4, 5, and 6 above.

## Key Rule: `<center>` HTML blocks are safe

Equations wrapped in `<center>...</center>` are treated as HTML blocks by Goldmark. Their content is NOT processed as markdown, so `\\`, `\_`, `*`, `\{` all pass through to KaTeX correctly. This is why many equations in pbctf2020.md work fine despite having `\\` row separators in matrices.
