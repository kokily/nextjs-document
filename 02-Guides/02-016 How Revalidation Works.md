# How revalidation works in Next.js

<span style="color: #808080;">Last updated June 23, 2026</span>

[캐싱](/docs/app/getting-started/caching) 페이지에서는 `use cache`, `cacheTag` 및 `cacheLife` 사용 방법을 다룹니다. 이 페이지에서는 [맞춤 캐시 처리기](/docs/app/api-reference/config/next-config-js/cacheHandlers)를 구현하거나 재검증 동작을 디버그하기 위해 시스템을 이해해야 하는 플랫폼 엔지니어 및 고급 사용자를 위해 **재검증이 내부적으로 작동하는 방식**을 설명합니다.

## The Revalidation Model
Next.js의 대부분의 경로는 요청 시 재검증될 수 있습니다. 여기에는 ISR/사전 렌더링 캐시 항목을 생성하는 앱 라우터 경로와 페이지 라우터 경로가 포함됩니다. 자동으로 정적으로 최적화된(순수한 정적 출력) 페이지 라우터 경로는 요청 시 재검증되지 않습니다. 재배포하지 않고 캐시된 콘텐츠를 업데이트하는 기능은 Next.js [렌더링 모델](/docs/app/guides/rendering-philosophy)의 핵심 부분입니다.

재검증에는 두 가지 유형이 있습니다:
* **Time-based revalidation** 재검증 중 오래된 패턴을 사용합니다. 캐시된 콘텐츠는 즉시 제공되며 콘텐츠의 수명이 [`cacheLife`](/docs/app/api-reference/functions/cacheLife) 또는 `재검증` 기간을 초과하면 백그라운드 재생성이 트리거됩니다. 오래된 콘텐츠는 새로운 콘텐츠가 준비될 때까지 계속 제공됩니다.
* **On-demand revalidation** [`revalidateTag()`](/docs/app/api-reference/functions/revalidateTag) 또는 [`revalidatePath()`](/docs/app/api-reference/functions/revalidatePath)를 호출하여 캐시된 콘텐츠를 명시적으로 무효화합니다. 해당 콘텐츠에 대한 다음 요청은 새로운 렌더링을 트리거합니다.

> **알아두면 좋은 점:** 페이지 라우터 주문형 ISR API(예: `res.revalidate()` 및 `x-prerender-revalidate` 흐름)는 계속 지원되며 서버 캐시 핸들러(`cacheHandler`, 단수형)를 사용합니다. `cacheHandlers` 옵션(복수형)은 `use cache` 지시문을 위한 것입니다.

## What Gets Revalidated
경로가 재검증되면 Next.js는 동일한 React 컴포넌트 트리에서 HTML 응답과 RSC 페이로드(React Server 컴포넌트 페이로드) **모두**를 다시 생성합니다. 두 아티팩트 모두 동일한 캐시 항목에 함께 저장됩니다.

RSC 페이로드는 클라이언트측 탐색에 사용되기 때문에 이러한 일관성이 중요합니다. 브라우저 탐색과 클라이언트측 탐색은 동일한 콘텐츠를 보유해야 합니다.

### What happens if they get out of sync
플랫폼의 캐시가 하나의 렌더링에서 HTML을 제공하고 다른 렌더링에서 RSC 페이로드를 제공하는 경우 사용자는 클라이언트 측 탐색 중에 오래되거나 일치하지 않는 콘텐츠를 볼 수 있습니다. 주요 완화 방법은 동일한 TTL 및 무효화 정책을 사용하여 HTML 및 RSC 응답을 캐시하고 Next.js가 설정하는 [`Vary` 헤더](/docs/app/guides/cdn-caching)를 준수하는 것입니다. 자세한 내용은 [CDN 캐싱](/docs/app/guides/cdn-caching)을 참조하세요.

별개이지만 관련된 문제는 **배포 간 왜곡**입니다. 롤링 배포 중에 배포 A로 구축된 클라이언트는 배포 B를 실행하는 서버로부터 응답을 받을 수 있습니다. [`deploymentId`](/docs/app/api-reference/config/next-config-js/deploymentId)는 이를 완화합니다. 클라이언트가 서버에서 다른 배포 ID를 감지하면 일관된 콘텐츠를 가져오기 위해 하드 탐색을 트리거합니다.

## Tag System Architecture
Next.js는 태그 기반 시스템을 사용하여 무효화해야 하는 캐시된 콘텐츠를 추적합니다. 태그에는 두 가지 유형이 있습니다:

### Explicit tags
명시적 태그는 개발자가 `usecache` 함수 내에서 [`cacheTag()`](/docs/app/api-reference/functions/cacheTag)를 사용하거나 `fetch` 호출에서 `next: {tag: [...] }`를 통해 설정합니다. [`revalidateTag('my-tag')`](/docs/app/api-reference/functions/revalidateTag)가 호출되면 해당 태그가 있는 모든 캐시 항목이 무효화됩니다.

### Soft tags
소프트 태그는 `_N_T_` 접두사가 붙은 경로 경로를 기반으로 Next.js에 의해 자동으로 생성됩니다. 예를 들어 `/blog/hello` 경로는 `_N_T_/layout`, `_N_T_/blog/layout`, `_N_T_/blog/hello/layout` 및 `_N_T_/blog/hello`와 같은 소프트 태그를 생성합니다. 경로의 각 세그먼트에는 레이아웃 태그와 리프 경로 자체가 포함됩니다.

소프트 태그를 사용하면 [`revalidatePath()`](/docs/app/api-reference/functions/revalidatePath)가 동일한 태그 기반 시스템을 통해 작동할 수 있습니다. `revalidatePath('/blog/hello')`가 호출되면 해당 경로의 리프 경로 태그 및 상위 레이아웃 소프트 태그(예: `_N_T_/layout`, `_N_T_/blog/layout`, `_N_T_/blog/hello/layout` 및 `_N_T_/blog/hello`)와 연결된 캐시 항목을 무효화합니다.

[캐시 처리기 API](/docs/app/api-reference/config/next-config-js/cacheHandlers)에서 소프트 태그는 `softTags` 매개변수로 `get()` 메서드에 전달됩니다. 처리기는 캐시 항목의 타임스탬프 이후 소프트 태그가 무효화되었는지 확인해야 합니다. `getExpiration()` 메소드는 제공된 모든 태그에서 가장 최근의 재검증 타임스탬프를 반환하거나 재검증된 태그가 없는 경우 `0`을 반환합니다. 또한 'Infinity'를 반환하여 소프트 태그를 대신 'get()'에 전달하고 그곳에서 만료를 확인해야 한다는 신호를 보낼 수도 있습니다. 반환된 타임스탬프가 항목 자체의 타임스탬프보다 최신인 경우 핸들러는 해당 항목을 오래된 항목으로 처리해야 합니다. 전체 의미는 [캐시 처리기 API 참조](/docs/app/api-reference/config/next-config-js/cacheHandlers#getexpiration)를 참조하세요.

## Multi-Instance Considerations
로드 밸런서 뒤에서 여러 Next.js 인스턴스를 실행할 때 재검증 이벤트는 기본적으로 로컬입니다. 인스턴스 A에서 `revalidateTag()`를 호출하면 해당 인스턴스의 캐시만 무효화됩니다. 다른 인스턴스는 무효화에 대해 알아볼 때까지 오래된 콘텐츠를 계속 제공합니다.

캐시 처리기 API는 분산 조정을 위한 두 가지 후크를 제공합니다:
* **`updateTags()`**는 `revalidateTag()`가 호출될 때 호출됩니다. 핸들러는 다른 인스턴스가 이를 검색할 수 있도록 공유 스토리지(예: Redis 또는 데이터베이스)에 무효화 이벤트를 작성해야 합니다.
* **`refreshTags()`**는 주기적으로 호출되지만 항상 새 요청을 시작하기 전에 호출됩니다. 핸들러는 공유 저장소에 최근 무효화 이벤트가 있는지 확인하고 이에 따라 로컬 태그 상태를 업데이트해야 합니다.

구현 세부정보 및 Redis 예시는 [사용자 정의 캐시 처리기](/docs/app/api-reference/config/next-config-js/cacheHandlers)를 참조하세요.

## Implementation Patterns for Platforms
### Single instance
기본 파일 시스템 캐시는 일관성을 자동으로 처리합니다. 캐시 쓰기는 로컬 파일 시스템에서 원자적으로 수행되며 태그 상태는 메모리에서 유지됩니다. 추가 구성이 필요하지 않습니다.

### Multi-instance with shared cache
조정이 없으면 각 인스턴스는 독립적으로 콘텐츠를 제공하고 로컬 캐시만 사용하여 재검증을 처리합니다. 요청을 처리하는 인스턴스에 따라 사용자마다 다른 콘텐츠가 표시될 수 있으며 주문형 재검증은 호출을 수신한 인스턴스에만 적용됩니다.

이 창을 줄이고 재검증이 인스턴스 전체에 전파되도록 하려면:

1. 공유 서비스(Redis, DynamoDB 또는 간단한 HTTP API)에 태그 무효화 타임스탬프를 저장합니다.
2. 공유 서비스에 쓰려면 `updateTags()`를 구현하세요.
3. 공유 서비스에서 읽으려면 `refreshTags()`를 구현하세요. 핸들러는 `refreshTags()`에서 오류를 포착해야 합니다. 오류가 발생하면 예외가 요청 실패로 전파됩니다. 오류를 포착하면 요청이 마지막으로 알려진 로컬 태그 상태로 계속 진행되어 연결이 복원될 때까지 잠재적으로 오래된 콘텐츠를 제공할 수 있습니다.
4. 공유 저장소에 캐시 항목(HTML RSC 페이로드)을 저장합니다. 원자성 쓰기는 불일치 창을 더욱 줄이지만 정확성을 위해 필요하지는 않습니다.

### CDN integration
CDN이 Next.js 응답을 캐시하는 경우 `Vary` 헤더와 Next.js가 설정하는 `Cache-Control` 지시문을 준수해야 합니다. 다른 TTL을 사용하여 HTML 및 RSC 페이로드 응답을 별도로 캐시하지 마십시오. 자세한 내용은 [CDN 캐싱](/docs/app/guides/cdn-caching)을 참조하세요.

## Graceful Degradation
재검증 시스템은 엄격한 일관성보다 가용성을 우선시합니다. 인프라 보장이 완전히 충족되지 않는 경우에도 콘텐츠는 항상 제공됩니다:
* **Cache write failure**: 쓰기는 비동기식이므로 응답은 계속 사용자에게 제공됩니다. 캐시 항목이 손실되고 다음 요청이 새로운 렌더링을 트리거합니다.
* **Cache read failure**: 핸들러는 내부 오류를 포착하고 `undefined`(캐시 누락 신호)을 반환해야 합니다. 그러면 경로가 서버에서 새로 렌더링됩니다. 발생한 오류는 캐시 누락으로 처리되지 않습니다. 이는 렌더링 오류로 전파되므로 누락을 알리기 위해 항상 `undefined`을 반환합니다.
* **HTML/RSC cache inconsistency**: CDN이 다른 TTL 또는 무효화 타이밍으로 HTML 및 RSC 응답을 캐시하는 경우 사용자는 클라이언트 측 탐색 중에 일치하지 않는 콘텐츠를 볼 수 있습니다. 이를 방지하려면 함께 캐시하고 `Vary` 헤더를 존중하세요.
* **Cross-deployment skew**: 롤링 배포 중에 빌드 ID 변경으로 인해 일관된 콘텐츠를 가져오기 위한 하드 탐색이 트리거되도록 [`deploymentId`](/docs/app/api-reference/config/next-config-js/deploymentId)를 구성하세요.

캐시 오류로 인해 애플리케이션이 손상되는 것이 아니라 성능 저하(오래된 콘텐츠, 추가 렌더링)가 발생합니다.