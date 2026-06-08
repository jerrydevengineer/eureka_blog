---
date: 2026-06-08
---
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
Tauri v2 Android 타겟은 놀라울 정도로 안정적입니다. 저는 앱 개발은 처음이고 tauri도 처음인데 window 먼저 테스트 후 android 테스트를 했는데 진짜 환경으로 인한 에러는 없고 폴더 접근만 수정하면 됐습니다. `invoke()` 브리지(TypeScript → Rust)는 Android에서도 동일하게 작동합니다. `npm run tauri android build` 명령어를 실행하면 제대로 된 네이티브 APK 파일을 생성할 수 있습니다.

**file I/O를 위한 Rust 선택은 괜찮았다**
TypeScript 프런트엔드는 파일 시스템에 직접 접근하지 않습니다. 모든 파일 읽기, 디렉토리 스캔 및 데이터베이스 접근은 Rust 명령어를 통해 이루어집니다. 이것이 올바른 아키텍처입니다. Rust는 오류를 적절하게 처리하고 코드 속도를 향상시킵니다. 다른 어플이 사용하는 환경은 무엇인지 모르겠지만 체감 상 훨씬 빠른 속도를 보여줍니다.

**SQLite는 로컬 앱 상태를 저장하는 데 적합합니다**
진행 상황, 북마크, 서가, 폴더 목록 등 모든 정보가 앱의 데이터 디렉터리에 저장된 하나의 SQLite 파일에 보관됩니다. 백엔드 서버도, 동기화도, 계정도 필요 없습니다. 그냥 사용하면 됩니다.

**바닐라 타입스크립트는 과소평가되어 있습니다**
React도, Vue도, 상태 관리 라이브러리도 사용하지 않았습니다. 오직 DOM API를 사용하는 TypeScript만 사용했습니다. 앱은 라이브러리 뷰와 리더 뷰로 구성되어 있으며, 상태는 몇 개의 모듈 레벨 변수에 저장됩니다. 이렇게 간단한 프로젝트에 프레임워크를 추가하면 복잡성만 가중될 뿐 이점은 없었습니다.

**안드로이드의 파일 시스템은 미로 같습니다**
기본 파일 선택기는 전자책 리더기에 적합하지 않습니다. 전자책 리더기에서는 단일 파일이 아니라 책이 담긴 *폴더*를 선택해야 하기 때문입니다. 또한 `content://` URI는 실제 경로를 제공하지 않습니다. 따라서 사용자 지정 파일 브라우저가 올바른 해결책이었습니다.

---

## 남은 작업

다음 목표는 **iOS**입니다. Tauri v2는 Android와 마찬가지로 iOS도 지원합니다. `npm run tauri ios init` 명령어를 실행하면 Xcode 프로젝트가 생성됩니다. 다만 한 가지 주의할 점은 iOS 컴파일에는 Mac이 필요하다는 것입니다(Xcode는 macOS에서만 실행됩니다). Mac에서 빌드하고 실제 기기에서 테스트할 계획입니다. 후... 안해봤지만 제 8기가 m1 에어에서는 안 될 거 같아서 잠시 보류 중입니다... 나중에 부자 되면 바로 빌드하고 테스트해서 아이패드에서도 마크다운 편하게 봐야겠어요....


혹시라도 이북리더기 쓰시는 분이 있으시다면 제 깃헙에서 다운 받고 빌드해서 apk 옮겨서 쓰시면 됩니다....