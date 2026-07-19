# Fetching Data
<span style="color: #808080;">Last updated March 13, 2026</span>

이 페이지에서는 서버 및 클라이언트 컴포넌트에서 데이터를 가져오는 방법과 캐시되지 않은 데이터에 의존하는 컴포넌트를 스트리밍하는 방법을 안내합니다.

---
## Fetching data
### 서버 컴포넌트
다음과 같은 비동기 I/O를 사용하여 서버 컴포넌트에서 데이터를 가져올 수 있습니다:

1. [Fetch API](https://nextjs.org/docs/app/getting-started/fetching-data#with-the-fetch-api)
2. An [ORM or Database](https://nextjs.org/docs/app/getting-started/fetching-data#with-an-orm-or-database)

#### `fetch` API 사용
To fetch data with the fetch API, turn your component into an asynchronous function, and await the fetch call. For example:

app/blog/page.tsx
```typescript
export default async function Page() {
  const data = await fetch('https://api.vercel.app/blog')
  const posts = await data.json()
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

> 알아두면 좋은 점
> - React 컴포넌트 트리에서 동일한 `fetch` 요청은 기본적으로 [메모되므로](https://nextjs.org/docs/app/glossary#memoization) props를 드릴링하는 대신 필요한 컴포넌트에서 데이터를 가져올 수 있습니다. 
> - `fetch` 요청은 기본적으로 캐시되지 않으며 요청이 완료될 때까지 페이지 렌더링을 차단합니다. [use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache) 지시문을 사용하여 결과를 캐시하거나 `<Suspense>`에서 가져오기 컴포넌트를 래핑하여 요청 시 새로운 데이터를 스트리밍합니다. 자세한 내용은 [캐싱](https://nextjs.org/docs/app/getting-started/caching)을 참조하세요.
> - 개발 중에 더 나은 가시성과 디버깅을 위해 가져오기 호출을 기록할 수 있습니다. [로깅 API 참조](https://nextjs.org/docs/app/api-reference/config/next-config-js/logging)를 확인하세요.

#### ORM 또는 데이터베이스 사용
서버 컴포넌트는 서버에서 렌더링되므로 자격 증명 및 쿼리 논리는 클라이언트 번들에 포함되지 않으므로 ORM 또는 데이터베이스 클라이언트를 사용하여 데이터베이스 쿼리를 안전하게 만들 수 있습니다.

app/blog/page.tsx
```typescript
import { db, posts } from '@/lib/db'
 
export default async function Page() {
  const allPosts = await db.select().from(posts)
  return (
    <ul>
      {allPosts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

요청이 적절하게 인증되고 승인되었는지 확인해야 합니다. 서버 측 데이터 액세스 보안에 대한 모범 사례는 [데이터 보안 가이드](https://nextjs.org/docs/app/guides/data-security)를 참조하세요.

#### Streaming
서버 컴포넌트에서 데이터를 가져오면 각 요청에 대해 서버에서 데이터를 가져오고 렌더링합니다. 데이터 요청 속도가 느린 경우 모든 데이터를 가져올 때까지 전체 경로의 렌더링이 차단됩니다.

초기 로드 시간과 사용자 경험을 개선하려면 페이지를 더 작은 청크로 나누고 점진적으로 해당 청크를 서버에서 클라이언트로 보낼 수 있습니다. 이것을 스트리밍이라고 합니다. HTTP 계약, 인프라 고려 사항 및 성능 균형을 포함하여 스트리밍 작동 방식에 대한 자세한 내용은 [스트리밍 가이드](https://nextjs.org/docs/app/guides/streaming)를 참조하세요.

![](./images/007/server-rendering-with-streaming.avif)

애플리케이션에서 스트리밍을 사용할 수 있는 두 가지 방법이 있습니다:

1. [loading.js 파일](https://nextjs.org/docs/app/getting-started/fetching-data#with-loadingjs)로 페이지 래핑하기
2. `<Suspense>`로 컴포넌트 래핑하기

#### loading.js 사용
페이지와 동일한 폴더에 `loading.js` 파일을 생성하여 데이터를 가져오는 동안 전체 페이지를 스트리밍할 수 있습니다. 예를 들어 `app/blog/page.js`를 스트리밍하려면 `app/blog` 폴더 내에 파일을 추가하세요.

![](./images/007/loading-file.avif)

app/blog/loading.tsx
```typescript
export default function Loading() {
  // Define the Loading UI here
  return <div>Loading...</div>
}
```

탐색 시 페이지가 렌더링되는 동안 사용자는 레이아웃과 [로딩 상태](https://nextjs.org/docs/app/getting-started/fetching-data#creating-meaningful-loading-states)를 즉시 확인할 수 있습니다. 렌더링이 완료되면 새 콘텐츠가 자동으로 교체됩니다.

![](./images/007/loading-ui.avif)

그 뒤에서 `loading.js`는 [layout.js 내에 중첩](https://nextjs.org/docs/app/getting-started/project-structure#component-hierarchy)되고 `<Suspense>` 경계에서 `page.js` 파일과 그 아래의 모든 하위 항목을 자동으로 래핑합니다.

![](./images/007/loading-overview.avif)

이로 인해 캐시되지 않은 데이터 또는 런타임 데이터(예: cookies(), headers() 또는 캐시되지 않은 가져오기)에 액세스하는 레이아웃은 동일한 경로 세그먼트 `loading.js`로 대체되지 않습니다. 대신 레이아웃 렌더링이 완료될 때까지 탐색을 차단합니다. [캐시 컴포넌트](https://nextjs.org/docs/app/getting-started/caching)는 빌드 시간 오류를 안내하여 이를 방지합니다.

이 문제를 해결하려면 폴백을 사용하여 캐시되지 않은 액세스를 자체 [<Suspense>](https://nextjs.org/docs/app/getting-started/fetching-data#with-suspense) 경계로 래핑하거나 데이터 가져오기를 `loading.js`가 처리할 수 있는 `page.js`로 이동하세요. 자세한 내용은 [loading.js](https://nextjs.org/docs/app/api-reference/file-conventions/loading)를 참조하세요.

이것이 바로 `loading.js`가 스트리밍 경로 세그먼트에 잘 작동하는 반면 `<Suspense>`를 런타임에 더 가깝게 사용하거나 캐시되지 않은 데이터 액세스를 권장하는 이유입니다.

#### `<Suspense>` 사용
`<Suspense>`를 사용하면 페이지에서 스트리밍할 부분을 더욱 세부적으로 설정할 수 있습니다. 예를 들어 `<Suspense>` 경계 외부에 있는 모든 페이지 콘텐츠를 즉시 표시하고 경계 내부의 블로그 게시물 목록을 스트리밍할 수 있습니다.

app/blog/page.tsx
```typescript
import { Suspense } from 'react'
import BlogList from '@/components/BlogList'
import BlogListSkeleton from '@/components/BlogListSkeleton'
 
export default function BlogPage() {
  return (
    <div>
      {/* This content will be sent to the client immediately */}
      <header>
        <h1>Welcome to the Blog</h1>
        <p>Read the latest posts below.</p>
      </header>
      <main>
        {/* If there's any dynamic content inside this boundary, it will be streamed in */}
        <Suspense fallback={<BlogListSkeleton />}>
          <BlogList />
        </Suspense>
      </main>
    </div>
  )
}
```

#### 의미 있는 로딩 상태 만들기
즉시 로딩 상태는 탐색 후 사용자에게 즉시 표시되는 대체 UI입니다. 최고의 사용자 경험을 위해 의미 있고 사용자가 앱이 응답하고 있음을 이해하는 데 도움이 되는 로드 상태를 디자인하는 것이 좋습니다. 예를 들어 스켈레톤과 스피너를 사용할 수도 있고 표지 사진, 제목 등 미래 화면의 작지만 의미 있는 부분을 사용할 수도 있습니다.

개발 중에는 [React Devtools](https://react.dev/learn/react-developer-tools)를 사용하여 컴포넌트의 로딩 상태를 미리 보고 검사할 수 있습니다.

### 클라이언트 컴포넌트
클라이언트 컴포넌트에서 데이터를 가져오는 방법에는 두 가지가 있습니다:

1. React의 [use API](https://react.dev/reference/react/use)
2. [SWR](https://swr.vercel.app/) 또는 [React Query](https://tanstack.com/query/latest)와 같은 커뮤니티 라이브러리

#### Streaming data with the use API
React의 [use API](https://react.dev/reference/react/use)를 사용하여 서버에서 클라이언트로 데이터를 [스트리밍](https://nextjs.org/docs/app/getting-started/fetching-data#streaming)할 수 있습니다. 서버 컴포넌트에서 데이터를 가져오는 것으로 시작하고 Promise를 클라이언트 컴포넌트에 props으로 전달합니다:

app/blog/page.tsx
```typescript
import Posts from '@/app/ui/posts'
import { Suspense } from 'react'
 
export default function Page() {
  // Don't await the data fetching function
  const posts = getPosts()
 
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Posts posts={posts} />
    </Suspense>
  )
}
```

그런 다음 클라이언트 컴포넌트에서 `use` API를 사용하여 Propmise을 읽습니다:

app/ui/posts.tsx
```typescript
'use client'
import { use } from 'react'
 
export default function Posts({
  posts,
}: {
  posts: Promise<{ id: string; title: string }[]>
}) {
  const allPosts = use(posts)
 
  return (
    <ul>
      {allPosts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

위의 예에서 `<Posts>` 컴포넌트는 [<Suspense> 바운더리](https://react.dev/reference/react/Suspense)로 래핑됩니다. 이는 Promise가 해결되는 동안 fallback이 표시된다는 의미입니다. [스트리밍](https://nextjs.org/docs/app/getting-started/fetching-data#streaming)에 대해 자세히 알아보세요.

#### 커뮤니티 라이브러리
[SWR](https://swr.vercel.app/) 또는 [React Query](https://tanstack.com/query/latest)와 같은 커뮤니티 라이브러리를 사용하여 클라이언트 구성 요소에서 데이터를 가져올 수 있습니다. 이러한 라이브러리에는 캐싱, 스트리밍 및 기타 기능에 대한 고유한 의미가 있습니다. 예를 들어 SWR의 경우:

```typescript
'use client'
import useSWR from 'swr'
 
const fetcher = (url) => fetch(url).then((r) => r.json())
 
export default function BlogPage() {
  const { data, error, isLoading } = useSWR(
    'https://api.vercel.app/blog',
    fetcher
  )
 
  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>
 
  return (
    <ul>
      {data.map((post: { id: string; title: string }) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

---
## Examples
### Sequential data fetching
순차적 데이터 가져오기는 한 요청이 다른 요청의 데이터에 의존할 때 발생합니다.

예를 들어 `<Playlists>`는 `<Artist>`가 완료된 후에만 데이터를 가져올 수 있습니다. `artistID`가 필요하기 때문입니다:

app/artist/[usename]/page.tsx
```typescript
export default async function Page({
  params,
}: {
  params: Promise<{ username: string }>
}) {
  const { username } = await params
  // 아티스트 정보 얻기
  const artist = await getArtist(username)
 
  return (
    <>
      <h1>{artist.name}</h1>
      {/* 재생 목록 컴포넌트가 로드되는 동안 대체 UI 표시 */}
      <Suspense fallback={<div>Loading...</div>}>
        {/* 아티스트 ID를 재생 목록 컴포넌트에 전달 */}
        <Playlists artistID={artist.id} />
      </Suspense>
    </>
  )
}
 
async function Playlists({ artistID }: { artistID: string }) {
  // 아티스트 ID를 사용하여 재생목록을 가져옵니다.
  const playlists = await getArtistPlaylists(artistID)
 
  return (
    <ul>
      {playlists.map((playlist) => (
        <li key={playlist.id}>{playlist.name}</li>
      ))}
    </ul>
  )
}
```

이 예에서 `<Suspense>`는 아티스트 데이터가 로드된 후 재생 목록이 스트리밍되도록 허용합니다. 그러나 페이지는 아무것도 표시하기 전에 여전히 아티스트 데이터를 기다립니다. 이를 방지하려면 전체 페이지 컴포넌트를 `<Suspense>` 경계로 래핑할 수 있습니다(예: [loading.js 파일](https://nextjs.org/docs/app/getting-started/fetching-data 사용))

데이터 소스가 다른 모든 요청을 차단하므로 첫 번째 요청을 신속하게 해결할 수 있는지 확인하세요. 요청을 더 이상 최적화할 수 없는 경우 데이터가 자주 변경되지 않으면 결과를 [캐싱](https://nextjs.org/docs/app/getting-started/caching)하는 것이 좋습니다.

### 병렬 데이터 가져오기
병렬 데이터 가져오기는 경로의 데이터 요청이 적극적으로 시작되고 동시에 시작될 때 발생합니다.

기본적으로 [레이아웃과 페이지](https://nextjs.org/docs/app/getting-started/layouts-and-pages)는 병렬로 렌더링됩니다. 따라서 각 세그먼트는 가능한 한 빨리 데이터 가져오기를 시작합니다.

그러나 모든 컴포넌트 내에서 여러 `async`/`await` 요청이 다른 요청 뒤에 배치되면 여전히 순차적일 수 있습니다. 예를 들어 `getAlbums`는 `getArtist`가 해결될 때까지 차단됩니다:

app/artist/[username]/page.tsx
```typescript
import { getArtist, getAlbums } from '@/app/lib/data'
 
export default async function Page({ params }) {
  // These requests will be sequential
  const { username } = await params
  const artist = await getArtist(username)
  const albums = await getAlbums(username)
  return <div>{artist.name}</div>
}
```

`fetch`를 호출하여 여러 요청을 시작한 다음 [Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)을 사용하여 기다립니다. `fetch`가 호출되자마자 요청이 시작됩니다.

app/artist/[username]/page.tsx
```typescript
import Albums from './albums'
 
async function getArtist(username: string) {
  const res = await fetch(`https://api.example.com/artist/${username}`)
  return res.json()
}
 
async function getAlbums(username: string) {
  const res = await fetch(`https://api.example.com/artist/${username}/albums`)
  return res.json()
}
 
export default async function Page({
  params,
}: {
  params: Promise<{ username: string }>
}) {
  const { username } = await params
 
  // Initiate requests
  const artistData = getArtist(username)
  const albumsData = getAlbums(username)
 
  const [artist, albums] = await Promise.all([artistData, albumsData])
 
  return (
    <>
      <h1>{artist.name}</h1>
      <Albums list={albums} />
    </>
  )
}
```

> 알아두면 좋은 점: `Promise.all`을 사용할 때 하나의 요청이 실패하면 전체 작업이 실패합니다. 이를 처리하려면 [Promise.allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) 메서드를 대신 사용할 수 있습니다.

### 컨텍스트 및 `React.cache`를 사용하여 데이터 공유
[React.cache](https://react.dev/reference/react/cache)를 컨텍스트 공급자와 결합하여 서버와 클라이언트 컴포넌트 모두에서 가져온 데이터를 공유할 수 있습니다.

데이터를 가져오는 캐시된 함수 만들기:

app/lib/user.ts
```typescript
import { cache } from 'react'
 
export const getUser = cache(async () => {
  const res = await fetch('https://api.example.com/user')
  return res.json()
})
```

Promise을 저장하는 컨텍스트 공급자 만들기:

app/user-provider.tsx
```typescript
'use client'
 
import { createContext } from 'react'
 
type User = {
  id: string
  name: string
}
 
export const UserContext = createContext<Promise<User> | null>(null)
 
export default function UserProvider({
  children,
  userPromise,
}: {
  children: React.ReactNode
  userPromise: Promise<User>
}) {
  return <UserContext value={userPromise}>{children}</UserContext>
}
```

레이아웃에서는 기다리지 않고 프로미스를 공급자에게 전달합니다:

app/layout.tsx
```typescript
import UserProvider from './user-provider'
import { getUser } from './lib/user'
 
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const userPromise = getUser() // Don't await
 
  return (
    <html>
      <body>
        <UserProvider userPromise={userPromise}>{children}</UserProvider>
      </body>
    </html>
  )
}
```

클라이언트 컴포넌트는 [use()](https://react.dev/reference/react/use)를 사용하여 폴백 UI를 위해 `<Suspense>`로 래핑된 컨텍스트에서 Promise를 확인합니다:

app/ui/profile.tsx
```typescript
'use client'
 
import { use, useContext } from 'react'
import { UserContext } from '../user-provider'
 
export function Profile() {
  const userPromise = useContext(UserContext)
  if (!userPromise) {
    throw new Error('useContext must be used within a UserProvider')
  }
  const user = use(userPromise)
  return <p>Welcome, {user.name}</p>
}
```

app/page.tsx
```typescript
import { Suspense } from 'react'
import { Profile } from './ui/profile'
 
export default function Page() {
  return (
    <Suspense fallback={<div>Loading profile...</div>}>
      <Profile />
    </Suspense>
  )
}
```

서버 컴포넌트는 `getUser()`를 직접 호출할 수도 있습니다:

app/dashboard/page.tsx
```typescript
import { getUser } from '../lib/user'
 
export default async function DashboardPage() {
  const user = await getUser() // 캐시됨 - 동일한 요청, 중복 가져오기 없음
  return <h1>Dashboard for {user.name}</h1>
}
```

`getUser`는 `React.cache`로 래핑되므로 동일한 요청 내의 여러 호출은 서버 컴포넌트에서 직접 호출되거나 클라이언트 컴포넌트의 컨텍스트를 통해 해결되는지 여부에 관계없이 동일한 메모된 결과를 반환합니다.

> 알아두면 좋은 점: `React.cache`는 현재 요청으로만 범위가 지정됩니다. 각 요청은 요청 간 공유 없이 자체 메모 범위를 갖습니다.