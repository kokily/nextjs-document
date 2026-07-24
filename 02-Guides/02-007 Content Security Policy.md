# How to set a Content Security Policy (CSP) for your Next.js application
<span style="color: #808080;">Last updated March 20, 2026</span>

[콘텐츠 보안 정책(CSP)](https://developer.mozilla.org/docs/Web/HTTP/CSP)은 XSS(교차 사이트 스크립팅), 클릭재킹 및 기타 코드 삽입 공격과 같은 다양한 보안 위협으로부터 Next.js 애플리케이션을 보호하는 데 중요합니다.

CSP를 사용하여 개발자는 콘텐츠 소스, 스크립트, 스타일시트, 이미지, 글꼴, 개체, 미디어(오디오, 비디오), iframe 등에 허용되는 원본을 지정할 수 있습니다.

> Examples ([Strict CSP](https://github.com/vercel/next.js/tree/canary/examples/with-strict-csp))

## Nonces
[Nonce](https://developer.mozilla.org/docs/Web/HTML/Global_attributes/nonce)는 일회용으로 생성된 고유하고 임의의 문자열입니다. 엄격한 CSP 지시어를 우회하여 특정 인라인 스크립트나 스타일을 선택적으로 실행할 수 있도록 CSP와 함께 사용됩니다.

### Why use a nonce?
CSP는 공격을 방지하기 위해 인라인 및 외부 스크립트를 모두 차단할 수 있습니다. nonce를 사용하면 특정 스크립트가 일치하는 nonce 값을 포함하는 경우에만 안전하게 실행되도록 허용할 수 있습니다.

공격자가 페이지에 스크립트를 로드하려면 nonce 값을 추측해야 합니다. 그렇기 때문에 Nonce는 모든 요청에 ​​대해 예측할 수 없고 고유해야 합니다.

### Adding a nonce with Proxy
[Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)를 사용하면 페이지가 렌더링되기 전에 헤더를 추가하고 nonce를 생성할 수 있습니다.

페이지를 볼 때마다 새로운 nonce가 생성되어야 합니다. 즉, nonce를 추가하려면 [동적 렌더링](https://nextjs.org/docs/app/glossary#dynamic-rendering)을 사용해야 합니다.

예를 들어:
> 알아두면 좋은 점: 개발 중에는 React가 브라우저에서 서버 측 오류 스택 재구성과 같은 향상된 디버깅 정보를 제공하기 위해 `eval`을 사용하기 때문에 `unsafe-eval`이 필요합니다. `unsafe-eval`은 프로덕션에 필요하지 않습니다. React나 Next.js 모두 기본적으로 프로덕션에서 `eval`을 사용하지 않습니다.

proxy.ts
```ts
import { NextRequest, NextResponse } from 'next/server'
 
export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
  const isDev = process.env.NODE_ENV === 'development'
  const cspHeader = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic'${isDev ? " 'unsafe-eval'" : ''};
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' blob: data:;
    font-src 'self';
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
`
  // 개행 문자 및 공백 바꾸기
  const contentSecurityPolicyHeaderValue = cspHeader
    .replace(/\s{2,}/g, ' ')
    .trim()
 
  const requestHeaders = new Headers(request.headers)
  requestHeaders.set('x-nonce', nonce)
 
  requestHeaders.set(
    'Content-Security-Policy',
    contentSecurityPolicyHeaderValue
  )
 
  const response = NextResponse.next({
    request: {
      headers: requestHeaders,
    },
  })
  response.headers.set(
    'Content-Security-Policy',
    contentSecurityPolicyHeaderValue
  )
 
  return response
}
```

기본적으로 프록시는 모든 요청에 ​​대해 실행됩니다. [`matcher`](https://nextjs.org/docs/app/api-reference/file-conventions/proxy#matcher)를 사용하여 특정 경로에서 실행되도록 프록시를 필터링할 수 있습니다.

CSP 헤더가 필요하지 않은 정적 자산과 일치하는 프리페치(`next/link`)를 무시하는 것이 좋습니다.

proxy.ts
```ts
export const config = {
  matcher: [
    /*
     * 다음으로 시작하는 경로를 제외한 모든 요청 경로와 일치합니.:
     * - api (API routes)
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     */
    {
      source: '/((?!api|_next/static|_next/image|favicon.ico).*)',
      missing: [
        { type: 'header', key: 'next-router-prefetch' },
        { type: 'header', key: 'purpose', value: 'prefetch' },
      ],
    },
  ],
}
```

### How nonces work in Next.js
nonce를 사용하려면 페이지가 동적으로 렌더링되어야 합니다. 이는 Next.js가 요청에 있는 CSP 헤더를 기반으로 서버 측 렌더링 중에 nonce를 적용하기 때문입니다. 요청 또는 응답 헤더가 없는 빌드 시 정적 페이지가 생성되므로 Nonce를 삽입할 수 없습니다.

동적으로 렌더링된 페이지에서 nonce 지원이 작동하는 방식은 다음과 같습니다:
1. 프록시가 nonce를 생성합니다. 프록시는 요청에 대한 고유한 nonce를 생성하고 이를 `Content-Security-Policy` 헤더에 추가하며 사용자 지정 `x-nonce` 헤더에도 설정합니다.
2. Next.js는 nonce를 추출합니다. Next.js는 렌더링하는 동안 Content-Security-Policy 헤더를 구문 분석하고 'nonce-{value}' 패턴을 사용하여 nonce를 추출합니다.
3. Nonce가 자동으로 적용됩니다: Next.js는 nonce를 다음에 첨부합니다:
4. Framework scripts (React, Next.js runtime)
5. Page-specific JavaScript bundles
6. Next.js에서 생성된 인라인 스타일 및 스크립트
7. `nonce` prop을 사용하는 모든 `<Script>` 컴포넌트

이러한 자동 동작으로 인해 각 태그에 nonce를 수동으로 추가할 필요가 없습니다.

### Forcing dynamic rendering
nonce를 사용하는 경우 페이지를 동적 렌더링으로 명시적으로 선택해야 할 수도 있습니다:

app/page.tsx
```tsx
import { connection } from 'next/server'
 
export default async function Page() {
  // 이 페이지를 렌더링하기 위해 들어오는 요청을 기다립니다.
  await connection()
  // 귀하의 페이지 콘텐츠
}
```

### Reading the nonce
[`headers`](https://nextjs.org/docs/app/api-reference/functions/headers)를 사용하여 [서버 컴포넌트](https://nextjs.org/docs/app/getting-started/server-and-client-components)에서 nonce를 읽을 수 있습니다:

app/page.tsx
```tsx
import { headers } from 'next/headers'
import Script from 'next/script'
 
export default async function Page() {
  const nonce = (await headers()).get('x-nonce')
 
  return (
    <Script
      src="https://www.googletagmanager.com/gtag/js"
      strategy="afterInteractive"
      nonce={nonce}
    />
  )
}
```

## Static vs Dynamic Rendering with CSP
Nonce를 사용하면 Next.js 애플리케이션이 렌더링되는 방식에 중요한 영향을 미칩니다:

### Dynamic Rendering Requirement
CSP에서 nonce를 사용하면 모든 페이지가 동적으로 렌더링되어야 합니다. 이는 다음을 의미합니다:

- 페이지는 성공적으로 빌드되지만 동적 렌더링이 제대로 구성되지 않은 경우 런타임 오류가 발생할 수 있습니다.
- 각 요청은 새로운 nonce를 사용하여 새로운 페이지를 생성합니다.
- 정적 최적화 및 증분 정적 재생(ISR)이 비활성화되었습니다.
- 추가 구성 없이는 CDN에서 페이지를 캐시할 수 없습니다.
- PPR(부분 사전 렌더링)은 정적 셸 스크립트가 nonce에 액세스할 수 없으므로 nonce 기반 CSP와 호환되지 않습니다.

### Performance Implications
정적 렌더링에서 동적 렌더링으로의 전환은 성능에 영향을 미칩니다:
- 느린 초기 페이지 로드: 각 요청마다 페이지를 생성해야 합니다.
- 서버 로드 증가: 모든 요청에는 서버 측 렌더링이 필요합니다.
- CDN 캐싱 없음: 기본적으로 동적 페이지를 에지에서 캐시할 수 없습니다.
- 호스팅 비용 증가: 동적 렌더링에 더 많은 서버 리소스 필요

### When to use nonces
다음과 같은 경우 nonce를 고려하세요:
- `unsafe-inline`을 금지하는 엄격한 보안 요구 사항이 있습니다.
- 애플리케이션이 민감한 데이터를 처리합니다.
- 특정 인라인 스크립트는 허용하고 다른 스크립트는 차단해야 합니다.
- 규정 준수 요구 사항은 엄격한 CSP를 요구합니다.

## Without Nonces
nonce가 필요하지 않은 애플리케이션의 경우 [`next.config.js`](https://nextjs.org/docs/app/api-reference/config/next-config-js) 파일에서 직접 CSP 헤더를 설정할 수 있습니다:

next.config.js
```js
const isDev = process.env.NODE_ENV === 'development'
 
const cspHeader = `
    default-src 'self';
    script-src 'self' 'unsafe-inline'${isDev ? " 'unsafe-eval'" : ''};
    style-src 'self' 'unsafe-inline';
    img-src 'self' blob: data:;
    font-src 'self';
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
`
 
module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: cspHeader.replace(/\n/g, ''),
          },
        ],
      },
    ]
  },
}
```

## Subresource Integrity (Experimental)
Nonce의 대안으로 Next.js는 SRI(Subresource Integrity)를 사용하여 해시 기반 CSP에 대한 실험적 지원을 제공합니다. 이 접근 방식을 사용하면 엄격한 CSP를 유지하면서 정적 생성을 유지할 수 있습니다.

> 알아두면 좋은 점: 이 기능은 실험적이며 앱 라우터 애플리케이션에서 사용할 수 있습니다.

### How SRI works
nonce를 사용하는 대신 SRI는 빌드 시 JavaScript 파일의 암호화 해시를 생성합니다. 이러한 해시는 스크립트 태그에 `무결성(integrity)` 속성으로 추가되어 브라우저가 전송 중에 파일이 수정되지 않았는지 확인할 수 있습니다.

### Enabling SRI
`next.config.js`에 실험적인 SRI 구성을 추가하세요:

next.config.js
```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    sri: {
      algorithm: 'sha256', // or 'sha384' or 'sha512'
    },
  },
}
 
module.exports = nextConfig
```

### CSP configuration with SRI
SRI가 활성화되면 기존 CSP 정책을 계속 사용할 수 있습니다. SRI는 자산에 무결성 속성을 추가하여 독립적으로 작동합니다:

> 알아두면 좋은 점: 동적 렌더링 시나리오의 경우 SRI 무결성 특성과 임시 기반 CSP 접근 방식을 결합하여 필요한 경우 프록시를 사용하여 계속 임시를 생성할 수 있습니다.

next.config.js
```js
const isDev = process.env.NODE_ENV === 'development'
 
const cspHeader = `
    default-src 'self';
    script-src 'self'${isDev ? " 'unsafe-eval'" : ''};
    style-src 'self';
    img-src 'self' blob: data:;
    font-src 'self';
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
`
 
module.exports = {
  experimental: {
    sri: {
      algorithm: 'sha256',
    },
  },
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: cspHeader.replace(/\n/g, ''),
          },
        ],
      },
    ]
  },
}
```

### Benefits of SRI over nonces
- 정적 생성: 페이지를 정적으로 생성하고 캐시할 수 있습니다.
- CDN 호환성: 정적 페이지는 CDN 캐싱과 함께 작동합니다.
- 더 나은 성능: 각 요청에 대해 서버 측 렌더링이 필요하지 않습니다.
- 빌드 시간 보안: 빌드 시간에 해시가 생성되어 무결성을 보장합니다.

### Limitations of SRI
- 실험적: 기능이 변경되거나 제거될 수 있음
- 앱 라우터만 해당: 페이지 라우터에서는 지원되지 않습니다.
- 빌드 타임에만 해당: 동적으로 생성된 스크립트를 처리할 수 없습니다.

## Development vs Production Considerations
CSP 구현은 개발 환경과 프로덕션 환경에서 다릅니다:

### Development Environment
개발 중에는 React가 `eval`을 사용하여 서버에서 오류가 발생한 위치를 보여주기 위해 브라우저에서 서버 측 오류 스택을 재구성하는 등 향상된 디버깅 정보를 제공하기 때문에 `unsafe-eval`을 활성화해야 합니다:

proxy.ts
```ts
export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
  const isDev = process.env.NODE_ENV === 'development'
 
  const cspHeader = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic' ${isDev ? "'unsafe-eval'" : ''};
    style-src 'self' ${isDev ? "'unsafe-inline'" : `'nonce-${nonce}'`};
    img-src 'self' blob: data:;
    font-src 'self';
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
`
 
  // Rest of proxy implementation
}
```

### Production Deployment
프로덕션의 일반적인 문제:
- **Nonce가 적용되지 않음**: 필요한 모든 경로에서 프록시가 실행되는지 확인하세요.
- **정적 자산 차단**: CSP가 Next.js 정적 자산을 허용하는지 확인하세요.
- **타사 스크립트**: CSP 정책에 필요한 도메인을 추가합니다.

## Troubleshooting
### Third-party Scripts
CSP에서 타사 스크립트를 사용하는 경우:

app/layout.tsx
```tsx
import { GoogleTagManager } from '@next/third-parties/google'
import { headers } from 'next/headers'
 
export default async function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const nonce = (await headers()).get('x-nonce')
 
  return (
    <html lang="en">
      <body>
        {children}
        <GoogleTagManager gtmId="GTM-XYZ" nonce={nonce} />
      </body>
    </html>
  )
}
```

타사 도메인을 허용하도록 CSP를 업데이트하세요:

proxy.ts
```ts
const cspHeader = `
  default-src 'self';
  script-src 'self' 'nonce-${nonce}' 'strict-dynamic' https://www.googletagmanager.com;
  connect-src 'self' https://www.google-analytics.com;
  img-src 'self' data: https://www.google-analytics.com;
`
```

### Common CSP Violations
1. **inline styles**: nonce를 지원하거나 스타일을 외부 파일로 이동하는 CSS-in-JS 라이브러리를 사용합니다.
2. **Dynamic imports**: script-src 정책에서 동적 가져오기가 허용되는지 확인하세요.
3. **WebAssembly**: WebAssembly를 사용하는 경우 `wasm-unsafe-eval` 추가
4. **Service workers**: 서비스 워커 스크립트에 적합한 정책 추가

## Version History
|Version|Changes|
|---|---|
|`v14.0.0`|해시 기반 CSP에 실험적인 SRI 지원이 추가되었습니다|
|`v13.4.20`|적절한 nonce 처리 및 CSP 헤더 구문 분석에 권장됩니다|
