# Claude랑 Tauri로 이북 어플 만들기

저는 이북 리더기로 boox 포크 6를 쓰는데 킨들, readera, eBoox를 씁니다. 근데 셋 다 쓸 때마다 너무 느린거 같아서 잡다한 기능 다 빼고 그냥 딱 보기만 하기 위한 어플을 만들어보기로 했습니다. 처음부터 클로드 코드랑 만들었고 코드는 보고 이상한 부분 말하는 정도로 했습니다. Rust와 Tauri v2를 사용해서 만들었습니다.

---

## 기능

현재는 윈도우 데스크탑, 안드로이드에서 테스트 완료됐습니다. PDF, epub, md 파일들 테스트 완료됐습니다.

- **PDF** — page-by-page 렌더링, zoom in/out, 전문 검색
- **EPUB** — 챕터 네비게이션, 목차, 전문 검색
- **Markdown** — 마크다운 형식 뷰어, 검색

읽기 기능 외에도 기본적인 도서관 관리 시스템을 갖추고 있습니다:

- 디바이스 내 폴더를 스캔할 수 있습니다.
- **Shelves** — 책장에 이름을 붙이고 원하는 책을 넣을 수 있습니다.
- **Reading progress** — 어느 페이지/챕터에서 나갔는지 기억합니다.
- **Bookmarks** — 책에 책갈피를 만들 수 있습니다.
- **Multi-select** — 여러 책 선택 후 책장에 넣을 수도 있습니다.
- **Font size control** (for EPUB/Markdown) and **zoom** (for PDF)

---

## 스택

| Layer | Technology |
|---|---|
| Framework | [Tauri v2](https://tauri.app/) |
| Frontend | TypeScript (vanilla — no React/Vue) |
| Backend | Rust |
| Database | SQLite via `rusqlite` |
| Targets | Windows, Android |

Tauri는 WebView가 포함된 네이티브 창을 제공합니다. 프런트엔드(TypeScript)는 모든 UI를 처리하고, 백엔드(Rust)는 파일 입출력, PDF 렌더링, EPUB 파싱, Markdown 렌더링 및 SQLite 데이터베이스와 같은 핵심 기능을 처리합니다.

---

## 만든 과정

앱을 만들기 전에 계획을 페이즈 별로 세웠는데, 러프하게 보여드리면:

**Phase 1 — First commit**
기본 골격으로 Tauri가 뜨고 창이 보이는 세팅만 했습니다.

**Phases 2–5 — Core reader**
- PDF 렌더링을 Rust로 만들고, TypeScript로 찾아갈 수 있게 했습니다.
- EPUB 파일 파싱하고 챕터 렌더링을 했습니다.
- Markdown 렌더링을 했고 코드 블럭에서는 `highlight.js`를 사용했습니다.
- 폴더 스캔하는 부분이랑 책 리스트 부분을 만들었습니다.

**Android port**
Tauri v2는 안드로이드를 기본적으로 지원합니다. `npm run tauri android init` 명령어를 실행하면 안드로이드 스튜디오 프로젝트가 생성됩니다. 대부분의 작업은 "테스트한 부분은 잘 작동"했으며, WebView는 안드로이드에서도 동일하게 렌더링됩니다. 주요 문제는 파일 시스템 접근이었습니다.

**SD card fix**
저는 모든 책들을 SD 카드에 저장했습니다. 안드로이드의 표준 파일 선택기는 Tauri가 직접 읽을 수 없는 `content://` URI를 반환합니다. 해결책은 Rust의 `list_directory` 명령과 통신하는 맞춤형 앱 내 파일 브라우저를 사용하는 것입니다. 이 브라우저는 저장소 루트(내부 저장소의 경우 `/storage/emulated/0`, SD 카드의 경우 `/storage/XXXX`)를 감지하고 실제 파일 시스템을 탐색할 수 있도록 합니다.

**Select mode**
라이브러리에 다중 선택 기능을 추가했습니다. 이런거 안하려고 했는데 이거마저 안하면 사용성이 너무 떨어졌습니다.

**Bookmarks and progress bar**
책갈피는 SQLite에 저장됩니다. 진행률 표시줄은 화면 위에 표시되며 페이지/챕터를 넘길 때마다 업데이트됩니다.

**App icon**
PowerShell의 `System.Drawing`을 사용하여 1024x1024 크기의 마스터 아이콘을 생성했습니다. Photoshop이나 npm 패키지는 사용하지 않았습니다. 파란색 모서리가 둥근 직사각형 배경에 펼쳐진 책과 노란색 책갈피 리본이 있는 아이콘입니다. 그런 다음 `npm run tauri icon icon_source.png` 명령어를 실행하여 모든 플랫폼(Windows `.ico`, macOS `.icns`, Android `mipmap-*`, iOS `AppIcon-*`)에 대한 50개 이상의 다양한 크기 아이콘을 자동으로 생성했습니다.

---

## 느낀 점

**Tauri는 모바일로 진짜 좋다**
Tauri v2 Android 타겟은 놀라울 정도로 안정적입니다. 저는 앱 개발은 처음이고 tauri도 처음인데 window 먼저 테스트 후 android 테스트를 했는데 진짜 환경 `invoke()` 브리지(TypeScript → Rust)는 Android에서도 동일하게 작동합니다. `npm run tauri android build` 명령어를 실행하면 제대로 된 네이티브 APK 파일을 생성할 수 있습니다.

**Rust for file I/O is the right call**
The TypeScript frontend never touches the filesystem directly. All file reading, directory scanning, and database access goes through Rust commands. This is the right architecture — Rust handles errors properly and the code is fast.

**SQLite is perfect for local app state**
Progress, bookmarks, shelves, folder list — all in one SQLite file stored in the app's data directory. No backend server, no sync, no account required. It just works.

**Vanilla TypeScript is underrated**
No React, no Vue, no state management library. Just TypeScript with DOM APIs. The app has a library view and a reader view — the state fits in a few module-level variables. Adding a framework would have added complexity without benefit.

**Android's filesystem is a maze**
The standard file picker doesn't work well for e-readers because you want to pick a *folder* of books, not a single file. And `content://` URIs don't give you real paths. The custom file browser was the right solution.

---

## What's Next

The next target is **iOS**. Tauri v2 supports iOS in the same way it supports Android — `npm run tauri ios init` generates an Xcode project. The main catch: iOS requires a Mac to compile (Xcode only runs on macOS). The plan is to build on a Mac and test on a real device.

The code is essentially ready. It's the build environment that needs to move.