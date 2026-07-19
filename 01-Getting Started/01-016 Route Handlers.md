# Route Handlers

<span style="color: #808080;">Last updated March 3, 2026</span>

---
## Route Handlers
경로 핸들러를 사용하면 웹 [요청](https://developer.mozilla.org/docs/Web/API/Request) 및 [응답](https://developer.mozilla.org/docs/Web/API/Response) API를 사용하여 특정 경로에 대한 사용자 정의 요청 핸들러를 생성할 수 있습니다.

![](./images/016/route-special-file.avif)

> 알아두면 좋은 점: 경로 처리기는 `app` 디렉터리 내에서만 사용할 수 있습니다. `pages` 디렉터리 내부의 [API 경로](https://nextjs.org/docs/pages/building-your-application/routing/api-routes)와 동일하므로 API 경로와 경로 처리기를 함께 사용할 필요가 없습니다.

### Convention
경로 핸들러는 `app` 디렉터리 내의 [route.js|ts 파일](https://nextjs.org/docs/app/api-reference/file-conventions/route)에 정의되어 있습니다:

app/api/route.ts
```ts
export async function GET(request: Request) {}
```

라우트 핸들러는 `page.js` 및 `layout.js`와 유사하게 `app` 디렉토리 내 어디에나 중첩될 수 있습니다. 그러나 `page.js`와 동일한 경로 세그먼트 수준에는 `route.js` 파일이 있을 수 없습니다.

### Supported HTTP Methods
`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD` 및 `OPTIONS`와 같은 [HTTP 메서드](https://developer.mozilla.org/docs/Web/HTTP/Methods)가 지원됩니다. 지원되지 않는 메서드가 호출되면 Next.js는 `405 Method Not Allowed` 응답을 반환합니다.

### Extended `NextRequest` and `Nextresponse` APIs
기본 [요청](https://developer.mozilla.org/docs/Web/API/Request) 및 [응답](https://developer.mozilla.org/docs/Web/API/Response) API를 지원하는 것 외에도 Next.js는 [`NextRequest`](https://nextjs.org/docs/app/api-reference/functions/next-request) 및 [`NextResponse`](https://nextjs.org/docs/app/api-reference/functions/next-response)로 이를 확장하여 고급 사용 사례에 편리한 도우미를 제공합니다.

### Caching
경로 처리기는 기본적으로 캐시되지 않습니다. 그러나 `GET` 메서드에 대한 캐싱을 선택할 수 있습니다. 지원되는 다른 HTTP 메소드는 캐시되지 않습니다. `GET` 메서드를 캐시하려면 경로 핸들러 파일에서 `export const dynamic = 'force-static'`과 같은 [경로 구성 옵션](https://nextjs.org/docs/app/guides/caching-without-cache-components#dynamic)을 사용하십시오.

app/items/route.ts
```ts
export const dynamic = 'force-static'
 
export async function GET() {
  const res = await fetch('https://data.mongodb-api.com/...', {
    headers: {
      'Content-Type': 'application/json',
      'API-Key': process.env.DATA_API_KEY,
    },
  })
  const data = await res.json()
 
  return Response.json({ data })
}
```

> 알아두면 좋은 점: 지원되는 다른 HTTP 메서드는 캐시된 `GET` 메서드와 함께 동일한 파일에 배치되더라도 캐시되지 않습니다.

#### With Cache components
[캐시 컴포넌트](https://nextjs.org/docs/app/getting-started/caching)가 활성화되면 `GET` 경로 핸들러는 애플리케이션의 일반 UI 경로와 동일한 모델을 따릅니다. 기본적으로 요청 시 실행되고, 캐시되지 않은 데이터나 런타임 데이터에 액세스하지 않을 때 사전 렌더링될 수 있으며, `use cache`로 정적 응답에 캐시되지 않은 데이터를 포함할 수 있습니다.

정적 예 - 캐시되지 않은 데이터 또는 런타임 데이터에 액세스하지 않으므로 빌드 시 사전 렌더링됩니다.:

app/api/project-info/route.ts
```ts
export async function GET() {
  return Response.json({
    projectName: 'Next.js',
  })
}
```

동적 예 - 비결정적 작업에 액세스합니다. 빌드 중에 `Math.random()`이 호출되면 사전 렌더링이 중지되고 요청 시 렌더링이 연기됩니다:

app/api/random-number/route.ts
```ts
export async function GET() {
  return Response.json({
    randomNumber: Math.random(),
  })
}
```

런타임 데이터 예 - 요청별 데이터에 액세스합니다. `headers()`와 같은 런타임 API가 호출되면 사전 렌더링이 종료됩니다:

app/api/user-agent/route.ts
```ts
import { headers } from 'next/headers'
 
export async function GET() {
  const headersList = await headers()
  const userAgent = headersList.get('user-agent')
 
  return Response.json({ userAgent })
}
```

> 알아두면 좋은 점: `GET` 핸들러가 네트워크 요청, 데이터베이스 쿼리, 비동기 파일 시스템 작업, 요청 개체 속성(예: `req.url`, `request.headers`, `request.cookies`, `request.body`), [cookies()](https://nextjs.org/docs/app/api-reference/functions/cookies), [headers()]*(https://nextjs.org/docs/app/api-reference/functions/headers), [connection()](https://nextjs.org/docs/app/api-reference/functions/connection)과 같은 런타임 API 또는 비결정적 작업에 액세스하는 경우 사전 렌더링이 중지됩니다.

캐시된 예 - 캐시되지 않은 데이터(데이터베이스 쿼리)에 액세스하지만 캐시를 사용하여 캐시하여 사전 렌더링된 응답에 포함될 수 있도록 합니다:

app/api/products/route.ts
```ts
import { cacheLife } from 'next/cache'
 
export async function GET() {
  const products = await getProducts()
  return Response.json(products)
}
 
async function getProducts() {
  'use cache'
  cacheLife('hours')
 
  return await db.query('SELECT * FROM products')
}
```

> 알아두면 좋은 점: `use cache`은 경로 핸들러 본문 내에서 직접 사용할 수 없습니다. 도우미 함수로 추출합니다. 캐시된 응답은 새 요청이 도착하면 `cacheLife`에 따라 재검증됩니다.

### Special Route Handlers
[sitemap.ts](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap), [opengraph-image.tsx](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/opengraph-image) 및 [icon.tsx](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/app-icons)와 같은 특수 경로 처리기 및 기타 [메타데이터 파일](https://nextjs.org/docs/app/api-reference/file-conventions/metadata)은 요청 시간 API 또는 동적 구성 옵션을 사용하지 않는 한 기본적으로 정적으로 유지됩니다.

### Route Resolution
경로를 가장 낮은 수준의 `라우팅` 기본 요소로 간주할 수 있습니다.

- `page`와 같은 레이아웃이나 클라이언트 측 탐색에는 참여하지 않습니다.
- `page.js`와 동일한 경로에 `route.js` 파일이 있을 수 없습니다.

|Page|Route|Result|
|---|---|---|
|`app/page.js`|`app/route.js`|X Conflict|
|`app/page.js`|`app/api/route.js`|O Valid|
|`app/[user]/page.js`|`app/api/route.js`|O Valid|

각 `route.js` 또는 `page.js` 파일은 해당 경로에 대한 모든 HTTP 동사를 인수합니다.

app/page.ts
```ts
export default function Page() {
  return <h1>Hello, Next.js!</h1>
}
 
// Conflict
// `app/route.ts`
export async function POST(request: Request) {}
```

경로 처리기가 [프런트엔드 애플리케이션을 보완](https://nextjs.org/docs/app/guides/backend-for-frontend)하는 방법에 대해 자세히 알아보거나 경로 처리기 [API 참조](https://nextjs.org/docs/app/api-reference/file-conventions/route)를 살펴보세요.

### Route Context Helper
TypeScript에서는 전역적으로 사용 가능한 `RouteContext` 도우미를 사용하여 경로 처리기에 대한 `context` 매개 변수를 입력할 수 있습니다:

app/users/[id]/route.ts
```ts
import type { NextRequest } from 'next/server'
 
export async function GET(_req: NextRequest, ctx: RouteContext<'/users/[id]'>) {
  const { id } = await ctx.params
  return Response.json({ id })
}
```

> 알아두면 좋은 점
> - 유형은 `next dev`, `next build` 또는 `next typegen` 중에 생성됩니다.