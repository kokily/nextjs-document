# Caching
<span style="color: #808080;">Last updated May 13, 2026</span>

> 이 페이지에서는 `next.config.ts` 파일에서 [캐시 컴포넌트](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)를 true로 설정하여 활성화되는 캐시 컴포넌트를 사용한 캐싱을 다룹니다. 캐시 컴포넌트를 사용하지 않는 경우 [캐싱 및 유효성 재검사(이전 모델) 가이드](https://nextjs.org/docs/app/guides/caching-without-cache-components)를 참조하세요.

캐싱은 데이터 가져오기 및 기타 계산 결과를 저장하여 동일한 데이터에 대한 향후 요청이 작업을 다시 수행하지 않고도 더 빠르게 처리될 수 있도록 하는 기술입니다.

---
## 캐시 컴포넌트 활성화
다음 구성 파일에 캐시 컴포넌트 옵션을 추가하여 캐시 컴포넌트를 활성화할 수 있습니다:

next.config.js
```typescript
import type { NextConfig } from 'next'
 
const nextConfig: NextConfig = {
  cacheComponents: true,
}
 
export default nextConfig
```

> 알아두면 좋은 점: 캐시 컴포넌트가 활성화되면 'GET' 경로 핸들러는 페이지와 동일한 사전 렌더링 모델을 따릅니다. [캐시 컴포넌트가 있는 경로 핸들러](https://nextjs.org/docs/app/getting-started/route-handlers)를 참조하세요.

---
## 사용방법
`use cache` 지시문은 비동기 함수 및 컴포넌트의 반환 값을 캐시합니다. 두 가지 수준으로 적용할 수 있습니다:
- Data-level: 데이터를 가져오거나 계산하는 함수를 캐시합니다.(예 `getProducts()`, `getUser(id)`)
- UI-level: 전체 컴포넌트 또는 페이지 캐시 (예 `async function BlogPosts()`)

> 상위 범위의 인수 및 닫힌 값은 자동으로 캐시 키의 일부가 됩니다. 즉, 서로 다른 입력이 별도의 캐시 항목을 생성한다는 의미입니다. 이를 통해 개인화되거나 매개변수화된 캐시 콘텐츠가 가능해집니다. 캐시할 수 있는 항목과 인수 작동 방식에 대한 자세한 내용은 직렬화 요구 사항 및 제약 조건을 참조하세요.

### Data-level caching
데이터를 가져오는 비동기 함수를 캐시하려면 함수 본문 상단에 `use cache` 지시문을 추가하세요:

app/lib/data.ts
```typescript
import { cacheLife } from 'next/cache'
 
export async function getUsers() {
  'use cache'
  cacheLife('hours')
  return db.query('SELECT * FROM users')
}
```

데이터 수준 캐싱은 동일한 데이터가 여러 컴포넌트에서 사용되거나 UI와 독립적으로 데이터를 캐시하려는 경우에 유용합니다.

### UI-level caching
전체 컴포넌트, 페이지 또는 레이아웃을 캐시하려면 컴포넌트 또는 페이지 본문 상단에 `use cache` 지시문을 추가하세요:

app/page.tsx
```typescript
import { cacheLife } from 'next/cache'
 
export default async function Page() {
  'use cache'
  cacheLife('hours')
 
  const users = await db.query('SELECT * FROM users')
 
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}
```

> 파일 상단에 "use cache"을 추가하면 파일에 내보낸 모든 함수가 캐시됩니다.

### 캐시되지 않은 데이터 스트리밍
API, 데이터베이스 또는 기타 비동기 작업과 같은 비동기 소스에서 데이터를 가져오고 모든 요청에 ​​새로운 데이터가 필요한 구성 요소의 경우 `"use cache"`을 사용하지 마세요.

대신 `<Suspense>`로 컴포넌트를 래핑하고 대체 UI를 제공하세요. 요청 시 React는 먼저 fallback을 렌더링한 다음 비동기 작업이 완료되면 해결된 콘텐츠를 스트리밍합니다.

page.tsx
```typescript
import { Suspense } from 'react'
 
async function LatestPosts() {
  const data = await fetch('https://api.example.com/posts')
  const posts = await data.json()
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
 
export default function Page() {
  return (
    <>
      <h1>My Blog</h1>
      <Suspense fallback={<p>Loading posts...</p>}>
        <LatestPosts />
      </Suspense>
    </>
  )
}
```

fallback(`<p>게시물 로드 중...</p>`)은 정적 셸에 포함되며 컴포넌트의 콘텐츠는 요청 시 스트리밍됩니다.

`<Suspense>`는 비동기 작업이 완료되는 동안 대체 UI를 제공하지만 자체적으로 컴포넌트를 동적 렌더링으로 선택하지는 않습니다. 컴포넌트가 동기 작업만 수행하는 경우 `<Suspense>`에 래핑되었는지 여부에 관계없이 사전 렌더링 중에 완료됩니다.

---
## 런타임 API 작업
런타임 API에는 사용자가 요청할 때만 사용할 수 있는 정보가 필요합니다. 여기에는 다음이 포함됩니다:
- [cookies](https://nextjs.org/docs/app/api-reference/functions/cookies) - 사용자의 쿠키 데이터
- [headers](https://nextjs.org/docs/app/api-reference/functions/headers) - 요청 헤더
- [searchParams](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional) - URL 쿼리 매개변수
- [params](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional) - 동적 경로 매개변수 (최소한 하나의 샘플이 다음을 통해 제공되지 않는 한 [generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params))

런타임 API에 액세스하는 컴포넌트는 `<Suspense>`로 래핑되어야 합니다:

page.tsx
```typescript
import { cookies } from 'next/headers'
import { Suspense } from 'react'
 
async function UserGreeting() {
  const cookieStore = await cookies()
  const theme = cookieStore.get('theme')?.value || 'light'
  return <p>Your theme: {theme}</p>
}
 
export default function Page() {
  return (
    <>
      <h1>Dashboard</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <UserGreeting />
      </Suspense>
    </>
  )
}
```

### 캐시된 함수에 런타임 값 전달
런타임 API에서 값을 추출하여 캐시된 함수에 인수로 전달할 수 있습니다:

app/profile/page.tsx
```typescript
import { cookies } from 'next/headers'
import { Suspense } from 'react'
 
export default function Page() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <ProfileContent />
    </Suspense>
  )
}
 
// 캐시되지 않은 컴포넌트가 런타임 데이터를 읽습니다.
async function ProfileContent() {
  const session = (await cookies()).get('session')?.value
  return <CachedContent sessionId={session} />
}
 
// 캐시된 컴포넌트는 추출된 값을 prop으로 받습니다.
async function CachedContent({ sessionId }: { sessionId: string }) {
  'use cache'
  // sessionId는 캐시 키의 일부가 됩니다.
  const data = await fetchUserData(sessionId)
  return <div>{data}</div>
}
```

요청 시 일치하는 캐시 항목이 없으면 `CachedContent`가 실행되고 동일한 `sessionId`를 사용하여 향후 요청에 대한 결과를 저장합니다.

기본적으로 `use cache`는 항목을 [메모리](https://nextjs.org/docs/app/api-reference/directives/use-cache#runtime-caching-considerations)에 저장합니다. 요청 전반에 걸쳐 메모리가 유지되지 않는 서버리스 환경에서는 `CachedContent`가 모든 요청을 다시 평가할 수 있습니다. 내구성 있는 공유 캐싱을 위해 [use cache: remote](https://nextjs.org/docs/app/api-reference/directives/use-cache-remote)을 고려하세요.

---
### 비결정적 작업 작업
`Math.random()`, `Date.now()` 또는 `crypto.randomUUID()`와 같은 작업은 실행될 때마다 다른 값을 생성합니다. 캐시 컴포넌트에서는 이를 명시적으로 처리해야 합니다.

요청별로 고유한 값을 생성하려면 이러한 작업 전에 [connection()](https://nextjs.org/docs/app/api-reference/functions/connection)을 호출하여 요청 시간을 연기하고 `<Suspense>`에 구성 요소를 래핑합니다:

page.tsx
```typescript
import { connection } from 'next/server'
import { Suspense } from 'react'

async function UniqueContent() {
  await connection()
  const uuid = crypto.randomUUID()
  return <p>Request ID: {uuid}</p>
}

export default function Page() {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UniqueContent />
    </Suspense>
  )
}
```

또는 재검증될 때까지 모든 사용자가 동일한 값을 볼 수 있도록 결과를 캐시할 수 있습니다:

page.tsx
```typescript
export default async function Page() {
  'use cache'
  const buildId = crypto.randomUUID()
  return <p>Build ID: {buildId}</p>
}
```

---
### 결정론적 작업 작업
동기식 I/O, 모듈 가져오기, 순수 계산과 같은 작업은 사전 렌더링 중에 완료될 수 있습니다. 이러한 작업만 사용하는 컴포넌트에는 렌더링된 출력이 정적 HTML 셸에 자동으로 포함됩니다.

page.tsx
```typescript
import fs from 'node:fs'

export default async function Page() {
  const content = fs.readFileSync('./config.json', 'utf-8')
  const constants = await import('./constants.json')
  const processed = JSON.parse(content).items.map((item) => item.value * 2)

  return (
    <div>
      <h1>{constants.appName}</h1>
      <ul>
        {processed.map((value, i) => (
          <li key={i}>{value}</li>
        ))}
      </ul>
    </div>
  )
}
```

> 알아두면 좋은 점: 여기에는 `better-sqlite3` 또는 Node.js의 내장 [node:sqlite](https://nodejs.org/api/sqlite.html)와 같은 동기 API를 사용하여 내장된 데이터베이스에 대한 쿼리가 포함됩니다. 동기 소스의 요청별 데이터가 필요한 경우 쿼리 전에 [connection()](https://nextjs.org/docs/app/api-reference/functions/connection)을 호출하세요.

---
### 렌더링 작동 방식
빌드 시 Next.js는 경로의 컴포넌트 트리를 렌더링합니다. 각 컴포넌트가 처리되는 방식은 사용하는 API에 따라 다릅니다:

- [use cache](https://nextjs.org/docs/app/getting-started/caching#usage): 결과는 캐시되어 정적 셸에 포함됩니다.
- [<Suspense>](https://nextjs.org/docs/app/getting-started/caching#streaming-uncached-data): 대체 UI는 요청 시 콘텐츠가 스트리밍되는 동안 정적 셸에 포함됩니다.
- [결정론적 작업](https://nextjs.org/docs/app/getting-started/caching#working-with-deterministic-operations): 순수 계산 및 모듈 가져오기와 같은 작업이 정적 셸에 자동으로 포함됩니다.

이는 초기 페이지 로드를 위한 HTML과 클라이언트측 탐색을 위한 직렬화된 [RSC 페이로드](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)로 구성된 정적 셸을 생성하여 사용자가 URL로 직접 이동하거나 다른 페이지에서 전환하는 경우 브라우저가 완전히 렌더링된 콘텐츠를 즉시 수신하도록 보장합니다.

![](./images/009/thinking-in-ppr.avif)

이 렌더링 접근 방식을 `PPR(부분 사전 렌더링)`이라고 하며 이는 캐시 컴포넌트의 기본 동작입니다.

> [빌드 출력 요약](https://nextjs.org/docs/app/api-reference/cli/next#next-build-options)을 확인하여 경로가 완전히 사전 렌더링되었는지 확인할 수 있습니다. 또는 브라우저에서 페이지 소스를 확인하여 페이지의 정적 셸에 어떤 콘텐츠가 추가되었는지 확인하세요.

![](./images/009/server-rendering-with-streaming.avif)

Next.js에서는 사전 렌더링 중에 완료할 수 없는 컴포넌트를 명시적으로 처리해야 합니다. `<Suspense>`로 래핑되지 않거나 `use cache`로 표시되지 않은 경우 개발 및 빌드 시간 동안 [<Suspense> 외부에서 캐시되지 않은 데이터에 액세스](https://nextjs.org/docs/messages/blocking-route)했습니다.라는 오류가 표시됩니다.

> 🎥 시청: 부분 사전 렌더링의 이유 및 작동 방식 → [YouTube(10분)](https://www.youtube.com/watch?v=MTcPrTIBkpA)

#### 정적 셸 선택 해제
루트 레이아웃의 문서 본문 위에 빈 대체가 있는 `<Suspense>` 경계를 배치하면 전체 앱이 요청 시간을 연기하게 됩니다. 대체가 비어 있기 때문에 즉시 보낼 정적 셸이 없으므로 페이지가 완전히 렌더링될 때까지 모든 요청이 차단됩니다. 이를 특정 경로로 제한하려면 [여러 루트 레이아웃](https://nextjs.org/docs/app/api-reference/file-conventions/layout#root-layout)을 사용하십시오.

app/layout.tsx
```typescript
import { Suspense } from 'react'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html>
      <Suspense fallback={null}>
        <body>{children}</body>
      </Suspense>
    </html>
  )
}
```

> 알아두면 좋은 점: `generateViewport`가 캐시되지 않은 동적 데이터에 액세스할 때에도 이와 동일한 패턴이 적용됩니다. 자세한 예는 [캐시 컴포넌트가 있는 뷰포트](https://nextjs.org/docs/app/api-reference/functions/generate-viewport#with-cache-components)를 참조하세요.

#### 모든 것을 종합하면
다음은 단일 페이지에서 함께 작동하는 정적 콘텐츠, 캐시된 동적 콘텐츠, 스트리밍 동적 콘텐츠를 보여주는 완전한 예입니다:

app/blog/page.tsx
```typescript
import { Suspense } from 'react'
import { cookies } from 'next/headers'
import { cacheLife, cacheTag, updateTag } from 'next/cache'
import Link from 'next/link'

export default function BlogPage() {
  return (
    <>
      {/* 정적 콘텐츠 - 자동으로 사전 렌더링됨 */}
      <header>
        <h1>Out Blog</h1>
        <nav>
          <Link href="/">Home</Link> | <Link href="/about">About</Link>
        </nav>
      </header>

      {/* 캐시된 동적 콘텐츠 - 정적 셸에 포함됨 */}
      <BlogPosts />

      {/* 런타임 동적 콘텐츠 - 요청 시 스트리밍 */}
      <Suspense fallback={<p>Loading...</p>}>
        <CreatePost />
      </Suspense>

      {/* Mutation - 캐시를 재검증하는 서버 작업 */}
      <Suspense fallback={<p>Loading...</p>}>
        <CreatePost />
      </Suspense>
    </>
  )
}

// 모든 사람이 동일한 블로그 게시물을 봅니다(매시간 재검증됨)
async function BlogPosts() {
  'use cache'
  cacheLife('hours')
  cacheTag('posts')

  const res = await fetch('https://api.vercel.app/blog')
  const posts = await res.json()

  return (
    <section>
      <h2>Latest Posts</h2>
      <ul>
        {posts.slice(0, 5).map((post: any) => (
          <li key={post.id}>
            <h3>{post.title}</h3>
            <p>
              By {post.author} on {post.date}
            </p>
          </li>
        ))}
      </ul>
    </section>
  )
}

// 쿠키를 기반으로 사용자별로 맞춤화됨
async function UserPreferences() {
  const theme = (await cookies()).get('theme')?.value || 'light'
  const favoriteCategory = (await cookies()).get('category')?.value

  return (
    <aside>
      <p>Your theme: {theme}</p>
      {favoriteCategory && <p>Favorite category: {favoriteCategory}</p>}
    </aside>
  )
}

// 게시물을 작성하고 캐시를 재검증하는 관리자 전용 양식
async function CreatePost() {
  const isAdmin = (await cookies()).get('role')?.value === 'admin'
  if (!isAdmin) return null

  async function createPost(formData: FormData) {
    'use server'
    await db.post.create({ data: { title: formData.get('title') } })
    updateTag('posts')
  }

  return (
    <form action={createPost}>
      <input name="title" placeholder="Post title" required />
      <button type="submit">Publish</button>
    </form>
  )
}
```

사전 렌더링 중에 헤더(정적) 및 블로그 게시물(`use cache`로 캐시됨)은 사용자 기본 설정에 대한 대체 UI와 함께 정적 셸의 일부가 됩니다. 요청 시 개인화된 기본 설정만 스트리밍됩니다. 관리자가 새 게시물을 게시하면 [updateTag](https://nextjs.org/docs/app/getting-started/revalidating#updatetag) 호출은 다음 방문자가 볼 수 있도록 블로그 게시물 캐시를 즉시 만료합니다.

알아두면 좋은 점: `generateMetadata` 및 `generateViewport`는 페이지와 별도로 런타임 데이터 액세스를 추적합니다. 이를 처리하는 방법은 [캐시 컴포넌트가 있는 메타데이터](https://nextjs.org/docs/app/api-reference/functions/generate-metadata#with-cache-components) 및 [캐시 컴포넌트가 있는 뷰포트](https://nextjs.org/docs/app/api-reference/functions/generate-viewport#with-cache-components)를 참조하세요.