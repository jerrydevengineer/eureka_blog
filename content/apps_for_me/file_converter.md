# AI랑 Rust로 Markdown-to-PDF Converter 만들기

저는 기록용(절대 다시 안 읽지만)으로 옵시디언을 사용 중입니다. 옵시디언은 마크다운으로 파일을 저장하는데 아이패드나 이북리더기에서 읽기 불편한 단점이 있었습니다. 그래서 옵시디언 기본 pdf로 변환을 자주 썼는데 이렇게 만들면 이북리더기에서는 보기가 좀 힘들더라구요. 그래서 클코랑 rust를 사용해서 같이 하나 만들어 봤습니다.

---

## 문제점

저는 저를 위해서 작성하는 기술 노트들이 너무 많아지고 있었어요. 마크다운은 작성하고 모니터에서 보는건 편한데 이북리더기에서는 읽을 수 있는 어플도 찾기 힘들더라구요. 그래서 옵시디언 기본 기능으로 pdf를 만들면 이북리더기에서 읽기에는 글씨도 작고 코드 블록이나 이런게 다 이상하게 나오고 색깔도 옵시디언 현재 색상을 따라가 버려서 이북리더기에서 읽을 수가 없었습니다!

그래서 필요한 점만 말하면:
- 깔끔하고 조절 가능한 글씨
- pdf로 변환했을 때 깨지지 않는 코드 블럭
- 설정 파일을 만들어서 원할 때만 적용 가능
- 폴더에서 커맨드 라인으로 실행 가능

정도였습니다.

## Why Rust


솔직히 말하면 그냥 써보고 싶었습니다. 저는 백엔드 개발자인데 Rust는 한 번도 다뤄본 적이 없었거든요. 이 프로젝트는 완성하기에 충분히 작으면서도, 실제로 배울 수 있을 만큼 충분히 복잡해 보였어요.

또 다른 이유는 실용적인 측면이었습니다. Rust는 런타임이 필요 없는 단일 네이티브 바이너리를 생성합니다. 따라서 빌드가 완료되면 Node, Python 또는 JVM을 설치하지 않고도 어디서든 `file-converter`를 실행할 수 있습니다.

## 어떻게 돌아가는지

파이프라인은 간단합니다:

```
Markdown → HTML (pulldown-cmark) → PDF (headless Chrome)
```

반대도 되긴 합니다다:

```
PDF → text + font sizes (PDFium) → classified blocks → Markdown
```

순방향 처리를 위해 저는 [pulldown-cmark](https://github.com/raphlinus/pulldown-cmark)를 사용하여 Markdown을 HTML로 파싱하고 이를 전체 CSS 템플릿에 삽입한 다음, Chrome 개발자 도구 프로토콜을 통해 헤드리스 Chrome에 HTML을 전달합니다. Chrome의 PDF 인쇄 기능은 진짜 좋더라구요. 대부분의 경우 페이지 매김, `@page` 규칙, `page-break-inside: avoid`를 정확하게 처리합니다.

반대 방향(PDF → Markdown)으로 변환할 때는 PDFium을 사용하여 각 텍스트 객체를 실제 글꼴 크기로 읽고, 글꼴 크기 임계값을 기준으로 블록을 제목 또는 본문으로 분류한 다음 Markdown 구조를 재구성합니다.

JSON config file에서 디자인 수정 가능합니다:

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

이 파일은 아무데나 넣고 마지막에 경로로 옵션 추가할 수 있습니다. pdf 파일도 원하는 경로에 적으면 거기에 생성됩니다.

## Commands

```bash
# Single file
file-converter md2pdf notes.md notes.pdf --config ebook-theme.json

# Merge multiple files into one PDF (ebook 만들 때 좋음, ebook-theme-big.json을 저는 씁니다)
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
