# AI랑 Rust로 Markdown-to-PDF Converter 만들기

저는 기록용(절대 다시 안 읽지만)으로 옵시디언을 사용 중입니다.
I had a simple problem: I write technical notes in Markdown, but I wanted to read them comfortably on my ebook reader. Every tool I tried either produced ugly PDFs or gave me no control over the styling. So I decided to build my own — in Rust, a language I had never used before.

This is the story of how that went, what I learned, and the tool I ended up with.

---

## The Problem

I have a growing collection of technical notes — database internals, system design references, guides I write for myself. Markdown is great for writing. But reading a raw `.md` file on an ebook reader is not great. And most Markdown-to-PDF tools produce output that looks fine on a monitor but is painful on an e-ink display: tiny fonts, harsh white backgrounds, code blocks that overflow the page.

I wanted:

- Clean, readable typography (a serif font like Georgia, warm background)
- Code blocks that don't break mid-line or mid-diagram
- Full control over every font size, color, and margin — via a config file
- A tool I could run from the command line on any folder

## Why Rust

Honestly, partly curiosity. I'm a backend developer and had never touched Rust. This felt like a project small enough to finish but complex enough to actually learn from.

The other reason was practical: Rust produces a single native binary with no runtime. Once built, I can run `file-converter` anywhere without installing Node, Python, or JVM.

## How the Tool Works

The pipeline is simple:

```
Markdown → HTML (pulldown-cmark) → PDF (headless Chrome)
```

And in reverse:

```
PDF → text + font sizes (PDFium) → classified blocks → Markdown
```

For the forward direction, I use [pulldown-cmark](https://github.com/raphlinus/pulldown-cmark) to parse Markdown into HTML, inject it into a full CSS template, and hand the HTML to headless Chrome via the Chrome DevTools Protocol. Chrome's print-to-PDF is surprisingly good — it handles pagination, `@page` rules, and `page-break-inside: avoid` correctly in most cases.

For the reverse direction (PDF → Markdown), I use PDFium to read each text object with its actual font size, classify blocks as headings or body text based on font size thresholds, and reconstruct the Markdown structure.

All styling is driven by a single JSON config file:

```json
{
  "font_family": "Georgia, 'Times New Roman', serif",
  "body_font_size": 19.0,
  "code_font_size": 11.0,
  "background_color": "#faf9f7",
  "margin_left": "12mm",
  "margin_right": "12mm"
}
```

You can place this config file anywhere and reference it by full path. The output PDF can go anywhere too — no need to copy config files around.

## Commands

```bash
# Single file
file-converter md2pdf notes.md notes.pdf --config ebook-theme.json

# Merge multiple files into one PDF (great for ebooks)
file-converter merge2pdf ch1.md ch2.md ch3.md --output book.pdf --config ebook-theme.json

# Extract text from a PDF back to Markdown
file-converter pdf2md report.pdf recovered.md
```

---

## The Hard Parts

### Code Blocks on Ebook Readers

This took more debugging than I expected. There are three separate problems that all interact:

**1. ASCII art breaks at the wrong character.**

I had `word-break: break-all` in my CSS, which breaks at *any* character — including mid-draw in box-drawing characters like `──────────┐`. Switching to `overflow-wrap: break-word` fixes this: it only breaks at whitespace when it must.

**2. Code blocks are too narrow.**

Body padding plus page margins leave maybe 544px of usable width — about 70 characters at 13px mono font. Database diagrams and SQL examples routinely hit 80-90 characters. The fix: negative margins on `<pre>` blocks extend them beyond the body content area, reclaiming the padding on both sides.

```css
pre {
    margin: 10px -44px 14px;  /* extends 88px wider than the body */
}
```

**3. Large code font size breaks pagination.**

I was using 17px for code to match a large body font. A 34-line ASCII diagram at 17px is about 895px tall — taller than a single A4 page (≈625px usable). Chrome ignores `page-break-inside: avoid` when an element physically cannot fit on one page, so it splits the diagram across two pages mid-draw.

The solution is to keep `code_font_size` at 11px regardless of how large the body font is. Code is read structurally, not word-by-word — alignment and whitespace matter more than size. At 11px you get ≈98 characters per line, and even a 34-line block fits on one page.

### Learning Rust Without a Rust Background

I had no Rust experience going into this. The ownership model felt strange at first — every function signature is explicit about who owns what and who can borrow. But it turns out this is exactly what makes the code reliable: the compiler catches a whole class of bugs at compile time that I would have spent hours debugging in other languages.

The patterns I relied on most:

- `Option<T>` and `Result<T, E>` instead of null/exception — every fallible operation is explicit in the type
- The `?` operator for propagating errors without nested `match` blocks
- `serde` with derive macros for JSON config deserialization in about 5 lines of code
- `clap` derive macros for the CLI — the entire argument parser is just struct definitions with attributes

---

## The Ebook Theme

After a lot of iteration, my go-to config for comfortable long reading:

| Setting | Value | Why |
|---|---|---|
| Font | Georgia serif | Reduces eye strain vs sans-serif for long text |
| Background | `#faf9f7` | Warm white — less harsh than pure `#ffffff` |
| Body size | 19px | Comfortable on a 6" ebook display |
| Code size | 11px | Keeps diagrams intact and lines from wrapping |
| Code background | `#f1f5f9` | Light theme — better on e-ink than dark |
| Margins | 12mm left/right | More content width, essential for code blocks |

---

## What I'd Do Differently

**Start with the CSS constraints, not the design.** I spent time making the output look good on screen before testing on the ebook reader. The constraints of an ebook display (e-ink, A4 page, limited width for code) should drive the CSS decisions from day one.

**Keep code font size independent of body font size.** These two sizes serve completely different purposes and should not scale together.

**Test with real content, not toy examples.** The ASCII diagrams and long SQL lines in my actual notes exposed problems that a simple "hello world" Markdown file would never hit.

---

## The Result

A single Rust binary that:

- Converts any Markdown file (or multiple files merged) to a well-styled PDF
- Accepts a JSON theme config for full typographic control
- Produces PDFs that are genuinely comfortable to read on an ebook reader
- Can reverse-convert a PDF back to rough Markdown using font-size-aware heading detection

The whole project is about 500 lines of Rust across five files. Most of the complexity lives in the CSS and the config system, not the Rust itself.

---

## Source

The full source is at `github.com/jerrydevengineer/file-converter` *(update this link)*.

The two theme configs I use most are `ebook-theme.json` (13px body, compact) and `ebook-theme-big.json` (19px body, for when I want larger text with 11px code).

---

*Built with Rust, headless Chrome, PDFium, and a lot of trial and error on a 6" ebook display.*
