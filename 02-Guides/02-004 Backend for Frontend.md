# Next.js를 프런트엔드의 백엔드로 사용하는 방법
<span style="color: #808080;">Last updated June 23, 2026</span>

Next.js는 "프런트엔드용 백엔드" 패턴을 지원합니다. 이를 통해 HTTP 요청을 처리하고 HTML뿐만 아니라 모든 콘텐츠 유형을 반환하는 공개 끝점을 만들 수 있습니다. 또한 데이터 소스에 액세스하고 원격 데이터 업데이트와 같은 부작용을 수행할 수도 있습니다.

새 프로젝트를 시작하는 경우 `--api` 플래그와 함께 `create-next-app`을 사용하면 새 프로젝트의 `app/` 폴더에 예제 `route.ts`가 자동으로 포함되어 API 엔드포인트를 생성하는 방법을 보여줍니다.

```bash
npx create-next-app@latest --api
```

> 알아두면 좋은 점: Next.js 백엔드 기능은 완전한 백엔드 대체가 아닙니다. 이는 API 계층 역할을 합니다:
> - 공개적으로 접근 가능
> - 모든 HTTP 요청을 처리합니다.
> - 모든 콘텐츠 유형을 반환할 수 있습니다.

이 패턴을 구현하려면 다음을 사용하세요:
- [Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route)
- [`proxy`](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)
- In Pages Router, [API Routes](https://nextjs.org/docs/pages/building-your-application/routing/api-routes)

## Public Endpoints
경로 처리기는 공개 HTTP 끝점입니다. 모든 클라이언트가 액세스할 수 있습니다.

`route.ts` 또는 `route.js` 파일 규칙을 사용하여 경로 처리기를 생성합니다:

```ts
export function GET(request: Request) {}
```

이는 `/api`로 전송된 `GET` 요청을 처리합니다.

예외가 발생할 수 있는 작업에는 `try`/`catch` 블록을 사용하세요:

/app/api/route.ts
```ts
import { submit } from '@/lib/submit'

export async function POST(request: Request) {
  try {
    await submit(request)
    return new Response(null, { status: 204 })
  } catch (reason) {
    const message = reason instanceof Error ? reason.message : 'Unexpected error'
    return new Response(message, { status: 500 })
  }
}
```

클라이언트에 전송되는 오류 메시지에 민감한 정보가 노출되지 않도록 하세요.

액세스를 제한하려면 인증 및 승인을 구현하세요. [Authentication](https://nextjs.org/docs/app/guides/authentication)

## Content types
경로 핸들러를 사용하면 JSON, XML, 이미지, 파일, 일반 텍스트 등 UI가 아닌 응답을 제공할 수 있습니다.

Next.js는 공통 엔드포인트에 대한 파일 규칙을 사용합니다.:
- [`sitemap.xml`](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap)
- [`opengraph-image.jpg`](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/opengraph-image), [`twitter-image`](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/opengraph-image)
- [favicon](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/app-icons), [app icon](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/app-icons), and [apple-icon](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/app-icons)
- [`manifest.json`](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/manifest)
- [`robots.txt`](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/robots)

다음과 같은 사용자 정의 항목을 정의할 수도 있습니다:
- `llms.txt`
- `rss.xml`
- `.well-known`

예를 들어 `app/rss.xml/route.ts`는 `rss.xml`에 대한 경로 처리기를 생성합니다.

/app/rss.xml/route.ts
```ts
export async function GET(request: Request) {
  const rssResponse = await fetch(/* rss endpoint */)
  const rssData = await rssResponse.json()
  const rssFeed = `<?xml version="1.0" encoding="UTF-8" ?>
    <rss version="2.0">
    <channel>
      <title>${rssData.title}</title>
      <description>${rssData.description}</description>
      <link>${rssData.link}</link>
      <copyright>${rssData.copyright}</copyright>
      ${rssData.items.map((item) => {
        return `<item>
          <title>${item.title}</title>
          <description>${item.description}</description>
          <link>${item.link}</link>
          <pubData>${item.publishDate}</pubData>
          <guid isPermaLink="false">${item.guid}</guid>
        </item>`
      })}
    </channel>
  </rss>`

  const headers = new Headers({ 'content-type': 'application/xml' })

  return new Response(rssFeed, { headers })
}
```

마크업을 생성하는 데 사용된 모든 입력을 삭제합니다.

### Content negotiation
요청의 `Accept` 헤더를 기반으로 동일한 URL에서 다양한 콘텐츠 유형을 제공하기 위해 헤더 일치와 함께 [다시 쓰기](https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites)를 사용할 수 있습니다. 이를 [콘텐츠 협상](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation)이라고 합니다.

예를 들어 문서 사이트는 브라우저에 HTML 페이지를 제공하고 동일한 `/docs/…` URL의 AI 에이전트에 원시 마크다운을 제공할 수 있습니다.

1. `Accept` 헤더와 일치하는 재작성 구성:
.next.config.js
```js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/docs/:slug*',
        destination: '/docs/md/:slug*',
        has: [
          {
            type: 'header',
            key: 'accept',
            value: '(.*)text/markdown(.*)',
          }
        ]
      }
    ]
  }
}
```

`/docs/getting-started`에 대한 요청에 `Accept: text/markdown`이 포함된 경우 재작성은 이를 `/docs/md/getting-started`로 라우팅합니다. 해당 경로의 경로 처리기는 Markdown 응답을 반환합니다. `Accept` 헤더에 `text/markdown`을 보내지 않는 클라이언트는 계속해서 일반 HTML 페이지를 받습니다.

2. Markdown 응답을 위한 경로 처리기 만들기:
app/docs/md/[...slug]/route.ts
```ts
import { getDocsMd, generateDocsStaticParams } from '@/lib/docs'

export async function generateStaticParams() {
  return generateDocsStaticParams()
}

export async function GET(_: Request, ctx: RouteContext<'/docs/md/[...slug]'>) {
  const { slug } = await ctx.params
  const mdDoc = await getDocsMd({ slug })

  if (mdDoc == null) {
    return new Response(null, { status: 404 })
  }

  return new Response(mdDoc, {
    headers: {
      'Content-Type': 'text/markdown; charset=utf-8',
      Vary: 'Accept',
    },
  })
}
```

[Vary: Accept](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary) 응답 헤더는 응답 본문이 `Accept` 요청 헤더에 따라 달라짐을 캐시에 알려줍니다. 이것이 없으면 공유 캐시는 캐시된 Markdown 응답을 브라우저에 제공할 수 있습니다(또는 그 반대). 대부분의 호스팅 공급자는 이미 캐시 키에 `Accept` 헤더를 포함하고 있지만 `Vary`를 명시적으로 설정하면 모든 CDN 및 프록시 캐시에서 올바른 동작이 보장됩니다.

`generateStaticParams`를 사용하면 빌드 시 Markdown 변형을 사전 렌더링할 수 있으므로 모든 요청에서 원본 서버에 도달하지 않고도 에지에서 제공될 수 있습니다.

3. `curl`로 테스트해보세요:
```bash
# Returns Markdown
curl -H "Accept: text/markdown" https://example.com/docs/getting-started

# Returns the normal HTML page
curl https://example.com/docs/getting-started
```

> Good to know:
> - `/docs/md/...` 경로는 다시 작성하지 않고도 계속해서 직접 액세스할 수 있습니다. 재작성을 통해서만 제공되도록 제한하려면 [proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)를 사용하여 예상되는 `Accept` 헤더를 포함하지 않는 직접 요청을 차단하세요.
> - 고급 협상 논리의 경우 재작성 대신 [proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)를 사용하여 유연성을 높일 수 있습니다.

### 요청 페이로드 사용
`.json()`, `.formData()` 또는 `.text()`와 같은 요청 [인스턴스 메서드](https://developer.mozilla.org/en-US/docs/Web/API/Request#instance_methods)를 사용하여 요청 본문에 액세스합니다.

`GET` 및 `HEAD` 요청은 본문을 전달하지 않습니다.

/app/api/echo-body/route.ts
```ts
export async function POST(request: Request) {
  const res = await request.json()
  return Response.json({ res })
}
```

> 알아두면 좋은 점: 데이터를 다른 시스템에 전달하기 전에 데이터 유효성을 검사하세요.

/app/api/send-mail/route.ts
```ts
import { sendMail, validateInputs } from '@/lib/email-transporter'
 
export async function POST(request: Request) {
  const formData = await request.formData()
  const email = formData.get('email')
  const contents = formData.get('contents')
 
  try {
    await validateInputs({ email, contents })
    const info = await sendMail({ email, contents })
 
    return Response.json({ messageId: info.messageId })
  } catch (reason) {
    const message =
      reason instanceof Error ? reason.message : 'Unexpected exception'
 
    return new Response(message, { status: 500 })
  }
}
```

요청 본문은 한 번만 읽을 수 있습니다. 요청을 다시 읽어야 하는 경우 요청을 복제하세요:

/app/api/clone/route.ts
```ts
export async function POST(request: Request) {
  try {
    const clonedRequest = request.clone()
 
    await request.text()
    await clonedRequest.text()
    await request.text() // Throws error
 
    return new Response(null, { status: 204 })
  } catch {
    return new Response(null, { status: 500 })
  }
}
```

## Manipulating data
Route Handlers can transform, filter, and aggregate data from one or more sources. This keeps logic out of the frontend and avoids exposing internal systems.

You can also offload heavy computations to the server and reduce client battery and data usage.

```ts
import { parseWeatherData } from '@/lib/weather'
 
export async function POST(request: Request) {
  const body = await request.json()
  const searchParams = new URLSearchParams({ lat: body.lat, lng: body.lng })
 
  try {
    const weatherResponse = await fetch(`${weatherEndpoint}?${searchParams}`)
 
    if (!weatherResponse.ok) {
      /* handle error */
    }
 
    const weatherData = await weatherResponse.text()
    const payload = parseWeatherData.asJSON(weatherData)
 
    return new Response(payload, { status: 200 })
  } catch (reason) {
    const message =
      reason instanceof Error ? reason.message : 'Unexpected exception'
 
    return new Response(message, { status: 500 })
  }
}
```

> 알아두면 좋은 점: 이 예에서는 `POST`를 사용하여 URL에 지리적 위치 데이터를 입력하지 않습니다. `GET` 요청은 캐시되거나 기록될 수 있으며 이로 인해 민감한 정보가 노출될 수 있습니다.

## Proxying to a backend
경로 핸들러를 다른 백엔드에 대한 `proxy`로 사용할 수 있습니다. 요청을 전달하기 전에 유효성 검사 논리를 추가하세요.

/app/api/[...slug]/route.ts
```ts
import { isValidRequest } from '@/lib/utils'

export async function POST(request: Request, { params }) {
  const clonedRequest = request.clone()
  const isValid = await isValidRequest(clonedRequest)

  if (!isValid) {
    return new Response(null, { status: 400, statusText: 'Bad Request' })
  }

  const { slug } = await params;
  const pathname = slug.join('/')
  const proxyURL = new URL(pathname, 'https://nextjs.org')
  const proxyRequest = new Request(proxyURL, request)

  try {
    return fetch(proxyRequest)
  } catch (reason) {
    const message = reason instanceof Error ? reason.message : 'Unexpected error'
    return new Response(message, { status: 500 })
  }
}
```

또는 :
- `proxy` [rewrites](https://nextjs.org/docs/app/guides/backend-for-frontend#proxy)
- `next.config.js`의 [`rewrites`](https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites)

## NextRequest and NextResponse
Next.js는 일반적인 작업을 단순화하는 방법으로 [Request](https://developer.mozilla.org/en-US/docs/Web/API/Request) 및 [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) 웹 API를 확장합니다. 이러한 확장은 경로 처리기와 프록시 모두에서 사용할 수 있습니다.

둘 다 쿠키를 읽고 조작하는 방법을 제공합니다.

`NextRequest`에는 들어오는 요청에서 구문 분석된 값을 노출하는 [nextUrl](https://nextjs.org/docs/app/api-reference/functions/next-request#nexturl) 속성이 포함되어 있습니다. 예를 들어 요청 경로 이름 및 검색 매개변수에 더 쉽게 액세스할 수 있습니다.

`NextResponse`는 `next()`, `json()`, `redirect()` 및 `rewrite()`와 같은 도우미를 제공합니다.

`요청`이 필요한 모든 함수에 `NextRequest`를 전달할 수 있습니다. 마찬가지로 `응답`이 예상되는 경우 `NextResponse`를 반환할 수 있습니다.

/app/echo-pathname/route.ts
```ts
import { type NextRequest, NextResponse } from 'next/server'
 
export async function GET(request: NextRequest) {
  const nextUrl = request.nextUrl
 
  if (nextUrl.searchParams.get('redirect')) {
    return NextResponse.redirect(new URL('/', request.url))
  }
 
  if (nextUrl.searchParams.get('rewrite')) {
    return NextResponse.rewrite(new URL('/', request.url))
  }
 
  return NextResponse.json({ pathname: nextUrl.pathname })
}
```

[NextRequest](https://nextjs.org/docs/app/api-reference/functions/next-request) 및 [NextResponse](https://nextjs.org/docs/app/api-reference/functions/next-response)에 대해 자세히 알아보세요.

## Webhooks and callback URLs
경로 처리기를 사용하여 타사 애플리케이션으로부터 이벤트 알림을 받습니다.

예를 들어 CMS에서 콘텐츠가 변경되면 경로의 유효성을 다시 검사합니다. 변경 시 특정 끝점을 호출하도록 CMS를 구성합니다.

/app/webhook/route.ts
```ts
import { type NextRequest, NextResponse } from 'next/server'
 
export async function GET(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('token')
 
  if (token !== process.env.REVALIDATE_SECRET_TOKEN) {
    return NextResponse.json({ success: false }, { status: 401 })
  }
 
  const tag = request.nextUrl.searchParams.get('tag')
 
  if (!tag) {
    return NextResponse.json({ success: false }, { status: 400 })
  }
 
  revalidateTag(tag)
 
  return NextResponse.json({ success: true })
}
```

콜백 URL은 또 다른 사용 사례입니다. 사용자가 제3자 흐름을 완료하면 제3자는 이를 콜백 URL로 보냅니다. 경로 처리기를 사용하여 응답을 확인하고 사용자를 리디렉션할 위치를 결정합니다.

/app/auth/callback/route.ts
```ts
import { type NextRequest, NextResponse } from 'next/server'
 
export async function GET(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('session_token')
  const redirectUrl = request.nextUrl.searchParams.get('redirect_url')
 
  const destination = new URL(redirectUrl ?? '/', request.url)
  // 공개 리디렉션 방지: 동일한 출처의 대상만 허용
  if (destination.origin !== request.nextUrl.origin) {
    return new Response('Invalid redirect', { status: 400 })
  }
 
  const response = NextResponse.redirect(destination)
 
  response.cookies.set({
    value: token,
    name: '_token',
    path: '/',
    secure: true,
    httpOnly: true,
    expires: undefined, // session cookie
  })
 
  return response
}
```

## Redirects
app/api/route.ts
```ts
import { redirect } from 'next/navigation'
 
export async function GET(request: Request) {
  redirect('https://nextjs.org/')
}
```

[`redirect`](https://nextjs.org/docs/app/api-reference/functions/redirect) 및 [`permanentRedirect`](https://nextjs.org/docs/app/api-reference/functions/permanentRedirect)의 리디렉션에 대해 자세히 알아보세요.

## Proxy
프로젝트당 하나의 `proxy` 파일만 허용됩니다. 특정 경로를 대상으로 하려면 `config.matcher`를 사용하세요. [proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)에 대해 자세히 알아보세요.

요청이 경로 경로에 도달하기 전에 `proxy`를 사용하여 응답을 생성합니다.

proxy.ts
```ts
import { isAuthenticated } from '@lib/auth'
 
export const config = {
  matcher: '/api/:function*',
}
 
export function proxy(request: Request) {
  if (!isAuthenticated(request)) {
    return Response.json(
      { success: false, message: 'authentication failed' },
      { status: 401 }
    )
  }
}
```

프록시를 사용하여 요청을 프록시할 수도 있습니다:

proxy.ts
```ts
import { NextResponse } from 'next/server'
 
export function proxy(request: Request) {
  if (request.nextUrl.pathname === '/proxy-this-path') {
    const rewriteUrl = new URL('https://nextjs.org')
    return NextResponse.rewrite(rewriteUrl)
  }
}
```

생성할 수 있는 또 다른 유형의 응답 프록시는 리디렉션입니다:

proxy.ts
```ts
import { NextResponse } from 'next/server'
 
export function proxy(request: Request) {
  if (request.nextUrl.pathname === '/v1/docs') {
    request.nextUrl.pathname = '/v2/docs'
    return NextResponse.redirect(request.nextUrl)
  }
}
```

## Security
### Working with headers
헤더가 어디로 갈지 신중하게 생각하고 들어오는 요청 헤더를 나가는 응답으로 직접 전달하지 마세요.

- 업스트림 요청 헤더: 프록시에서 `NextResponse.next({ request: { headers } })`는 서버가 수신하는 헤더를 수정하고 이를 클라이언트에 노출하지 않습니다.
- 응답 헤더: `new Response(..., { headers })`, `NextResponse.json(..., { headers })`, `NextResponse.next({ headers })` 또는 `response.headers.set(...)`는 헤더를 클라이언트에 다시 보냅니다. 이러한 헤더에 중요한 값이 추가된 경우 클라이언트에 표시됩니다.

[proxy의 NextResponse 헤더](https://nextjs.org/docs/app/api-reference/functions/next-response#next)에서 자세히 알아보세요.

### Rate limiting
Next.js 백엔드에서 속도 제한을 구현할 수 있습니다. 코드 기반 검사 외에도 호스트에서 제공하는 속도 제한 기능을 활성화하십시오.

/app/resource/route.ts
```ts
import { NextResponse } from 'next/server'
import { checkRateLimit } from '@/lib/rate-limit'
 
export async function POST(request: Request) {
  const { rateLimited } = await checkRateLimit(request)
 
  if (rateLimited) {
    return NextResponse.json({ error: 'Rate limit exceeded' }, { status: 429 })
  }
 
  return new Response(null, { status: 204 })
}
```

### Verify payloads
들어오는 요청 데이터를 절대 신뢰하지 마십시오. 콘텐츠 유형과 크기를 검증하고 사용하기 전에 XSS를 기준으로 정리합니다.

남용을 방지하고 서버 리소스를 보호하려면 시간 초과를 사용하십시오.

사용자가 생성한 정적 자산을 전용 서비스에 저장합니다. 가능하다면 브라우저에서 업로드하고 반환된 URI를 데이터베이스에 저장하여 요청 크기를 줄이세요.

### Access to protected resources
액세스 권한을 부여하기 전에 항상 자격 증명을 확인하세요. 인증 및 권한 부여를 위해 프록시에만 의존하지 마십시오.

응답 및 백엔드 로그에서 민감하거나 불필요한 데이터를 제거합니다.

자격 증명과 API 키를 정기적으로 교체하세요.

## Preflight Requests
실행 전 요청은 `OPTIONS` 메서드를 사용하여 원본, 메서드 및 헤더를 기반으로 요청이 허용되는지 서버에 묻습니다.

`OPTIONS`가 정의되지 않은 경우 Next.js는 이를 자동으로 추가하고 정의된 다른 메서드를 기반으로 `Allow` 헤더를 설정합니다.
- [CORS](https://nextjs.org/docs/app/api-reference/file-conventions/route#cors)

## Library patterns
커뮤니티 라이브러리는 종종 경로 핸들러에 대한 팩토리 패턴을 사용합니다.

/app/api/[...path]/route.ts
```ts
import { createHandler } from 'third-party-library'
 
const handler = createHandler({
  /* library-specific options */
})
 
export const GET = handler
// or
export { handler as POST }
```

그러면 `GET` 및 `POST` 요청에 대한 공유 핸들러가 생성됩니다. 라이브러리는 요청의 `method`와 `pathname`을 기반으로 동작을 사용자 정의합니다.

라이브러리는 `proxy` 팩토리도 제공할 수 있습니다.

proxy.ts
```ts
import { createMiddleware } from 'third-party-library'
 
export default createMiddleware()
```

> 알아두면 좋은 점: 타사 라이브러리에서는 여전히 프록시를 미들웨어로 참조할 수 있습니다.

## More examples
[라우터 핸들러](https://nextjs.org/docs/app/api-reference/file-conventions/route#examples) 및 [`proxy`](https://nextjs.org/docs/app/api-reference/file-conventions/proxy#examples) API 참조 사용에 대한 추가 예제를 확인하세요.

이러한 예에는 [Cookies](https://nextjs.org/docs/app/api-reference/file-conventions/route#cookies), [Headers](https://nextjs.org/docs/app/api-reference/file-conventions/route#headers), [Streaming](https://nextjs.org/docs/app/api-reference/file-conventions/route#streaming), Proxy [negative matching](https://nextjs.org/docs/app/api-reference/file-conventions/proxy#negative-matching) 및 기타 유용한 코드 조각 작업이 포함됩니다.

## Caveats
### Server Components
경로 처리기를 통하지 않고 소스에서 직접 서버 컴포넌트의 데이터를 가져옵니다.

빌드 시 사전 렌더링된 서버 컴포넌트의 경우 경로 처리기를 사용하면 빌드 단계가 실패합니다. 이는 구축하는 동안 이러한 요청을 수신하는 서버가 없기 때문입니다.

요청 시 렌더링되는 서버 컴포넌트의 경우 핸들러와 렌더링 프로세스 간의 추가 HTTP 왕복으로 인해 경로 핸들러에서 가져오는 속도가 느려집니다.

> 서버 측 `fetch` 요청은 절대 URL을 사용합니다. 이는 외부 서버로의 HTTP 왕복을 의미합니다. 개발 중에는 자체 개발 서버가 외부 서버 역할을 합니다. 빌드 시에는 서버가 없으며 런타임 시 공개 도메인을 통해 서버를 사용할 수 있습니다.

서버 컴포넌트는 대부분의 데이터 가져오기 요구 사항을 충족합니다. 그러나 클라이언트측에서 데이터를 가져오는 작업이 필요할 수 있습니다:
- 클라이언트 전용 Web API에 의존하는 데이터:
  - Geo-location API
  - Storage API
  - Audio API
  - File API
- 자주 폴링되는 데이터

이를 위해서는 [`swr`](https://swr.vercel.app/) 또는 [`react-query`](https://tanstack.com/query/latest/docs/framework/react/overview)와 같은 커뮤니티 라이브러리를 사용하세요.

### Server Actions
[Server Action](https://nextjs.org/docs/app/guides/server-actions)을 사용하면 클라이언트에서 서버측 코드를 실행할 수 있습니다. 주요 목적은 프런트엔드 클라이언트의 데이터를 변경하는 것입니다.

Server Action이 대기열에 추가되었습니다. 데이터를 가져오는 데 이를 사용하면 순차적 실행이 발생합니다.

### `export` mode
`export` 모드는 런타임 서버 없이 정적 사이트를 출력합니다. Next.js 런타임이 필요한 기능은 [지원되지 않습니다](https://nextjs.org/docs/app/guides/static-exports#unsupported-features) 이 모드는 정적 사이트를 생성하고 런타임 서버를 생성하지 않기 때문입니다.

`export` 모드에서는 `'force-static'`으로 설정된 [`dynamic`](https://nextjs.org/docs/app/guides/caching-without-cache-components#dynamic) 경로 세그먼트 구성과 함께 GET 경로 핸들러만 지원됩니다.

이는 정적 HTML, JSON, TXT 또는 기타 파일을 생성하는 데 사용할 수 있습니다.

app/hello-world/route.ts
```ts
export const dynamic = 'force-static'
 
export function GET() {
  return new Response('Hello World', { status: 200 })
}
```

### Deployment environment
일부 호스트는 경로 처리기를 람다 함수로 배포합니다. 이는 다음을 의미합니다:
- 경로 처리기는 요청 간에 데이터를 공유할 수 없습니다.
- 환경이 파일 시스템에 쓰기를 지원하지 않을 수 있습니다.
- 시간 초과로 인해 장기 실행 핸들러가 종료될 수 있습니다.
- 시간 초과 시 또는 응답이 생성된 후에 연결이 닫히기 때문에 WebSocket이 작동하지 않습니다.