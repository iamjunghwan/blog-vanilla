# blog-vanilla 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `../blog`(Next.js 16)의 공개 사이트를 프레임워크 없이 다시 만들고, 같은 글로 빌드한 Next판과 결과물·숫자·코드를 비교한다.

**Architecture:** 직접 만든 JSX→문자열 런타임으로 서버 컴포넌트를 1:1 이식하고, 빌드 스크립트가 라우트 표를 돌며 `dist/`에 HTML을 쓴다. 클라이언트 동작은 `data-island` 요소에 붙는 바닐라 TS 모듈이고, `tsc`가 번들 없이 ES 모듈로 내보낸다. `lib/`(글 파이프라인)은 blog에서 가져온 그대로 쓴다.

**Tech Stack:** Node 24, pnpm 10, TypeScript 5.8, tsx 4, markdown-it, prismjs, gray-matter, cheerio, Tailwind CSS 3.4(CLI), subset-font, node:test

**설계 문서:** `docs/superpowers/specs/2026-11-20-blog-vanilla-design.md`

## Global Constraints

- 저장소 루트: `c:\Users\oT26116\Desktop\my\blog-vanilla`. 원본은 `../blog` (읽기만 하고 고치지 않는다)
- 프레임워크(Next, React, 템플릿 엔진, 번들러, 개발 서버 도구)를 의존성에 넣지 않는다
- 허용 의존성: `markdown-it` `prismjs` `gray-matter` `cheerio` `tailwindcss` `@tailwindcss/typography` `subset-font` `tsx` `typescript` 와 `@types/*`
- import는 **상대 경로 + 확장자**(`./x.ts`, `./X.tsx`)로 쓴다. `@/` 별칭은 없다. 예외는 JSX 런타임(`#jsx`)뿐이다
- URL은 blog와 같아야 한다. 출력 파일은 `/` → `index.html`, `/<경로>` → `<경로>.html` (Next `trailingSlash: false`와 같은 규칙)
- 서버, API, 런타임 데이터 요청이 없다. 환경변수는 `NODE_ENV`, `GA_MEASURE_ID`(blog와 같다), `BLOG_DEV`(dev 표시)만 읽는다
- 클라이언트 동작용 속성은 전부 `data-island`로 시작한다. 비교 스크립트가 이 접두사의 속성을 지운다
- 썸네일 생성, `admin`, `lib/analytics`, `new-post`, `icons` 스크립트는 범위 밖이다
- 커밋 메시지는 gitmoji + `-다`체 (예: `✨ feat: JSX 런타임을 만든다`), 끝에 `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`
- 커밋은 사용자가 승인한 범위에서만 한다

## 파일 구조

```text
blog-vanilla/
├─ package.json  tsconfig.json  tsconfig.client.json  tailwind.config.js  .gitignore  README.md
├─ content/                    ← sync로 복사 (content/SOURCE에 원본 커밋 기록)
├─ public/                     ← sync로 복사 (+ app 아이콘 3개)
├─ src/
│  ├─ jsx/html.ts              SafeHtml, raw, escapeHtml
│  ├─ jsx/jsx-runtime.ts       jsx, jsxs, jsxDEV, Fragment, JSX 타입
│  ├─ lib/                     blog app/lib 이식 (posts, i18n, blog, carousel, date, seo, site, processHtml)
│  ├─ lib/i18n/switch.ts       언어 전환 대상 계산 (새로 씀)
│  ├─ components/              blog components 이식
│  ├─ pages/                   HomePage, ArticlePage, PostListPage, AboutPage, NotFoundPage
│  ├─ head.tsx                 resolveMeta + HeadTags (generateMetadata 대체)
│  ├─ routes.ts                RouteSpec 목록 + outputFile
│  ├─ render.tsx               RouteSpec → 완성 HTML 문자열
│  ├─ build/build.ts           빌드 단계 묶음
│  ├─ build/sitemap.ts         sitemap.xml, robots.txt
│  ├─ build/links.ts           깨진 내부 링크 경고
│  ├─ build/serve.ts           정적 서버 (dev, metrics가 같이 씀)
│  ├─ client/                  theme, search, menu, date, tag-list, article, carousel
│  ├─ styles/globals.css
│  └─ static/_headers
├─ scripts/
│  ├─ sync-content.ts  rewrite-imports.ts  build.ts  dev.ts  fonts.ts
│  └─ compare/normalize.ts  output.ts  metrics.ts  code.ts
└─ docs/comparison.md  docs/comparison/*.md
```

---

### Task 1: 프로젝트 뼈대와 콘텐츠 동기화

**Files:**
- Create: `package.json`, `tsconfig.json`, `tsconfig.client.json`, `.gitignore`, `README.md`, `scripts/sync-content.ts`

**Interfaces:**
- Produces: `pnpm sync [원본경로]` — `content/`, `public/`을 원본으로 교체하고 `content/SOURCE`에 원본 커밋 해시를 쓴다. `#jsx/jsx-runtime` import 경로(Task 2가 파일을 채운다)

- [ ] **Step 1: package.json 작성**

```json
{
  "name": "blog-vanilla",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "imports": {
    "#jsx/jsx-runtime": "./src/jsx/jsx-runtime.ts",
    "#jsx/jsx-dev-runtime": "./src/jsx/jsx-runtime.ts"
  },
  "scripts": {
    "build": "tsx scripts/build.ts",
    "dev": "tsx scripts/dev.ts",
    "test": "tsx --test \"src/**/*.test.ts\" \"src/**/*.test.tsx\" \"scripts/**/*.test.ts\"",
    "typecheck": "tsc --noEmit -p tsconfig.json && tsc --noEmit -p tsconfig.client.json",
    "sync": "tsx scripts/sync-content.ts",
    "fonts": "tsx scripts/fonts.ts",
    "compare:output": "tsx scripts/compare/output.ts",
    "compare:metrics": "tsx scripts/compare/metrics.ts",
    "compare:code": "tsx scripts/compare/code.ts"
  },
  "packageManager": "pnpm@10.28.2",
  "engines": {
    "node": ">=24"
  },
  "dependencies": {
    "@tailwindcss/typography": "^0.5.16",
    "cheerio": "^1.2.0",
    "gray-matter": "^4.0.3",
    "markdown-it": "^15.0.0",
    "prismjs": "^1.30.0",
    "tailwindcss": "^3.4.17"
  },
  "devDependencies": {
    "@types/markdown-it": "^14.1.2",
    "@types/node": "^24",
    "@types/prismjs": "^1.26.6",
    "subset-font": "^2.9.0",
    "tsx": "^4.23.12",
    "typescript": "~5.8.3"
  }
}
```

- [ ] **Step 2: tsconfig 두 개 작성**

`tsconfig.json` (서버·스크립트·테스트, 타입 검사 전용):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "ES2023"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "jsxImportSource": "#jsx",
    "types": ["node"]
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "scripts/**/*.ts", "scripts/**/*.tsx"]
}
```

`tsconfig.client.json` (브라우저 모듈 출력용):

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": false,
    "rewriteRelativeImportExtensions": true,
    "rootDir": "src",
    "outDir": "dist/js",
    "types": []
  },
  "include": ["src/client/**/*.ts"]
}
```

`rewriteRelativeImportExtensions`가 `./x.ts` import를 출력에서 `./x.js`로 바꾼다. 브라우저는 확장자가 있어야 모듈을 찾는다.

- [ ] **Step 3: .gitignore, README.md 작성**

`.gitignore`:

```text
node_modules/
dist/
.cache/
*.log
```

`README.md`:

```markdown
# blog-vanilla

[iaman 블로그](https://iaman.kr/)를 Next.js 없이 직접 구현한 버전. 원본은 `../blog`.

| 명령 | 하는 일 |
|---|---|
| `pnpm sync [../blog]` | 원본의 글·public을 복사한다 |
| `pnpm dev` | 빌드 + 정적 서버(:3000) + 자동 새로고침 |
| `pnpm build` | `dist/` 생성 |
| `pnpm test` / `pnpm typecheck` | 테스트 / 타입 검사 |
| `pnpm fonts` | 폰트 서브셋 재생성 |
| `pnpm compare:output` · `compare:metrics` · `compare:code` | Next판과 비교 |

설계: `docs/superpowers/specs/`, 비교 결과: `docs/comparison.md`
```

- [ ] **Step 4: sync-content 스크립트 작성**

`scripts/sync-content.ts`:

```ts
/**
 * 원본 블로그(../blog)의 글과 정적 파일을 복사한다.
 *
 * content/, public/ 을 통째로 갈아 끼운다. 원본의 public/_headers 는 Next 경로
 * (/_next/static/*)를 가리켜서 가져오지 않는다 — 우리 것은 src/static/_headers 다.
 * Next가 app/ 에서 내보내던 아이콘 세 개는 public 루트로 옮긴다.
 * 어느 커밋을 복사했는지 content/SOURCE 에 남겨 비교 때 같은 글인지 확인한다.
 */
import fs from "node:fs";
import path from "node:path";
import { execFileSync } from "node:child_process";

const from = path.resolve(process.argv[2] ?? "../blog");
const APP_ICONS = ["favicon.ico", "icon.png", "apple-icon.png"];

if (!fs.existsSync(path.join(from, "content", "posts"))) {
  throw new Error(`${from} 에 content/posts 가 없다. 원본 경로를 확인한다.`);
}

function replaceDir(src: string, dest: string): void {
  fs.rmSync(dest, { recursive: true, force: true });
  fs.cpSync(src, dest, { recursive: true });
}

replaceDir(path.join(from, "content"), "content");
replaceDir(path.join(from, "public"), "public");
fs.rmSync(path.join("public", "_headers"), { force: true });

for (const icon of APP_ICONS) {
  fs.copyFileSync(path.join(from, "app", icon), path.join("public", icon));
}

const commit = execFileSync("git", ["-C", from, "rev-parse", "--short", "HEAD"], {
  encoding: "utf8",
}).trim();
const dirty =
  execFileSync("git", ["-C", from, "status", "--porcelain", "content", "public"], {
    encoding: "utf8",
  }).trim() !== "";
fs.writeFileSync(path.join("content", "SOURCE"), `${commit}${dirty ? "-dirty" : ""}\n`);

const count = (dir: string): number =>
  fs.readdirSync(dir, { recursive: true }).filter((f) => String(f).endsWith(".md")).length;

console.log(`복사 완료: ${from} @ ${commit}${dirty ? " (커밋 안 된 변경 있음)" : ""}`);
console.log(`  글 ${count(path.join("content", "posts"))}편, 아이콘 ${APP_ICONS.length}개`);
```

- [ ] **Step 5: 설치하고 동기화 실행**

Run: `pnpm install` 그리고 `pnpm sync`
Expected: `복사 완료: C:\...\blog @ <해시>` 와 `글 49편, 아이콘 3개` (글 수는 원본에 따라 다를 수 있다). `content/posts/ko`, `content/featured.json`, `public/uploads`, `public/fonts`, `public/favicon.ico`가 생긴다.

- [ ] **Step 6: 커밋**

```bash
git add .
git commit -m "🎉 init: 프로젝트 뼈대를 세우고 원본 글을 복사한다"
```

`content/`와 `public/`도 함께 커밋한다(설계 Q2-A: 복사해서 독립시킨다).

---

### Task 2: JSX 런타임

**Files:**
- Create: `src/jsx/html.ts`, `src/jsx/jsx-runtime.ts`
- Test: `src/jsx/jsx-runtime.test.tsx`

**Interfaces:**
- Produces:
  - `class SafeHtml { readonly value: string; toString(): string }`
  - `raw(html: string): SafeHtml` — 이스케이프하지 않고 통과
  - `escapeHtml(text: string): string` — `& < > " '` → `&amp; &lt; &gt; &quot; &#x27;` (React와 같다)
  - `type Child = SafeHtml | string | number | boolean | null | undefined | Child[]`
  - `jsx(type, props): SafeHtml`, `Fragment`. JSX 표현식의 값은 `SafeHtml`이고 `String(el)`로 HTML을 얻는다
  - 이벤트 핸들러(`onClick` 등)나 `dangerouslySetInnerHTML`을 넘기면 **던진다**

- [ ] **Step 1: 실패하는 테스트 작성**

`src/jsx/jsx-runtime.test.tsx`:

```tsx
import { test } from "node:test";
import assert from "node:assert/strict";
import { raw, SafeHtml } from "./html.ts";

test("텍스트와 속성 값을 이스케이프한다", () => {
  const el = <p title={`"a" & 'b'`}>{"<script>alert(1)</script>"}</p>;
  assert.equal(
    String(el),
    `<p title="&quot;a&quot; &amp; &#x27;b&#x27;">&lt;script&gt;alert(1)&lt;/script&gt;</p>`,
  );
});

test("raw()는 이스케이프하지 않는다", () => {
  assert.equal(String(<div>{raw("<b>x</b>")}</div>), "<div><b>x</b></div>");
});

test("className·htmlFor·SVG 속성 이름을 HTML 이름으로 바꾼다", () => {
  const el = (
    <svg className="i" viewBox="0 0 24 24" strokeWidth={1.5}>
      <path strokeLinecap="round" strokeLinejoin="round" d="M0 0" />
    </svg>
  );
  assert.equal(
    String(el),
    `<svg class="i" viewBox="0 0 24 24" stroke-width="1.5"><path stroke-linecap="round" stroke-linejoin="round" d="M0 0"></path></svg>`,
  );
  assert.equal(String(<label htmlFor="q">x</label>), `<label for="q">x</label>`);
  assert.equal(String(<time dateTime="2026-01-01">d</time>), `<time datetime="2026-01-01">d</time>`);
});

test("boolean 속성: true는 이름만, false·null·undefined는 생략, aria-*는 문자열", () => {
  assert.equal(
    String(<button disabled={true} hidden={false} title={undefined} aria-current={true} />),
    `<button disabled aria-current="true"></button>`,
  );
});

test("void 태그는 닫는 태그가 없다", () => {
  assert.equal(String(<img src="/a.png" alt="" />), `<img src="/a.png" alt=""/>`);
  assert.equal(String(<meta charSet="utf-8" />), `<meta charset="utf-8"/>`);
});

test("style 객체를 CSS 문자열로 바꾼다", () => {
  assert.equal(
    String(<li style={{ transform: "translateX(0px)", opacity: 1, zIndex: 5 }} />),
    `<li style="transform:translateX(0px);opacity:1;z-index:5"></li>`,
  );
});

test("함수 컴포넌트, Fragment, 배열, 숫자, falsy children", () => {
  const Item = ({ n, children }: { n: number; children?: string }) => (
    <li data-n={n}>{children}</li>
  );
  const el = (
    <ul>
      <>
        {[1, 2].map((n) => (
          <Item n={n}>{`#${n}`}</Item>
        ))}
      </>
      {false}
      {null}
      {0}
    </ul>
  );
  assert.equal(String(el), `<ul><li data-n="1">#1</li><li data-n="2">#2</li>0</ul>`);
});

test("null을 돌려주는 컴포넌트는 빈 문자열이 된다", () => {
  const Nothing = () => null;
  assert.equal(String(<div><Nothing /></div>), "<div></div>");
});

test("이벤트 핸들러는 던진다 — 클라이언트 모듈로 옮겨야 한다", () => {
  assert.throws(() => <button onClick={() => {}} />, /이벤트 핸들러/);
});

test("JSX 결과는 SafeHtml이다", () => {
  assert.ok(<i /> instanceof SafeHtml);
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test src/jsx/jsx-runtime.test.tsx`
Expected: FAIL — `Cannot find module` (`#jsx/jsx-runtime` 또는 `./html.ts`)

- [ ] **Step 3: html.ts 작성**

`src/jsx/html.ts`:

```ts
/** 이미 안전한(이스케이프가 끝났거나 신뢰하는) HTML 조각. JSX 표현식의 값이다. */
export class SafeHtml {
  constructor(readonly value: string) {}
  toString(): string {
    return this.value;
  }
}

/** 신뢰하는 HTML을 그대로 통과시킨다. 렌더된 글 본문, JSON-LD 같은 곳에만 쓴다. */
export function raw(html: string): SafeHtml {
  return new SafeHtml(html);
}

const ESCAPES: Record<string, string> = {
  "&": "&amp;",
  "<": "&lt;",
  ">": "&gt;",
  '"': "&quot;",
  "'": "&#x27;",
};

/** React와 같은 규칙으로 이스케이프한다. 비교 때 차이가 나지 않게 한다. */
export function escapeHtml(text: string): string {
  return text.replace(/[&<>"']/g, (char) => ESCAPES[char]);
}
```

- [ ] **Step 4: jsx-runtime.ts 작성**

`src/jsx/jsx-runtime.ts`:

```ts
/****************************************
 *
 * JSX → HTML 문자열 런타임
 *
 * tsconfig 의 jsxImportSource("#jsx")로 React 대신 이 파일이 불린다.
 * 훅·상태·컨텍스트는 없다. 서버에서 한 번 그리고 끝난다.
 * 동작이 필요한 곳은 data-island 속성을 달고 src/client/ 모듈이 붙는다.
 *
 ****************************************/

import { SafeHtml, escapeHtml } from "./html.ts";

export type Child = SafeHtml | string | number | boolean | null | undefined | Child[];
type Props = Record<string, unknown> & { children?: Child };
type Component = (props: any) => SafeHtml | null;

const VOID_TAGS = new Set([
  "area", "base", "br", "col", "embed", "hr", "img", "input",
  "link", "meta", "source", "track", "wbr",
]);

/** React 프롭 이름 → HTML 속성 이름. 여기 없는 이름은 그대로 쓴다(viewBox, aria-*, data-*). */
const ATTR_NAMES: Record<string, string> = {
  className: "class",
  htmlFor: "for",
  tabIndex: "tabindex",
  hrefLang: "hreflang",
  dateTime: "datetime",
  charSet: "charset",
  crossOrigin: "crossorigin",
  fetchPriority: "fetchpriority",
  autoComplete: "autocomplete",
  readOnly: "readonly",
  srcSet: "srcset",
  strokeWidth: "stroke-width",
  strokeLinecap: "stroke-linecap",
  strokeLinejoin: "stroke-linejoin",
  strokeDasharray: "stroke-dasharray",
  fillRule: "fill-rule",
  clipRule: "clip-rule",
};

function renderChild(child: Child): string {
  if (child === null || child === undefined || child === false || child === true) {
    return "";
  }
  if (Array.isArray(child)) {
    return child.map(renderChild).join("");
  }
  if (child instanceof SafeHtml) {
    return child.value;
  }
  return escapeHtml(String(child));
}

/** 숫자에 px를 붙이지 않는다. 이 블로그가 쓰는 숫자 style은 opacity·zIndex뿐이다. */
function styleToString(style: Record<string, unknown>): string {
  return Object.entries(style)
    .filter(([, value]) => value !== null && value !== undefined && value !== "")
    .map(([key, value]) => {
      const name = key.startsWith("--") ? key : key.replace(/[A-Z]/g, (c) => `-${c.toLowerCase()}`);
      return `${name}:${value}`;
    })
    .join(";");
}

function renderAttributes(tag: string, props: Props): string {
  let out = "";
  for (const [key, value] of Object.entries(props)) {
    if (key === "children" || key === "key" || key === "ref") {
      continue;
    }
    if (/^on[A-Z]/.test(key)) {
      throw new Error(
        `<${tag} ${key}>: 이벤트 핸들러는 서버 렌더에 쓸 수 없다. data-island 를 달고 src/client/ 모듈로 옮긴다`,
      );
    }
    if (key === "dangerouslySetInnerHTML") {
      throw new Error(`<${tag}>: dangerouslySetInnerHTML 대신 children 에 raw() 를 넣는다`);
    }
    if (value === null || value === undefined || value === false) {
      continue;
    }
    const name = ATTR_NAMES[key] ?? key;
    if (value === true) {
      out += /^(aria|data)-/.test(name) ? ` ${name}="true"` : ` ${name}`;
      continue;
    }
    const text =
      key === "style" && typeof value === "object"
        ? styleToString(value as Record<string, unknown>)
        : String(value);
    out += ` ${name}="${escapeHtml(text)}"`;
  }
  return out;
}

export function jsx(type: string | Component, props: Props): SafeHtml {
  if (typeof type === "function") {
    return type(props) ?? new SafeHtml("");
  }
  const attributes = renderAttributes(type, props);
  if (VOID_TAGS.has(type)) {
    return new SafeHtml(`<${type}${attributes}/>`);
  }
  return new SafeHtml(`<${type}${attributes}>${renderChild(props.children)}</${type}>`);
}

export const jsxs = jsx;
export const jsxDEV = jsx;

export function Fragment(props: { children?: Child }): SafeHtml {
  return new SafeHtml(renderChild(props.children));
}

export namespace JSX {
  export type Element = SafeHtml;
  export interface IntrinsicElements {
    [tag: string]: Record<string, unknown>;
  }
  export interface ElementChildrenAttribute {
    children: {};
  }
}
```

- [ ] **Step 5: 테스트 통과 확인**

Run: `pnpm exec tsx --test src/jsx/jsx-runtime.test.tsx`
Expected: PASS, 10개 테스트 통과

만약 `#jsx/jsx-runtime`을 찾지 못한다는 오류가 나면 tsx가 package.json `imports`를 못 읽는 것이다. 이때는 tsconfig에 `"paths": { "#jsx/*": ["./src/jsx/*"] }`를 더하고 다시 돌린다.

- [ ] **Step 6: 타입 검사**

Run: `pnpm exec tsc --noEmit -p tsconfig.json`
Expected: 오류 없음

- [ ] **Step 7: 커밋**

```bash
git add src/jsx
git commit -m "✨ feat: JSX를 HTML 문자열로 바꾸는 런타임을 만든다"
```

---

### Task 3: lib 이식과 import 재작성 도구

**Files:**
- Create: `scripts/rewrite-imports.ts`, `scripts/rewrite-imports.test.ts`
- Create (복사): `src/lib/{posts,i18n,blog,carousel,date}/**`, `src/lib/processHtml.ts`, `src/lib/site.ts`, `src/lib/seo/json-ld.ts`
- Modify: `src/lib/posts/repository.test.ts`, `src/lib/posts/content.test.ts`, `src/lib/seo/json-ld.ts`

**Interfaces:**
- Produces:
  - `rewriteSpecifier(fromFile, spec, exists, root?): string`, `rewriteSource(fromFile, source, exists, root?): string`
  - CLI: `pnpm exec tsx scripts/rewrite-imports.ts <파일·폴더...>` — `@/app/lib/` `@/components/` `@/scripts/` 별칭과 확장자 없는 상대 import를 확장자 있는 상대 경로로 바꾼다
  - `src/lib/` 아래 모든 모듈 (blog의 `app/lib`와 같은 export). 특히 `allPosts()` `LANGS` `featuredSlugs()` (repository), `postsIn` `postBySlug` `postListParams` `postListUrls` `bilingualSlugs` `counterpart` `searchIndex` `filterByTitle` `allTags` `paginate` `postsByTag` `featuredPosts` (queries), `renderPostBody` (render), `localePath` (i18n/paths), `messages` (i18n/messages), `SITE_COPY` (i18n/site-copy), `SITE_URL` `absoluteUrl` (site), `websiteJsonLd` `blogPostingJsonLd` (seo/json-ld)

- [ ] **Step 1: 실패하는 테스트 작성**

`scripts/rewrite-imports.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import path from "node:path";
import { rewriteSource, rewriteSpecifier } from "./rewrite-imports.ts";

const root = path.resolve("/repo");
const files = new Set(
  [
    "src/lib/posts/types.ts",
    "src/lib/posts/queries.ts",
    "src/lib/seo/json-ld.ts",
    "src/components/Card/CardDateArea.tsx",
    "src/components/SvgIcon/index.ts",
    "scripts/fonts.ts",
  ].map((f) => path.join(root, f)),
);
const exists = (p: string) => files.has(p);
const from = path.join(root, "src", "lib", "posts", "queries.ts");

test("@/app/lib 별칭을 확장자 있는 상대 경로로 바꾼다", () => {
  assert.equal(rewriteSpecifier(from, "@/app/lib/posts/types", exists, root), "./types.ts");
  assert.equal(rewriteSpecifier(from, "@/app/lib/seo/json-ld", exists, root), "../seo/json-ld.ts");
});

test("@/components, @/scripts, index 파일, .tsx", () => {
  assert.equal(
    rewriteSpecifier(from, "@/components/Card/CardDateArea", exists, root),
    "../../components/Card/CardDateArea.tsx",
  );
  assert.equal(
    rewriteSpecifier(from, "@/components/SvgIcon/index", exists, root),
    "../../components/SvgIcon/index.ts",
  );
  assert.equal(rewriteSpecifier(from, "@/components/SvgIcon", exists, root), "../../components/SvgIcon/index.ts");
  assert.equal(rewriteSpecifier(from, "@/scripts/fonts", exists, root), "../../../scripts/fonts.ts");
});

test("확장자 없는 상대 import에 확장자를 붙이고, 패키지는 그대로 둔다", () => {
  assert.equal(rewriteSpecifier(from, "./types", exists, root), "./types.ts");
  assert.equal(rewriteSpecifier(from, "./types.ts", exists, root), "./types.ts");
  assert.equal(rewriteSpecifier(from, "markdown-it", exists, root), "markdown-it");
  assert.equal(rewriteSpecifier(from, "node:fs", exists, root), "node:fs");
});

test("못 찾으면 던진다", () => {
  assert.throws(() => rewriteSpecifier(from, "@/app/lib/nope", exists, root), /찾을 수 없다/);
});

test("소스 안의 import·export from 을 모두 바꾼다", () => {
  const source = [
    `import type { Post } from "@/app/lib/posts/types";`,
    `import {`,
    `  x,`,
    `} from '@/app/lib/seo/json-ld';`,
    `import "prismjs/components/prism-jsx";`,
    `export { y } from "./types";`,
  ].join("\n");
  assert.equal(
    rewriteSource(from, source, exists, root),
    [
      `import type { Post } from "./types.ts";`,
      `import {`,
      `  x,`,
      `} from '../seo/json-ld.ts';`,
      `import "prismjs/components/prism-jsx";`,
      `export { y } from "./types.ts";`,
    ].join("\n"),
  );
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test scripts/rewrite-imports.test.ts`
Expected: FAIL — `Cannot find module './rewrite-imports.ts'`

- [ ] **Step 3: rewrite-imports.ts 작성**

`scripts/rewrite-imports.ts`:

```ts
/**
 * blog에서 복사해 온 파일의 import를 이 저장소 규칙으로 바꾼다.
 *
 * blog는 `@/` 별칭과 확장자 없는 import를 쓴다(번들러가 풀어 준다).
 * 여기는 번들러가 없어서 브라우저와 Node가 직접 파일을 찾아야 하므로
 * 상대 경로 + 확장자로 적는다.
 */
import fs from "node:fs";
import path from "node:path";
import { fileURLToPath } from "node:url";

const ALIASES: [string, string][] = [
  ["@/app/lib/", "src/lib/"],
  ["@/components/", "src/components/"],
  ["@/scripts/", "scripts/"],
];
const CANDIDATE_SUFFIXES = ["", ".ts", ".tsx", "/index.ts", "/index.tsx"];

export function rewriteSpecifier(
  fromFile: string,
  spec: string,
  exists: (file: string) => boolean,
  root: string = process.cwd(),
): string {
  const alias = ALIASES.find(([prefix]) => spec.startsWith(prefix));
  let target: string;
  if (alias !== undefined) {
    target = path.join(root, alias[1], spec.slice(alias[0].length));
  } else if (spec.startsWith("./") || spec.startsWith("../")) {
    target = path.join(path.dirname(fromFile), spec);
  } else {
    return spec;
  }

  const found = CANDIDATE_SUFFIXES.map((suffix) => target + suffix).find(exists);
  if (found === undefined) {
    throw new Error(`${fromFile}: '${spec}' 를 찾을 수 없다`);
  }

  const relative = path.relative(path.dirname(fromFile), found).split(path.sep).join("/");
  return relative.startsWith(".") ? relative : `./${relative}`;
}

export function rewriteSource(
  fromFile: string,
  source: string,
  exists: (file: string) => boolean,
  root: string = process.cwd(),
): string {
  return source.replace(
    /(\bfrom\s*|\bimport\s*)(["'])([^"']+)\2/g,
    (_match, lead: string, quote: string, spec: string) =>
      `${lead}${quote}${rewriteSpecifier(fromFile, spec, exists, root)}${quote}`,
  );
}

function isFile(file: string): boolean {
  return fs.existsSync(file) && fs.statSync(file).isFile();
}

function collect(target: string): string[] {
  if (isFile(target)) {
    return /\.tsx?$/.test(target) ? [target] : [];
  }
  return fs
    .readdirSync(target, { recursive: true })
    .map((entry) => path.join(target, String(entry)))
    .filter((file) => /\.tsx?$/.test(file) && isFile(file));
}

function main(): void {
  const targets = process.argv.slice(2);
  if (targets.length === 0) {
    throw new Error("사용법: tsx scripts/rewrite-imports.ts <파일·폴더...>");
  }
  let changed = 0;
  for (const file of targets.flatMap((t) => collect(path.resolve(t)))) {
    const before = fs.readFileSync(file, "utf8");
    const after = rewriteSource(file, before, isFile);
    if (after !== before) {
      fs.writeFileSync(file, after);
      changed += 1;
    }
  }
  console.log(`import 재작성: ${changed}개 파일`);
}

if (path.resolve(process.argv[1] ?? "") === fileURLToPath(import.meta.url)) {
  main();
}
```

- [ ] **Step 4: 테스트 통과 확인**

Run: `pnpm exec tsx --test scripts/rewrite-imports.test.ts`
Expected: PASS, 5개 테스트 통과

- [ ] **Step 5: lib 복사**

```bash
mkdir -p src/lib/seo
cp -r ../blog/app/lib/posts ../blog/app/lib/i18n ../blog/app/lib/blog ../blog/app/lib/carousel ../blog/app/lib/date src/lib/
cp ../blog/app/lib/processHtml.ts ../blog/app/lib/site.ts src/lib/
cp ../blog/app/lib/seo/json-ld.tsx src/lib/seo/json-ld.ts
```

`analytics`와 `fonts`(테스트만 있음, Task 12에서 옮긴다)는 가져오지 않는다.

- [ ] **Step 6: json-ld.ts에서 JSX 컴포넌트 떼기**

`src/lib/seo/json-ld.ts`에서 `JsonLd` 함수(주석 `JSON.stringify 는 XSS에...`부터 함수 끝까지)를 지운다. 이 파일에는 데이터 함수(`websiteJsonLd`, `blogPostingJsonLd`)만 남긴다. `JsonLd` 컴포넌트는 Task 6에서 `src/components/JsonLd.tsx`로 새로 쓴다. 파일 머리 주석의 `next/script 가 아니라 평범한 <script> 로 넣는다` 문단은 그대로 둬도 된다.

- [ ] **Step 7: 테스트 파일 경로 고치기**

`src/lib/posts/repository.test.ts`:

```ts
// 바꾸기 전
const FIXTURES = path.join(process.cwd(), "app", "lib", "posts", "__fixtures__");
// 바꾼 뒤
const FIXTURES = path.join(process.cwd(), "src", "lib", "posts", "__fixtures__");
```

`src/lib/posts/content.test.ts`:
- `import { MOTIF_BY_SLUG } from "@/scripts/thumbnail/registry";` 줄을 지운다
- `test("공개된 글마다 고유한 썸네일 그림이 배정돼 있다", ...)` 블록 전체를 지운다 (썸네일 생성은 범위 밖이다)

- [ ] **Step 8: import 재작성**

Run: `pnpm exec tsx scripts/rewrite-imports.ts src/lib`
Expected: `import 재작성: N개 파일` (N은 30 안팎)

Run: `grep -rn '"@/' src/lib`
Expected: 출력 없음

- [ ] **Step 9: 테스트와 타입 검사**

Run: `pnpm exec tsx --test "src/lib/**/*.test.ts"`
Expected: 전부 PASS. `content.test.ts`는 실제 `content/`와 `public/`을 읽는다(Task 1에서 복사했다).

Run: `pnpm exec tsc --noEmit -p tsconfig.json`
Expected: 오류 없음. `repository.ts`의 `process.env.NODE_ENV` 관련 주석은 Next 타입 얘기라 무시해도 된다.

- [ ] **Step 10: 커밋**

```bash
git add scripts/rewrite-imports.ts scripts/rewrite-imports.test.ts src/lib
git commit -m "♻️ refactor: blog의 lib를 옮기고 import를 상대 경로로 바꾼다"
```

---

### Task 4: head 메타데이터

**Files:**
- Create: `src/head.tsx`
- Test: `src/head.test.tsx`

**Interfaces:**
- Consumes: `localePath(lang, path)`, `SITE_URL`, `SITE_COPY[lang].siteName`
- Produces:
  - `interface MetadataOptions { title; description; lang: Lang; imageUrl?; type?: "article" | "website"; path?; hasCounterpart? }` (blog `app/utils/metadata.ts`와 같다)
  - `interface ResolvedMeta { title: string; description: string; canonical?: string; languages?: Record<string, string>; siteName: string; imageUrl: string; locale: string; type: "article" | "website" }`
  - `resolveMeta(options: MetadataOptions): ResolvedMeta`
  - `HeadTags({ meta }: { meta: ResolvedMeta }): SafeHtml` — Next가 내보내는 순서대로 `<title>`부터 아이콘 `<link>`까지

- [ ] **Step 1: 실패하는 테스트 작성**

`src/head.test.tsx`:

```tsx
import { test } from "node:test";
import assert from "node:assert/strict";
import { HeadTags, resolveMeta } from "./head.tsx";

test("path가 있으면 canonical과 양쪽 hreflang을 만든다", () => {
  const meta = resolveMeta({ title: "T", description: "D", lang: "en", path: "/about", hasCounterpart: true });
  assert.equal(meta.canonical, "https://iaman.kr/en/about");
  assert.deepEqual(meta.languages, { ko: "https://iaman.kr/about", en: "https://iaman.kr/en/about" });
  assert.equal(meta.locale, "en_US");
  assert.equal(meta.imageUrl, "https://iaman.kr/default-thumbnail.png");
  assert.equal(meta.type, "article");
});

test("반대판이 없으면 자기 언어만, path가 없으면 둘 다 없다", () => {
  const one = resolveMeta({ title: "T", description: "D", lang: "ko", path: "/x" });
  assert.deepEqual(one.languages, { ko: "https://iaman.kr/x" });
  const none = resolveMeta({ title: "T", description: "D", lang: "ko" });
  assert.equal(none.canonical, undefined);
  assert.equal(none.languages, undefined);
});

test("HeadTags는 Next와 같은 태그를 같은 순서로 낸다", () => {
  const html = String(
    <HeadTags meta={resolveMeta({ title: "A's", description: "d", lang: "ko", path: "/", hasCounterpart: true })} />,
  );
  const expected = [
    `<title>A&#x27;s</title>`,
    `<meta name="description" content="d"/>`,
    `<meta name="robots" content="index, follow"/>`,
    `<meta name="googlebot" content="index, follow"/>`,
    `<link rel="canonical" href="https://iaman.kr/"/>`,
    `<link rel="alternate" hreflang="ko" href="https://iaman.kr/"/>`,
    `<link rel="alternate" hreflang="en" href="https://iaman.kr/en"/>`,
    `<meta property="og:title" content="A&#x27;s"/>`,
    `<meta property="og:description" content="d"/>`,
    `<meta property="og:site_name" content="iaman"/>`,
    `<meta property="og:locale" content="ko_KR"/>`,
    `<meta property="og:image" content="https://iaman.kr/default-thumbnail.png"/>`,
    `<meta property="og:image:alt" content="A&#x27;s"/>`,
    `<meta property="og:type" content="article"/>`,
    `<meta name="twitter:card" content="summary_large_image"/>`,
    `<meta name="twitter:title" content="A&#x27;s"/>`,
    `<meta name="twitter:description" content="d"/>`,
    `<meta name="twitter:image" content="https://iaman.kr/default-thumbnail.png"/>`,
    `<meta name="twitter:image:alt" content="A&#x27;s"/>`,
    `<link rel="icon" href="/favicon.ico" sizes="96x96" type="image/x-icon"/>`,
    `<link rel="icon" href="/icon.png" sizes="192x192" type="image/png"/>`,
    `<link rel="apple-touch-icon" href="/apple-icon.png" sizes="180x180" type="image/png"/>`,
  ].join("");
  assert.equal(html, expected);
});
```

`og:site_name`의 기대값 `iaman`은 `SITE_COPY.ko.siteName`이다. 다르면 `src/lib/i18n/site-copy.ts`의 값으로 테스트를 맞춘다.

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test src/head.test.tsx`
Expected: FAIL — `Cannot find module './head.tsx'`

- [ ] **Step 3: head.tsx 작성**

`src/head.tsx`:

```tsx
/****************************************
 *
 * <head> 메타 태그 — Next의 generateMetadata 대체
 *
 * resolveMeta는 blog app/utils/metadata.ts 의 generateCommonMetadata 와 같은 규칙이다.
 * HeadTags는 Next 16이 실제로 내보낸 태그(out/about.html에서 확인)를 같은 순서로 쓴다.
 * twitter:* 는 Next가 openGraph에서 자동으로 만들던 것이다.
 *
 ****************************************/

import { SITE_URL } from "./lib/site.ts";
import { localePath } from "./lib/i18n/paths.ts";
import { SITE_COPY } from "./lib/i18n/site-copy.ts";
import type { Lang } from "./lib/posts/types.ts";

export interface MetadataOptions {
  title: string;
  description: string;
  lang: Lang;
  imageUrl?: string;
  type?: "article" | "website";
  /** 언어 접두사를 뺀 경로. 있으면 canonical·hreflang을 만든다. */
  path?: string;
  hasCounterpart?: boolean;
}

export interface ResolvedMeta {
  title: string;
  description: string;
  canonical?: string;
  languages?: Record<string, string>;
  siteName: string;
  imageUrl: string;
  locale: string;
  type: "article" | "website";
}

const OG_LOCALE: Record<Lang, string> = { ko: "ko_KR", en: "en_US" };

export function resolveMeta(options: MetadataOptions): ResolvedMeta {
  const {
    title,
    description,
    lang,
    imageUrl = `${SITE_URL}/default-thumbnail.png`,
    type = "article",
    path,
    hasCounterpart = false,
  } = options;

  const absolute = (l: Lang) => `${SITE_URL}${localePath(l, path ?? "/")}`;

  return {
    title,
    description,
    ...(path === undefined
      ? {}
      : {
          canonical: absolute(lang),
          languages: hasCounterpart ? { ko: absolute("ko"), en: absolute("en") } : { [lang]: absolute(lang) },
        }),
    siteName: SITE_COPY[lang].siteName,
    imageUrl,
    locale: OG_LOCALE[lang],
    type,
  };
}

export function HeadTags({ meta }: { meta: ResolvedMeta }) {
  return (
    <>
      <title>{meta.title}</title>
      <meta name="description" content={meta.description} />
      <meta name="robots" content="index, follow" />
      <meta name="googlebot" content="index, follow" />
      {meta.canonical !== undefined && <link rel="canonical" href={meta.canonical} />}
      {Object.entries(meta.languages ?? {}).map(([lang, href]) => (
        <link rel="alternate" hrefLang={lang} href={href} />
      ))}
      <meta property="og:title" content={meta.title} />
      <meta property="og:description" content={meta.description} />
      <meta property="og:site_name" content={meta.siteName} />
      <meta property="og:locale" content={meta.locale} />
      <meta property="og:image" content={meta.imageUrl} />
      <meta property="og:image:alt" content={meta.title} />
      <meta property="og:type" content={meta.type} />
      <meta name="twitter:card" content="summary_large_image" />
      <meta name="twitter:title" content={meta.title} />
      <meta name="twitter:description" content={meta.description} />
      <meta name="twitter:image" content={meta.imageUrl} />
      <meta name="twitter:image:alt" content={meta.title} />
      <link rel="icon" href="/favicon.ico" sizes="96x96" type="image/x-icon" />
      <link rel="icon" href="/icon.png" sizes="192x192" type="image/png" />
      <link rel="apple-touch-icon" href="/apple-icon.png" sizes="180x180" type="image/png" />
    </>
  );
}
```

- [ ] **Step 4: 테스트 통과 확인**

Run: `pnpm exec tsx --test src/head.test.tsx`
Expected: PASS, 3개 테스트 통과

- [ ] **Step 5: 커밋**

```bash
git add src/head.tsx src/head.test.tsx
git commit -m "✨ feat: 메타데이터를 head 태그로 그린다"
```

---

### Task 5: 라우트 표

**Files:**
- Create: `src/routes.ts`
- Test: `src/routes.test.ts`

**Interfaces:**
- Consumes: `LANGS` (repository), `postsIn`, `postListParams(posts): { slug: string; page: string[] }[]` (queries), `localePath`
- Produces:
  - `type RouteSpec = { kind: "home"; lang; path } | { kind: "about"; lang; path } | { kind: "article"; lang; path; slug } | { kind: "list"; lang; path; tag; page: number }` — `path`는 언어 접두사가 붙은 실제 URL 경로
  - `routes(posts: Post[]): RouteSpec[]`
  - `outputFile(urlPath: string): string` — `/` → `index.html`, `/en` → `en.html`, `/post/all/1` → `post/all/1.html` (항상 `/` 구분자)

- [ ] **Step 1: 실패하는 테스트 작성**

`src/routes.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { outputFile, routes } from "./routes.ts";
import type { Lang, Post } from "./lib/posts/types.ts";

function post(lang: Lang, slug: string, date: string, tags: string[]): Post {
  return { slug, title: slug, date, tags, draft: false, lang, fileName: `${slug}.md`, body: `# ${slug}` };
}

test("홈·소개는 언어마다, 글과 목록은 그 언어의 글에서 만든다", () => {
  const posts = [
    post("ko", "b", "2026-02-01", ["x", "y"]),
    post("ko", "a", "2026-01-01", ["x"]),
    post("en", "a", "2026-01-01", ["x"]),
  ];
  const paths = routes(posts).map((r) => r.path).sort();
  assert.deepEqual(paths, [
    "/",
    "/a",
    "/about",
    "/b",
    "/en",
    "/en/a",
    "/en/about",
    "/en/post/all/1",
    "/en/post/x/1",
    "/post/all/1",
    "/post/x/1",
    "/post/y/1",
  ]);
});

test("목록은 PAGE_SIZE(5)로 나뉜다", () => {
  const posts = Array.from({ length: 6 }, (_, i) => post("ko", `p${i}`, `2026-01-0${i + 1}`, ["t"]));
  const lists = routes(posts).filter((r) => r.kind === "list").map((r) => r.path).sort();
  assert.deepEqual(lists, ["/post/all/1", "/post/all/2", "/post/t/1", "/post/t/2"]);
});

test("RouteSpec이 페이지에 필요한 값을 담는다", () => {
  const specs = routes([post("ko", "a", "2026-01-01", ["x"])]);
  assert.deepEqual(specs.find((r) => r.path === "/a"), { kind: "article", lang: "ko", path: "/a", slug: "a" });
  assert.deepEqual(specs.find((r) => r.path === "/post/x/1"), {
    kind: "list", lang: "ko", path: "/post/x/1", tag: "x", page: 1,
  });
});

test("outputFile은 Next export와 같은 파일 이름을 쓴다", () => {
  assert.equal(outputFile("/"), "index.html");
  assert.equal(outputFile("/en"), "en.html");
  assert.equal(outputFile("/about"), "about.html");
  assert.equal(outputFile("/post/all/1"), "post/all/1.html");
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test src/routes.test.ts`
Expected: FAIL — `Cannot find module './routes.ts'`

- [ ] **Step 3: routes.ts 작성**

먼저 `src/lib/posts/queries.ts`에서 `postListParams`의 반환 타입이 `{ slug: string; page: string[] }[]`인지 확인한다(blog 라우트가 `Number(page[0])`로 읽는다). 다르면 아래 코드의 구조 분해를 맞춘다.

`src/routes.ts`:

```ts
/****************************************
 *
 * 라우트 표 — generateStaticParams 대체
 *
 * blog의 app/(ko)·app/(en) 폴더 구조가 하던 일을 목록 하나로 적는다.
 * 홈과 소개는 글 수와 상관없이 언어마다 생긴다(Next도 그렇다).
 *
 ****************************************/

import { LANGS } from "./lib/posts/repository.ts";
import { postListParams, postsIn } from "./lib/posts/queries.ts";
import { localePath } from "./lib/i18n/paths.ts";
import type { Lang, Post } from "./lib/posts/types.ts";

export type RouteSpec =
  | { kind: "home"; lang: Lang; path: string }
  | { kind: "about"; lang: Lang; path: string }
  | { kind: "article"; lang: Lang; path: string; slug: string }
  | { kind: "list"; lang: Lang; path: string; tag: string; page: number };

export function routes(posts: Post[]): RouteSpec[] {
  const specs: RouteSpec[] = [];

  for (const lang of LANGS) {
    const inLang = postsIn(posts, lang);

    specs.push({ kind: "home", lang, path: localePath(lang, "/") });
    specs.push({ kind: "about", lang, path: localePath(lang, "/about") });

    for (const post of inLang) {
      specs.push({ kind: "article", lang, path: localePath(lang, `/${post.slug}`), slug: post.slug });
    }

    for (const { slug, page } of postListParams(inLang)) {
      const n = Number(page[0]);
      specs.push({ kind: "list", lang, path: localePath(lang, `/post/${slug}/${n}`), tag: slug, page: n });
    }
  }

  return specs;
}

/** URL 경로 → dist 안의 파일. Next `output: "export"`(trailingSlash 없음)와 같다. */
export function outputFile(urlPath: string): string {
  return urlPath === "/" ? "index.html" : `${urlPath.slice(1)}.html`;
}
```

- [ ] **Step 4: 테스트 통과 확인**

Run: `pnpm exec tsx --test src/routes.test.ts`
Expected: PASS, 4개 테스트 통과

- [ ] **Step 5: 커밋**

```bash
git add src/routes.ts src/routes.test.ts
git commit -m "✨ feat: 생성할 URL 목록을 라우트 표로 만든다"
```

---

### Task 6: 공용 컴포넌트 이식

**Files:**
- Create (복사 후 수정): `src/components/Icon/*.tsx`, `src/components/SvgIcon/*`, `src/components/Card/*.tsx`, `src/components/PostArticle/*.tsx`, `src/components/Layout/{InnerHeader,Article}.tsx`, `src/components/Tags/{Tags,TagList}.tsx`, `src/components/Home/MostRead.tsx`, `src/components/About/AboutPanel.tsx`, `src/components/Article/*.tsx`, `src/components/PostPageContent.tsx`
- Create (새로 씀): `src/components/JsonLd.tsx`
- Test: `src/components/components.test.tsx`

**Interfaces:**
- Consumes: `jsx` 런타임(Task 2), `src/lib/*`(Task 3)
- Produces: blog와 같은 이름·props의 컴포넌트. 바뀐 점:
  - `CardDateArea` → `<time data-island="date" ...>` (상대 날짜는 `src/client/date.ts`가 붙인다)
  - `TagList` → 칩마다 `data-island-href`, 바깥에 `data-island="tag-list"`
  - `ArticleInteractions` → 정적 `<div id="article-content">` (제목 클릭은 `src/client/article.ts`)
  - `ArticleContent` → `raw()`로 본문을 넣는다
  - `JsonLd({ data })` → `<script type="application/ld+json">`
  - 타입 `Child`는 `src/jsx/jsx-runtime.ts`에서 가져온다

**이식 규칙** (모든 파일에 적용):
1. `"use client"` 줄과 `react` import를 지운다
2. `import Link from "next/link"`를 지우고 `<Link ...>` → `<a ...>`로 바꾼다
3. `React.ReactNode` → `Child` (`import type { Child } from "<상대경로>/jsx/jsx-runtime.ts"`), `React.FC<P>` → `(props: P) =>`, `React.SVGProps<SVGSVGElement>` → `Record<string, unknown>`
4. `key={...}` 속성을 지운다
5. `import "@/app/globals.css"`를 지운다
6. 마크업(태그, className, 텍스트)은 **바꾸지 않는다**. 비교의 기준이다

- [ ] **Step 1: 그대로 옮길 파일 복사**

```bash
mkdir -p src/components
cp -r ../blog/components/Icon ../blog/components/SvgIcon ../blog/components/Card ../blog/components/PostArticle ../blog/components/Home ../blog/components/About ../blog/components/Article ../blog/components/Tags src/components/
mkdir -p src/components/Layout
cp ../blog/components/Layout/InnerHeader.tsx ../blog/components/Layout/Article.tsx src/components/Layout/
cp ../blog/components/PostPageContent.tsx src/components/
```

- [ ] **Step 2: 이식 규칙 1~5를 각 파일에 적용**

대상: `Icon/*.tsx`(4개), `SvgIcon/*.tsx`(2개), `Card/{CardExcerptArea,CardImageArea,CardTagsArea,CardTitleArea}.tsx`, `PostArticle/*.tsx`(4개), `Home/MostRead.tsx`, `About/AboutPanel.tsx`, `Article/ArticleMeta.tsx`, `Tags/Tags.tsx`, `Layout/{InnerHeader,Article}.tsx`, `PostPageContent.tsx`.

예시 — `src/components/Layout/Article.tsx`는 이렇게 된다:

```tsx
import type { Child } from "../../jsx/jsx-runtime.ts";

const Article = ({ children, className }: { children: Child; className?: string }) => {
  return <main className={`mb-auto ${className || ""}`}>{children}</main>;
};
export default Article;
```

예시 — `src/components/Icon/SearchIcon.tsx`:

```tsx
/** 돋보기. stroke가 currentColor라 글자색을 따라간다. */
export default function SearchIcon(props: Record<string, unknown>) {
  return (
    <svg
      xmlns="http://www.w3.org/2000/svg"
      fill="none"
      viewBox="0 0 24 24"
      strokeWidth={1.5}
      stroke="currentColor"
      {...props}
    >
      <path
        strokeLinecap="round"
        strokeLinejoin="round"
        d="m21 21-5.197-5.197m0 0A7.5 7.5 0 1 0 5.196 5.196a7.5 7.5 0 0 0 10.607 10.607Z"
      />
    </svg>
  );
}
```

예시 — `src/components/Card/CardTagsArea.tsx`의 본문:

```tsx
    <div className={className}>
      {tags.map((tag: string) => (
        <a href={localePath(lang, `/post/${tag}/1`)} className={linkClassName}>
          {tag}
        </a>
      ))}
    </div>
```

- [ ] **Step 3: 동작이 있던 컴포넌트 다시 쓰기**

`src/components/Card/CardDateArea.tsx` (전체 교체):

```tsx
/****************************************
 *
 * CardDateArea : 아티클 날짜
 *
 * 절대 날짜는 빌드 때, 상대 표기("2일 전")는 브라우저에서 붙인다.
 * 빌드 때 계산하면 배포 순간의 값이 HTML에 박힌다. src/client/date.ts 가
 * data-island="date" 인 time 을 찾아 " (N일 전)" 을 덧붙인다.
 *
 ****************************************/

import type { Lang } from "../../lib/posts/types.ts";

interface CardDateAreaProps {
  date: string;
  className?: string;
  lang: Lang;
}

const CardDateArea = ({ date, className = "flex items-center mb-4" }: CardDateAreaProps) => {
  return (
    <div className={className}>
      <time className="text-gray-500 dark:text-gray-400" dateTime={date} data-island="date">
        {date}
      </time>
    </div>
  );
};

export default CardDateArea;
```

`lang` prop은 호출부와 맞추려고 남긴다. 클라이언트는 `<html lang>`에서 언어를 읽는다.

`src/components/Tags/TagList.tsx` (전체 교체):

```tsx
/****************************************
 *
 * TagList : 목록 상단의 태그 칩
 *
 * blog는 div 클릭을 router.push 로 처리했다. 여기서는 이동할 주소를
 * data-island-href 에 적고 src/client/tag-list.ts 가 클릭을 받는다.
 * 지금 태그는 주소가 없다 — 눌러도 아무 일이 없어야 한다.
 *
 ****************************************/

import { ALL_TAG } from "../../lib/posts/constants.ts";
import { localePath } from "../../lib/i18n/paths.ts";
import type { Lang } from "../../lib/posts/types.ts";

interface TagListProps {
  currTag: string;
  tagNames: string[];
  lang: Lang;
}

export default function TagList({ currTag, tagNames, lang }: TagListProps) {
  return (
    <div
      className="w-full flex overflow-x-auto space-x-1 whitespace-nowrap mt-4 pb-2 px-2"
      data-island="tag-list"
    >
      {[ALL_TAG, ...tagNames].map((tag) => (
        <div
          data-island-href={tag === currTag ? undefined : localePath(lang, `/post/${tag}/1`)}
          className={`shrink-0 px-3 py-1 rounded-full text-sm uppercase cursor-pointer ${
            currTag === tag ? "bg-blue-600 text-white" : "bg-gray-200 text-gray-800"
          }`}
        >
          {tag}
        </div>
      ))}
    </div>
  );
}
```

`src/components/Article/ArticleInteractions.tsx` (전체 교체):

```tsx
/****************************************
 *
 * ArticleInteractions : 본문을 감싼다
 *
 * 제목(h2) 클릭 → 부드러운 스크롤은 src/client/article.ts 가 이 div 안에서 찾아 붙인다.
 *
 ****************************************/

import type { Child } from "../../jsx/jsx-runtime.ts";

export default function ArticleInteractions({ children }: { children: Child }) {
  return <div id="article-content">{children}</div>;
}
```

`src/components/Article/ArticleContent.tsx` (전체 교체):

```tsx
import ArticleInteractions from "./ArticleInteractions.tsx";
import ArticleMeta from "./ArticleMeta.tsx";
import { META_MARKER } from "../../lib/processHtml.ts";
import { raw } from "../../jsx/html.ts";
import type { Lang } from "../../lib/posts/types.ts";

// 제목 바로 뒤에서 본문을 둘로 가르고 그 사이에 날짜·태그를 끼운다.
// 표시가 없으면(마크다운에 h1이 없는 경우) 통째로 하나로 내보낸다.
export default function ArticleContent({
  content,
  date,
  tags,
  lang,
}: {
  content: string;
  date: string;
  tags: string[];
  lang: Lang;
}) {
  const at = content.indexOf(META_MARKER);
  const head = at === -1 ? content : content.slice(0, at);
  const rest = at === -1 ? "" : content.slice(at + META_MARKER.length);

  return (
    <article>
      <ArticleInteractions>
        <div>{raw(head)}</div>
        <ArticleMeta date={date} tags={tags} lang={lang} />
        {rest !== "" && <div className="article-rest">{raw(rest)}</div>}
      </ArticleInteractions>
    </article>
  );
}
```

`src/components/JsonLd.tsx` (새 파일):

```tsx
import { raw } from "../jsx/html.ts";

/**
 * 구조화 데이터. JSON.stringify 는 </script> 를 막아 주지 않으므로
 * `<` 를 유니코드로 바꿔 태그가 닫히지 않게 한다.
 */
export function JsonLd({ data }: { data: object }) {
  return (
    <script type="application/ld+json">
      {raw(JSON.stringify(data).replace(/</g, "\\u003c"))}
    </script>
  );
}
```

- [ ] **Step 4: import 재작성**

Run: `pnpm exec tsx scripts/rewrite-imports.ts src/components`
Expected: `import 재작성: N개 파일`

Run: `grep -rn 'next/\|from "react"\|"@/\|use client\|dangerouslySetInnerHTML\|<Link' src/components`
Expected: 출력 없음

- [ ] **Step 5: 컴포넌트 테스트 작성**

`src/components/components.test.tsx`:

```tsx
import { test } from "node:test";
import assert from "node:assert/strict";
import CardTagsArea from "./Card/CardTagsArea.tsx";
import CardDateArea from "./Card/CardDateArea.tsx";
import TagList from "./Tags/TagList.tsx";
import ArticleContent from "./Article/ArticleContent.tsx";
import { JsonLd } from "./JsonLd.tsx";
import { META_MARKER } from "../lib/processHtml.ts";

test("태그 링크에 언어 접두사가 붙는다", () => {
  const html = String(<CardTagsArea tags={["react"]} lang="en" />);
  assert.match(html, /<a href="\/en\/post\/react\/1"/);
});

test("날짜는 time 하나로 그리고 상대 표기는 비워 둔다", () => {
  assert.equal(
    String(<CardDateArea date="2026-11-20" lang="ko" />),
    `<div class="flex items-center mb-4"><time class="text-gray-500 dark:text-gray-400" datetime="2026-11-20" data-island="date">2026-11-20</time></div>`,
  );
});

test("TagList: 지금 태그는 강조하고 주소를 달지 않는다", () => {
  const html = String(<TagList currTag="all" tagNames={["react"]} lang="ko" />);
  assert.match(html, /<div class="[^"]*bg-blue-600 text-white">all<\/div>/);
  assert.match(html, /data-island-href="\/post\/react\/1"[^>]*bg-gray-200/);
});

test("ArticleContent: 제목 뒤에 날짜·태그를 끼운다", () => {
  const html = String(
    <ArticleContent content={`<h1>T</h1>${META_MARKER}<p>body</p>`} date="2026-01-01" tags={[]} lang="ko" />,
  );
  assert.match(html, /<div><h1>T<\/h1><\/div><div class="article-meta">.*<\/div><div class="article-rest"><p>body<\/p><\/div>/);
});

test("JsonLd는 < 를 이스케이프한다", () => {
  assert.equal(
    String(<JsonLd data={{ a: "</script>" }} />),
    `<script type="application/ld+json">{"a":"\\u003c/script>"}</script>`,
  );
});
```

- [ ] **Step 6: 테스트와 타입 검사**

Run: `pnpm exec tsx --test src/components/components.test.tsx`
Expected: PASS, 5개 테스트 통과

Run: `pnpm exec tsc --noEmit -p tsconfig.json`
Expected: 오류 없음. `MenuToggleButton`·`Theme`·`Search`·`Carousel`·`Layout/Header` 등은 아직 없다(Task 7, 8). 복사한 파일이 그것들을 import하면 안 된다.

- [ ] **Step 7: 커밋**

```bash
git add src/components
git commit -m "♻️ refactor: 서버 컴포넌트를 React 없이 옮긴다"
```

---

### Task 7: 레이아웃과 헤더

**Files:**
- Create: `src/lib/i18n/switch.ts`, `src/lib/i18n/switch.test.ts`
- Create: `src/components/Layout/{Document,Header,Footer,LanguageSwitch}.tsx`, `src/components/Search/Search.tsx`, `src/components/Button/{Theme,MenuToggleButton}.tsx`
- Create: `src/styles/globals.css`, `tailwind.config.js`, `src/static/_headers`
- Test: `src/components/layout.test.tsx`

**Interfaces:**
- Consumes: `HeadTags`, `ResolvedMeta` (Task 4), `allPosts`, `bilingualSlugs`, `postsIn`, `searchIndex`, `messages`, `localePath`
- Produces:
  - `languageSwitchTarget(lang: Lang, pathname: string, bilingualSlugs: string[]): string | null` — 반대 언어 주소. 반대판이 없는 글이면 `null`
  - `Document({ lang, path, meta, scripts, children }): SafeHtml` — `scripts`는 페이지 전용 모듈 이름 배열(예: `["carousel"]`). 기본 모듈 `theme` `search` `menu` `date`는 항상 붙는다. 모듈 경로는 `/js/client/<name>.js`
  - `THEME_INIT_SCRIPT: string` — `<head>` 인라인 테마 스크립트 (NotFound도 쓴다)
  - `ASSET_HEAD: SafeHtml`을 만드는 `AssetLinks()` — charset, viewport, 폰트 preload, `/styles.css`
  - 마크업 계약 (Task 10의 클라이언트가 의존):
    - 테마 버튼 `[data-island="theme-toggle"]`
    - 검색 열기 버튼 `[data-island="search-open"]`, 모달 `[data-island="search-modal"]`(처음엔 `hidden` 클래스), 안쪽 `[data-island-role="container"|"close"|"input"|"results"|"index"]`
    - 메뉴 열기 `[data-island="menu-open"]`, 오버레이 `[data-island="menu"]`(처음엔 `hidden` 클래스), 안쪽 `[data-island-role="backdrop"|"close"]`

`hidden` 속성 대신 `hidden` **클래스**를 쓴다. Tailwind preflight의 `[hidden]{display:none}`는 같은 특이도의 `.flex`에 진다.

- [ ] **Step 1: 언어 전환 계산 테스트 작성**

`src/lib/i18n/switch.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { languageSwitchTarget } from "./switch.ts";

const both = ["a"];

test("홈·소개는 같은 자리의 반대 언어로 간다", () => {
  assert.equal(languageSwitchTarget("ko", "/", both), "/en");
  assert.equal(languageSwitchTarget("en", "/en", both), "/");
  assert.equal(languageSwitchTarget("ko", "/about", both), "/en/about");
  assert.equal(languageSwitchTarget("en", "/en/about", both), "/about");
});

test("글은 반대판이 있을 때만", () => {
  assert.equal(languageSwitchTarget("ko", "/a", both), "/en/a");
  assert.equal(languageSwitchTarget("en", "/en/a", both), "/a");
  assert.equal(languageSwitchTarget("ko", "/b", both), null);
});

test("목록은 반대 언어의 전체 목록 1페이지로", () => {
  assert.equal(languageSwitchTarget("ko", "/post/react/3", both), "/en/post/all/1");
  assert.equal(languageSwitchTarget("en", "/en/post/all/2", both), "/post/all/1");
});

test("/english 같은 slug를 접두사로 오인하지 않는다", () => {
  assert.equal(languageSwitchTarget("ko", "/english", ["english"]), "/en/english");
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test src/lib/i18n/switch.test.ts`
Expected: FAIL — `Cannot find module './switch.ts'`

- [ ] **Step 3: switch.ts 작성**

`src/lib/i18n/switch.ts`:

```ts
/**
 * 언어 전환 버튼이 가리킬 주소.
 *
 * blog는 클라이언트에서 usePathname()으로 계산했다. 여기는 빌드 때 페이지 경로를
 * 알고 있으므로 HTML에 주소를 박는다 — 이 기능에 필요한 JS가 0이 된다.
 * 규칙은 blog components/Layout/LanguageSwitch.tsx 와 같다.
 */
import { ALL_TAG } from "../posts/constants.ts";
import { localePath } from "./paths.ts";
import type { Lang } from "../posts/types.ts";

export function languageSwitchTarget(
  lang: Lang,
  pathname: string,
  bilingualSlugs: string[],
): string | null {
  const other: Lang = lang === "ko" ? "en" : "ko";
  const neutral = lang === "en" ? pathname.replace(/^\/en(?=\/|$)/, "") || "/" : pathname;

  if (neutral.startsWith("/post/")) {
    return localePath(other, `/post/${ALL_TAG}/1`);
  }

  const segments = neutral.split("/").filter((s) => s !== "");
  const slug = segments.length === 1 && segments[0] !== "about" ? segments[0] : null;
  if (slug !== null && !bilingualSlugs.includes(slug)) {
    return null;
  }

  return localePath(other, neutral);
}
```

- [ ] **Step 4: 통과 확인**

Run: `pnpm exec tsx --test src/lib/i18n/switch.test.ts`
Expected: PASS, 4개 테스트 통과

- [ ] **Step 5: 헤더 부품 작성**

`src/components/Layout/LanguageSwitch.tsx`:

```tsx
import { languageSwitchTarget } from "../../lib/i18n/switch.ts";
import { messages } from "../../lib/i18n/messages.ts";
import type { Lang } from "../../lib/posts/types.ts";

export default function LanguageSwitch({
  lang,
  path,
  bilingualSlugs,
}: {
  lang: Lang;
  path: string;
  bilingualSlugs: string[];
}) {
  const target = languageSwitchTarget(lang, path, bilingualSlugs);
  if (target === null) {
    return null;
  }
  const other: Lang = lang === "ko" ? "en" : "ko";
  return (
    <a className="p-4 text-sm font-semibold uppercase" href={target} hrefLang={other} title={messages(lang).switchTo}>
      {other.toUpperCase()}
    </a>
  );
}
```

`src/components/Button/Theme.tsx`:

```tsx
/**
 * 테마 토글. blog는 마운트 전까지 아무것도 그리지 않았다(테마를 서버가 몰라서).
 * 여기서는 두 아이콘을 다 그리고 html.dark 클래스로 하나만 보인다 — 깜빡임이 없다.
 * 클릭은 src/client/theme.ts.
 */
import DarkModeIcon from "../Icon/DarkModeIcon.tsx";
import LightModeIcon from "../Icon/LightModeIcon.tsx";

export default function Theme() {
  return (
    <div className="p-4">
      <button type="button" aria-label="ThemeButton" data-island="theme-toggle">
        <span className="dark:hidden">
          <LightModeIcon />
        </span>
        <span className="hidden dark:inline">
          <DarkModeIcon />
        </span>
      </button>
    </div>
  );
}
```

`src/components/Button/MenuToggleButton.tsx`:

```tsx
import MenuToggleIcon from "../Icon/MenuToggleIcon.tsx";
import { localePath } from "../../lib/i18n/paths.ts";
import { messages } from "../../lib/i18n/messages.ts";
import type { Lang } from "../../lib/posts/types.ts";

/** 모바일 메뉴. 오버레이를 미리 그려 두고 src/client/menu.ts 가 hidden 클래스를 켜고 끈다. */
export default function MenuToggleButton({ lang }: { lang: Lang }) {
  const m = messages(lang);
  return (
    <>
      <div className="p-4">
        <button
          type="button"
          className="rounded-md hover:bg-gray-100 dark:hover:bg-gray-800 flex justify-center items-center"
          aria-label={m.openMenu}
          data-island="menu-open"
        >
          <MenuToggleIcon className="w-6 h-6" />
        </button>
      </div>

      <div className="hidden sm:hidden fixed inset-0 z-50 bg-black/50 dark:bg-white/50" data-island="menu">
        <div className="absolute inset-0 bg-black bg-opacity-50" data-island-role="backdrop" />
        <div className="absolute top-0 right-0 w-64 h-full bg-gray-900 dark:bg-white  shadow-lg">
          <div className="flex justify-end p-4">
            <button
              className="p-2 rounded-md text-white dark:text-black hover:bg-gray-200 dark:hover:bg-gray-200"
              aria-label={m.closeMenu}
              data-island-role="close"
            >
              <svg className="w-6 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>
          <nav className="px-4 py-6">
            <div className="space-y-4">
              <a
                href={localePath(lang, "/post/all/1")}
                className="block p-3 text-lg  text-white hover:bg-gray-300 dark:text-black  rounded-md"
              >
                Posts
              </a>
              <a
                href={localePath(lang, "/about")}
                className="block p-3 text-lg text-white hover:bg-gray-300 dark:text-black  rounded-md"
              >
                About
              </a>
            </div>
          </nav>
        </div>
      </div>
    </>
  );
}
```

`src/components/Search/Search.tsx`:

```tsx
/****************************************
 *
 * 검색 — 버튼은 헤더에 두 번(데스크톱·모바일), 모달은 문서에 한 번
 *
 * 모달 마크업과 검색 인덱스(JSON)를 미리 실어 두고 src/client/search.ts 가
 * 열기·닫기·필터링을 한다. 인덱스는 그 언어의 글만 담는다(blog와 같다).
 *
 ****************************************/

import SearchIcon from "../Icon/SearchIcon.tsx";
import { messages } from "../../lib/i18n/messages.ts";
import { raw } from "../../jsx/html.ts";
import type { Lang, SearchEntry } from "../../lib/posts/types.ts";

export function SearchButton({ lang }: { lang: Lang }) {
  return (
    <div className="p-4">
      <button type="button" aria-label={messages(lang).searchLabel} data-island="search-open">
        <SearchIcon className="w-6 h-6" />
      </button>
    </div>
  );
}

export function SearchModal({ lang, index }: { lang: Lang; index: SearchEntry[] }) {
  const m = messages(lang);
  return (
    <div
      className="hidden fixed inset-0 z-50 items-start justify-center px-4 pt-[10vh] bg-black/40"
      data-island="search-modal"
    >
      <div className="w-full max-w-xl bg-white rounded-lg shadow-lg p-6 relative" data-island-role="container">
        <button
          type="button"
          className="absolute top-2 right-2 text-gray-400 hover:text-gray-600"
          data-island-role="close"
        >
          ✕
        </button>
        <div className="text-lg font-bold mb-4 dark:text-gray-500">{m.searchTitle}</div>
        <input
          type="search"
          placeholder={m.searchPlaceholder}
          className="w-full border rounded px-3 py-2 mb-4 "
          data-island-role="input"
        />
        <div className="text-gray-500 text-sm max-h-[min(400px,50vh)] w-full overflow-y-auto" data-island-role="results">
          <div className="text-center text-gray-400 px-6 py-10">{m.searchHint}</div>
        </div>
      </div>
      <script type="application/json" data-island-role="index">
        {raw(JSON.stringify(index).replace(/</g, "\\u003c"))}
      </script>
    </div>
  );
}
```

`src/components/Layout/Header.tsx`:

```tsx
import LanguageSwitch from "./LanguageSwitch.tsx";
import Theme from "../Button/Theme.tsx";
import MenuToggleButton from "../Button/MenuToggleButton.tsx";
import { SearchButton } from "../Search/Search.tsx";
import { allPosts } from "../../lib/posts/repository.ts";
import { bilingualSlugs } from "../../lib/posts/queries.ts";
import { localePath } from "../../lib/i18n/paths.ts";
import type { Lang } from "../../lib/posts/types.ts";

export default function Header({ lang, path }: { lang: Lang; path: string }) {
  const bothLangs = bilingualSlugs(allPosts());

  return (
    <header className="pt-10 pb-10 flex justify-between items-center">
      <a href={localePath(lang, "/")}>
        <div className="flex items-center">
          <img
            src="/iaman.png"
            alt="blog logo"
            className="w-10 h-10 rounded-full mr-2"
            aria-label="blogLogo"
            width={100}
            height={100}
          />
          <div className="flex max-[500px]:hidden">
            <span className="italic"> {"iaman"}</span>
          </div>
        </div>
      </a>
      <div className="flex justify-between items-center">
        <div className="hidden sm:flex items-center">
          <a className="p-4" href={localePath(lang, "/post/all/1")}>
            {"Posts"}
          </a>
          <a className="p-4" href={localePath(lang, "/about")}>
            {"About"}
          </a>
          <SearchButton lang={lang} />
          <LanguageSwitch lang={lang} path={path} bilingualSlugs={bothLangs} />
          <Theme />
        </div>
        <div className="sm:hidden flex items-center">
          <SearchButton lang={lang} />
          <LanguageSwitch lang={lang} path={path} bilingualSlugs={bothLangs} />
          <Theme />
          <MenuToggleButton lang={lang} />
        </div>
      </div>
    </header>
  );
}
```

`src/components/Layout/Footer.tsx`:

```tsx
/** blog는 마운트 뒤에만 그렸다(서버 HTML에 없음). 여기서는 바로 그린다 — 막을 이유가 없다. */
import { localePath } from "../../lib/i18n/paths.ts";
import type { Lang } from "../../lib/posts/types.ts";

export default function Footer({ lang }: { lang: Lang }) {
  return (
    <footer>
      <div className="pt-10 pb-10 flex flex-col items-center">
        <div className="flex">
          <div className="mr-2"> © 2026</div>
          <div className="w-px bg-[#d3d5d7] mx-2"></div>
          <div className="mr-2"> iaman </div>
          <div className="w-px bg-[#d3d5d7] mx-2"></div>
          <div>
            <a href={localePath(lang, "/")}>https://iaman.kr</a>
          </div>
        </div>
      </div>
    </footer>
  );
}
```

- [ ] **Step 6: Document 작성**

`src/components/Layout/Document.tsx`:

```tsx
/****************************************
 *
 * Document : <html> 전체 — blog RootShell + Next가 대신 그리던 <head>
 *
 ****************************************/

import Header from "./Header.tsx";
import Footer from "./Footer.tsx";
import Article from "./Article.tsx";
import { SearchModal } from "../Search/Search.tsx";
import { HeadTags, type ResolvedMeta } from "../../head.tsx";
import { raw } from "../../jsx/html.ts";
import type { Child } from "../../jsx/jsx-runtime.ts";
import { allPosts } from "../../lib/posts/repository.ts";
import { postsIn, searchIndex } from "../../lib/posts/queries.ts";
import type { Lang } from "../../lib/posts/types.ts";

/**
 * next-themes 와 같은 동작: 저장값(없으면 light)을 html 클래스에 적용한다.
 * "system"이면 OS 설정을 따른다. CSS보다 먼저 돌아야 깜빡임이 없어서 인라인이다.
 */
export const THEME_INIT_SCRIPT =
  `(function(){try{var d=document.documentElement;var t=localStorage.getItem("blog-Theme")||"light";` +
  `if(t==="system"){t=matchMedia("(prefers-color-scheme: dark)").matches?"dark":"light"}` +
  `d.classList.remove("light","dark");d.classList.add(t);d.style.colorScheme=t}catch(e){}})()`;

const BASE_SCRIPTS = ["theme", "search", "menu", "date"];

export function AssetLinks() {
  return (
    <>
      <meta charSet="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <link rel="preload" href="/fonts/pretendard-subset.woff2" as="font" crossOrigin="" type="font/woff2" />
      <link rel="stylesheet" href="/styles.css" />
    </>
  );
}

function Analytics() {
  if (process.env.NODE_ENV !== "production") {
    return null;
  }
  const id = process.env.GA_MEASURE_ID ?? "";
  return (
    <>
      <script async src={`https://www.googletagmanager.com/gtag/js?id=${id}`}></script>
      <script>
        {raw(
          `window.dataLayer=window.dataLayer||[];function gtag(){dataLayer.push(arguments);}` +
            `gtag('js',new Date());gtag('config','${id}',{page_path:window.location.pathname});`,
        )}
      </script>
    </>
  );
}

export function Document({
  lang,
  path,
  meta,
  scripts = [],
  children,
}: {
  lang: Lang;
  path: string;
  meta: ResolvedMeta;
  scripts?: string[];
  children: Child;
}) {
  return (
    <html lang={lang}>
      <head>
        <AssetLinks />
        <Analytics />
        <HeadTags meta={meta} />
        <script>{raw(THEME_INIT_SCRIPT)}</script>
        {[...BASE_SCRIPTS, ...scripts].map((name) => (
          <script type="module" src={`/js/client/${name}.js`}></script>
        ))}
        {process.env.BLOG_DEV === "1" && <script type="module" src="/__dev/reload.js"></script>}
      </head>
      <body className="font-custom min-h-full bg-white dark:bg-black text-black dark:text-white">
        <div className="mx-auto max-w-3xl px-6">
          <div className="flex flex-col justify-between min-h-screen">
            <Header lang={lang} path={path} />
            <Article>{children}</Article>
            <Footer lang={lang} />
          </div>
        </div>
        <SearchModal lang={lang} index={searchIndex(postsIn(allPosts(), lang))} />
      </body>
    </html>
  );
}
```

- [ ] **Step 7: 스타일 설정 복사**

```bash
mkdir -p src/styles src/static
cp ../blog/app/globals.css src/styles/globals.css
cp ../blog/tailwind.config.js tailwind.config.js
```

`tailwind.config.js`의 `content`를 바꾼다:

```js
  content: ["./src/**/*.{ts,tsx}"],
```

`src/styles/globals.css`의 `@tailwind utilities;` 바로 아래에 넣는다 (Next의 `next/font/local`이 하던 일):

```css
/* blog app/utils/fonts.ts(next/font/local)를 대신한다. 변수 폰트 하나가 100~900을 맡는다. */
@font-face {
  font-family: "Pretendard Subset";
  src: url("/fonts/pretendard-subset.woff2") format("woff2");
  font-weight: 100 900;
  font-style: normal;
  font-display: swap;
}

:root {
  --font-custom: "Pretendard Subset", Pretendard, -apple-system, BlinkMacSystemFont, system-ui,
    "Segoe UI", "Apple SD Gothic Neo", "Malgun Gothic", "Noto Sans KR", sans-serif;
}
```

`src/static/_headers`:

```text
/*
  Cache-Control: public, max-age=0, must-revalidate

/images/*
  Cache-Control: public, max-age=31536000, immutable
```

blog의 `/_next/static/*` 규칙은 뺐다. 파일 이름에 해시가 없어서 오래 캐시하면 배포 후 옛 JS가 남는다.

- [ ] **Step 8: 레이아웃 테스트 작성**

`src/components/layout.test.tsx`:

```tsx
import { test } from "node:test";
import assert from "node:assert/strict";
import { Document } from "./Layout/Document.tsx";
import { resolveMeta } from "../head.tsx";

const render = (path: string, scripts?: string[]) =>
  String(
    <Document lang="ko" path={path} meta={resolveMeta({ title: "T", description: "D", lang: "ko" })} scripts={scripts}>
      <p>child</p>
    </Document>,
  );

test("문서 뼈대: lang, 본문은 main 안, 기본 모듈과 페이지 모듈", () => {
  const html = render("/", ["carousel"]);
  assert.match(html, /^<html lang="ko"><head><meta charset="utf-8"\/>/);
  assert.match(html, /<main class="mb-auto "><p>child<\/p><\/main>/);
  for (const name of ["theme", "search", "menu", "date", "carousel"]) {
    assert.ok(html.includes(`<script type="module" src="/js/client/${name}.js"></script>`), name);
  }
});

test("헤더의 언어 전환은 빌드 때 주소가 정해진다", () => {
  assert.ok(render("/about").includes(`href="/en/about" hreflang="en"`));
});

test("검색 모달과 메뉴는 처음에 hidden 클래스로 숨는다", () => {
  const html = render("/");
  assert.match(html, /<div class="hidden fixed inset-0 z-50 items-start[^"]*" data-island="search-modal">/);
  assert.match(html, /<div class="hidden sm:hidden fixed[^"]*" data-island="menu">/);
  assert.match(html, /<script type="application\/json" data-island-role="index">\[/);
});
```

- [ ] **Step 9: 테스트와 타입 검사**

Run: `pnpm exec tsx --test src/components/layout.test.tsx src/lib/i18n/switch.test.ts`
Expected: PASS, 7개 테스트 통과

Run: `pnpm exec tsc --noEmit -p tsconfig.json`
Expected: 오류 없음

- [ ] **Step 10: 커밋**

```bash
git add src tailwind.config.js
git commit -m "✨ feat: 문서 뼈대와 헤더를 정적 마크업으로 그린다"
```

---

### Task 8: 캐러셀과 페이지, 라우트 렌더

**Files:**
- Modify: `src/lib/carousel/geometry.ts` (상수 추가)
- Create: `src/components/Carousel/{CarouselCard,CarouselControls,LatestCarousel}.tsx`
- Create: `src/pages/{HomePage,ArticlePage,PostListPage,AboutPage,NotFoundPage}.tsx`, `src/render.tsx`
- Test: `src/render.test.tsx`

**Interfaces:**
- Consumes: `RouteSpec` (Task 5), `Document` `AssetLinks` `THEME_INIT_SCRIPT` (Task 7), `resolveMeta` (Task 4), `JsonLd` (Task 6)
- Produces:
  - `BASE_CARD_WIDTH = 320` (geometry.ts에서 export)
  - `renderRoute(spec: RouteSpec): string` — `<!DOCTYPE html>`로 시작하는 완성 문서
  - `renderNotFound(): string`
  - 캐러셀 마크업 계약 (Task 10이 의존): `section[data-island="carousel"]`, 스테이지 `[data-island-role="stage"]`, 카드 `li[data-index]`, `[data-island-role="prev"|"next"|"count"|"live"]`, 점 버튼 `[data-island-dot="<i>"]`

- [ ] **Step 1: BASE_CARD_WIDTH 추가**

`src/lib/carousel/geometry.ts` 끝에 더한다:

```ts
/**
 * 서버가 첫 HTML을 그릴 때 쓰는 카드 폭. CSS --card-w 의 최대값과 같다.
 * 클라이언트는 붙은 뒤 실제 폭을 재어 다시 칠한다. (blog useCoverflow.ts 에 있던 값)
 */
export const BASE_CARD_WIDTH = 320;
```

- [ ] **Step 2: 캐러셀 컴포넌트 작성**

`src/components/Carousel/CarouselCard.tsx`: `../blog/components/Carousel/CarouselCard.tsx`를 복사하고 Task 6의 이식 규칙을 적용한다(마크업은 그대로).

```bash
mkdir -p src/components/Carousel
cp ../blog/components/Carousel/CarouselCard.tsx src/components/Carousel/
pnpm exec tsx scripts/rewrite-imports.ts src/components/Carousel
```

`src/components/Carousel/CarouselControls.tsx`:

```tsx
/**
 * 화살표 · 점 · 번호. 첫 상태(0번째)로 그려 두고, 움직임은 src/client/carousel.ts 가
 * disabled · is-current · aria-current · 번호를 고친다.
 */
import { messages } from "../../lib/i18n/messages.ts";
import type { Lang } from "../../lib/posts/types.ts";

export default function CarouselControls({ lang, total }: { lang: Lang; total: number }) {
  const m = messages(lang);
  const current = 0;

  return (
    <>
      <button
        type="button"
        className="carousel-arrow carousel-arrow-prev"
        aria-label={m.prevSlide}
        disabled={current === 0}
        data-island-role="prev"
      >
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" aria-hidden="true">
          <path d="M15 5 8 12l7 7" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" />
        </svg>
      </button>

      <button
        type="button"
        className="carousel-arrow carousel-arrow-next"
        aria-label={m.nextSlide}
        disabled={current === total - 1}
        data-island-role="next"
      >
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" aria-hidden="true">
          <path d="m9 5 7 7-7 7" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" />
        </svg>
      </button>

      <div className="mt-4 flex items-center justify-center gap-4">
        <ul className="flex list-none items-center">
          {Array.from({ length: total }, (_, i) => (
            <li>
              <button
                type="button"
                className={i === current ? "carousel-dot is-current" : "carousel-dot"}
                aria-label={m.goToSlide(i + 1)}
                aria-current={i === current ? "true" : undefined}
                data-island-dot={i}
              />
            </li>
          ))}
        </ul>
        <span className="text-sm tabular-nums text-gray-500 dark:text-gray-400" data-island-role="count">
          {current + 1} / {total}
        </span>
      </div>
    </>
  );
}
```

점 버튼에 `data-index`를 쓰지 않는다. 클릭 처리기가 `closest("[data-index]")`로 카드를 찾기 때문에 점이 카드로 오인된다.

`src/components/Carousel/LatestCarousel.tsx`:

```tsx
/****************************************
 *
 * LatestCarousel : 메인 최신 글 코버플로우 (마크업만)
 *
 * 첫 렌더에서 변환값을 inline style 로 박는다. JS가 붙기 전 첫 페인트에
 * 카드가 가운데 겹쳐 보이지 않게 한다. 움직임은 src/client/carousel.ts.
 *
 ****************************************/

import CarouselCard from "./CarouselCard.tsx";
import CarouselControls from "./CarouselControls.tsx";
import { BASE_CARD_WIDTH, slideTransform, transformCss } from "../../lib/carousel/geometry.ts";
import { messages } from "../../lib/i18n/messages.ts";
import type { CarouselSlide, Lang } from "../../lib/posts/types.ts";

function initialStyle(index: number) {
  const t = slideTransform(index, BASE_CARD_WIDTH);
  return { transform: transformCss(t), opacity: t.opacity, zIndex: t.zIndex };
}

export default function LatestCarousel({
  slides,
  lang,
  labelledBy,
}: {
  slides: CarouselSlide[];
  lang: Lang;
  labelledBy?: string;
}) {
  const total = slides.length;
  const m = messages(lang);

  return (
    <section
      className="carousel"
      aria-roledescription="carousel"
      aria-labelledby={labelledBy}
      aria-label={labelledBy ? undefined : m.carouselLabel}
      data-island="carousel"
    >
      <div className="carousel-stage" data-island-role="stage">
        <ul className="list-none">
          {slides.map((slide, i) => (
            <li
              data-index={i}
              className="carousel-slide"
              style={initialStyle(i)}
              aria-roledescription="slide"
              aria-label={m.slidePosition(i + 1, total)}
            >
              <CarouselCard slide={slide} lang={lang} priority={i === 0} />
            </li>
          ))}
        </ul>
      </div>

      <CarouselControls lang={lang} total={total} />

      <p className="sr-only" aria-live="polite" data-island-role="live">
        {m.slidePosition(1, total)}
      </p>
    </section>
  );
}
```

- [ ] **Step 3: 페이지 복사와 수정**

```bash
mkdir -p src/pages
cp ../blog/components/pages/HomePage.tsx ../blog/components/pages/ArticlePage.tsx ../blog/components/pages/PostListPage.tsx ../blog/components/pages/AboutPage.tsx src/pages/
```

각 파일에 Task 6 이식 규칙을 적용하고 다음을 고친다:
- `import { notFound } from "next/navigation"`을 지우고, `notFound()` 호출을 `throw new Error(\`글이 없다: ${lang}/${slug}\`)`(ArticlePage), `throw new Error(\`빈 목록: ${lang}/${tag}/${page}\`)`(PostListPage)로 바꾼다. 라우트 표가 있는 것만 만들므로 여기 오면 버그다
- `ArticlePage`의 `<JsonLd .../>` import를 `../components/JsonLd.tsx`로 바꾸고 `@/app/lib/seo/json-ld`에서는 `blogPostingJsonLd`만 가져온다
- `HomePage`의 `<Link>`를 `<a>`로

그다음:

Run: `pnpm exec tsx scripts/rewrite-imports.ts src/pages`

`src/pages/NotFoundPage.tsx` (새로 씀):

```tsx
/**
 * 어떤 라우트에도 없는 주소의 페이지. 언어를 알 수 없어 영어로 적고 양쪽 홈으로 보낸다.
 * blog는 Next 구조 때문에 <html lang>·<title>을 스크립트로 채웠다. 여기서는 그냥 쓴다.
 */
import { AssetLinks, THEME_INIT_SCRIPT } from "../components/Layout/Document.tsx";
import { raw } from "../jsx/html.ts";

export default function NotFoundPage() {
  return (
    <html lang="en">
      <head>
        <AssetLinks />
        <title>404 | iaman</title>
        <script>{raw(THEME_INIT_SCRIPT)}</script>
      </head>
      <body className="font-custom bg-white dark:bg-black text-black dark:text-white">
        <div className="mx-auto max-w-3xl px-6 flex flex-col items-center justify-center min-h-screen gap-6">
          <p className="text-5xl font-black">404</p>
          <p className="font-bold text-2xl">Can Not Find This Page !</p>
          <div className="flex gap-3">
            <a href="/">
              <button className="bg-yellow-200 rounded-lg text-base py-4 px-8 text-black">
                <span lang="ko">한국어 홈</span>
              </button>
            </a>
            <a href="/en">
              <button className="bg-yellow-200 rounded-lg text-base py-4 px-8 text-black">English home</button>
            </a>
          </div>
        </div>
      </body>
    </html>
  );
}
```

- [ ] **Step 4: render 테스트 작성**

`src/render.test.tsx`:

```tsx
import { test } from "node:test";
import assert from "node:assert/strict";
import { renderNotFound, renderRoute } from "./render.tsx";
import { allPosts } from "./lib/posts/repository.ts";
import { postsIn } from "./lib/posts/queries.ts";

const firstKo = postsIn(allPosts(), "ko")[0];

test("홈: DOCTYPE, canonical, WebSite JSON-LD, 캐러셀 모듈", () => {
  const html = renderRoute({ kind: "home", lang: "ko", path: "/" });
  assert.ok(html.startsWith("<!DOCTYPE html><html lang=\"ko\">"));
  assert.ok(html.includes(`<link rel="canonical" href="https://iaman.kr/"/>`));
  assert.ok(html.includes(`"@type":"WebSite"`));
  assert.ok(html.includes(`/js/client/carousel.js`));
  assert.ok(html.includes(`data-island="carousel"`));
});

test("글: prose 래퍼, 본문, BlogPosting, 제목이 title", () => {
  const html = renderRoute({ kind: "article", lang: "ko", path: `/${firstKo.slug}`, slug: firstKo.slug });
  assert.ok(html.includes(`<div class="prose dark:prose-invert"><script type="application/ld+json">`));
  assert.ok(html.includes(`"@type":"BlogPosting"`));
  assert.ok(html.includes(`<div id="article-content">`));
  assert.ok(html.includes("/js/client/article.js"));
});

test("목록: canonical 없음, 태그 칩", () => {
  const html = renderRoute({ kind: "list", lang: "ko", path: "/post/all/1", tag: "all", page: 1 });
  assert.ok(!html.includes(`rel="canonical"`));
  assert.ok(html.includes(`<title>Posts by iaman</title>`));
  assert.ok(html.includes(`data-island="tag-list"`));
});

test("소개와 404", () => {
  assert.ok(renderRoute({ kind: "about", lang: "en", path: "/en/about" }).includes(`<title>About by iaman</title>`));
  const notFound = renderNotFound();
  assert.ok(notFound.startsWith(`<!DOCTYPE html><html lang="en">`));
  assert.ok(notFound.includes("<title>404 | iaman</title>"));
});
```

- [ ] **Step 5: 실패 확인**

Run: `pnpm exec tsx --test src/render.test.tsx`
Expected: FAIL — `Cannot find module './render.tsx'`

- [ ] **Step 6: render.tsx 작성**

`src/render.tsx`:

```tsx
/****************************************
 *
 * RouteSpec → 완성 HTML
 *
 * blog의 라우트 폴더마다 있던 page.tsx + layout.tsx(generateMetadata, 감싸는 div)를
 * 한곳에 모았다. 감싸는 div 모양은 비교를 위해 blog와 똑같이 둔다.
 *
 ****************************************/

import { Document } from "./components/Layout/Document.tsx";
import { JsonLd } from "./components/JsonLd.tsx";
import HomePage from "./pages/HomePage.tsx";
import ArticlePage from "./pages/ArticlePage.tsx";
import PostListPage from "./pages/PostListPage.tsx";
import AboutPage from "./pages/AboutPage.tsx";
import NotFoundPage from "./pages/NotFoundPage.tsx";
import { resolveMeta } from "./head.tsx";
import { allPosts } from "./lib/posts/repository.ts";
import { counterpart, postBySlug, postsIn } from "./lib/posts/queries.ts";
import { excerptOf } from "./lib/posts/excerpt.ts";
import { thumbnailOf } from "./lib/posts/thumbnail.ts";
import { absoluteUrl } from "./lib/site.ts";
import { SITE_COPY } from "./lib/i18n/site-copy.ts";
import { websiteJsonLd } from "./lib/seo/json-ld.ts";
import type { RouteSpec } from "./routes.ts";

const DOCTYPE = "<!DOCTYPE html>";

export function renderRoute(spec: RouteSpec): string {
  const { lang, path } = spec;

  switch (spec.kind) {
    case "home":
      return (
        DOCTYPE +
        String(
          <Document
            lang={lang}
            path={path}
            scripts={["carousel"]}
            meta={resolveMeta({ title: "iaman", description: SITE_COPY[lang].home, lang, path: "/", hasCounterpart: true })}
          >
            <JsonLd data={websiteJsonLd(lang)} />
            <HomePage lang={lang} />
          </Document>,
        )
      );

    case "about":
      return (
        DOCTYPE +
        String(
          <Document
            lang={lang}
            path={path}
            meta={resolveMeta({
              title: "About by iaman",
              description: SITE_COPY[lang].about,
              lang,
              path: "/about",
              hasCounterpart: true,
            })}
          >
            <div>
              <AboutPage lang={lang} />
            </div>
          </Document>,
        )
      );

    case "article": {
      const post = postBySlug(postsIn(allPosts(), lang), spec.slug);
      if (post === undefined) {
        throw new Error(`글이 없다: ${lang}/${spec.slug}`);
      }
      return (
        DOCTYPE +
        String(
          <Document
            lang={lang}
            path={path}
            scripts={["article"]}
            meta={resolveMeta({
              title: post.title,
              description: excerptOf(post),
              imageUrl: absoluteUrl(thumbnailOf(post)),
              lang,
              path: `/${post.slug}`,
              hasCounterpart: counterpart(allPosts(), post.slug, lang) !== undefined,
            })}
          >
            <div className="prose dark:prose-invert">
              <ArticlePage lang={lang} slug={spec.slug} />
            </div>
          </Document>,
        )
      );
    }

    case "list":
      return (
        DOCTYPE +
        String(
          <Document
            lang={lang}
            path={path}
            scripts={["tag-list"]}
            meta={resolveMeta({ title: "Posts by iaman", description: SITE_COPY[lang].postList, lang })}
          >
            <div>
              <PostListPage lang={lang} tag={spec.tag} page={spec.page} />
            </div>
          </Document>,
        )
      );
  }
}

export function renderNotFound(): string {
  return DOCTYPE + String(<NotFoundPage />);
}
```

- [ ] **Step 7: 테스트 통과 확인**

Run: `pnpm exec tsx --test src/render.test.tsx`
Expected: PASS, 4개 테스트 통과

`글: prose 래퍼` 테스트가 실패하면 `ArticlePage`가 `<>JsonLd, ArticleContent</>` 순서로 내는지 확인한다(blog와 같아야 한다).

- [ ] **Step 8: 전체 테스트와 타입 검사**

Run: `pnpm test` 그리고 `pnpm exec tsc --noEmit -p tsconfig.json`
Expected: 전부 PASS, 타입 오류 없음

- [ ] **Step 9: 커밋**

```bash
git add src
git commit -m "✨ feat: 페이지를 조립해 라우트마다 HTML을 만든다"
```

---

### Task 9: 빌드 스크립트

**Files:**
- Create: `src/build/sitemap.ts`, `src/build/links.ts`, `src/build/build.ts`, `scripts/build.ts`
- Test: `src/build/sitemap.test.ts`, `src/build/links.test.ts`

**Interfaces:**
- Consumes: `routes`, `outputFile` (Task 5), `renderRoute`, `renderNotFound` (Task 8)
- Produces:
  - `sitemapXml(posts: Post[]): string`, `robotsTxt(): string` — Next 출력과 같은 문자열
  - `findBrokenLinks(dir: string): { file: string; url: string }[]`
  - `build(options: { outDir?: string; dev?: boolean; only?: "html" | "css" | "js" }): Promise<void>` — `only`가 없으면 전체 빌드(디렉터리를 비우고 다시 만든다)
  - CLI: `pnpm build` (`--dev`, `--only=html|css|js`)

- [ ] **Step 1: sitemap·robots 테스트 작성**

`src/build/sitemap.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { robotsTxt, sitemapXml } from "./sitemap.ts";
import type { Lang, Post } from "../lib/posts/types.ts";

function post(lang: Lang, slug: string, date: string): Post {
  return { slug, title: slug, date, tags: ["x"], draft: false, lang, fileName: `${slug}.md`, body: "" };
}

test("robots.txt는 Next 출력과 같다", () => {
  assert.equal(
    robotsTxt(),
    "User-Agent: *\nAllow: /\nDisallow: /admin\n\nHost: https://iaman.kr\nSitemap: https://iaman.kr/sitemap.xml\n",
  );
});

test("sitemap: 양쪽 다 있는 주소에만 alternate, 목록에는 최신 날짜", () => {
  const xml = sitemapXml([post("ko", "a", "2026-02-01"), post("ko", "b", "2026-01-01"), post("en", "a", "2026-02-01")]);
  const lines = xml.split("\n");
  assert.equal(lines[0], `<?xml version="1.0" encoding="UTF-8"?>`);
  assert.equal(
    lines[1],
    `<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9" xmlns:xhtml="http://www.w3.org/1999/xhtml">`,
  );
  assert.ok(
    xml.includes(
      [
        "<url>",
        "<loc>https://iaman.kr/a</loc>",
        `<xhtml:link rel="alternate" hreflang="ko" href="https://iaman.kr/a" />`,
        `<xhtml:link rel="alternate" hreflang="en" href="https://iaman.kr/en/a" />`,
        "<lastmod>2026-02-01</lastmod>",
        "</url>",
      ].join("\n"),
    ),
  );
  assert.ok(xml.includes(["<url>", "<loc>https://iaman.kr/b</loc>", "<lastmod>2026-01-01</lastmod>", "</url>"].join("\n")));
  assert.ok(xml.includes(["<url>", "<loc>https://iaman.kr/post/all/1</loc>", "<lastmod>2026-02-01</lastmod>", "</url>"].join("\n")));
  assert.ok(xml.trimEnd().endsWith("</urlset>"));
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test src/build/sitemap.test.ts`
Expected: FAIL — `Cannot find module './sitemap.ts'`

- [ ] **Step 3: sitemap.ts 작성**

`src/build/sitemap.ts`:

```ts
/**
 * sitemap.xml · robots.txt — blog app/sitemap.ts, app/robots.ts 와 같은 내용.
 * 형식은 Next 16이 실제로 쓴 파일(out/sitemap.xml)을 그대로 따른다.
 */
import { SITE_URL } from "../lib/site.ts";
import { LANGS } from "../lib/posts/repository.ts";
import { bilingualSlugs, postListUrls, postsIn } from "../lib/posts/queries.ts";
import { localePath } from "../lib/i18n/paths.ts";
import { escapeHtml } from "../jsx/html.ts";
import type { Post } from "../lib/posts/types.ts";

interface Entry {
  url: string;
  lastModified?: string;
  alternates?: { ko: string; en: string };
}

function alternatesFor(path: string, both: boolean): Pick<Entry, "alternates"> {
  return both
    ? { alternates: { ko: `${SITE_URL}${localePath("ko", path)}`, en: `${SITE_URL}${localePath("en", path)}` } }
    : {};
}

function entries(posts: Post[]): Entry[] {
  const both = new Set(bilingualSlugs(posts));
  const bothLangsHavePosts = LANGS.every((lang) => postsIn(posts, lang).length > 0);
  const out: Entry[] = [];

  for (const lang of LANGS) {
    const inLang = postsIn(posts, lang);
    if (inLang.length === 0) {
      continue;
    }
    const latestDate = inLang[0]?.date;

    out.push({ url: `${SITE_URL}${localePath(lang, "/")}`, lastModified: latestDate, ...alternatesFor("/", bothLangsHavePosts) });
    out.push({ url: `${SITE_URL}${localePath(lang, "/about")}`, ...alternatesFor("/about", bothLangsHavePosts) });

    for (const post of inLang) {
      out.push({
        url: `${SITE_URL}${localePath(lang, `/${post.slug}`)}`,
        lastModified: post.date,
        ...alternatesFor(`/${post.slug}`, both.has(post.slug)),
      });
    }
    for (const path of postListUrls(inLang)) {
      out.push({ url: `${SITE_URL}${localePath(lang, path)}`, lastModified: latestDate });
    }
  }
  return out;
}

export function sitemapXml(posts: Post[]): string {
  const lines = [
    `<?xml version="1.0" encoding="UTF-8"?>`,
    `<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9" xmlns:xhtml="http://www.w3.org/1999/xhtml">`,
  ];
  for (const entry of entries(posts)) {
    lines.push("<url>", `<loc>${escapeHtml(entry.url)}</loc>`);
    if (entry.alternates !== undefined) {
      for (const [lang, href] of Object.entries(entry.alternates)) {
        lines.push(`<xhtml:link rel="alternate" hreflang="${lang}" href="${escapeHtml(href)}" />`);
      }
    }
    if (entry.lastModified !== undefined) {
      lines.push(`<lastmod>${entry.lastModified}</lastmod>`);
    }
    lines.push("</url>");
  }
  lines.push("</urlset>");
  return `${lines.join("\n")}\n`;
}

export function robotsTxt(): string {
  return `User-Agent: *\nAllow: /\nDisallow: /admin\n\nHost: ${SITE_URL}\nSitemap: ${SITE_URL}/sitemap.xml\n`;
}
```

- [ ] **Step 4: 통과 확인**

Run: `pnpm exec tsx --test src/build/sitemap.test.ts`
Expected: PASS, 2개 테스트 통과

- [ ] **Step 5: 링크 검사 테스트 작성**

`src/build/links.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import { findBrokenLinks } from "./links.ts";

test("없는 내부 주소만 보고한다. 외부·앵커·mailto는 건너뛴다", () => {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), "links-"));
  fs.mkdirSync(path.join(dir, "post", "all"), { recursive: true });
  fs.writeFileSync(path.join(dir, "post", "all", "1.html"), "");
  fs.writeFileSync(path.join(dir, "a.png"), "");
  fs.writeFileSync(
    path.join(dir, "index.html"),
    `<a href="/post/all/1">ok</a><a href="/missing">x</a><img src="/a.png"><img src="/b.png">` +
      `<a href="https://x.com/">e</a><a href="#top">t</a><a href="mailto:a@b.c">m</a><a href="/">home</a>`,
  );
  assert.deepEqual(findBrokenLinks(dir), [
    { file: "index.html", url: "/missing" },
    { file: "index.html", url: "/b.png" },
  ]);
});
```

- [ ] **Step 6: links.ts 작성**

`src/build/links.ts`:

```ts
/** dist 안 HTML의 내부 링크·이미지가 실제 파일로 이어지는지 본다. 빌드는 막지 않고 경고만 한다. */
import fs from "node:fs";
import path from "node:path";

export function resolvesTo(dir: string, url: string): boolean {
  const clean = decodeURIComponent(url.split(/[?#]/)[0]);
  if (clean === "/") {
    return fs.existsSync(path.join(dir, "index.html"));
  }
  const base = path.join(dir, clean);
  return [base, `${base}.html`, path.join(base, "index.html")].some(
    (file) => fs.existsSync(file) && fs.statSync(file).isFile(),
  );
}

export function findBrokenLinks(dir: string): { file: string; url: string }[] {
  const broken: { file: string; url: string }[] = [];
  const htmlFiles = fs
    .readdirSync(dir, { recursive: true })
    .map(String)
    .filter((f) => f.endsWith(".html"))
    .sort();

  for (const file of htmlFiles) {
    const html = fs.readFileSync(path.join(dir, file), "utf8");
    for (const match of html.matchAll(/\s(?:href|src)="(\/[^"]*)"/g)) {
      const url = match[1];
      if (url.startsWith("//") || resolvesTo(dir, url)) {
        continue;
      }
      broken.push({ file: file.split(path.sep).join("/"), url });
    }
  }
  return broken;
}
```

Run: `pnpm exec tsx --test src/build/links.test.ts`
Expected: PASS

- [ ] **Step 7: build.ts 작성**

`src/build/build.ts`:

```ts
/****************************************
 *
 * 빌드 — next build 대체
 *
 * 1. public/ + src/static/ 복사   2. 라우트마다 HTML   3. 404·sitemap·robots
 * 4. Tailwind CLI → styles.css    5. tsc → js/        6. 깨진 링크 경고
 *
 * NODE_ENV 는 글 목록을 읽기 전에 정한다. production 이면 draft 글이 빠진다(blog와 같다).
 *
 ****************************************/

import fs from "node:fs";
import path from "node:path";
import { execFileSync } from "node:child_process";

export interface BuildOptions {
  outDir?: string;
  dev?: boolean;
  only?: "html" | "css" | "js";
}

const TAILWIND_CLI = path.join("node_modules", "tailwindcss", "lib", "cli.js");
const TSC = path.join("node_modules", "typescript", "bin", "tsc");

function writeFile(outDir: string, file: string, content: string): void {
  const full = path.join(outDir, file);
  fs.mkdirSync(path.dirname(full), { recursive: true });
  fs.writeFileSync(full, content);
}

async function buildHtml(outDir: string): Promise<number> {
  // NODE_ENV 를 정한 뒤에 불러와야 한다 — 그래서 동적 import.
  const { allPosts } = await import("../lib/posts/repository.ts");
  const { routes, outputFile } = await import("../routes.ts");
  const { renderRoute, renderNotFound } = await import("../render.tsx");
  const { sitemapXml, robotsTxt } = await import("./sitemap.ts");

  const posts = allPosts();
  const written = new Map<string, string>();

  for (const spec of routes(posts)) {
    const file = outputFile(spec.path);
    const previous = written.get(file);
    if (previous !== undefined) {
      throw new Error(`같은 파일을 두 라우트가 만든다: ${file} (${previous}, ${spec.path})`);
    }
    if (fs.existsSync(path.join(outDir, file))) {
      throw new Error(`라우트가 public 파일을 덮어쓴다: ${file} (${spec.path})`);
    }
    let html: string;
    try {
      html = renderRoute(spec);
    } catch (error) {
      throw new Error(`${spec.path} 렌더 실패: ${(error as Error).message}`, { cause: error });
    }
    writeFile(outDir, file, html);
    written.set(file, spec.path);
  }

  writeFile(outDir, "404.html", renderNotFound());
  writeFile(outDir, "sitemap.xml", sitemapXml(posts));
  writeFile(outDir, "robots.txt", robotsTxt());
  return written.size;
}

function buildCss(outDir: string, dev: boolean): void {
  execFileSync(
    process.execPath,
    [TAILWIND_CLI, "-c", "tailwind.config.js", "-i", path.join("src", "styles", "globals.css"),
      "-o", path.join(outDir, "styles.css"), ...(dev ? [] : ["--minify"])],
    { stdio: ["ignore", "ignore", "inherit"] },
  );
}

function buildJs(outDir: string): void {
  execFileSync(process.execPath, [TSC, "-p", "tsconfig.client.json", "--outDir", path.join(outDir, "js")], {
    stdio: "inherit",
  });
}

export async function build({ outDir = "dist", dev = false, only }: BuildOptions = {}): Promise<void> {
  const started = performance.now();
  process.env.NODE_ENV = dev ? "development" : "production";
  if (dev) {
    process.env.BLOG_DEV = "1";
  }

  if (only === "css") {
    buildCss(outDir, dev);
  } else if (only === "js") {
    buildJs(outDir);
    buildCss(outDir, dev); // 클라이언트가 만드는 클래스(검색 결과 등)도 CSS에 들어가야 한다
  } else if (only === "html") {
    await buildHtml(outDir);
  } else {
    fs.rmSync(outDir, { recursive: true, force: true });
    fs.cpSync("public", outDir, { recursive: true });
    fs.cpSync(path.join("src", "static"), outDir, { recursive: true });
    const pages = await buildHtml(outDir);
    buildCss(outDir, dev);
    buildJs(outDir);

    const broken = await import("./links.ts").then((m) => m.findBrokenLinks(outDir));
    for (const { file, url } of broken) {
      console.warn(`⚠ 깨진 링크: ${file} → ${url}`);
    }
    console.log(`페이지 ${pages}개 + 404, 깨진 링크 ${broken.length}개`);
  }

  console.log(`빌드 ${only ?? "전체"} 완료: ${Math.round(performance.now() - started)}ms → ${outDir}/`);
}
```

`scripts/build.ts`:

```ts
import { build } from "../src/build/build.ts";

const dev = process.argv.includes("--dev");
const only = process.argv.find((arg) => arg.startsWith("--only="))?.slice("--only=".length) as
  | "html"
  | "css"
  | "js"
  | undefined;

try {
  await build({ dev, only });
} catch (error) {
  console.error(error instanceof Error ? (error.stack ?? error.message) : error);
  process.exitCode = 1;
}
```

- [ ] **Step 8: 실제 빌드 확인**

`src/client/`가 아직 비어 있으면 tsc가 "No inputs were found"로 실패한다. 빈 모듈을 먼저 만든다:

```bash
mkdir -p src/client
for name in theme search menu date tag-list article carousel; do echo "export {};" > src/client/$name.ts; done
```

Run: `pnpm build`
Expected:
- `페이지 N개 + 404, 깨진 링크 0개` (N은 blog `out/`의 `.html` 수에서 `404.html`, `_not-found.html`을 뺀 수와 같아야 한다. blog에 admin 페이지가 있으므로 `admin.html`도 뺀다)
- `dist/index.html`, `dist/en.html`, `dist/about.html`, `dist/post/all/1.html`, `dist/styles.css`, `dist/js/client/theme.js`, `dist/sitemap.xml`, `dist/robots.txt`, `dist/_headers`가 있다

깨진 링크가 나오면 원인(글 본문의 오타인지, 라우트 누락인지)을 확인하고 `docs/comparison.md`에 적는다. blog에도 같은 링크가 있으면 원본 문제다.

Run: `grep -c "pretendard" dist/styles.css`
Expected: 1 이상 (`@font-face`가 들어갔다)

- [ ] **Step 9: 커밋**

```bash
git add src scripts/build.ts
git commit -m "✨ feat: 라우트·sitemap·CSS·JS를 dist로 빌드한다"
```

---

### Task 10: 클라이언트 모듈

**Files:**
- Modify (빈 모듈을 채운다): `src/client/{theme,date,tag-list,article,menu,search,carousel}.ts`

**Interfaces:**
- Consumes: Task 6~8의 마크업 계약(`data-island*`), `relativeLabel(date: string, now: Date, lang: Lang): string | null`, `filterByTitle(index, query)`, `localePath`, `messages`, `slideTransform` `transformCss` `clamp` `BASE_CARD_WIDTH`, `dragPosition` `isDrag` `releaseTarget`
- Produces: `dist/js/client/*.js` (브라우저 ES 모듈). 각 모듈은 자기 요소가 없으면 아무것도 하지 않는다

이 작업은 DOM에 붙는 코드라 node:test로 돌리지 않는다. 계산은 이미 `lib/`의 테스트가 덮는다(`relative`, `queries`, `carousel/drag`, `carousel/geometry`). 검증은 타입 검사와 Step 9의 브라우저 체크리스트로 한다.

- [ ] **Step 1: theme.ts**

```ts
/** 테마 토글 — next-themes setTheme 대체. 저장 키와 값은 blog와 같다. */
const KEY = "blog-Theme";

function apply(theme: "light" | "dark"): void {
  const root = document.documentElement;
  root.classList.remove("light", "dark");
  root.classList.add(theme);
  root.style.colorScheme = theme;
  try {
    localStorage.setItem(KEY, theme);
  } catch {
    // 저장소가 막혀 있어도 이번 페이지에서는 바뀐다.
  }
}

for (const button of document.querySelectorAll<HTMLButtonElement>('[data-island="theme-toggle"]')) {
  button.addEventListener("click", () => {
    apply(document.documentElement.classList.contains("dark") ? "light" : "dark");
  });
}
```

- [ ] **Step 2: date.ts**

```ts
/** 빌드 때 박힌 절대 날짜 뒤에 " (N일 전)" 을 붙인다. blog CardDateArea 의 useEffect 대체. */
import { relativeLabel } from "../lib/date/relative.ts";
import type { Lang } from "../lib/posts/types.ts";

const lang = document.documentElement.lang as Lang;
const now = new Date();

for (const time of document.querySelectorAll<HTMLTimeElement>('time[data-island="date"]')) {
  const label = relativeLabel(time.dateTime, now, lang);
  if (label !== null) {
    time.textContent = `${time.dateTime} (${label})`;
  }
}
```

`relativeLabel`의 실제 시그니처를 `src/lib/date/relative.ts`에서 확인하고 다르면 맞춘다.

- [ ] **Step 3: tag-list.ts, article.ts**

`src/client/tag-list.ts`:

```ts
/** 태그 칩 클릭 → 그 태그 목록 1페이지. blog TagList 의 router.push 대체. */
const ACTIVE = ["bg-blue-600", "text-white"];
const IDLE = ["bg-gray-200", "text-gray-800"];

for (const list of document.querySelectorAll<HTMLElement>('[data-island="tag-list"]')) {
  const chips = Array.from(list.children) as HTMLElement[];
  for (const chip of chips) {
    const href = chip.dataset.islandHref;
    if (href === undefined) {
      continue; // 지금 태그
    }
    chip.addEventListener("click", () => {
      for (const other of chips) {
        other.classList.remove(...ACTIVE);
        other.classList.add(...IDLE);
      }
      chip.classList.remove(...IDLE);
      chip.classList.add(...ACTIVE);
      location.href = href;
    });
  }
}
```

`src/client/article.ts`:

```ts
/** 본문 h2 클릭 → 그 제목으로 부드럽게 스크롤. blog ArticleInteractions 대체. */
const container = document.getElementById("article-content");

for (const heading of container?.querySelectorAll("h2") ?? []) {
  heading.addEventListener("click", () => {
    document.getElementById(heading.id)?.scrollIntoView({ behavior: "smooth", block: "start" });
  });
}
```

- [ ] **Step 4: menu.ts**

```ts
/** 모바일 메뉴 열기·닫기. 배경·닫기 버튼·링크를 누르면 닫힌다. */
const overlay = document.querySelector<HTMLElement>('[data-island="menu"]');
const openButton = document.querySelector<HTMLElement>('[data-island="menu-open"]');

if (overlay !== null && openButton !== null) {
  const setOpen = (open: boolean) => overlay.classList.toggle("hidden", !open);

  openButton.addEventListener("click", () => setOpen(overlay.classList.contains("hidden")));
  for (const el of overlay.querySelectorAll('[data-island-role="backdrop"], [data-island-role="close"], a')) {
    el.addEventListener("click", () => setOpen(false));
  }
}
```

- [ ] **Step 5: search.ts**

```ts
/****************************************
 *
 * 검색 모달 — blog Search.tsx + useSearchData + useSearchEvent 대체
 *
 * 인덱스는 페이지에 실린 JSON. 필터 규칙은 lib/posts/queries.ts 의 filterByTitle
 * 그대로라 테스트가 덮는다. 여기는 DOM 배선만 한다.
 *
 ****************************************/

import { filterByTitle } from "../lib/posts/queries.ts";
import { localePath } from "../lib/i18n/paths.ts";
import { messages } from "../lib/i18n/messages.ts";
import type { Lang, SearchEntry } from "../lib/posts/types.ts";

const FOCUSABLE = 'a[href], input:not([disabled]), button:not([disabled]), [tabindex]:not([tabindex="-1"])';
const modal = document.querySelector<HTMLElement>('[data-island="search-modal"]');

if (modal !== null) {
  setup(modal);
}

function setup(modal: HTMLElement): void {
  const lang = document.documentElement.lang as Lang;
  const m = messages(lang);
  const part = <T extends HTMLElement>(role: string) =>
    modal.querySelector<T>(`[data-island-role="${role}"]`) as T;

  const container = part<HTMLElement>("container");
  const input = part<HTMLInputElement>("input");
  const results = part<HTMLElement>("results");
  const index = JSON.parse(part<HTMLElement>("index").textContent ?? "[]") as SearchEntry[];

  const row = (entry: SearchEntry): HTMLElement => {
    const el = document.createElement("div");
    el.className =
      "flex cursor-pointer justify-between px-4 py-2 text-gray-700 dark:text-gray-100 bg-transparent hover:bg-yellow-100";
    el.tabIndex = 0;
    const date = document.createElement("div");
    date.className = "text-gray-400 text-sm";
    date.textContent = entry.date;
    const title = document.createElement("div");
    title.className = "dark:text-gray-500 truncate ml-10 flex-1";
    title.textContent = entry.title;
    el.append(date, title);
    el.addEventListener("click", () => {
      close();
      location.href = localePath(lang, `/${entry.slug}`);
    });
    return el;
  };

  const message = (text: string): HTMLElement => {
    const el = document.createElement("div");
    el.className = "text-center text-gray-400 px-6 py-10";
    el.textContent = text;
    return el;
  };

  const render = (): void => {
    const found = filterByTitle(index, input.value);
    results.replaceChildren(
      ...(found.length > 0
        ? found.map(row)
        : [message(input.value.trim() !== "" ? m.noResults : m.searchHint)]),
    );
  };

  const open = (): void => {
    modal.classList.remove("hidden");
    modal.classList.add("flex");
    input.value = "";
    render();
    input.focus();
  };

  const close = (): void => {
    modal.classList.add("hidden");
    modal.classList.remove("flex");
  };

  input.addEventListener("input", render);
  part<HTMLElement>("close").addEventListener("click", close);
  for (const button of document.querySelectorAll('[data-island="search-open"]')) {
    button.addEventListener("click", open);
  }

  // 포커스 가두기와 Esc — useSearchEvent 와 같다.
  container.addEventListener("keydown", (e) => {
    if (e.key === "Escape") {
      e.preventDefault();
      close();
      return;
    }
    if (e.key !== "Tab") {
      return;
    }
    const focusable = Array.from(container.querySelectorAll<HTMLElement>(FOCUSABLE));
    if (focusable.length === 0) {
      return;
    }
    const first = focusable[0];
    const last = focusable[focusable.length - 1];
    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();
    }
  });
}
```

`filterByTitle`의 시그니처가 `(index: SearchEntry[], query: string) => SearchEntry[]`인지 `src/lib/posts/queries.ts`에서 확인한다.

- [ ] **Step 6: carousel.ts**

```ts
/****************************************
 *
 * 코버플로우 캐러셀 — blog useCoverflow.ts 를 DOM 이벤트로 옮긴 것
 *
 * 산수는 lib/carousel/(geometry·drag)에 있고 테스트가 돈다.
 * React state 였던 current 는 지역 변수, ref 였던 값들도 지역 변수가 된다.
 * 리렌더가 없으니 화살표·점·번호는 renderControls()가 직접 고친다.
 *
 ****************************************/

import { BASE_CARD_WIDTH, clamp, slideTransform, transformCss } from "../lib/carousel/geometry.ts";
import { dragPosition, isDrag, releaseTarget } from "../lib/carousel/drag.ts";
import { messages } from "../lib/i18n/messages.ts";
import type { Lang } from "../lib/posts/types.ts";

for (const section of document.querySelectorAll<HTMLElement>('[data-island="carousel"]')) {
  setup(section);
}

/** 이 노드가 몇 번째 카드 안에 있는지. 카드 밖이면 null. */
function slideIndexOf(el: HTMLElement): number | null {
  const slide = el.closest<HTMLElement>("[data-index]");
  if (slide === null) {
    return null;
  }
  const index = Number(slide.dataset.index);
  return Number.isInteger(index) ? index : null;
}

function setup(section: HTMLElement): void {
  const m = messages(document.documentElement.lang as Lang);
  const part = <T extends HTMLElement>(role: string) =>
    section.querySelector<T>(`[data-island-role="${role}"]`) as T;

  const stage = part<HTMLElement>("stage");
  const prev = part<HTMLButtonElement>("prev");
  const next = part<HTMLButtonElement>("next");
  const count = part<HTMLElement>("count");
  const live = part<HTMLElement>("live");
  const slides = Array.from(stage.querySelectorAll<HTMLLIElement>("li[data-index]"));
  const dots = Array.from(section.querySelectorAll<HTMLButtonElement>("[data-island-dot]"));
  const total = slides.length;

  let pos = 0;
  let current = 0;
  let cardWidth = BASE_CARD_WIDTH;
  const drag = { active: false, startX: 0, startPos: 0, lastX: 0, lastT: 0, velocity: 0, moved: 0, captured: false };

  const paint = (): void => {
    slides.forEach((el, i) => {
      const t = slideTransform(i - pos, cardWidth);
      el.style.transform = transformCss(t);
      el.style.opacity = String(t.opacity);
      el.style.zIndex = String(t.zIndex);
    });
  };

  const renderControls = (): void => {
    prev.disabled = current === 0;
    next.disabled = current === total - 1;
    dots.forEach((dot, i) => {
      dot.className = i === current ? "carousel-dot is-current" : "carousel-dot";
      if (i === current) {
        dot.setAttribute("aria-current", "true");
      } else {
        dot.removeAttribute("aria-current");
      }
    });
    count.textContent = `${current + 1} / ${total}`;
    live.textContent = m.slidePosition(current + 1, total);
  };

  const settle = (target: number): void => {
    pos = clamp(Math.round(target), 0, total - 1);
    current = pos;
    renderControls();
    paint();
  };

  const measure = (): void => {
    // offsetWidth 는 레이아웃 폭이라 transform(scale)의 영향을 받지 않는다.
    if (slides[0] !== undefined) {
      cardWidth = slides[0].offsetWidth;
    }
    paint();
  };
  measure();
  window.addEventListener("resize", measure);

  prev.addEventListener("click", () => settle(current - 1));
  next.addEventListener("click", () => settle(current + 1));
  dots.forEach((dot, i) => dot.addEventListener("click", () => settle(i)));

  stage.addEventListener("pointerdown", (e) => {
    if (e.pointerType === "mouse" && e.button !== 0) {
      return;
    }
    Object.assign(drag, {
      active: true, startX: e.clientX, startPos: pos, lastX: e.clientX,
      lastT: e.timeStamp, velocity: 0, moved: 0, captured: false,
    });
    stage.classList.add("is-dragging");
  });

  stage.addEventListener("pointermove", (e) => {
    if (!drag.active) {
      return;
    }
    const dx = e.clientX - drag.startX;
    drag.moved = Math.max(drag.moved, Math.abs(dx));
    // 클릭 문턱을 넘은 뒤에 붙잡는다. 누르자마자 잡으면 Chrome이 click target 을
    // 스테이지로 바꿔 카드 링크가 안 눌린다(blog에서 실측).
    if (!drag.captured && isDrag(drag.moved)) {
      stage.setPointerCapture(e.pointerId);
      drag.captured = true;
    }
    pos = dragPosition(drag.startPos, dx, cardWidth, total);
    paint();
    const dt = e.timeStamp - drag.lastT;
    if (dt > 0) {
      drag.velocity = (e.clientX - drag.lastX) / dt;
      drag.lastX = e.clientX;
      drag.lastT = e.timeStamp;
    }
  });

  const release = (): void => {
    if (!drag.active) {
      return;
    }
    drag.active = false;
    stage.classList.remove("is-dragging"); // settle 전에 꺼야 전환 애니메이션이 붙는다
    settle(releaseTarget(pos, drag.velocity, cardWidth));
  };
  stage.addEventListener("pointerup", release);
  stage.addEventListener("pointercancel", release);
  stage.addEventListener("dragstart", (e) => e.preventDefault());

  // Tab 으로 옆 카드 링크에 들어오면 그 카드를 가운데로. 마우스 포커스는 거른다.
  section.addEventListener(
    "focus",
    (e) => {
      const target = e.target as HTMLElement;
      if (!target.matches(":focus-visible")) {
        return;
      }
      const index = slideIndexOf(target);
      if (index !== null && index !== current) {
        settle(index);
      }
    },
    true,
  );

  // 드래그 꼬리의 click 을 막고, 옆 카드 클릭은 글을 열지 않고 가운데로 데려온다.
  section.addEventListener(
    "click",
    (e) => {
      if (isDrag(drag.moved)) {
        e.preventDefault();
        e.stopPropagation();
        drag.moved = 0;
        return;
      }
      const index = slideIndexOf(e.target as HTMLElement);
      if (index === null || index === current) {
        return;
      }
      e.preventDefault();
      settle(index);
    },
    true,
  );

  section.addEventListener("keydown", (e) => {
    if (e.altKey || e.ctrlKey || e.metaKey || e.shiftKey) {
      return;
    }
    if (e.key === "ArrowLeft") {
      e.preventDefault();
      settle(current - 1);
    } else if (e.key === "ArrowRight") {
      e.preventDefault();
      settle(current + 1);
    }
  });
}
```

- [ ] **Step 7: 타입 검사와 빌드**

Run: `pnpm typecheck`
Expected: 오류 없음 (서버용·클라이언트용 tsconfig 둘 다)

Run: `pnpm build`
Expected: `dist/js/client/carousel.js`가 `import ... from "../lib/carousel/geometry.js"`처럼 `.js` 경로를 쓴다. `dist/js/lib/carousel/geometry.js`가 있다.

Run: `grep -l "node:" -r dist/js`
Expected: 출력 없음 (브라우저 모듈 그래프에 Node 모듈이 섞이지 않았다)

- [ ] **Step 8: 커밋**

```bash
git add src/client
git commit -m "✨ feat: 검색·테마·메뉴·캐러셀을 바닐라 모듈로 붙인다"
```

- [ ] **Step 9: 브라우저 확인 (Task 11의 dev 서버로)**

Task 11을 끝낸 뒤 `pnpm dev`로 띄우고 아래를 하나씩 확인한다. 결과는 `docs/comparison.md`의 "기능 확인" 표에 적는다.

| 페이지 | 확인할 것 |
|---|---|
| 모든 페이지 | 다크 토글 → 새로고침해도 유지, 깜빡임 없음 |
| 모든 페이지 | 검색 열기 → 입력 즉시 필터, Esc 닫기, Tab이 모달 안에서 돈다, 결과 클릭 → 이동 |
| 모든 페이지 (폭 640px 미만) | 메뉴 열기·닫기, 배경 클릭 닫기 |
| 모든 페이지 | 날짜 옆 "(N일 전)" (30일 이내 글) |
| 홈 | 캐러셀 드래그, 플릭, 화살표, 점, ←/→ 키, 옆 카드 클릭 → 가운데로, 가운데 카드 클릭 → 글 열림 |
| 글 | h2 클릭 → 스크롤, 언어 전환 링크(반대판 없으면 숨김) |
| 목록 | 태그 칩 클릭 → 이동, 페이지 이동 |

---

### Task 11: 개발 서버

**Files:**
- Create: `src/build/serve.ts`, `scripts/dev.ts`
- Test: `src/build/serve.test.ts`

**Interfaces:**
- Produces:
  - `resolveFile(root: string, urlPath: string): string | null` — Cloudflare Pages와 같은 규칙: `/` → `index.html`, 파일 그대로, `<p>.html`, `<p>/index.html`. 루트 밖은 `null`
  - `createStaticServer(root: string, extra?: (req, res) => boolean): http.Server` — `extra`가 `true`를 돌려주면 그 요청은 처리된 것으로 본다. 못 찾으면 `404.html`을 404로
  - CLI: `pnpm dev` (포트 `PORT` 환경변수, 기본 3000)

- [ ] **Step 1: 실패하는 테스트 작성**

`src/build/serve.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import { resolveFile } from "./serve.ts";

const root = fs.mkdtempSync(path.join(os.tmpdir(), "serve-"));
for (const file of ["index.html", "about.html", "en.html", "post/all/1.html", "js/client/theme.js"]) {
  fs.mkdirSync(path.dirname(path.join(root, file)), { recursive: true });
  fs.writeFileSync(path.join(root, file), file);
}

test("Cloudflare Pages와 같은 주소 → 파일 규칙", () => {
  assert.equal(resolveFile(root, "/"), path.join(root, "index.html"));
  assert.equal(resolveFile(root, "/about"), path.join(root, "about.html"));
  assert.equal(resolveFile(root, "/about/"), path.join(root, "about.html"));
  assert.equal(resolveFile(root, "/en"), path.join(root, "en.html"));
  assert.equal(resolveFile(root, "/post/all/1?x=1"), path.join(root, "post", "all", "1.html"));
  assert.equal(resolveFile(root, "/js/client/theme.js"), path.join(root, "js", "client", "theme.js"));
});

test("없는 파일과 루트 밖은 null", () => {
  assert.equal(resolveFile(root, "/nope"), null);
  assert.equal(resolveFile(root, "/../secret"), null);
  assert.equal(resolveFile(root, "/%2e%2e/secret"), null);
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test src/build/serve.test.ts`
Expected: FAIL — `Cannot find module './serve.ts'`

- [ ] **Step 3: serve.ts 작성**

`src/build/serve.ts`:

```ts
/** dist 정적 서버. dev 와 compare:metrics(Lighthouse)가 함께 쓴다. Node 내장 http 만 쓴다. */
import fs from "node:fs";
import http from "node:http";
import path from "node:path";

const MIME: Record<string, string> = {
  ".html": "text/html; charset=utf-8",
  ".css": "text/css; charset=utf-8",
  ".js": "text/javascript; charset=utf-8",
  ".json": "application/json; charset=utf-8",
  ".xml": "application/xml; charset=utf-8",
  ".txt": "text/plain; charset=utf-8",
  ".png": "image/png",
  ".jpg": "image/jpeg",
  ".jpeg": "image/jpeg",
  ".gif": "image/gif",
  ".webp": "image/webp",
  ".svg": "image/svg+xml",
  ".ico": "image/x-icon",
  ".woff2": "font/woff2",
};

const isFile = (file: string) => fs.existsSync(file) && fs.statSync(file).isFile();

export function resolveFile(root: string, urlPath: string): string | null {
  let clean: string;
  try {
    clean = decodeURIComponent(urlPath.split(/[?#]/)[0]);
  } catch {
    return null;
  }
  if (clean.length > 1 && clean.endsWith("/")) {
    clean = clean.slice(0, -1);
  }
  const base = path.resolve(root, `.${clean}`);
  const rootResolved = path.resolve(root);
  if (base !== rootResolved && !base.startsWith(rootResolved + path.sep)) {
    return null;
  }
  const candidates = clean === "/" ? [path.join(base, "index.html")] : [base, `${base}.html`, path.join(base, "index.html")];
  return candidates.find(isFile) ?? null;
}

export function createStaticServer(
  root: string,
  extra?: (req: http.IncomingMessage, res: http.ServerResponse) => boolean,
): http.Server {
  return http.createServer((req, res) => {
    if (extra?.(req, res)) {
      return;
    }
    const file = resolveFile(root, req.url ?? "/");
    if (file === null) {
      const notFound = path.join(root, "404.html");
      res.writeHead(404, { "content-type": MIME[".html"] });
      res.end(isFile(notFound) ? fs.readFileSync(notFound) : "404");
      return;
    }
    res.writeHead(200, {
      "content-type": MIME[path.extname(file)] ?? "application/octet-stream",
      "cache-control": "no-store",
    });
    fs.createReadStream(file).pipe(res);
  });
}
```

- [ ] **Step 4: 통과 확인**

Run: `pnpm exec tsx --test src/build/serve.test.ts`
Expected: PASS, 2개 테스트 통과

- [ ] **Step 5: dev.ts 작성**

`scripts/dev.ts`:

```ts
/****************************************
 *
 * 개발 서버 — next dev 대체
 *
 * 빌드는 매번 자식 프로세스로 돌린다. 같은 프로세스에서 다시 import 하면
 * Node 모듈 캐시 때문에 고친 코드가 반영되지 않는다.
 * 바뀐 파일에 따라 html / css / js 중 필요한 것만 다시 만든다.
 *
 ****************************************/

import fs from "node:fs";
import path from "node:path";
import http from "node:http";
import { spawn } from "node:child_process";
import { createStaticServer } from "../src/build/serve.ts";

const PORT = Number(process.env.PORT ?? 3000);
const OUT = "dist";
type Kind = "full" | "html" | "css" | "js";

const RELOAD_JS = `
const events = new EventSource("/__dev/events");
events.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type === "reload") location.reload();
  if (msg.type === "error") {
    let el = document.getElementById("__dev-error");
    if (!el) {
      el = document.createElement("pre");
      el.id = "__dev-error";
      el.style.cssText = "position:fixed;inset:0;z-index:9999;margin:0;padding:24px;overflow:auto;" +
        "background:rgba(30,0,0,.94);color:#fdd;font:13px/1.5 monospace;white-space:pre-wrap";
      document.body.append(el);
    }
    el.textContent = msg.message;
  }
};
`;

const clients = new Set<http.ServerResponse>();
let lastError: string | null = null;
let running = false;
let queued: Kind | null = null;

function send(res: http.ServerResponse, msg: object): void {
  res.write(`data: ${JSON.stringify(msg)}\n\n`);
}

function broadcast(msg: object): void {
  for (const res of clients) send(res, msg);
}

function merge(a: Kind | null, b: Kind): Kind {
  if (a === null || a === b) return b;
  return "full";
}

function runBuild(kind: Kind): void {
  if (running) {
    queued = merge(queued, kind);
    return;
  }
  running = true;
  const args = ["--import", "tsx", path.join("scripts", "build.ts"), "--dev", ...(kind === "full" ? [] : [`--only=${kind}`])];
  const child = spawn(process.execPath, args, { stdio: ["ignore", "inherit", "pipe"] });
  let stderr = "";
  child.stderr.on("data", (chunk) => {
    stderr += chunk;
    process.stderr.write(chunk);
  });
  child.on("exit", (code) => {
    running = false;
    lastError = code === 0 ? null : stderr || `빌드 실패 (exit ${code})`;
    broadcast(lastError === null ? { type: "reload" } : { type: "error", message: lastError });
    if (queued !== null) {
      const next = queued;
      queued = null;
      runBuild(next);
    }
  });
}

function kindOf(file: string): Kind {
  const f = file.split(path.sep).join("/");
  if (f.startsWith("src/client/")) return "js";
  if (f.startsWith("src/styles/") || f === "tailwind.config.js") return "css";
  if (f.startsWith("public/") || f.startsWith("src/static/")) return "full";
  return "html";
}

let pending: Kind | null = null;
let timer: NodeJS.Timeout | undefined;
function onChange(file: string): void {
  pending = merge(pending, kindOf(file));
  clearTimeout(timer);
  timer = setTimeout(() => {
    const kind = pending ?? "full";
    pending = null;
    console.log(`변경 감지 → ${kind} 빌드`);
    runBuild(kind);
  }, 100);
}

for (const dir of ["content", "src", "public"]) {
  fs.watch(dir, { recursive: true }, (_event, name) => {
    if (name !== null) onChange(path.join(dir, String(name)));
  });
}
fs.watch("tailwind.config.js", () => onChange("tailwind.config.js"));

const server = createStaticServer(OUT, (req, res) => {
  if (req.url === "/__dev/reload.js") {
    res.writeHead(200, { "content-type": "text/javascript; charset=utf-8" });
    res.end(RELOAD_JS);
    return true;
  }
  if (req.url === "/__dev/events") {
    res.writeHead(200, { "content-type": "text/event-stream", "cache-control": "no-store", connection: "keep-alive" });
    clients.add(res);
    if (lastError !== null) send(res, { type: "error", message: lastError });
    req.on("close", () => clients.delete(res));
    return true;
  }
  return false;
});

runBuild("full");
server.listen(PORT, () => console.log(`dev: http://localhost:${PORT}`));
```

`--only=html`은 Task 9의 `build()`에서 dist를 비우지 않고 HTML만 다시 쓴다. 단, 글이나 라우트가 **줄어들면** 지워진 페이지의 옛 HTML이 남는다. dev에서만 생기는 일이라 그대로 두고, 이상하면 서버를 다시 띄운다(전체 빌드).

- [ ] **Step 6: 수동 확인**

Run: `pnpm dev` (백그라운드)
Expected: 콘솔에 빌드 완료 후 `dev: http://localhost:3000`

확인:
1. 브라우저에서 `http://localhost:3000/`, `/en`, `/about`, `/post/all/1`이 열린다. 없는 주소는 404 페이지
2. `content/posts/ko/...` 아무 글의 본문 한 줄을 고치고 저장 → 브라우저가 새로고침되고 바뀐 내용이 보인다. 되돌린다
3. `src/components/Layout/Footer.tsx`의 `© 2026`을 `© 2026!`로 바꾸고 저장 → 새로고침. 되돌린다
4. `src/render.tsx`에 문법 오류(`<div>` 하나 지우기)를 넣고 저장 → 브라우저에 빨간 오류 화면, 서버는 살아 있다. 되돌리면 정상 화면으로 돌아온다
5. 이어서 Task 10 Step 9의 체크리스트를 진행한다

- [ ] **Step 7: 커밋**

```bash
git add src/build/serve.ts src/build/serve.test.ts scripts/dev.ts
git commit -m "✨ feat: 감시·정적 서버·자동 새로고침으로 dev 서버를 만든다"
```

---

### Task 12: 폰트 서브셋 이식

**Files:**
- Create (복사 후 수정): `scripts/fonts.ts`, `scripts/fonts.test.ts`

**Interfaces:**
- Produces: `usedCharacters(): string`, CLI `pnpm fonts` — `public/fonts/pretendard-subset.woff2`와 `.manifest.json`을 다시 만든다

- [ ] **Step 1: 복사**

```bash
cp ../blog/scripts/fonts.ts scripts/fonts.ts
cp ../blog/app/lib/fonts/subset.test.ts scripts/fonts.test.ts
```

- [ ] **Step 2: 경로 수정**

`scripts/fonts.ts`:
- `const CACHE = path.join(".next", "cache", "pretendard-variable.woff2");` → `const CACHE = path.join(".cache", "pretendard-variable.woff2");`
- `UI_SOURCES`를 바꾼다:

```ts
const UI_SOURCES = [
  path.join("src", "lib", "i18n", "messages.ts"),
  path.join("src", "lib", "i18n", "site-copy.ts"),
  path.join("src", "pages", "NotFoundPage.tsx"),
  path.join("src", "components", "Layout", "Footer.tsx"),
];
```

- 캐시를 쓰는 곳 앞에 `fs.mkdirSync(path.dirname(CACHE), { recursive: true });`가 있는지 확인하고 없으면 넣는다

`scripts/fonts.test.ts`:
- `import { usedCharacters } from "@/scripts/fonts";` → `import { usedCharacters } from "./fonts.ts";`

- [ ] **Step 3: 테스트**

Run: `pnpm exec tsx --test scripts/fonts.test.ts`
Expected: PASS, 3개 테스트 통과. `본문에 쓰인 글자가 서브셋에 전부 들어 있다`가 실패하면 바닐라판 UI 소스에 새 글자가 생긴 것이다 → `pnpm fonts`를 돌린다(원본 폰트를 내려받으므로 네트워크가 필요하다)

- [ ] **Step 4: 커밋**

```bash
git add scripts/fonts.ts scripts/fonts.test.ts
git commit -m "♻️ refactor: 폰트 서브셋 스크립트를 옮긴다"
```

---

### Task 13: 결과물 동일성 비교 (compare:output)

**Files:**
- Create: `scripts/compare/normalize.ts`, `scripts/compare/output.ts`
- Test: `scripts/compare/normalize.test.ts`
- 출력: `docs/comparison/output.md`

**Interfaces:**
- Produces:
  - `interface PageDigest { meta: Record<string, string[]>; jsonLd: string[]; main: string }`
  - `digest(html: string): PageDigest`
  - `htmlPages(dir: string): string[]` — 비교 대상 `.html` 상대 경로(정렬, `/` 구분자). `_not-found.html`, `_next/` 아래, `404.html`, `admin.html`은 뺀다
  - CLI: `pnpm compare:output [--next ../blog/out] [--vanilla dist]`

정규화 규칙(이 목록이 "Next 전용이라 지우는 것"의 전부다. 늘릴 때는 이유를 주석에 적는다):
1. `<script>`, HTML 주석(`<!-- -->`, `<!--$-->`)을 지운다 — React 스트리밍 표시와 실행 코드
2. `data-island`로 시작하는 속성을 지운다 — 바닐라판의 클라이언트 연결점
3. `class`는 공백으로 나눠 빈 조각을 버리고 한 칸으로 잇는다. 비면 속성을 지운다
4. `style`은 `;`로 나눠 앞뒤 공백을 지우고 다시 잇는다
5. 요소마다 속성을 이름순으로 정렬한다

- [ ] **Step 1: 실패하는 테스트 작성**

`scripts/compare/normalize.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { digest } from "./normalize.ts";

const page = (head: string, main: string) => `<!DOCTYPE html><html><head>${head}</head><body><main>${main}</main></body></html>`;

test("Next 전용 흔적과 data-island 차이는 같다고 본다", () => {
  const next = page(
    "",
    `<p class="a  b ">x<!-- -->y</p><script>self.__next_f.push(1)</script><time dateTime="d">d</time><li style="opacity:1; z-index:5"></li>`,
  );
  const vanilla = page("", `<p class="a b">xy</p><time data-island="date" datetime="d">d</time><li style="opacity:1;z-index:5"></li>`);
  assert.equal(digest(next).main, digest(vanilla).main);
});

test("속성 순서는 무시하고 값 차이는 잡는다", () => {
  assert.equal(digest(page("", `<a href="/x" class="c">t</a>`)).main, digest(page("", `<a class="c" href="/x">t</a>`)).main);
  assert.notEqual(digest(page("", `<a href="/x">t</a>`)).main, digest(page("", `<a href="/y">t</a>`)).main);
});

test("메타 태그와 JSON-LD를 모은다", () => {
  const d = digest(
    page(
      `<title>T</title><meta name="description" content="D"/><meta property="og:title" content="T"/>` +
        `<link rel="canonical" href="https://iaman.kr/"/><link rel="alternate" hrefLang="en" href="https://iaman.kr/en"/>`,
      `<script type="application/ld+json">{"b":1,"a":2}</script>`,
    ),
  );
  assert.deepEqual(d.meta, {
    "alternate:en": ["https://iaman.kr/en"],
    canonical: ["https://iaman.kr/"],
    description: ["D"],
    "og:title": ["T"],
    title: ["T"],
  });
  assert.deepEqual(d.jsonLd, [`{"b":1,"a":2}`]);
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm exec tsx --test scripts/compare/normalize.test.ts`
Expected: FAIL — `Cannot find module './normalize.ts'`

- [ ] **Step 3: normalize.ts 작성**

`scripts/compare/normalize.ts`:

```ts
/**
 * Next판과 바닐라판 HTML을 같은 잣대로 펴서 비교할 수 있게 한다.
 * 규칙은 계획 문서 Task 13의 목록과 같다. 늘릴 때는 이유를 적는다.
 */
import * as cheerio from "cheerio";
import type { Element } from "domhandler";
import fs from "node:fs";
import path from "node:path";

export interface PageDigest {
  meta: Record<string, string[]>;
  jsonLd: string[];
  main: string;
}

function push(meta: Record<string, string[]>, key: string, value: string | undefined): void {
  if (value === undefined) return;
  (meta[key] ??= []).push(value);
}

export function digest(html: string): PageDigest {
  const $ = cheerio.load(html);
  const meta: Record<string, string[]> = {};

  push(meta, "title", $("head title").first().text());
  $("head meta[name], head meta[property]").each((_, el) => {
    const key = $(el).attr("name") ?? $(el).attr("property");
    if (key !== undefined && key !== "viewport" && key !== "next-size-adjust") push(meta, key, $(el).attr("content"));
  });
  push(meta, "canonical", $('head link[rel="canonical"]').attr("href"));
  $('head link[rel="alternate"][hreflang]').each((_, el) => {
    push(meta, `alternate:${$(el).attr("hreflang")}`, $(el).attr("href"));
  });
  for (const values of Object.values(meta)) values.sort();
  const sortedMeta = Object.fromEntries(Object.entries(meta).sort(([a], [b]) => a.localeCompare(b)));

  const jsonLd = $('script[type="application/ld+json"]')
    .map((_, el) => JSON.stringify(JSON.parse($(el).text())))
    .get();

  const main = $("main").first();
  main.find("script").remove();
  main.find("*").contents().filter((_, node) => node.type === "comment").remove();
  main.contents().filter((_, node) => node.type === "comment").remove();

  main.find("*").each((_, node) => {
    const el = node as Element;
    const attribs = Object.entries(el.attribs)
      .filter(([name]) => !name.startsWith("data-island"))
      .map(([name, value]): [string, string] => {
        if (name === "class") return [name, value.split(/\s+/).filter(Boolean).join(" ")];
        if (name === "style") return [name, value.split(";").map((s) => s.trim()).filter(Boolean).join(";")];
        return [name, value];
      })
      .filter(([name, value]) => !(name === "class" && value === ""))
      .sort(([a], [b]) => a.localeCompare(b));
    el.attribs = Object.fromEntries(attribs);
  });

  return { meta: sortedMeta, jsonLd, main: $.html(main) ?? "" };
}

const EXCLUDED = /^(_next\/|_not-found\.html$|404\.html$|admin\.html$)/;

export function htmlPages(dir: string): string[] {
  return fs
    .readdirSync(dir, { recursive: true })
    .map((f) => String(f).split(path.sep).join("/"))
    .filter((f) => f.endsWith(".html") && !EXCLUDED.test(f))
    .sort();
}
```

`domhandler`는 cheerio의 의존성이라 따로 설치하지 않는다. 타입을 못 찾으면 `type Element = { attribs: Record<string, string> }`로 바꾼다.

- [ ] **Step 4: 통과 확인**

Run: `pnpm exec tsx --test scripts/compare/normalize.test.ts`
Expected: PASS, 3개 테스트 통과

- [ ] **Step 5: output.ts 작성**

`scripts/compare/output.ts`:

```ts
/**
 * Next판(../blog/out)과 바닐라판(dist)의 결과물을 비교해 docs/comparison/output.md 로 남긴다.
 * 차이가 있어도 실패로 끝내지 않는다 — 차이마다 이유를 적는 것이 목적이다.
 */
import fs from "node:fs";
import path from "node:path";
import { execFileSync } from "node:child_process";
import { digest, htmlPages } from "./normalize.ts";

function arg(name: string, fallback: string): string {
  const i = process.argv.indexOf(`--${name}`);
  return i === -1 ? fallback : process.argv[i + 1];
}

const nextDir = path.resolve(arg("next", "../blog/out"));
const vanillaDir = path.resolve(arg("vanilla", "dist"));
const REPORT = path.join("docs", "comparison", "output.md");

for (const dir of [nextDir, vanillaDir]) {
  if (!fs.existsSync(dir)) throw new Error(`${dir} 가 없다. 먼저 빌드한다.`);
}

const source = fs.readFileSync(path.join("content", "SOURCE"), "utf8").trim();
const blogHead = execFileSync("git", ["-C", path.dirname(nextDir), "rev-parse", "--short", "HEAD"], { encoding: "utf8" }).trim();
const sameSource = source === blogHead;

/** 처음 달라지는 곳 앞뒤를 잘라 보여 준다. */
function firstDiff(a: string, b: string, width = 160): { next: string; vanilla: string } {
  let i = 0;
  while (i < a.length && i < b.length && a[i] === b[i]) i++;
  const from = Math.max(0, i - 40);
  return { next: a.slice(from, from + width), vanilla: b.slice(from, from + width) };
}

const nextPages = new Set(htmlPages(nextDir));
const vanillaPages = new Set(htmlPages(vanillaDir));
const onlyNext = [...nextPages].filter((p) => !vanillaPages.has(p));
const onlyVanilla = [...vanillaPages].filter((p) => !nextPages.has(p));
const common = [...nextPages].filter((p) => vanillaPages.has(p));

const metaDiffs: string[] = [];
const jsonLdDiffs: string[] = [];
const mainDiffs: { page: string; next: string; vanilla: string }[] = [];

for (const page of common) {
  const a = digest(fs.readFileSync(path.join(nextDir, page), "utf8"));
  const b = digest(fs.readFileSync(path.join(vanillaDir, page), "utf8"));
  if (JSON.stringify(a.meta) !== JSON.stringify(b.meta)) {
    const keys = new Set([...Object.keys(a.meta), ...Object.keys(b.meta)]);
    for (const key of keys) {
      if (JSON.stringify(a.meta[key]) !== JSON.stringify(b.meta[key])) {
        metaDiffs.push(`| ${page} | ${key} | ${JSON.stringify(a.meta[key] ?? null)} | ${JSON.stringify(b.meta[key] ?? null)} |`);
      }
    }
  }
  if (JSON.stringify(a.jsonLd) !== JSON.stringify(b.jsonLd)) jsonLdDiffs.push(page);
  if (a.main !== b.main) mainDiffs.push({ page, ...firstDiff(a.main, b.main) });
}

const sameFile = (file: string) =>
  fs.existsSync(path.join(nextDir, file)) &&
  fs.readFileSync(path.join(nextDir, file), "utf8").trim() === fs.readFileSync(path.join(vanillaDir, file), "utf8").trim();

const escapeCell = (s: string) => s.replace(/\|/g, "\\|").replace(/\n/g, " ");
const lines = [
  "# 결과물 비교",
  "",
  `- 실행: ${new Date().toISOString()}`,
  `- 글 원본: content/SOURCE=${source}, blog HEAD=${blogHead}${sameSource ? "" : " ⚠ 다르다 — pnpm sync 후 다시 돌린다"}`,
  "",
  "| 항목 | 결과 |",
  "|---|---|",
  `| URL (Next / 바닐라 / 공통) | ${nextPages.size} / ${vanillaPages.size} / ${common.length} |`,
  `| Next에만 있음 | ${onlyNext.length} |`,
  `| 바닐라에만 있음 | ${onlyVanilla.length} |`,
  `| 메타 태그 차이 (항목 수) | ${metaDiffs.length} |`,
  `| JSON-LD 차이 (페이지) | ${jsonLdDiffs.length} |`,
  `| 본문(main) 차이 (페이지) | ${mainDiffs.length} |`,
  `| sitemap.xml 동일 | ${sameFile("sitemap.xml") ? "예" : "아니오"} |`,
  `| robots.txt 동일 | ${sameFile("robots.txt") ? "예" : "아니오"} |`,
  "",
  ...(onlyNext.length ? ["## Next에만 있는 URL", "", ...onlyNext.map((p) => `- ${p}`), ""] : []),
  ...(onlyVanilla.length ? ["## 바닐라에만 있는 URL", "", ...onlyVanilla.map((p) => `- ${p}`), ""] : []),
  ...(metaDiffs.length ? ["## 메타 태그 차이", "", "| 페이지 | 키 | Next | 바닐라 |", "|---|---|---|---|", ...metaDiffs, ""] : []),
  ...(jsonLdDiffs.length ? ["## JSON-LD 차이", "", ...jsonLdDiffs.map((p) => `- ${p}`), ""] : []),
  ...(mainDiffs.length
    ? [
        "## 본문 차이 (처음 달라지는 곳)",
        "",
        "| 페이지 | Next | 바닐라 |",
        "|---|---|---|",
        ...mainDiffs.map((d) => `| ${d.page} | \`${escapeCell(d.next)}\` | \`${escapeCell(d.vanilla)}\` |`),
        "",
      ]
    : []),
];

fs.mkdirSync(path.dirname(REPORT), { recursive: true });
fs.writeFileSync(REPORT, `${lines.join("\n")}\n`);
console.log(lines.slice(0, 18).join("\n"));
console.log(`\n→ ${REPORT}`);
```

- [ ] **Step 6: 실제 비교 실행**

먼저 같은 글로 맞춘다:

Run: `pnpm sync` → `cd ../blog && pnpm build && cd ../blog-vanilla` → `pnpm build` → `pnpm compare:output`
Expected: 표가 출력되고 `docs/comparison/output.md`가 생긴다. `글 원본` 줄에 경고가 없다.

- [ ] **Step 7: 차이 해소 또는 기록**

설계의 통과 기준:
- URL, 메타 태그, sitemap, robots: **완전히 일치**. 차이가 있으면 바닐라판을 고친다(마크업을 Next에 맞춘다)
- 본문: 차이가 0이거나, 남은 차이마다 이유를 `docs/comparison.md`의 "알려진 차이" 표에 적는다

이미 예상되는 본문 차이(맞추지 않고 기록한다):
- `<footer>`는 main 밖이라 비교에 안 잡힌다. 다만 Next는 서버 HTML에 footer가 없고 바닐라는 있다
- 헤더의 테마 버튼, 검색·메뉴 모달도 main 밖이다

본문 차이를 고친 뒤에는 Step 6을 다시 돌린다.

- [ ] **Step 8: 커밋**

```bash
git add scripts/compare/normalize.ts scripts/compare/normalize.test.ts scripts/compare/output.ts docs/comparison/output.md src
git commit -m "✅ test: Next판과 결과물을 비교하는 스크립트를 만든다"
```

---

### Task 14: 숫자·코드 비교와 정리 문서

**Files:**
- Create: `scripts/compare/metrics.ts`, `scripts/compare/code.ts`, `docs/comparison.md`, `docs/worklog/2026-11-20-blog-vanilla.md`
- 출력: `docs/comparison/metrics.md`, `docs/comparison/code.md`

**Interfaces:**
- Consumes: `createStaticServer` (Task 11), `htmlPages` (Task 13), `allPosts` `postsIn`
- Produces: CLI `pnpm compare:metrics [--runs 5] [--skip-build-time] [--lighthouse]`, `pnpm compare:code`

- [ ] **Step 1: metrics.ts 작성**

`scripts/compare/metrics.ts`:

```ts
/**
 * 숫자 비교: 빌드 시간, 의존성, 출력 크기, 페이지당 JS, (선택) Lighthouse.
 * 결과는 docs/comparison/metrics.md.
 */
import fs from "node:fs";
import path from "node:path";
import os from "node:os";
import zlib from "node:zlib";
import { execSync } from "node:child_process";
import * as cheerio from "cheerio";
import { createStaticServer } from "../../src/build/serve.ts";
import { allPosts } from "../../src/lib/posts/repository.ts";
import { postsIn } from "../../src/lib/posts/queries.ts";

const BLOG = path.resolve("../blog");
const VANILLA = path.resolve(".");
const RUNS = Number(process.argv[process.argv.indexOf("--runs") + 1]) || 5;
const SKIP_BUILD = process.argv.includes("--skip-build-time");
const LIGHTHOUSE = process.argv.includes("--lighthouse");
const REPORT = path.join("docs", "comparison", "metrics.md");

const projects = [
  { name: "Next", root: BLOG, out: path.join(BLOG, "out"), clean: [".next", "out"] },
  { name: "바닐라", root: VANILLA, out: path.join(VANILLA, "dist"), clean: ["dist"] },
];

const kb = (bytes: number) => `${(bytes / 1024).toFixed(1)} KB`;
const mb = (bytes: number) => `${(bytes / 1024 / 1024).toFixed(1)} MB`;

function median(values: number[]): number {
  const s = [...values].sort((a, b) => a - b);
  return s[Math.floor(s.length / 2)];
}

function buildTime(root: string, clean: string[]): number {
  const times: number[] = [];
  for (let i = 0; i < RUNS; i++) {
    for (const dir of clean) fs.rmSync(path.join(root, dir), { recursive: true, force: true });
    const start = performance.now();
    execSync("pnpm build", { cwd: root, stdio: "ignore" });
    times.push(performance.now() - start);
    console.log(`  ${path.basename(root)} ${i + 1}/${RUNS}: ${Math.round(times[i])}ms`);
  }
  return median(times);
}

/** 심볼릭 링크를 따라가지 않고 실제 파일 크기만 더한다(pnpm 저장소 중복 방지). */
function dirSize(dir: string): number {
  let total = 0;
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name);
    if (entry.isSymbolicLink()) continue;
    total += entry.isDirectory() ? dirSize(full) : fs.statSync(full).size;
  }
  return total;
}

function deps(root: string): { packages: number; size: number } {
  const store = path.join(root, "node_modules", ".pnpm");
  const packages = fs.readdirSync(store).filter((n) => n !== "node_modules" && n !== "lock.yaml").length;
  return { packages, size: dirSize(store) };
}

function outputSizes(out: string): Record<string, number> {
  const sizes: Record<string, number> = { html: 0, js: 0, css: 0, "rsc(.txt)": 0, 기타: 0, 합계: 0 };
  for (const file of fs.readdirSync(out, { recursive: true }).map(String)) {
    const full = path.join(out, file);
    if (!fs.statSync(full).isFile()) continue;
    const size = fs.statSync(full).size;
    const ext = path.extname(file);
    const key = ext === ".html" ? "html" : ext === ".js" ? "js" : ext === ".css" ? "css" : ext === ".txt" && !file.endsWith("robots.txt") ? "rsc(.txt)" : "기타";
    sizes[key] += size;
    sizes.합계 += size;
  }
  return sizes;
}

/** 페이지가 처음에 받는 JS: <script src> + 거기서 정적 import 로 끌려오는 모듈. */
function pageJs(out: string, page: string): { raw: number; gzip: number; files: number } {
  const $ = cheerio.load(fs.readFileSync(path.join(out, page), "utf8"));
  const queue = $("script[src]").map((_, el) => $(el).attr("src")).get().filter((src) => src.startsWith("/"));
  const seen = new Set<string>();
  let raw = 0;
  let gzip = 0;
  while (queue.length > 0) {
    const url = queue.shift()!.split("?")[0];
    if (seen.has(url)) continue;
    seen.add(url);
    const file = path.join(out, url);
    if (!fs.existsSync(file)) continue;
    const code = fs.readFileSync(file);
    raw += code.length;
    gzip += zlib.gzipSync(code).length;
    for (const m of code.toString().matchAll(/(?:import|from)\s*["'](\.{1,2}\/[^"']+)["']/g)) {
      queue.push(path.posix.join(path.posix.dirname(url), m[1]));
    }
  }
  return { raw, gzip, files: seen.size };
}

async function lighthouse(out: string, port: number, pages: string[]): Promise<Record<string, string>> {
  const server = createStaticServer(out);
  await new Promise<void>((resolve) => server.listen(port, resolve));
  const scores: Record<string, string> = {};
  try {
    for (const page of pages) {
      const url = `http://localhost:${port}${page}`;
      const report = path.join(os.tmpdir(), `lh-${port}-${page.replace(/\W/g, "_")}.json`);
      try {
        execSync(
          `npx --yes lighthouse "${url}" --quiet --output=json --output-path="${report}" ` +
            `--only-categories=performance,accessibility,best-practices,seo --chrome-flags="--headless=new"`,
          { stdio: "ignore", timeout: 180_000 },
        );
        const cats = JSON.parse(fs.readFileSync(report, "utf8")).categories;
        scores[page] = ["performance", "accessibility", "best-practices", "seo"]
          .map((c) => Math.round(cats[c].score * 100))
          .join(" / ");
      } catch (error) {
        scores[page] = `실행 불가: ${(error as Error).message.split("\n")[0]}`;
      }
    }
  } finally {
    server.close();
  }
  return scores;
}

const latest = postsIn(allPosts(), "ko")[0].slug;
const samplePages = [
  { label: "홈", file: "index.html", url: "/" },
  { label: "글", file: `${latest}.html`, url: `/${latest}` },
  { label: "목록", file: "post/all/1.html", url: "/post/all/1" },
];

const lines: string[] = ["# 숫자 비교", "", `- 실행: ${new Date().toISOString()}, ${os.cpus()[0].model}, Node ${process.version}`, ""];

if (!SKIP_BUILD) {
  console.log(`빌드 시간 측정 (${RUNS}회씩)…`);
  const times = projects.map((p) => buildTime(p.root, p.clean));
  lines.push("## 빌드 시간 (콜드, 중앙값)", "", "| | Next | 바닐라 |", "|---|---|---|");
  lines.push(`| ${RUNS}회 중앙값 | ${(times[0] / 1000).toFixed(1)}s | ${(times[1] / 1000).toFixed(1)}s |`, "");
}

const d = projects.map((p) => deps(p.root));
lines.push("## 의존성", "", "| | Next | 바닐라 |", "|---|---|---|");
lines.push(`| 설치된 패키지 수 | ${d[0].packages} | ${d[1].packages} |`);
lines.push(`| node_modules/.pnpm 크기 | ${mb(d[0].size)} | ${mb(d[1].size)} |`, "");

const sizes = projects.map((p) => outputSizes(p.out));
lines.push("## 출력 크기", "", "| 종류 | Next | 바닐라 |", "|---|---|---|");
for (const key of Object.keys(sizes[0])) lines.push(`| ${key} | ${kb(sizes[0][key])} | ${kb(sizes[1][key])} |`);
lines.push("");

lines.push("## 페이지당 JS (처음 받는 것)", "", "| 페이지 | Next (원본 / gzip / 파일) | 바닐라 (원본 / gzip / 파일) |", "|---|---|---|");
for (const page of samplePages) {
  const [a, b] = projects.map((p) => pageJs(p.out, page.file));
  lines.push(`| ${page.label} | ${kb(a.raw)} / ${kb(a.gzip)} / ${a.files} | ${kb(b.raw)} / ${kb(b.gzip)} / ${b.files} |`);
}
lines.push("");

if (LIGHTHOUSE) {
  console.log("Lighthouse…");
  const [a, b] = [await lighthouse(projects[0].out, 4101, samplePages.map((p) => p.url)), await lighthouse(projects[1].out, 4102, samplePages.map((p) => p.url))];
  lines.push("## Lighthouse (성능 / 접근성 / 권장사항 / SEO)", "", "| 페이지 | Next | 바닐라 |", "|---|---|---|");
  for (const page of samplePages) lines.push(`| ${page.label} | ${a[page.url]} | ${b[page.url]} |`);
  lines.push("");
}

fs.mkdirSync(path.dirname(REPORT), { recursive: true });
fs.writeFileSync(REPORT, `${lines.join("\n")}\n`);
console.log(lines.join("\n"));
```

- [ ] **Step 2: 측정 실행**

Run: `pnpm compare:metrics --lighthouse`
Expected: 빌드를 Next 5회, 바닐라 5회 돌린 뒤 표를 출력하고 `docs/comparison/metrics.md`를 쓴다. Lighthouse 칸에 `실행 불가`가 나오면 Chrome·Edge가 없거나 npx가 막힌 것이다 — 그대로 기록한다.

이 명령은 blog의 `.next`와 `out`을 지우고 다시 만든다. 끝나면 `out/`은 최신 빌드로 남는다.

- [ ] **Step 3: code.ts 작성**

`scripts/compare/code.ts`:

```ts
/** 코드 줄 수·파일 수 비교. 빈 줄은 세지 않는다. 테스트와 범위 밖 기능은 뺀다. */
import fs from "node:fs";
import path from "node:path";

const REPORT = path.join("docs", "comparison", "code.md");
const SOURCE_EXT = /\.(ts|tsx|js|mjs|css)$/;
const TEST = /\.test\.tsx?$/;

interface Group {
  label: string;
  next: string[];
  vanilla: string[];
}

/** 경로 접두사 기준. 범위 밖(admin, analytics, thumbnail, new-post, icons)은 Next 쪽에서 뺀다. */
const GROUPS: Group[] = [
  { label: "글 파이프라인·공용 로직 (lib)", next: ["app/lib/"], vanilla: ["src/lib/"] },
  { label: "컴포넌트", next: ["components/"], vanilla: ["src/components/"] },
  {
    label: "라우트·페이지·메타데이터",
    next: ["app/(ko)/", "app/(en)/", "app/not-found.tsx", "app/sitemap.ts", "app/robots.ts", "app/utils/"],
    vanilla: ["src/pages/", "src/routes.ts", "src/render.tsx", "src/head.tsx", "src/build/sitemap.ts"],
  },
  { label: "클라이언트 동작", next: [], vanilla: ["src/client/"] },
  {
    label: "프레임워크 대체 (직접 만든 것)",
    next: [],
    vanilla: ["src/jsx/", "src/build/build.ts", "src/build/links.ts", "src/build/serve.ts", "scripts/build.ts", "scripts/dev.ts"],
  },
  { label: "스타일·설정", next: ["app/globals.css", "tailwind.config.js", "next.config.ts", "postcss.config.mjs"], vanilla: ["src/styles/", "tailwind.config.js", "tsconfig.client.json"] },
  { label: "스크립트 (폰트·동기화)", next: ["scripts/fonts.ts"], vanilla: ["scripts/fonts.ts", "scripts/sync-content.ts", "scripts/rewrite-imports.ts"] },
];

const NEXT_EXCLUDE = [/^app\/lib\/analytics\//, /^components\/Admin\//, /^app\/\(ko\)\/admin\//, /^app\/lib\/posts\/__fixtures__\//];

function files(root: string, prefixes: string[], exclude: RegExp[] = []): string[] {
  const all = fs
    .readdirSync(root, { recursive: true })
    .map((f) => String(f).split(path.sep).join("/"))
    .filter((f) => !f.startsWith("node_modules/") && SOURCE_EXT.test(f) && !TEST.test(f));
  return all.filter((f) => prefixes.some((p) => f === p || f.startsWith(p)) && !exclude.some((r) => r.test(f)));
}

function lines(root: string, list: string[]): number {
  return list.reduce(
    (sum, f) => sum + fs.readFileSync(path.join(root, f), "utf8").split("\n").filter((l) => l.trim() !== "").length,
    0,
  );
}

const BLOG = path.resolve("../blog");
const out = ["# 코드 비교", "", "빈 줄 제외, 테스트 제외. Next 쪽은 범위 밖 기능(admin, analytics, 썸네일)을 뺐다.", "", "| 영역 | Next 줄 / 파일 | 바닐라 줄 / 파일 |", "|---|---|---|"];
let totalNext = 0;
let totalVanilla = 0;
for (const g of GROUPS) {
  const a = files(BLOG, g.next, NEXT_EXCLUDE);
  const b = files(".", g.vanilla);
  const la = lines(BLOG, a);
  const lb = lines(".", b);
  totalNext += la;
  totalVanilla += lb;
  out.push(`| ${g.label} | ${la} / ${a.length} | ${lb} / ${b.length} |`);
}
out.push(`| **합계** | **${totalNext}** | **${totalVanilla}** |`, "");

fs.mkdirSync(path.dirname(REPORT), { recursive: true });
fs.writeFileSync(REPORT, `${out.join("\n")}\n`);
console.log(out.join("\n"));
```

Run: `pnpm compare:code`
Expected: 영역별 표 출력, `docs/comparison/code.md` 생성

- [ ] **Step 4: docs/comparison.md 작성**

세 보고서(`docs/comparison/output.md`, `metrics.md`, `code.md`)의 **실제 숫자**를 옮겨 아래 구조로 쓴다. 숫자를 추측해 채우지 않는다.

```markdown
# Next.js판 vs 바닐라판 비교

- 글 원본: blog @ <content/SOURCE 값>, 측정일 <날짜>
- 상세: [결과물](comparison/output.md) · [숫자](comparison/metrics.md) · [코드](comparison/code.md)

## 한눈에

| 항목 | Next | 바닐라 |
|---|---|---|
| 빌드 시간 | | |
| 설치 패키지 수 | | |
| 출력 전체 | | |
| 홈 JS (gzip) | | |
| 글 JS (gzip) | | |
| 코드 줄 수 | | |

## 결과물 동일성

(URL·메타·sitemap·robots 일치 여부, 본문 차이 수)

## 알려진 차이

| 차이 | 이유 | 맞출지 |
|---|---|---|
| footer가 서버 HTML에 있다 | Next판은 mounted 이후에만 그렸다. 바닐라는 막을 이유가 없어 바로 그린다 | 아니오 |
| 테마 버튼이 서버 HTML에 있다 | 두 아이콘을 다 그리고 CSS로 하나만 보인다 | 아니오 |
| 언어 전환 링크가 서버 HTML에 있다 | 빌드 때 경로를 알아서 계산한다. Next판은 usePathname | 아니오 |
| 페이지 이동이 전체 새로고침 | 클라이언트 라우팅(Link 프리페치)을 구현하지 않았다 | 아니오 |
| `<html>` class 없음 | next/font의 변수 클래스 대신 :root에 CSS 변수 | 아니오 |
| (compare:output이 찾은 나머지) | | |

## 기능 확인

(Task 10 Step 9 체크리스트 결과)

## 기능별 난이도와 소감

| 기능 | 쉬웠나 | 메모 |
|---|---|---|
| JSX 런타임 | | |
| 라우트·메타데이터 | | |
| 캐러셀 | | |
| 검색 | | |
| dev 서버 | | |

## 결론
```

빈 칸은 측정·작업 결과로 채운다. 채울 근거가 없는 칸은 "측정 안 함"이라고 적는다.

- [ ] **Step 5: 작업 기록**

`docs/worklog/2026-11-20-blog-vanilla.md`에 남긴다: 무엇을 만들었는지, 계획과 달라진 점, 남은 일(예: GitHub 원격 연결, Cloudflare Pages 배포 여부), 비교 결과 요약 세 줄.

- [ ] **Step 6: 전체 확인**

Run: `pnpm test` → `pnpm typecheck` → `pnpm build`
Expected: 테스트 전부 PASS, 타입 오류 없음, 빌드 성공(깨진 링크 수는 비교 문서와 일치)

- [ ] **Step 7: 커밋**

```bash
git add scripts/compare docs
git commit -m "📝 docs: Next판과 숫자·코드를 비교한 결과를 남긴다"
```
