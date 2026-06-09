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

# PDF에서 Markdown(고질적인 문제점은 여전히 있으니 완벽하진 않아요! 특히 표나 코드 블럭 안에 그림림)
file-converter pdf2md report.pdf recovered.md
```

---

## 어려웠던 점

### Code Blocks on Ebook Readers

생각보다 이 부분에서 디버깅이 많았습니다. 3가지 문제가 있었습니다:

**1. ASCII 아트가 잘못된 문자에서 끊어짐.**

제 CSS에 `word-break: break-all`이라는 속성이 있었는데, 이 속성은 `──────────┐`와 같은 박스 드로잉 문자를 포함하여 *모든* 문자에서 줄 바꿈을 발생시켰습니다. `overflow-wrap: break-word`로 바꾸니 이 문제가 해결되었습니다. 필요한 경우에만 공백에서 줄 바꿈이 적용됩니다.

**2. Code block이 너무 좁음**

본문 패딩과 페이지 여백을 더하면 실제로 사용할 수 있는 너비는 약 544px 정도입니다. 이는 13px 모노폰트 기준으로 약 70자 정도에 해당합니다. 데이터베이스 다이어그램이나 SQL 예제는 보통 80~90자를 넘기곤 합니다. 해결책은 `<pre>` 블록에 음수 여백을 적용하여 본문 콘텐츠 영역을 벗어나도록 확장하는 것입니다. 이렇게 하면 양쪽 패딩을 되찾을 수 있습니다.

```css
pre {
    margin: 10px -44px 14px;  /* extends 88px wider than the body */
}
```

**3. 코드 글꼴 크기가 너무 크면 페이지네이션이 깨집니다.**

본문 글꼴 크기를 크게 하려고 코드 크기를 17px로 설정했습니다. 17px 크기로 34줄짜리 ASCII 다이어그램을 그리면 높이가 약 895px가 되는데, 이는 A4 용지 한 장(실사용 가능 면적 약 625px)보다 높습니다. 크롬은 요소가 한 페이지에 물리적으로 맞지 않을 경우 `page-break-inside: avoid` 설정을 무시하고 다이어그램을 중간에 두 페이지로 나눠 표시합니다.

그래서 본문 글꼴 크기에 관계없이 `code_font_size`를 11px로 유지하는 것으로 해결했습니다. 코드는 단어 단위로 읽는 것이 아니라 구조적으로 읽히므로 글꼴 크기보다 정렬과 공백이 더 중요하다고 생각했습니다. 11px로 설정하면 한 줄에 약 98자를 표시할 수 있으며, 34줄짜리 코드 블록도 한 페이지에 들어갑니다.

### Rust에 대한 배경지식 없이 Rust 배우기

저는 Rust 경험이 전혀 없는 상태에서 이 프로젝트를 시작했습니다. 소유권 모델이 처음에는 낯설게 느껴졌습니다. 모든 함수 시그니처에 누가 무엇을 소유하고 누가 빌릴 수 있는지 명시되어 있었기 때문입니다. 하지만 알고 보니 바로 이 점 때문에 코드가 안정적이었습니다. 컴파일러가 컴파일 시점에 다른 언어였다면 몇 시간씩 디버깅해야 했을 다양한 종류의 버그를 잡아냈습니다. 진짜 런타임에서 잡는거보다 편하더라구요...

제가 가장 많이 의존했던 패턴들:

- null/예외 대신 `Option<T>` 및 `Result<T, E>` 사용 — 모든 오류 발생 가능한 연산은 타입에 명시적으로 표현됩니다.
- 중첩된 `match` 블록 없이 오류를 전파하기 위한 `?` 연산자
- `serde`와 `derive` 매크로를 사용하면 약 5줄의 코드로 JSON 설정 역직렬화를 수행할 수 있습니다.
- `clap`은 CLI용 매크로를 파생합니다. 전체 인자 파서는 속성을 가진 구조체 정의로만 구성됩니다.

---

## The Ebook Theme

수많은 시행착오 끝에, 저에게 장시간 독서에 편안한 최적의 설정은 이렇게 했습니다:

| Setting | Value | Why |
|---|---|---|
| Font | Georgia serif | 긴 글 읽을 때 눈이 편함 |
| Background | `#faf9f7` | 백색광 - `#ffffff`보다 편합니다 |
| Body size | 19px | 6" ebook display에서 적당한 듯... |
| Code size | 11px | 도표를 그대로 유지하고 선이 겹치는 것을 방지합니다. |
| Code background | `#f1f5f9` | 까만 코드 블럭은 이북리더기에서 잘 안보이더라구요 |
| Margins | 12mm left/right | 코드 블럭을 위해 좀 더 넓게 했습니다다 |

---

## 작업할 때 신경 쓴 점

**디자인이 아닌 CSS 제약 조건부터 시작했습니다.** 전자책 리더기에서 테스트하기 전에 화면에서 결과물이 보기 좋게 나오도록 시간을 들였습니다. 전자책 디스플레이의 제약 조건(전자잉크, A4 용지, 코드의 제한된 너비)을 고려하여 처음부터 CSS를 결정해야 했습니다.

**코드 글꼴 크기와 본문 글꼴 크기는 별개로 유지했습니다.** 이 두 크기는 완전히 다른 목적을 가지고 있으므로 함께 확대/축소되어서는 안 됩니다.

**실제 콘텐츠로 테스트하세요. 간단한 예제로 하면 안됩니다.** 제 실제 노트에 있는 ASCII 다이어그램과 긴 SQL 코드는 간단한 "hello world" 마크다운 파일에서는 절대 발생하지 않을 문제들을 드러냈습니다.

---

## 결과

단일 Rust 바이너리:

- 모든 마크다운 파일(또는 여러 파일을 병합한 파일)을 스타일이 잘 적용된 PDF로 변환합니다.
- 완벽한 타이포그래피 제어를 위해 JSON 테마 설정을 지원합니다.
- 전자책 리더에서 읽기 편한 PDF를 생성합니다.
- 글꼴 크기를 인식하는 제목 감지 기능을 사용하여 PDF를 다시 마크다운 형식으로 변환할 수 있습니다.

전체 프로젝트는 5개의 파일에 걸쳐 약 500줄의 Rust 코드로 구성되어 있습니다. 대부분의 복잡성은 Rust 코드 자체보다는 CSS와 설정 시스템에 있었습니다.

---

## Source

full source는 [`github.com/jerrydevengineer/md2pdf`](https://github.com/jerrydevengineer/md2pdf)에 있습니다.

주로 쓰는 테마는`ebook-theme.json` (13px body, compact) and `ebook-theme-big.json` (19px body)입니다.

---

*Built with Rust, headless Chrome, PDFium, and a lot of trial and error on a 6" ebook display.*
