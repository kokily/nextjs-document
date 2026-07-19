# 서버 및 클라이언트 컴포넌트

<span style="color: #808080;">Last updated June 23, 2026</span>

기본적으로 레이아웃과 페이지는 [서버 컴포넌트](https://react.dev/reference/rsc/server-components)로, 이를 통해 서버에서 데이터를 가져오고 UI의 일부를 렌더링할 수 있으며 선택적으로 결과를 캐시하고 클라이언트로 스트리밍할 수 있습니다. 대화형 작업이나 브라우저 API가 필요한 경우 [클라이언트 컴포넌트](https://react.dev/reference/rsc/use-client)를 사용하여 기능을 계층화할 수 있습니다.

이 페이지에서는 서버 및 클라이언트 컴포넌트가 Next.js에서 작동하는 방식과 이를 사용하는 시기를 설명하고 애플리케이션에서 컴포넌트를 함께 구성하는 방법에 대한 예를 제공합니다.

---
## 서버 및 클라이언트 컴포넌트를 언제 사용하나요?
클라이언트와 서버 환경은 서로 다른 기능을 가지고 있습니다. 서버 및 클라이언트 컴포넌트를 사용하면 사용 사례에 따라 각 환경에서 논리를 실행할 수 있습니다.

아래 항목이 필요할 때 클라이언트 컴포넌트를 사용하세요:
- 상태, 이벤트 핸들러(onClick, onChange)
- 라이프사이클 로직(useEffect)
- 브라우저 전용 API들(localStorage, window, Navigator.geolocation)
- 커스텀 훅

아래 항목이 필요할 때 서버 컴포넌트를 사용하세요:
- 데이터베이스 데이터 페칭, 소스에 가까운 API들
- API 키, 토큰 및 기타 비밀을 클라이언트에 노출하지 않고 사용하세요.
- 브라우저로 전송되는 JavaScript의 양을 줄입니다.
- FCP(First Contentful Paint)를 개선하고 콘텐츠를 클라이언트에 점진적으로 스트리밍합니다.

예를 들어 `<Page>` 컴포넌트는 게시물에 대한 데이터를 가져오고 이를 클라이언트 측 상호 작용을 처리하는 `<LikeButton>`에 props로 전달하는 서버 컴포넌트입니다.

app/[id]/page.tsx
```typeScript
import LikeButton from '@/app/ui/like-button'
import { getPost } from '@/lib/data'
 
export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const post = await getPost(id)
 
  return (
    <div>
      <main>
        <h1>{post.title}</h1>
        {/* ... */}
        <LikeButton likes={post.likes} />
      </main>
    </div>
  )
}
```

app/ui/like-button.tsx
```typeScript
'use client'
 
import { useState } from 'react'
 
export default function LikeButton({ likes }: { likes: number }) {
  // ...
}
```

---
## Next.js에서 서버 및 클라이언트 컴포넌트는 어떻게 작동하나요?
### 서버에서는
서버에서 Next.js는 React의 API를 사용하여 렌더링을 조정합니다. 렌더링 작업은 개별 경로 세그먼트([레이아웃 및 페이지](https://nextjs.org/docs/app/getting-started/layouts-and-pages))별로 청크로 분할됩니다:
- 서버 컴포넌트는 RSC 페이로드(React Server Component Payload)라는 특수 데이터 형식으로 렌더링됩니다.
- 클라이언트 컴포넌트와 RSC 페이로드는 HTML을 사전 렌더링하는 데 사용됩니다.

```
React 서버 컴포넌트 페이로드(RSC)란 무엇입니까?

RSC 페이로드는 렌더링된 React 서버 컴포넌트 트리의 압축된 바이너리 표현입니다. 클라이언트의 React에서 브라우저의 DOM을 업데이트하는 데 사용됩니다. RSC 페이로드에는 다음이 포함됩니다:

- 서버 컴포넌트의 렌더링 결과
- 클라이언트 컴포넌트를 렌더링해야 하는 위치 및 해당 JavaScript 파일에 대한 참조에 대한 자리 표시자
- 서버 컴포넌트에서 클라이언트 컴포넌트로 전달되는 모든 소품
```

### 클라이언트에서는 (첫 번째 로드)
그런 다음 클라이언트에서:

- HTML은 사용자에게 경로의 빠른 비대화형 미리보기를 즉시 표시하는 데 사용됩니다.
- RSC 페이로드는 클라이언트 및 서버 컴포넌트 트리를 조정하는 데 사용됩니다.
- JavaScript는 클라이언트 컴포넌트를 하이드레이션하고 애플리케이션을 대화형으로 만드는 데 사용됩니다.

```
하이드레이션이란 무엇입니까?

하이드레이션은 정적 HTML을 대화형으로 만들기 위해 이벤트 핸들러를 DOM에 연결하는 React의 프로세스입니다.
```

### 후속 탐색
후속 탐색에서:
- RSC 페이로드는 즉각적인 탐색을 위해 미리 가져오고 캐시됩니다.
- 클라이언트 컴포넌트는 서버에서 렌더링된 HTML 없이 클라이언트에서 완전히 렌더링됩니다.

---
## Examples
### 클라이언트 컴포넌트 사용
파일 상단, 가져오기 위에 "`use client`" 지시어를 추가하여 클라이언트 컴포넌트를 생성할 수 있습니다.

app/ui/counter.tsx
```typeScript
'use client'
 
import { useState } from 'react'
 
export default function Counter() {
  const [count, setCount] = useState(0)
 
  return (
    <div>
      <p>{count} likes</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  )
}
```

"use client"는 서버와 클라이언트 모듈 그래프(트리) 사이의 경계를 선언하는 데 사용됩니다.

파일이 "use client"로 표시되면 모든 가져오기 및 직접 렌더링하는 컴포넌트가 클라이언트 번들에 포함됩니다. 이는 클라이언트를 대상으로 하는 모든 컴포넌트에 지시문을 추가할 필요가 없음을 의미합니다.

이 동작은 가져오는 모듈과 직접 렌더링하는 컴포넌트를 포함하는 클라이언트 컴포넌트의 모듈 그래프의 일부인 컴포넌트에 적용됩니다. 자식이나 다른 props로 전달된 서버 컴포넌트에는 적용되지 않습니다. 해당 컴포넌트는 클라이언트 컴포넌트의 [모듈 그래프](https://nextjs.org/docs/app/glossary#module-graph)로 가져오지 않습니다. 서버에서 렌더링되고 렌더링된 출력으로 클라이언트 컴포넌트에 전달됩니다.

서버 및 클라이언트 컴포넌트를 결합하는 방법은 [서버 및 클라이언트 컴포넌트 인터리빙](https://nextjs.org/docs/app/getting-started/server-and-client-components#interleaving-server-and-client-components)을 참조하세요.

### JS 번들 크기 줄이기
클라이언트 JavaScript 번들의 크기를 줄이려면 UI의 큰 부분을 클라이언트 컴포넌트로 표시하는 대신 특정 대화형 컴포넌트에 'use client'를 추가하세요.

예를 들어 `<Layout>` 컴포넌트에는 로고 및 탐색 링크와 같은 정적 요소가 대부분 포함되어 있지만 대화형 검색 창도 포함되어 있습니다. `<Search />`는 대화형이며 클라이언트 컴포넌트여야 하지만 나머지 레이아웃은 서버 컴포넌트로 남을 수 있습니다.

app/layout.tsx
```TypeScript
// 클라이언트 컴포넌트
import Search from './search'
// 서버 컴포넌트
import Logo from './logo'
 
// 레이아웃은 기본적으로 서버 컴포넌트입니다.
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <nav>
        <Logo />
        <Search />
      </nav>
      <main>{children}</main>
    </>
  )
}
```

app/ui/search.tsx
```TypeScript
'use client'
 
export default function Search() {
  // ...
}
```

### 서버에서 클라이언트 컴포넌트로 데이터 전달
props를 사용하여 서버 컴포넌트에서 클라이언트 컴포넌트로 데이터를 전달할 수 있습니다.

app/[id]/page.tsx
```TypeScript
import LikeButton from '@/app/ui/like-button'
import { getPost } from '@/lib/data'
 
export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const post = await getPost(id)
 
  return <LikeButton likes={post.likes} />
}
```

app/ui/like-button.tsx
```TypeScript
'use client'
 
export default function LikeButton({ likes }: { likes: number }) {
  // ...
}
```

또는 [API](https://react.dev/reference/react/use)를 사용하여 서버 컴포넌트에서 클라이언트 컴포넌트로 데이터를 스트리밍할 수 있습니다. [예](https://nextjs.org/docs/app/getting-started/fetching-data#streaming-data-with-the-use-api)를 참조하세요.

```
알아두면 좋은 점: 클라이언트 컴포넌트에 전달된 Prop은 React에서 직렬화할 수 있어야 합니다.
```

### 서버 및 클라이언트 컴포넌트 인터리빙
서버 컴포넌트를 클라이언트 컴포넌트에 props로 전달할 수 있습니다. 이를 통해 클라이언트 컴포넌트 내에 서버 렌더링 UI를 시각적으로 중첩할 수 있습니다.

일반적인 패턴은 `children`을 사용하여 `<ClientComponent>`에 슬롯을 만드는 것입니다. 예를 들어 서버에서 데이터를 가져오는 `<Cart>` 컴포넌트와 클라이언트 상태를 사용하여 가시성을 전환하는 `<Modal>` 컴포넌트가 있습니다.

app/ui/modal.tsx
```TypeScript
'use client'
 
export default function Modal({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>
}
```
그런 다음 상위 서버 컴포넌트(예:`<Page>`)에서 `<Cart>`를 `<Modal>`의 하위로 전달할 수 있습니다.

app/page.tsx
```typescript
import Modal from './ui/modal'
import Cart from './ui/cart'
 
export default function Page() {
  return (
    <Modal>
      <Cart />
    </Modal>
  )
}
```

이 패턴에서는 서버 컴포넌트가 클라이언트 컴포넌트에 props로 전달되는 경우에도 서버에서 미리 렌더링됩니다. React 서버 컴포넌트 페이로드에는 해당 서버 컴포넌트의 렌더링된 결과와 클라이언트 컴포넌트를 렌더링해야 하는 위치 표시자 및 해당 JavaScript 파일에 대한 참조가 포함되어 있습니다.

### 컨텍스트 제공자
[React 컨텍스트](https://react.dev/learn/passing-data-deeply-with-context)는 일반적으로 현재 테마와 같은 전역 상태를 공유하는 데 사용됩니다. 그러나 React 컨텍스트는 서버 컴포넌트에서 지원되지 않습니다.

컨텍스트를 사용하려면 하위 항목을 허용하는 클라이언트 컴포넌트를 생성하세요:

app/theme-provider.tsx
```TypeScript
'use client'
 
import { createContext } from 'react'
 
export const ThemeContext = createContext({})
 
export default function ThemeProvider({
  children,
}: {
  children: React.ReactNode
}) {
  return <ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>
}
```

그런 다음 서버 컴포넌트(예: layout)로 가져옵니다:

app/layout.tsx
```typescript
import ThemeProvider from './theme-provider'
 
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html>
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

이제 서버 컴포넌트가 공급자를 직접 렌더링할 수 있으며 앱 전체의 다른 모든 클라이언트 컴포넌트가 이 컨텍스트를 사용할 수 있습니다.

```
알아두면 좋은 점: 공급자를 트리에서 최대한 깊게 렌더링해야 합니다. `ThemeProvider`가 전체 <html> 문서 대신 {children}만 래핑하는 방법에 주목하세요. 이렇게 하면 Next.js가 서버 컴포넌트의 정적 부분을 더 쉽게 최적화할 수 있습니다.
```

### 타사 구성 요소
클라이언트 전용 기능에 의존하는 타사 컴포넌트를 사용하는 경우 이를 클라이언트 컴포넌트에 래핑하여 예상대로 작동하는지 확인할 수 있습니다.

예를 들어 `<Carousel />`은 `acme-carousel` 패키지에서 가져올 수 있습니다. 이 컴포넌트는 `useState`를 사용하지만 아직 "use client" 지시문이 없습니다.

클라이언트 컴포넌트 내에서 `<Carousel />`을 사용하면 예상대로 작동합니다:

app/gallery.tsx
```TypeScript
'use client'
 
import { useState } from 'react'
import { Carousel } from 'acme-carousel'
 
export default function Gallery() {
  const [isOpen, setIsOpen] = useState(false)
 
  return (
    <div>
      <button onClick={() => setIsOpen(true)}>View pictures</button>
      {/* Works, since Carousel is used within a Client Component */}
      {isOpen && <Carousel />}
    </div>
  )
}
```

그러나 서버 컴포넌트 내에서 직접 사용하려고 하면 오류가 표시됩니다. 이는 Next.js가 `<Carousel />`이 클라이언트 전용 기능을 사용하고 있다는 것을 모르기 때문입니다.

이 문제를 해결하려면 클라이언트 컴포넌트에 클라이언트 전용 기능을 사용하는 타사 컴포넌트를 래핑하면 됩니다:

app/carousel.tsx
```TypeScript
'use client'
 
import { Carousel } from 'acme-carousel'
 
export default Carousel
```

이제 서버 컴포넌트 내에서 `<Carousel />`을 직접 사용할 수 있습니다:

app/page.tsx
```TypeScript
import Carousel from './carousel'
 
export default function Page() {
  return (
    <div>
      <p>View pictures</p>
      {/*  Works, since Carousel is a Client Component */}
      <Carousel />
    </div>
  )
}
```

```
라이브러리 제작자를 위한 조언

컴포넌트 라이브러리를 구축하는 경우 클라이언트 전용 기능에 의존하는 진입점에 "use client" 지시문을 추가하세요. 이를 통해 사용자는 래퍼를 만들 필요 없이 컴포넌트를 서버 컴포넌트로 가져올 수 있습니다.

일부 번들러는 "use client" 지시어를 제거할 수 있다는 점은 주목할 가치가 있습니다. [React Wrap Balancer](https://github.com/shuding/react-wrap-balancer/blob/main/tsup.config.ts#L10-L13) 및 [Vercel Analytics](https://github.com/vercel/analytics/blob/main/packages/web/tsup.config.js#L26-L30) 저장소에 "use client" 지시문을 포함하도록 esbuild를 구성하는 방법에 대한 예를 찾을 수 있습니다.
```

### 환경중독 예방
JavaScript 모듈은 서버 및 클라이언트 컴포넌트 모듈 간에 공유될 수 있습니다. 이는 실수로 서버 전용 코드를 클라이언트로 가져올 수 있음을 의미합니다. 예를 들어 다음 함수를 고려해보세요:

lib/data.ts
```TypeScript
export async function getData() {
  const res = await fetch('https://external-service.com/data', {
    headers: {
      authorization: process.env.API_KEY,
    },
  })
 
  return res.json()
}
```

이 함수에는 클라이언트에 노출되어서는 안 되는 `API_KEY`가 포함되어 있습니다.

Next.js에서는 `NEXT_PUBLIC_` 접두사가 붙은 환경 변수만 클라이언트 번들에 포함됩니다. 변수에 접두사가 붙지 않으면 Next.js는 변수를 빈 문자열로 바꿉니다.

결과적으로 클라이언트에서 `getData()`를 가져와서 실행할 수 있더라도 예상대로 작동하지 않습니다.

클라이언트 컴포넌트에서 실수로 사용되는 것을 방지하려면 [서버 전용 패키지](https://www.npmjs.com/package/server-only)를 사용할 수 있습니다.

그런 다음 서버 전용 코드가 포함된 파일로 패키지를 가져옵니다:

lib/data.js
```typescript
import 'server-only'
 
export async function getData() {
  const res = await fetch('https://external-service.com/data', {
    headers: {
      authorization: process.env.API_KEY,
    },
  })
 
  return res.json()
}
```

이제 모듈을 클라이언트 컴포넌트로 가져오려고 하면 빌드 시간 오류가 발생합니다.

해당 클라이언트 전용 패키지를 사용하여 창 개체에 액세스하는 코드와 같은 클라이언트 전용 논리가 포함된 모듈을 표시할 수 있습니다.

Next.js에서 `서버 전용` 또는 `클라이언트 전용` 설치는 선택 사항입니다. 그러나 Linting 규칙이 외부 종속성을 표시하는 경우 문제를 방지하기 위해 해당 규칙을 설치할 수 있습니다.

```
npm add server-only
```

Next.js는 모듈이 잘못된 환경에서 사용될 때 더 명확한 오류 메시지를 제공하기 위해 `서버 전용` 및 `클라이언트 전용` 가져오기를 내부적으로 처리합니다. NPM의 이러한 패키지 내용은 Next.js에서 사용되지 않습니다.

Next.js는 또한 [noUncheckedSideEffectImports](https://www.typescriptlang.org/tsconfig/#noUncheckedSideEffectImports)가 활성화된 TypeScript 구성에 대해 `서버 전용` 및 `클라이언트 전용`에 대한 자체 유형 선언을 제공합니다.