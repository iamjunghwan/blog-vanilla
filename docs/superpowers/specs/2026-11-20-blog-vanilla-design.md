# blog-vanilla 설계

- 날짜: 2026-11-20
- 원본: `../blog` (https://iaman.kr/, Next.js 16 + React 19, `output: "export"`)

## 목적

iaman 블로그를 Next.js·React 없이 직접 구현하고, 같은 글로 만든 Next판 결과물과 비교한다.
비교 기준은 세 가지다.

1. 결과물 동일성 — URL, 메타 태그, 본문이 같은가
2. 숫자 — 빌드 시간, 의존성, 출력 크기, 페이지당 JS, Lighthouse
3. 코드 — 줄 수, 파일 수, 기능별 난이도

## 결정 사항

| 항목 | 결정 |
|---|---|
| 위치 | `c:\Users\oT26116\Desktop\my\blog-vanilla`, 독립 git 저장소. GitHub 원격은 필요할 때 연결 |
| 콘텐츠 | `../blog`의 `content/posts`, `public/`을 처음에 복사. 이후 `scripts/sync-content.ts`로 다시 복사 |
| 범위 | ko/en 공개 사이트 전부: 홈, 글, 태그 목록과 페이지네이션, about, 404, 검색, 다크모드, 캐러셀, 메뉴, 언어 전환, sitemap·robots·SEO 메타·JSON-LD, 폰트 서브셋 |
| 범위 밖 | 썸네일 생성(blog에서 만든 이미지를 복사해 쓴다), `admin` 페이지와 `lib/analytics`, `new-post`·`icons` 스크립트 |
| 의존성 원칙 | 프레임워크(Next, React, 템플릿 엔진, 번들러, 개발 서버 도구)는 쓰지 않는다. 지금 쓰는 부품 라이브러리는 그대로 쓴다: `markdown-it`, `prismjs`, `gray-matter`, `cheerio`, `tailwindcss`(CLI), `@tailwindcss/typography`, `subset-font`, `tsx`, `typescript` |
| 템플릿 방식 | 직접 만든 JSX → 문자열 런타임 |
| 패키지 관리 | pnpm, Node 24 (blog와 같은 버전) |

## 1. 구조

```text
blog-vanilla/
├─ content/posts/ko|en/**/*.md   blog에서 복사
├─ public/                       uploads, fonts, 아이콘, _headers 복사
├─ src/
│  ├─ jsx/          JSX 런타임 (jsx, jsxs, Fragment, raw, 이스케이프)
│  ├─ lib/          blog의 app/lib에서 가져온다
│  │  ├─ posts/     repository · parse · queries · render
│  │  ├─ i18n/ blog/ carousel/ date/ seo/ site.ts
│  │  └─ processHtml.ts
│  ├─ components/   서버 컴포넌트 (React 의존을 걷어내고 1:1 이식)
│  ├─ pages/        페이지 단위 (지금의 components/pages + layout)
│  ├─ routes.ts     URL → 페이지 함수 목록 (generateStaticParams 대체)
│  ├─ head.ts       metadata → <head> 태그 (generateMetadata 대체)
│  └─ client/       바닐라 TS 모듈
├─ scripts/
│  ├─ build.ts      빌드 진입점
│  ├─ dev.ts        감시 + 정적 서버 + 자동 새로고침
│  ├─ fonts.ts      폰트 서브셋
│  ├─ sync-content.ts
│  └─ compare/      Next판과 비교
├─ dist/            빌드 결과 → Cloudflare Pages (git에 넣지 않는다)
└─ tsconfig.json    jsxImportSource → src/jsx
```

### 빌드 흐름 (`pnpm build`)

```text
1. 글 읽기·검증 (lib/posts)            틀리면 빌드 중단
2. routes.ts가 URL 목록 생성             /, /<slug>, /post/<tag>/<page>, /about, 각각 /en/...
3. URL마다 페이지 함수 → HTML 문자열 → dist/<url>.html (/ 는 index.html)
4. sitemap.xml, robots.txt, 404.html
5. tailwindcss CLI → dist/styles.css
6. tsc → src/client/*.ts → dist/js/*.js   번들 없이 ES 모듈
7. public/ → dist/ 복사
```

- 서버, API, 환경변수, 런타임 데이터 요청이 없다. blog와 같다
- `lib/`은 Next 전용 import만 걷어내고 테스트까지 함께 가져온다
- URL 구조는 blog와 같아야 한다. 다르면 SEO가 깨진다
- 파일 이름도 Next export(`trailingSlash: false`)와 같게 `about.html`, `en.html`, `post/all/1.html`로 쓴다. `about/index.html`로 쓰면 Cloudflare Pages가 `/about`을 `/about/`으로 리다이렉트해 URL이 바뀐다

## 2. JSX 런타임과 클라이언트 동작

### JSX 런타임 (`src/jsx/`)

- tsx(esbuild)의 `jsxImportSource`로 React 대신 직접 만든 `jsx()`를 쓴다
- 함수 컴포넌트면 호출하고, 태그면 HTML 문자열을 만든다
- 반환값은 이스케이프가 끝난 `SafeHtml`. 일반 문자열은 항상 이스케이프한다
- 신뢰된 HTML(렌더된 글 본문 등)만 `raw(html)`로 통과시킨다 (`dangerouslySetInnerHTML` 대체)
- 지원: `class`·`className`, `style` 객체, boolean 속성, `null`·`undefined`·`false` 무시, 배열 children, void 태그
- 지원하지 않음: 훅, 상태, 컨텍스트, 비동기 컴포넌트. 데이터는 페이지 함수에서 미리 읽어 props로 넘긴다

### 클라이언트 동작 (`src/client/`)

서버가 마크업을 그리고, 작은 스크립트가 `data-island` 요소를 찾아 붙는다.

```text
HTML:  <div data-island="search" data-lang="ko">...</div>
JS:    document.querySelectorAll('[data-island="search"]') → 이벤트 연결
```

| 지금 (React) | 바닐라 |
|---|---|
| `next-themes` | `<head>` 인라인 스크립트로 깜빡임 없이 `dark` 클래스 적용, 토글은 `theme.js` |
| Search 모달 | 모달 마크업을 `hidden`으로 미리 렌더, `search.js`가 열기·닫기·필터링. 검색 인덱스는 지금처럼 페이지에 JSON으로 넣는다 |
| LatestCarousel | `lib/carousel`의 drag·geometry 순수 함수를 재사용, DOM 연결만 새로 쓴다 |
| 메뉴 토글, 언어 전환, 태그 목록, 글 상호작용, 카드 날짜 | 각각 작은 모듈 |

- 페이지마다 필요한 모듈만 `<script type="module">`로 붙인다
- Next의 `<Link>` 클라이언트 라우팅은 따라 하지 않는다. 바닐라판은 일반 페이지 이동이고, 이 차이는 비교 항목으로 남긴다

## 3. 개발 서버와 에러 처리

### 개발 서버 (`pnpm dev`)

```text
1. 전체 빌드 → dist/
2. node:http로 dist/ 서빙 (/foo → /foo.html, 없으면 404.html. Cloudflare Pages와 같은 규칙)
3. fs.watch로 감시
   content/, src/(client 제외)  → 전체 빌드를 자식 프로세스로 다시 실행
   src/client/                  → tsc만
   CSS 관련                      → tailwind만
4. 빌드가 끝나면 SSE(/__reload)로 브라우저 새로고침
```

- Node 내장 기능만 쓴다
- 자동 새로고침 스크립트는 dev 빌드에만 넣는다
- 증분 빌드는 하지 않는다. 느려지면 그때 고친다

### 에러 처리

| 상황 | 동작 |
|---|---|
| frontmatter가 틀림 | 파일명과 이유를 출력하고 빌드 실패 (`parse.ts` 재사용) |
| 렌더 중 예외 | URL을 붙여서 빌드 실패 |
| 같은 경로를 두 페이지가 생성 | 빌드 실패 |
| 글 안의 내부 링크·이미지가 dist에 없음 | 경고만 출력 |
| dev 중 빌드 실패 | 서버는 유지, 터미널과 브라우저 오버레이에 에러 표시. 고치면 자동 복구 |

### 테스트

- `pnpm test` = `node:test` + `tsx`
- 가져온 `lib/` 테스트와 `content.test.ts`(실제 글 전체 검사)를 유지한다
- 새로 쓰는 테스트: JSX 런타임(이스케이프, 속성, void 태그, raw), `routes.ts`(URL 목록), `head.ts`(메타 태그)

## 4. Next판과 비교

- 스크립트는 `scripts/compare/`, 결과는 `docs/comparison.md`
- 기준은 같은 글로 빌드한 `../blog/out/`이다. 비교 전에 `sync-content`로 글을 맞추고 blog에서 `pnpm build`를 돌린다

### 결과물 동일성 (`pnpm compare:output`)

| 검사 | 방법 | 통과 기준 |
|---|---|---|
| URL 목록 | `out/`과 `dist/`의 `*.html` 경로 집합 비교 | 완전히 일치 |
| 메타 태그 | 페이지별 title, description, canonical, og:*, hreflang, JSON-LD | 완전히 일치 |
| 본문 | cheerio로 `<main>`을 꺼내 공백·속성 순서·Next 전용 속성을 정규화한 뒤 diff | 차이가 0이거나 차이마다 이유를 기록 |
| sitemap.xml, robots.txt | 내용 비교 | 완전히 일치 |

바이트 단위 비교는 하지 않는다. Next가 넣는 스크립트와 RSC 데이터 때문에 의미가 없다.

### 숫자 (`pnpm compare:metrics`)

| 항목 | 방법 |
|---|---|
| 빌드 시간 | 각 5회, 캐시를 지운 콜드 빌드의 중앙값 |
| 의존성 | 설치된 패키지 수, `node_modules` 크기 |
| 출력 | 전체 크기, HTML·JS·CSS별 크기 |
| 페이지당 JS | 홈, 글, 목록 페이지가 받는 JS 합계 (원본과 gzip) |
| Lighthouse | 같은 3개 페이지, 로컬 정적 서버에서 `npx lighthouse`. Chrome이나 Edge가 있을 때만 |

### 코드

- 기능별 줄 수와 파일 수를 세는 스크립트 (외부 도구 없음)
- 기능별 난이도와 소감은 작업하면서 `docs/comparison.md`에 적는다
