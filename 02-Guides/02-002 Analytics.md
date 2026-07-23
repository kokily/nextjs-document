# Next.js 애플리케이션에 분석을 추가하는 방법
<span style="color: #808080;">Last updated May 13, 2025</span>

Next.js에는 성능 지표 측정 및 보고 기능이 내장되어 있습니다. [useReportWebVitals](https://nextjs.org/docs/app/api-reference/functions/use-report-web-vitals) 훅을 사용하여 직접 보고를 관리할 수도 있고, Vercel이 자동으로 측정항목을 수집하고 시각화하는 [관리형 서비스](https://vercel.com/analytics?utm_source=next-site&utm_medium=docs&utm_campaign=next-website)를 제공할 수도 있습니다.

## 클라이언트 계측
고급 분석 및 모니터링 요구 사항을 위해 Next.js는 애플리케이션의 프런트엔드 코드 실행이 시작되기 전에 실행되는 `instrumentation-client.js|ts` 파일을 제공합니다. 이는 글로벌 분석, 오류 추적 또는 성능 모니터링 도구를 설정하는 데 이상적입니다.

이를 사용하려면 애플리케이션의 루트 디렉터리에 `instrumentation-client.js` 또는 `instrumentation-client.ts` 파일을 만드세요:

instrumentation-client.js
```js
// 앱이 시작되기 전에 분석 초기화
console.log('Analytics initialized')

// 전역 오류 추적 설정
window.addEventListener('error', (event) => {
  // 오류 추적 서비스로 보내기
  reportError(event.error)
})
```

## Build Your Own

app/_components/web-vitals.js
```jsx
'use client'

import { useReportWebVitals } from 'next/web-vitals'

export function WebVitals() {
  useReportWebVitals((metric) => {
    console.log(metric)
  })
}
```

app/layout.js
```jsx
import { WebVitals } from './_components/web-vitals'

export default function Layout({ children }) {
  return (
    <html>
      <body>
        <WebVitals />
        {children}
      </body>
    </html>
  )
}
```

> `useReportWebVitals` 훅에는 `use client` 지시어가 필요하므로 가장 성능이 좋은 접근 방식은 루트 레이아웃이 가져오는 별도의 구성 요소를 만드는 것입니다. 이는 클라이언트 경계를 `WebVitals` 컴포넌트로만 제한합니다.

자세한 내용은 [API 참조](/docs/app/api-reference/functions/use-report-web-vitals)를 확인하세요.

## Web Vitals

[Web Vitals](https://web.dev/vitals/)는 웹페이지의 사용자 경험을 포착하는 것을 목표로 하는 유용한 측정항목 집합입니다. 다음 웹 바이탈이 모두 포함되어 있습니다:

* [Time to First Byte](https://developer.mozilla.org/docs/Glossary/Time_to_first_byte) (TTFB)
* [First Contentful Paint](https://developer.mozilla.org/docs/Glossary/First_contentful_paint) (FCP)
* [Largest Contentful Paint](https://web.dev/lcp/) (LCP)
* [First Input Delay](https://web.dev/fid/) (FID)
* [Cumulative Layout Shift](https://web.dev/cls/) (CLS)
* [Interaction to Next Paint](https://web.dev/inp/) (INP)

`name` 속성을 사용하여 이러한 측정항목의 모든 결과를 처리할 수 있습니다.

app/_components/web-vitals.tsx
```tsx
'use client'

import { useReportWebVitals } from 'next/web-vitals'

export function WebVitals() {
  useReportWebVitals((metric) => {
    switch (metric.name) {
      case 'FCP': {
        // handle FCP results
      }
      case 'LCP': {
        // handle LCP results
      }
      // ...
    }
  })
}
```

app/_components/web-vitals.js
```jsx
'use client'

import { useReportWebVitals } from 'next/web-vitals'

export function WebVitals() {
  useReportWebVitals((metric) => {
    switch (metric.name) {
      case 'FCP': {
        // handle FCP results
      }
      case 'LCP': {
        // handle LCP results
      }
      // ...
    }
  })
}
```

## Sending results to external systems

측정하고 추적하기 위해 결과를 모든 엔드포인트로 보낼 수 있습니다.
사이트의 실제 사용자 성과. 예를 들어:

```js
useReportWebVitals((metric) => {
  const body = JSON.stringify(metric)
  const url = 'https://example.com/analytics'

  // 가능하다면 `navigator.sendBeacon()`을 사용하고 `fetch()`로 대체합니다.
  if (navigator.sendBeacon) {
    navigator.sendBeacon(url, body)
  } else {
    fetch(url, { body, method: 'POST', keepalive: true })
  }
})
```

> **알아두면 좋은 점**: [Google Analytics](https://analytics.google.com/analytics/web/)를 사용하는 경우 `id` 값을 사용하면 측정항목 분포를 수동으로 구성(백분위수 계산 등)할 수 있습니다.

```js
useReportWebVitals((metric) => {
  // Use `window.gtag` if you initialized Google Analytics as this example:
  // https://github.com/vercel/next.js/blob/canary/examples/with-google-analytics
  window.gtag('event', metric.name, {
    value: Math.round(
      metric.name === 'CLS' ? metric.value * 1000 : metric.value
    ), // values must be integers
    event_label: metric.id, // id unique to current page load
    non_interaction: true, // avoids affecting bounce rate.
  })
})
```

> [Google Analytics로 결과 전송](https://github.com/GoogleChrome/web-vitals#send-the-results-to-google-analytics)에 대해 자세히 알아보세요.
