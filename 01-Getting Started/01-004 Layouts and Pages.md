# 레이아웃 및 페이지
<span style="color: #808080;">Last updated June 23, 2026</span>

Next.js는 파일 시스템 기반 라우팅을 사용합니다. 즉, 폴더와 파일을 사용하여 경로를 정의할 수 있습니다. 이 페이지에서는 레이아웃과 페이지를 생성하고 이들 사이를 연결하는 방법을 안내합니다.

---
## 페이지 생성
페이지는 특정 경로에서 렌더링되는 UI입니다. 페이지를 만들려면 `app` 디렉터리 내에 [page](https://nextjs.org/docs/app/api-reference/file-conventions/page) 파일을 추가하고 기본적으로 React 구성 요소를 내보냅니다. 예를 들어, 인덱스 페이지(`/`)를 생성하려면:

![](./images/004/page-special-file.avif)

```typescript
export default function Page() {
  return <h1>Hello Next.js!</h1>
}
```

---
## 레이아웃 생성
A layout is UI that is shared between multiple pages. On navigation, layouts preserve state, remain interactive, and do not rerender.

You can define a layout by default exporting a React component from a layout file. The component should accept a children prop which can be a page or another layout.

For example, to create a layout that accepts your index page as child, add a layout file inside the app directory:

![](./images/004/layout-special-file.avif)

```typescript
export default function DashboardLayout({
  children
}:{
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        {/* Layout UI */}
        {/* 페이지 또는 중첩 레이아웃을 렌더링하려는 위치에 하위 항목 배치 */}
      </body>
    </html>
  )
}
```

위의 레이아웃은 `app` 디렉터리의 루트에 정의되어 있으므로 [루트 레이아웃](https://nextjs.org/docs/app/api-reference/file-conventions/layout#root-layout)이라고 합니다. 루트 레이아웃은 필수이며 `html` 및 `body` 태그를 포함해야 합니다.

---
## 중첩 경로 만들기
중첩 경로는 여러 URL 세그먼트로 구성된 경로입니다. 예를 들어 `/blog/[slug]` 경로는 세 개의 세그먼트로 구성됩니다:

- `/` (Root Segment)
- `blog` (Segment)
- `[slug]` (Leaf Segment)

Next.js에서:

- 폴더는 URL 세그먼트에 매핑되는 경로 세그먼트를 정의하는 데 사용됩니다.
- `페이지` 및 `레이아웃`과 같은 파일은 세그먼트에 표시되는 UI를 만드는 데 사용됩니다.

중첩된 경로를 만들려면 폴더를 서로 중첩하면 됩니다. 예를 들어 `/blog`에 대한 경로를 추가하려면 `app` 디렉터리에 `blog`라는 폴더를 만듭니다. 그런 다음 `/blog`를 공개적으로 액세스할 수 있도록 하려면 `page.tsx` 파일을 추가하세요:

![](./images/004/blog-nested-route.avif)

```typescript
import { getPosts } from '@lib/posts';
import { Post } from '@/ui/post';

export default async function Page() {
  const posts = await getPosts();

  return (
    <ul>
      {posts.map((post) => (
        <Post key={post.id} post={post} />
      ))}
    </ul>
  )
}
```

계속해서 폴더를 중첩하여 중첩 경로를 만들 수 있습니다. 예를 들어, 특정 블로그 게시물에 대한 경로를 생성하려면 블로그 내에 새 `[slug]` 폴더를 생성하고 페이지 파일을 추가하세요.:

![](./images/004/blog-post-nested-route.avif)

```typescript
function generateStaticParams() {}

export default function Page() {
  return <h1>Hello, Blog Post Page!</h1>
}
```

폴더 이름을 대괄호(예: [slug])로 묶으면 데이터에서 여러 페이지를 생성하는 데 사용되는 [동적 경로 세그먼트](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes)가 생성됩니다. 예를 들어 블로그 게시물, 제품 페이지 등

---

## 중첩 레이아웃
기본적으로 폴더 계층 구조의 레이아웃도 중첩되어 있습니다. 즉, `하위 속성`을 통해 하위 레이아웃을 래핑합니다. 특정 경로 세그먼트(폴더) 내에 `레이아웃`을 추가하여 레이아웃을 중첩할 수 있습니다..

예를 들어 `/blog` 경로에 대한 레이아웃을 생성하려면 `블로그` 폴더 내에 새 `레이아웃` 파일을 추가합니다.

![](./images/004/nested-layouts.avif)

```typescript
export default function BlogLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return <section>{children}</section>
}
```

위의 두 레이아웃을 결합하는 경우 루트 레이아웃(`app/layout.js`)은 블로그 레이아웃(`app/blog/layout.js`)을 래핑하고, 이는 블로그(`app/blog/page.js`)와 블로그 게시물 페이지(`app/blog/[slug]/page.js`)를 래핑합니다.

---
## 동적 세그먼트 생성
[동적 세그먼트](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes)를 사용하면 데이터에서 생성된 경로를 만들 수 있습니다. 예를 들어, 각 개별 블로그 게시물에 대한 경로를 수동으로 생성하는 대신 동적 세그먼트를 생성하여 블로그 게시물 데이터를 기반으로 경로를 생성할 수 있습니다.

동적 세그먼트를 생성하려면 세그먼트(폴더) 이름을 대괄호로 묶습니다: [segmentName]. 예를 들어 app/blog/[slug]/page.tsx 경로에서 [slug]는 동적 세그먼트입니다.

```typescript
export default async function BlogPostPage({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params;
  const post = await getPost(slug);

  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </div>
  )
}
```

[동적 세그먼트](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes) 및 [params](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional) 소품에 대해 자세히 알아보세요.

[동적 세그먼트 내의 중첩 레이아웃](https://nextjs.org/docs/app/api-reference/file-conventions/layout#params-optional)도 `params` 소품에 액세스할 수 있습니다.

---
## 검색 매개변수를 사용한 렌더링
Server Component 페이지에서 [searchParams](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional) 소품을 사용하여 검색 매개변수에 액세스할 수 있습니다:

```typescript
export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}) {
  const filters = (await searchParams).filters
}
```

`searchParams`를 사용하면 검색 매개변수를 읽으려면 들어오는 요청이 필요하기 때문에 페이지를 [동적 렌더링](https://nextjs.org/docs/app/glossary#dynamic-rendering)으로 선택합니다.

클라이언트 구성 요소는 [useSearchParams](https://nextjs.org/docs/app/api-reference/functions/use-search-params) 훅을 사용하여 검색 매개변수를 읽을 수 있습니다.

[사전 렌더링](https://nextjs.org/docs/app/api-reference/functions/use-search-params#prerendering)되고 [동적으로 렌더링](https://nextjs.org/docs/app/api-reference/functions/use-search-params#dynamic-rendering)된 경로의 `useSearchParams`에 대해 자세히 알아보세요.

### 무엇을 언제 사용해야 하는가
- 페이지에 대한 데이터를 로드하기 위해 검색 매개변수가 필요한 경우(예: 페이지 매김, 데이터베이스 필터링) `searchParams` props를 사용하세요.
- 검색 매개변수가 클라이언트에서만 사용되는 경우(예: props를 통해 이미 로드된 목록 필터링) `useSearchParams`를 사용하세요.
- 작은 최적화로 콜백이나 이벤트 핸들러에서 `new URLSearchParams(window.location.search)`를 사용하여 다시 렌더링을 트리거하지 않고도 검색 매개변수를 읽을 수 있습니다.

---
## 페이지 간 연결
[`<Link>` 컴포넌트](https://nextjs.org/docs/app/api-reference/components/link)를 사용하여 경로 간을 탐색할 수 있습니다. `<Link>`는 HTML `<a>` 태그를 확장하여 [프리패치](https://nextjs.org/docs/app/getting-started/linking-and-navigating#prefetching) 및 [클라이언트측 탐색 기능](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions)을 제공하는 내장 Next.js 컴포넌트입니다.

예를 들어 블로그 게시물 목록을 생성하려면 `next/link`에서 `<Link>`를 가져오고 `href` 속성을 컴포넌트에 전달합니다:

```typescript
import Link from 'next/link'
import { getPosts } from '@/lib/posts'

export default async function Posts() {
  const posts = await getPosts()

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.slug}>
          <Link href={`/blog/${post.slug}`}>{post.title}</Link>
        </li>
      ))}
    </ul>
  )
}
```

```
알아두면 좋은 점: <Link>는 Next.js에서 경로 간을 탐색하는 기본 방법입니다. 고급 탐색을 위해 [useRouter 훅](https://nextjs.org/docs/app/api-reference/functions/use-router)을 사용할 수도 있습니다.
```

---
## Route Props 헬퍼
Next.js exposes utility types that infer params and named slots from your route structure:

PageProps: Props for page components, including params and searchParams.
LayoutProps: Props for layout components, including children and any named slots (e.g. folders like @analytics).
These are globally available helpers, generated when running either next dev, next build or next typegen.

```typescript
export default async function Page(props: PageProps<`/blog/[slug]`>) {
  const { slug } = await props.params
  return <h1>Blog post: {slug}</h1>
}
```

```typescript
export default function Layout(props: LayoutProps<`/dashboard`>) {
  return (
    <section>
      {props.children}
      {/* app/dashboard/@analytics가 있는 경우 입력된 슬롯으로 나타납니다.: */}
      {/* {props.analytics} */}
    </section>
  )
}
```

```
알아두면 좋은 점

- 정적 경로는 매개변수를 {}로 확인합니다.
- PageProps, LayoutProps는 전역 도우미이므로 import가 필요하지 않습니다.
- 타입은 next dev, next build 또는 next typegen 중에 생성됩니다.
```

---
