# Internationalization
<span style="color: #808080;">Last updated December 9, 2025</span>

Next.js를 사용하면 여러 언어를 지원하도록 콘텐츠의 라우팅 및 렌더링을 구성할 수 있습니다. 사이트를 다양한 로케일에 맞게 조정하려면 번역된 콘텐츠(현지화) 및 국제화된 경로가 포함됩니다.

## Terminology
* **Locale:** 언어 및 형식 지정 기본 설정 집합에 대한 식별자입니다. 여기에는 일반적으로 사용자가 선호하는 언어와 해당 지역이 포함됩니다.
  * `en-US`: 미국에서 사용되는 영어
  * `nl-NL`: 네덜란드에서 사용되는 네덜란드어
  * `nl`: 네덜란드어, 특정 지역 없음

## Routing Overview
사용할 로케일을 선택하려면 브라우저에서 사용자의 언어 기본 설정을 사용하는 것이 좋습니다. 기본 언어를 변경하면 애플리케이션에 수신되는 `Accept-Language` 헤더가 수정됩니다.

예를 들어, 다음 라이브러리를 사용하면 수신되는 `Request`을 보고 `Headers`, 지원하려는 로케일 및 기본 로케일을 기반으로 선택할 로케일을 결정할 수 있습니다.

proxy.js
```js
import { match } from '@formatjs/intl-localematcher'
import Negotiator from 'negotiator'

let headers = { 'accept-language': 'en-US,en;q=0.5' }
let languages = new Negotiator({ headers }).languages()
let locales = ['en-US', 'nl-NL', 'nl']
let defaultLocale = 'en-US'

match(languages, locales, defaultLocale) // -> 'en-US'
```

라우팅은 하위 경로(`/fr/products`) 또는 도메인(`my-site.fr/products`)을 기준으로 국제화될 수 있습니다. 이 정보를 사용하면 이제 [프록시](/docs/app/api-reference/file-conventions/proxy) 내부의 로케일을 기반으로 사용자를 리디렉션할 수 있습니다.

proxy.js
```js
import { NextResponse } from "next/server";

let locales = ['en-US', 'nl-NL', 'nl']

// 위와 유사하거나 라이브러리를 사용하여 선호하는 로케일을 가져옵니다.
function getLocale(request) { ... }

export function proxy(request) {
  // 경로 이름에 지원되는 로캘이 있는지 확인하세요.
  const { pathname } = request.nextUrl
  const pathnameHasLocale = locales.some(
    (locale) => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  )

  if (pathnameHasLocale) return

  // 로케일이 없으면 리디렉션
  const locale = getLocale(request)
  request.nextUrl.pathname = `/${locale}${pathname}`
  // 예를 들어 들어오는 요청은 /products입니다.
  // 이제 새 URL은 /en-US/products입니다.
  return NextResponse.redirect(request.nextUrl)
}

export const config = {
  matcher: [
    // 모든 내부 경로 건너뛰기(_next)
    '/((?!_next).*)',
    // 선택사항: 루트(/) URL에서만 실행
    // '/'
  ],
}
```

마지막으로 `app/` 내부의 모든 특수 파일이 `app/[lang]` 아래에 중첩되어 있는지 확인하세요. 이를 통해 Next.js 라우터는 경로의 다양한 로케일을 동적으로 처리하고 `lang` 매개변수를 모든 레이아웃과 페이지에 전달할 수 있습니다. 예를 들어:

app/[lang]/page.tsx
```tsx
// 이제 현재 로캘에 액세스할 수 있습니다.
// 예를 들어 /en-US/products -> `lang`은 "en-US"입니다.
export default async function Page({ params }: PageProps<'/[lang]'>) {
  const { lang } = await params
  return ...
}
```

> **알아두면 좋은 점:** `PageProps` 및 `LayoutProps`는 경로 매개변수에 대한 강력한 유형 지정을 제공하는 전역적으로 사용 가능한 TypeScript 도우미입니다. 자세한 내용은 [PageProps](/docs/app/api-reference/file-conventions/page#page-props-helper) 및 [LayoutProps](/docs/app/api-reference/file-conventions/layout#layout-props-helper)를 참조하세요.

루트 레이아웃은 새 폴더(예: `app/[lang]/layout.js`)에 중첩될 수도 있습니다.

## Localization
사용자가 선호하는 로케일 또는 현지화를 기반으로 표시되는 콘텐츠를 변경하는 것은 Next.js에만 국한된 것이 아닙니다. 아래 설명된 패턴은 모든 웹 애플리케이션에서 동일하게 작동합니다.

애플리케이션 내에서 영어와 네덜란드어 콘텐츠를 모두 지원한다고 가정해 보겠습니다. 우리는 일부 키에서 현지화된 문자열로의 매핑을 제공하는 객체인 두 개의 서로 다른 "사전"을 유지할 수 있습니다. 예를 들어:

dictionaries/en.json
```json
{
  "products": {
    "cart": "Add to Cart"
  }
}
```

dictionaries/nl.json
```json
{
  "products": {
    "cart": "Toevoegen aan Winkelwagen"
  }
}
```

그런 다음 `getDictionary` 함수를 생성하여 요청된 로케일에 대한 번역을 로드할 수 있습니다:

app/[lang]/dictionaries.ts
```ts
import 'server-only'

const dictionaries = {
  en: () => import('./dictionaries/en.json').then((module) => module.default),
  nl: () => import('./dictionaries/nl.json').then((module) => module.default),
}

export type Locale = keyof typeof dictionaries

export const hasLocale = (locale: string): locale is Locale =>
  locale in dictionaries

export const getDictionary = async (locale: Locale) => dictionaries[locale]()
```

현재 선택된 언어가 주어지면 레이아웃이나 페이지 내부에서 사전을 가져올 수 있습니다.

`lang`은 `string`으로 입력되므로 `hasLocale`을 사용하면 지원되는 로케일로 유형이 좁아집니다. 또한 번역이 누락된 경우 런타임 오류가 아닌 404가 반환되도록 보장합니다.

app/[lang]/page.tsx
```tsx
import { notFound } from 'next/navigation'
import { getDictionary, hasLocale } from './dictionaries'

export default async function Page({ params }: PageProps<'/[lang]'>) {
  const { lang } = await params

  if (!hasLocale(lang)) notFound()

  const dict = await getDictionary(lang)
  return <button>{dict.products.cart}</button> // 장바구니에 추가
}
```

`app/` 디렉토리의 모든 레이아웃과 페이지는 기본적으로 [Server Components](/docs/app/getting-started/server-and-client-comComponents)로 설정되어 있으므로 클라이언트측 JavaScript 번들 크기에 영향을 미치는 번역 파일의 크기에 대해 걱정할 필요가 없습니다. 이 코드는 **서버에서만 실행**되며 결과 HTML만 브라우저로 전송됩니다.

## Static Rendering
특정 로케일 세트에 대한 정적 경로를 생성하려면 페이지나 레이아웃에 `generateStaticParams`를 사용할 수 있습니다. 예를 들어 루트 레이아웃에서는 전역적일 수 있습니다:

app/[lang]/layout.tsx
```tsx
export async function generateStaticParams() {
  return [{ lang: 'en-US' }, { lang: 'de' }]
}

export default async function RootLayout({
  children,
  params,
}: LayoutProps<'/[lang]'>) {
  return (
    <html lang={(await params).lang}>
      <body>{children}</body>
    </html>
  )
}
```

## Resources
* [Minimal i18n routing and translations](https://github.com/vercel/next.js/tree/canary/examples/i18n-routing)
* [`next-intl`](https://next-intl.dev)
* [`next-international`](https://github.com/QuiiBz/next-international)
* [`next-i18n-router`](https://github.com/i18nexus/next-i18n-router)
* [`paraglide-next`](https://inlang.com/m/osslbuzt/paraglide-next-i18n)
* [`lingui`](https://lingui.dev)
* [`tolgee`](https://tolgee.io/apps-integrations/next)
* [`next-intlayer`](https://intlayer.org/doc/environment/nextjs)
* [`gt-next`](https://generaltranslation.com/en/docs/next)