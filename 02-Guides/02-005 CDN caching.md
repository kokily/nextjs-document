# Using a CDN with Next.js
<span style="color: #808080;">Last updated June 23, 2026</span>

Next.js는 CDN이 에지에서 응답을 캐시하는 데 사용할 수 있는 표준 `Cache-Control` 헤더를 설정합니다. 이 페이지에서는 CDN 캐싱이 어려운 현재 작동하는 작업과 사용자 정의 헤더 종속성을 제거하는 방향을 다룹니다.

## What Works Today
### Cache-Control headers
Next.js는 각 경로의 렌더링 전략에 따라 Cache-Control 헤더를 설정합니다:
- 정적 페이지(재검증 없음): `s-maxage=31536000`(1년)
- ISR 페이지(시간 기반 재검증): `s-maxage={revalidate}`, `stale-while-revalidate={expire - revalidate}`. 기본 만료 기간은 1년이므로 기본적으로 `stale-while-revalidate`이 응답 헤더에 포함됩니다. 이것을 캐시라이프([cacheLife](https://nextjs.org/docs/app/api-reference/functions/cacheLife))로 맞춤화할 수 있습니다.
- 동적 페이지(캐싱 없음): `private`, `no-cache`, `no-store`, `max-age=0`, `must-revalidate`

`s-maxage` 및 `stale-while-revalidate`를 준수하는 CDN은 edge에서 정적 페이지와 ISR 페이지를 캐시할 수 있습니다. 그러나 CDN 수준 캐싱만으로는 주문형 재검증([revalidateTag()](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) / [revalidatePath()](https://nextjs.org/docs/app/api-reference/functions/revalidatePath))을 지원하지 않습니다. 이러한 호출은 Next.js 서버 캐시를 무효화하지만 CDN은 `s-maxage` TTL이 만료될 때까지 캐시된 복사본을 계속 제공합니다. 주문형 재검증을 CDN에 전파하려면 재검증 호출과 함께 CDN 제거를 트리거하십시오. 일반적인 패턴은 `revalidateTag()`/`revalidatePath()`를 호출하여 Next.js 서버 캐시를 무효화한 다음 영향을 받는 키(HTML 및 RSC 변형 모두 포함)에 대해 CDN 제거 API를 호출하는 것입니다.

### Static assets
`/_next/static/`에서 제공되는 정적 자산(JavaScript, CSS, 이미지, 글꼴)은 파일 이름에 콘텐츠 해시를 포함하고 1년의 `max-age` 및 `immutable` 지시문을 갖습니다:
`public, max-age=31536000, immutable`

[`assetPrefix`](https://nextjs.org/docs/app/api-reference/config/next-config-js/assetPrefix)를 사용하여 다른 도메인 또는 CDN 출처의 정적 자산을 제공할 수 있습니다.

### Static prefetches (PPR-enabled routes)
경로에 부분 사전 렌더링이 활성화되어 있고 `next-router-prefetch` 헤더가 설정된 경우(정적 프리페치를 나타냄) 응답은 결정적입니다. 즉, 클라이언트의 라우터 상태에 관계없이 동일한 사전 렌더링된 콘텐츠를 반환합니다. `next-router-state-tree` 헤더는 이러한 요청에 대해 구문 분석되지 않으므로 응답에 영향을 미치지 않습니다.

PPR 지원 경로의 경우 CDN은 다음과 같은 경우 정적 프리페치 응답을 캐시할 수 있습니다:

1. 캐시 키에 `_rsc` 검색 매개변수를 포함합니다(프리페치 변형을 HTML 응답과 구별하기 위해).
2. 응답에 설정된 `Cache-Control` 헤더 Next.js를 존중합니다.

> 알아두면 좋은 점: PPR이 없는 경로의 경우 프리패치 요청 중에 `next-router-state-tree` 헤더를 읽어 포함할 세그먼트를 결정합니다. 그러면 현재 라우터 상태를 통과할 때 캐시 `vary`가 늘어납니다. 캐시 컴포넌트가 활성화되면 세그먼트 수준 프리페치가 이미 경로 이름 기반 경로(예: `/page.segments/_tree.segment.rsc`)를 사용하고 CDN은 표준 경로 이름 기반 캐시 키를 사용하여 이를 캐시할 수 있습니다.

## Where CDN Caching Is Challenging
앱 라우터 응답은 여러 사용자 정의 요청 헤더에 따라 달라질 수 있습니다. Next.js는 응답에 `Vary` 헤더를 설정하여 이를 CDN에 알립니다:
- `rsc` — 요청이 HTML 대신 RSC(React Server Components) 페이로드를 반환해야 하는지 여부
- `next-router-state-tree` — 동적 탐색 중 대상 세그먼트 업데이트에 사용되는 클라이언트의 현재 라우터 상태
- `next-router-prefetch` — 프리페치 요청인지 여부
- `next-router-segment-prefetch` — 프리페치되는 특정 세그먼트
- `next-url` — [차단 경로](https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes)를 사용하는 경로에만 추가되며 차단되는 URL을 전달합니다.

> 알아두면 좋은 점: [`proxy.js`](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)(이전 미들웨어)는 CDN 캐시보다 먼저 실행되어야 인증, 리디렉션 및 재작성을 위한 진실의 소스로 유지됩니다. 배포에서 CDN 뒤에 `proxy.js`를 배치하는 경우 `proxy.js` 결정에 의존하는 경로에 대한 캐싱을 우회하도록 캐시 계층을 구성합니다.

많은 CDN은 추가 구성 없이는 `Vary`를 지원하지 않습니다. Next.js는 `_rsc` 검색 매개변수(캐시 키 역할을 하는 관련 요청 헤더 값의 해시)를 사용하여 이 문제를 해결하며, 다양한 응답 변형이 다른 캐시 키를 얻도록 보장합니다. 이렇게 하면 `Vary`를 무시하는 CDN에서도 올바른 응답이 보장됩니다.

## Handling Headers at the CDN
### What you can safely ignore
프로토콜 오류를 일으키지 않고 특정한 경우 이러한 헤더를 생략할 수 있습니다. 서버는 여전히 구문 분석 가능한 응답을 반환하지만 특정 탐색에 대한 대상이 더 크거나 적을 수 있습니다:

`next-router-state-tree`: 프리페치가 아닌 RSC 요청에서 생략되면 서버는 대상 세그먼트 업데이트 대신 전체 페이로드를 반환합니다.

`next-router-segment-prefetch`: 프리페치 요청에서 생략되면 서버는 세그먼트별 페이로드 대신 더 광범위한 프리페치 페이로드로 대체됩니다.

`next-url`: 참조 페이지에 따라 응답을 변경하기 위해 [차단 경로](https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes)에 사용됩니다. 생략하면 서버가 어떤 원래 경로와 일치하는지 알 수 없으므로 차단 경로가 지원되지 않습니다. 반환된 응답은 `next-url`이 생략된 경우 일반 탐색에 대한 것입니다. 사용자는 가로채는 대상 페이지 대신 대상 페이지를 봅니다.

### What you must preserve
`rsc` 헤더는 클라이언트에서 서버로 전달되어야 합니다. 이 헤더는 HTML 대신 RSC 페이로드를 반환하도록 서버에 지시합니다. CDN이 이를 제거하면 클라이언트 측 라우터가 RSC 데이터를 예상할 때 서버가 HTML을 반환합니다. 이로 인해 클라이언트 측 탐색이 중단되고 대신 브라우저 탐색이 발생합니다. `Vary` 헤더와 `_rsc` 매개변수는 특히 CDN이 RSC 요청에 대해 캐시된 HTML 응답을 제공하는 것을 방지하기 위해(또는 그 반대) 존재합니다.

`next-router-prefetch`가 있는 경우 프리페치 헤더와 `_rsc` 검색 매개변수를 모두 유지하세요. 프리페치 흐름의 경우 `_rsc`는 필수 캐시 무효화 판별자이며 필수로 처리되어야 합니다.

`_rsc` 검색 매개변수는 캐시 키에 포함되어야 합니다. 이는 응답 변형(HTML 대 RSC, 다양한 프리페치 유형)을 구별합니다. 일부 CDN은 기본적으로 이를 수행하므로 CDN이 캐시 키에서 쿼리 매개변수를 제거하지 않는지 확인하세요. `experimental.validateRSCRequestHeaders` 옵션이 활성화되어 있고 RSC 요청이 올바른 `_rsc` 값 없이 도착하면 서버는 올바른 해시가 있는 URL에 대한 307 리디렉션으로 응답합니다. CDN은 이 리디렉션을 따라야 합니다. 해시 업스트림을 계산하는 플랫폼은 추가 왕복을 피하기 위해 전달하기 전에 올바른 `_rsc`를 포함하도록 요청을 다시 작성할 수 있습니다.

> 알아두면 좋은 점: 현재는 정적 프리페치 중에도 `next-url`이 `_rsc` 해시에 포함되어 있습니다. 이는 잠재적으로 캐시 누락이 발생하지 않고는 현재 체계에서 이를 안전하게 무시할 수 없음을 의미합니다. 아래에 설명된 경로 이름 기반 방향은 이러한 차이를 해결합니다.

## Direction: Pathname-Based Cache Keying
Next.js 팀은 캐시에 영향을 미치는 모든 입력을 URL 경로 이름으로 이동하여 사용자 정의 헤더에 대한 `Vary`의 필요성을 제거하고 `_rsc` 검색 매개변수를 제거하기 위해 노력하고 있습니다. 이는 위에서 설명한 CDN 캐싱 문제를 해결합니다.

### How it works
이 접근 방식은 현재 이미 사용하고 있는 [`output: 'export'`](https://nextjs.org/docs/app/guides/static-exports) 및 세그먼트 프리페치를 출력하는 라우팅 체계를 확장합니다. 경로 이름의 파일 확장자는 응답 유형을 식별합니다:
- 전체 페이지 RSC: `/my/page.rsc`는 전체 페이지에 대한 RSC 페이로드를 반환합니다.
- 세그먼트 RSC: `/my/page.segments/path/to/segment.segment.rsc`는 특정 세그먼트에 대한 RSC 페이로드를 반환합니다.

이 모델에서는:
- 경로 이름은 캐시 키를 결정합니다. 경로 이름의 모든 내용은 반환되는 응답 변형에 영향을 줍니다.
- 반환된 응답에 영향을 주지 않고 검색 매개변수를 안전하게 삭제할 수 있습니다.
- 표준 HTTP 캐시 헤더(`Cache-Control`, `max-age` 등)는 평소와 같이 존중됩니다.
- CDN에서는 `Vary` 지원이 필요하지 않습니다.

CDN은 경로 이름을 캐시 키로 사용하고 검색 매개변수를 무시하고 표준 `Cache-Control` 헤더를 존중하여 Next.js 응답을 캐시합니다. `Vary`를 이해하거나 사용자 정의 헤더를 검사하거나 에지 로직을 프로그래밍할 필요가 없습니다.

### What changes for interception routes
현재 체계에서는 `next-url`이 `_rsc` 해시에 기여하므로 이를 삭제하면 캐시 누락이 발생합니다. 경로명 기반 체계에서는 차단 가변성이 검색 매개변수(경로명이 아님)에 인코딩됩니다:
- CDN이 검색 매개변수를 유지하면 차단이 올바르게 작동합니다.
- CDN이 검색 매개변수를 삭제하는 경우 차단이 지원되지 않습니다. 가로채지 않는 페이지로 정상적으로 저하되며 클라이언트 측 탐색이 중단되지 않습니다.

이로 인해 차단 경로 지원이 요구사항이 아닌 옵트인 CDN 기능이 됩니다.

### Current status
이 방향은 코드베이스에서 이미 작동 중인 패턴을 확장합니다(세그먼트 프리페치 경로, 출력: `output: 'export'` 모드). 액티브한 디자인이에요.

## CDN Feature Compatibility
모든 주요 CDN(에지 컴퓨팅, 키-값 스토리지, Blob 스토리지, PPR 재개)에서 사용할 수 있는 인프라 기본 요소를 보여주는 전체 표를 보려면 [플랫폼에 배포](https://nextjs.org/docs/app/guides/deploying-to-platforms#cdn-infrastructure-compatibility)를 참조하세요.