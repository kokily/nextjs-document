# Linking and Navigating
<span style="color: #808080;">Last updated June 23, 2026</span>

Next.js에서는 기본적으로 경로가 서버에서 렌더링됩니다. 이는 종종 새 경로가 표시되기 전에 클라이언트가 서버 응답을 기다려야 함을 의미합니다. Next.js에는 [미리 가져오기](https://nextjs.org/docs/app/getting-started/linking-and-navigating#prefetching), [스트리밍](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming) 및 [클라이언트 측 전환 기능](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions)이 내장되어 있어 탐색 속도와 응답 속도가 빨라집니다.

이 가이드에서는 Next.js에서 탐색이 작동하는 방식과 [동적 라우팅](https://nextjs.org/docs/app/getting-started/linking-and-navigating#dynamic-routes-without-loadingtsx) 및 [느린 네트워크](https://nextjs.org/docs/app/getting-started/linking-and-navigating#slow-networks)에 대해 탐색을 최적화하는 방법을 설명합니다.

---
## 내비게이션 작동 방식
Next.js에서 탐색이 작동하는 방식을 이해하려면 다음 개념을 익히는 것이 도움이 됩니다:
- Server Rendering
- Prefetching
- Streaming
- Client-side transitions

### Server Rendering
Next.js에서 [레이아웃과 페이지](https://nextjs.org/docs/app/getting-started/layouts-and-pages)는 기본적으로 [React Server component](https://react.dev/reference/rsc/server-components) 입니다. 초기 및 후속 탐색 시 [Server Component Payload](https://nextjs.org/docs/app/getting-started/server-and-client-components#how-do-server-and-client-components-work-in-nextjs)는 클라이언트로 전송되기 전에 서버에서 생성됩니다.

서버 렌더링에는 발생 시기에 따라 두 가지 유형이 있습니다:
- 사전 렌더링은 빌드 시 또는 [재검증](https://nextjs.org/docs/app/getting-started/revalidating) 중에 발생하며 결과가 캐시됩니다.
- 동적 렌더링은 클라이언트 요청에 대한 응답으로 요청 시 발생합니다.

서버 렌더링의 단점은 새 경로가 표시되기 전에 클라이언트가 서버의 응답을 기다려야 한다는 것입니다. Next.js는 사용자가 방문할 가능성이 있는 경로를 [미리 가져오고](https://nextjs.org/docs/app/getting-started/linking-and-navigating#prefetching) [클라이언트 측 전환](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions)을 수행하여 이러한 지연을 해결합니다.

```알아두면 좋은 점: 최초 방문 시 HTML도 생성됩니다.```

### Prefetching
프리페칭은 사용자가 경로를 탐색하기 전에 백그라운드에서 경로를 로드하는 프로세스입니다. 이렇게 하면 사용자가 링크를 클릭할 때쯤에는 다음 경로를 렌더링할 데이터가 이미 클라이언트 측에서 사용 가능해지기 때문에 애플리케이션의 경로 간 탐색이 즉각적으로 느껴집니다.

Next.js는 사용자의 뷰포트에 들어갈 때 `<Link>` 컴포넌트와 연결된 경로를 자동으로 미리 가져옵니다.

```typescript
import Link from 'next/link'
 
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <nav>
          {/* Prefetched when the link is hovered or enters the viewport */}
          <Link href="/blog">Blog</Link>
          {/* No prefetching */}
          <a href="/contact">Contact</a>
        </nav>
        {children}
      </body>
    </html>
  )
}
```

미리 가져오는 경로의 양은 정적인지 동적인지에 따라 다릅니다:

- **Static Route**: 전체 경로가 프리페치됩니다.
- **Dynamic Route**: 프리페치를 건너뛰거나 [loading.tsx](https://nextjs.org/docs/app/api-reference/file-conventions/loading)가 있는 경우 경로가 부분적으로 프리페치됩니다.

Next.js는 동적 경로를 건너뛰거나 부분적으로 미리 가져옴으로써 사용자가 절대 방문하지 않을 경로에 대해 서버에서 불필요한 작업을 방지합니다. 그러나 탐색하기 전에 서버 응답을 기다리면 사용자에게 앱이 응답하지 않는다는 인상을 줄 수 있습니다.

![](./images/005/server-rendering-without-streaming.avif)

동적 경로에 대한 탐색 환경을 개선하기 위해 [스트리밍](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming)을 사용할 수 있습니다.

### Streaming
스트리밍을 사용하면 서버는 전체 경로가 렌더링될 때까지 기다리지 않고 준비가 되는 즉시 동적 경로의 일부를 클라이언트에 보낼 수 있습니다. 이는 페이지의 일부가 아직 로드 중이더라도 사용자가 더 빨리 내용을 볼 수 있음을 의미합니다. Next.js에서 스트리밍이 작동하는 방식에 대한 자세한 내용은 [스트리밍 가이드](https://nextjs.org/docs/app/guides/streaming)를 참조하세요.

동적 경로의 경우 부분적으로 미리 가져올 수 있음을 의미합니다. 즉, 공유 레이아웃과 로딩 스켈레톤을 미리 요청할 수 있습니다.

![](./images/005/server-rendering-with-streaming.avif)

스트리밍을 사용하려면 경로 폴더에 loading.tsx를 만듭니다.

![](./images/005/loading-special-file.avif)

```typescript
export default function Loading() {
  // 경로가 로드되는 동안 표시될 대체 UI를 추가합니다.
  return <LoadingSkeleton />
}
```

뒤에서 Next.js는 `<Suspense>` 경계에서 `page.tsx` 콘텐츠를 자동으로 래핑합니다. 경로가 로드되는 동안 미리 가져온 대체 UI가 표시되고, 준비되면 실제 콘텐츠로 교체됩니다.

```
알아두면 좋은 점: <Suspense>를 사용하여 중첩된 구성 요소에 대한 로딩 UI를 만들 수도 있습니다.
```

loading.tsx의 이점:

- 사용자를 위한 즉각적인 탐색 및 시각적 피드백.
- 공유 레이아웃은 대화형으로 유지되며 탐색이 중단될 수 있습니다.
- 향상된 핵심 웹 바이탈: [TTFB](https://web.dev/articles/ttfb), [FCP](https://web.dev/articles/fcp) 및 [TTI](https://web.dev/articles/tti).

탐색 경험을 더욱 향상시키기 위해 Next.js는 `<Link>` 컴포넌트를 사용하여 [클라이언트 측 전환](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions)을 수행합니다.

### 클라이언트측 전환
일반적으로 서버에서 렌더링된 페이지를 탐색하면 전체 페이지 로드가 트리거됩니다. 이렇게 하면 상태가 지워지고 스크롤 위치가 재설정되며 상호작용이 차단됩니다.

Next.js는 `<Link>` 컴포넌트를 사용하는 클라이언트 측 전환을 통해 이를 방지합니다. 페이지를 다시 로드하는 대신 다음을 통해 콘텐츠를 동적으로 업데이트합니다.

- 공유 레이아웃과 UI를 유지합니다.
- 현재 페이지를 미리 가져온 로드 상태로 바꾸거나 가능한 경우 새 페이지로 바꿉니다.

클라이언트 측 전환은 서버 렌더링 앱이 클라이언트 렌더링 앱처럼 느껴지도록 만드는 것입니다. 또한 [프리패치](https://nextjs.org/docs/app/getting-started/linking-and-navigating#prefetching) 및 [스트리밍](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming)과 결합하면 동적 경로의 경우에도 빠른 전환이 가능합니다.

Next.js는 클라이언트 측 전환 중에 [페이지 상단으로의 스크롤](https://nextjs.org/docs/app/api-reference/components/link#scroll)도 처리합니다. 탐색 후 콘텐츠가 고정 또는 고정 헤더 뒤로 스크롤되는 경우 CSS [scroll-padding-top](https://nextjs.org/docs/app/api-reference/components/link#scroll-offset-with-sticky-headers)을 사용하여 이 문제를 해결할 수 있습니다.

---
## 무엇이 전환을 느리게 만들 수 있나요?
이러한 Next.js 최적화를 통해 탐색이 빠르고 반응이 빨라집니다. 그러나 특정 조건에서는 전환이 여전히 느리게 느껴질 수 있습니다. 다음은 몇 가지 일반적인 원인과 사용자 경험을 개선하는 방법입니다:

### loading.tsx가 없는 동적 경로
동적 경로로 이동할 때 클라이언트는 결과를 표시하기 전에 서버 응답을 기다려야 합니다. 이는 사용자에게 앱이 응답하지 않는다는 인상을 줄 수 있습니다.

부분 미리 가져오기를 활성화하고, 즉각적인 탐색을 트리거하고, 경로가 렌더링되는 동안 로딩 UI를 표시하려면 동적 경로에 `loading.tsx`를 추가하는 것이 좋습니다.

```typescript
export default function Loading() {
  return <LoadingSkeleton />
}
```

```
알아두면 좋은 점: 개발 모드에서는 Next.js Devtools를 사용하여 경로가 정적인지 동적인지 식별할 수 있습니다. 자세한 내용은 [devIndicators](https://nextjs.org/docs/app/api-reference/config/next-config-js/devIndicators)를 참조하세요.
```

### generateStaticParams가 없는 동적 세그먼트
[동적 세그먼트](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes)를 사전 렌더링할 수 있지만 [generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)가 누락되어 그렇지 않은 경우 경로는 요청 시 동적 렌더링으로 대체됩니다.

`generateStaticParams`를 추가하여 빌드 시 경로가 정적으로 생성되는지 확인하세요.:

```typescript
export async function generateStaticParams() {
  const posts = await fetch('https://.../posts').then((res) => res.json())
 
  return posts.map((post) => ({
    slug: post.slug,
  }))
}
 
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  // ...
}
```

### 느린 네트워크
느리거나 불안정한 네트워크에서는 사용자가 링크를 클릭하기 전에 미리 가져오기가 완료되지 않을 수 있습니다. 이는 정적 경로와 동적 경로 모두에 영향을 미칠 수 있습니다. 이러한 경우 `loading.js` 대체가 아직 프리페치되지 않았기 때문에 즉시 표시되지 않을 수 있습니다.

인지된 성능을 향상시키기 위해 [useLinkStatus 훅](https://nextjs.org/docs/app/api-reference/functions/use-link-status)를 사용하여 전환이 진행되는 동안 즉각적인 피드백을 표시할 수 있습니다.

```typescript
'use client'
 
import { useLinkStatus } from 'next/link'
 
export default function LoadingIndicator() {
  const { pending } = useLinkStatus()
  return (
    <span aria-hidden className={`link-hint ${pending ? 'is-pending' : ''}`} />
  )
}
```

초기 애니메이션 지연(예: 100ms)을 추가하고 보이지 않는 상태로 시작하여(예: opacity: 0) 힌트를 "디바운스"할 수 있습니다. 즉, 탐색이 지정된 지연보다 오래 걸리는 경우에만 로딩 표시기가 표시됩니다. CSS 예제는 [useLinkStatus 참조](https://nextjs.org/docs/app/api-reference/functions/use-link-status#gracefully-handling-fast-navigation)를 참조하세요.

```
알아두면 좋은 점: 진행률 표시줄과 같은 다른 시각적 피드백 패턴을 사용할 수 있습니다. 여기에서 예를 확인하세요.
```

### 프리페치 비활성화
`<Link>` 컴포넌트에서 `prefetch` prop을 `false`로 설정하여 프리페치를 선택 해제할 수 있습니다. 이는 대규모 링크 목록(예: 무한 스크롤 테이블)을 렌더링할 때 불필요한 리소스 사용을 방지하는 데 유용합니다.

```typescript
<Link prefetch={false} href="/blog">
  Blog
</Link>
```

그러나 프리페치를 비활성화하면 장단점이 있습니다:

- 정적 경로는 사용자가 링크를 클릭할 때만 가져옵니다.
- 클라이언트가 서버로 이동할 수 있으려면 먼저 서버에서 동적 경로를 렌더링해야 합니다.

프리페치를 완전히 비활성화하지 않고 리소스 사용량을 줄이려면 마우스 오버 시에만 프리페치를 수행하면 됩니다. 이는 뷰포트의 모든 링크가 아닌 사용자가 방문할 가능성이 더 높은 경로로 미리 가져오기를 제한합니다.

```typescript
'use client'
 
import Link from 'next/link'
import { useState } from 'react'
 
function HoverPrefetchLink({
  href,
  children,
}: {
  href: string
  children: React.ReactNode
}) {
  const [active, setActive] = useState(false)
 
  return (
    <Link
      href={href}
      prefetch={active ? null : false}
      onMouseEnter={() => setActive(true)}
    >
      {children}
    </Link>
  )
}
```

### 하이드레이션이 완료되지 않았습니다.
`<Link>`는 클라이언트 구성 요소이며 경로를 미리 가져오기 전에 수화되어야 합니다. 처음 방문할 때 큰 JavaScript 번들은 하이드레이션을 지연시켜 미리 가져오기가 즉시 시작되지 못하게 할 수 있습니다.

React는 선택적 하이드레이션을 통해 이를 완화하고 다음을 통해 이를 더욱 개선할 수 있습니다.:

- [@next/bundle-analyzer](https://nextjs.org/docs/app/guides/package-bundling#nextbundle-analyzer-for-webpack) 플러그인을 사용하여 큰 종속성을 제거하여 번들 크기를 식별하고 줄입니다.
- 가능한 경우 논리를 클라이언트에서 서버로 이동합니다. 지침은 [서버 및 클라이언트 컴포넌트 문서](https://nextjs.org/docs/app/getting-started/server-and-client-components)를 참조하세요.

---
## 예제
### Native History API
Next.js를 사용하면 기본 [window.history.pushState](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) 및 [window.history.replaceState](https://developer.mozilla.org/en-US/docs/Web/API/History/replaceState) 메서드를 사용하여 페이지를 다시 로드하지 않고도 브라우저의 기록 스택을 업데이트할 수 있습니다.

`pushState` 및 `replacementState` 호출은 Next.js Router에 통합되어 [usePathname](https://nextjs.org/docs/app/api-reference/functions/use-pathname) 및 [useSearchParams](https://nextjs.org/docs/app/api-reference/functions/use-search-params)와 동기화할 수 있습니다.

`window.history.pushState`

이를 사용하여 브라우저 기록 스택에 새 항목을 추가합니다. 사용자는 이전 상태로 돌아갈 수 있습니다. 예를 들어 제품 목록을 정렬하려면:

```typescript
'use client'
 
import { useSearchParams } from 'next/navigation'
 
export default function SortProducts() {
  const searchParams = useSearchParams()
 
  function updateSorting(sortOrder: string) {
    const params = new URLSearchParams(searchParams.toString())
    params.set('sort', sortOrder)
    window.history.pushState(null, '', `?${params.toString()}`)
  }
 
  return (
    <>
      <button onClick={() => updateSorting('asc')}>Sort Ascending</button>
      <button onClick={() => updateSorting('desc')}>Sort Descending</button>
    </>
  )
}
```

`window.history.replaceState`

브라우저 기록 스택의 현재 항목을 바꾸는 데 사용합니다. 사용자는 이전 상태로 다시 이동할 수 없습니다. 예를 들어, 애플리케이션의 로케일을 전환하려면:

```typescript
'use client'
 
import { usePathname } from 'next/navigation'
 
export function LocaleSwitcher() {
  const pathname = usePathname()
 
  function switchLocale(locale: string) {
    // e.g. '/en/about' or '/fr/contact'
    const newPath = `/${locale}${pathname}`
    window.history.replaceState(null, '', newPath)
  }
 
  return (
    <>
      <button onClick={() => switchLocale('en')}>English</button>
      <button onClick={() => switchLocale('fr')}>French</button>
    </>
  )
}
```