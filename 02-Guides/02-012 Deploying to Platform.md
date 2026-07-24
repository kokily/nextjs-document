# Deploying Next.js to different platforms
<span style="color: #808080;">Last updated March 31, 2026</span>

Next.js는 컴포넌트 수준에서 [정적 및 ​​동적 콘텐츠를 스펙트럼으로 처리](/docs/app/guides/rendering-philosophy)합니다. 이 모델의 다양한 기능에는 다양한 플랫폼 기능이 필요합니다. 이 페이지는 플랫폼에서 지원해야 하는 사항과 배포 구성 방법을 이해하는 데 도움이 됩니다.

## Minimum Requirements
Next.js를 실행하려면 플랫폼에 **Node.js 서버**가 필요합니다. 그게 다야.

단일 `next start` 프로세스는 모든 Next.js 기능(서버 컴포넌트, ISR, PPR, 캐시 컴포넌트, Server Action, 프록시 및 `after()`)을 올바르게 처리합니다. 콘텐츠를 점진적으로 전달하려면 PPR 및 서버 컴포넌트와 같은 기능에 스트리밍 지원이 필요합니다. 스트리밍 지원이 없으면 응답이 버퍼링되어 전체적으로 전송되므로 여전히 작동하지만 스트리밍 성능 이점을 잃게 됩니다. 추가 인프라(CDN 캐싱, 에지 컴퓨팅, 공유 캐시)는 주로 성능과 다중 인스턴스 일관성을 향상시킵니다. 다중 인스턴스 배포에서는 공유 캐시와 태그 조정을 통해 인스턴스 간의 오래된 차이를 줄입니다. 유일한 추가 종속성은 [이미지 최적화](/docs/app/api-reference/comComponents/image)에 필요한 `sharp` 패키지입니다.

## Functional Fidelity vs. Performance Fidelity
Next.js에 대한 플랫폼 지원을 평가할 때 두 가지 수준을 구별하는 데 도움이 됩니다:

**기능 충실도**는 모든 Next.js 기능이 올바르게 작동함을 의미합니다. [어댑터 테스트 모음](/docs/app/api-reference/adapters/testing-adapters)은 계약입니다. 플랫폼의 어댑터가 테스트를 통과하면 Next.js를 지원합니다. 이것은 바이너리입니다. 통과하거나 실패합니다.

**성능 충실도**는 기능이 최적의 성능 특성을 달성한다는 것을 의미합니다. 예를 들어 원본 대기 시간이 아닌 CDN 대기 시간으로 제공되는 PPR의 정적 셸 또는 1초 미만의 재검증 전파를 통해 오래된 콘텐츠를 즉시 제공하는 ISR이 있습니다. 성능 충실도는 모든 플랫폼이 아키텍처에 따라 다르게 달성할 수 있는 스펙트럼입니다.

기능적 충실도를 달성하는 플랫폼은 Next.js가 완벽하게 지원되는 배포 대상입니다. 성능 충실도는 플랫폼을 차별화하는 방법이며 시간이 지남에 따라 점진적으로 향상됩니다.

이 렌즈를 통해 아래의 기능 매트릭스를 사용하십시오. "스트리밍 필요" 및 "공유 캐시 권장"은 기능 충실도에 필요한 사항을 설명합니다. "Edge Stitching"은 성능 충실도 최적화입니다.

## Feature Support Matrix
다양한 기능에는 다양한 인프라 기능이 필요합니다. "에지 스티칭" 열은 정확성 요구사항이 아니라 **성능 최적화**입니다. 모든 기능은 단일 원본 서버에서 올바르게 작동합니다.

|Feature|Streaming|Shared Cache|Edge Stitching|Notes|
|---|---|---|---|---|
|Server Components|Required|No|No|Basic streaming support|
|ISR (time-based)|No|Recommended|No|Works per-instance without shared cache|
|ISR (on-demand)|No|Recommended|No|[Tag propagation](/docs/app/guides/how-revalidation-works) needs shared cache for multi-instance|
|Partial Prerendering|Required|Recommended|Optional|[See PPR Platform Guide](/docs/app/guides/ppr-platform-guide)|Cache Components (`use cache`)|Required| Recommended|No|Shared cache enables cross-instance consistency|
|Proxy / Middleware|No|No|No|Runs at edge or origin|
|Server Actions|Required|No|No|POST requests with streaming response|
|`after()`|No|No|No|Requires [graceful shutdown](/docs/app/guides/self-hosting#after) support|

**스트리밍 필요**는 플랫폼이 청크 전송 인코딩 또는 HTTP/2 스트리밍을 지원해야 하며 클라이언트에 응답을 보내기 전에 응답을 버퍼링해서는 안 된다는 것을 의미합니다.

**공유 캐시 권장**은 여러 서버 인스턴스가 공유 캐시 백엔드를 활용하여 조정하는 이점을 의미합니다. ISR 및 서버 응답 캐싱의 경우 [`cacheHandler`](/docs/app/api-reference/config/next-config-js/incrementalCacheHandlerPath)를 사용하세요. `캐시 사용` 항목의 경우 [`cacheHandlers`](/docs/app/api-reference/config/next-config-js/cacheHandlers)를 사용하세요. 공유 캐시가 없으면 각 인스턴스는 자체 캐시를 독립적으로 유지 관리합니다. 기능은 각 인스턴스에서 계속 올바르게 작동하지만 재검증 이벤트는 인스턴스 전체에 전파되지 않습니다.

## CDN Infrastructure Compatibility
다음 표는 각 주요 CDN에 대한 인프라 기본 요소를 매핑합니다. 이는 완성된 통합이 아닌 사용 가능한 빌딩 블록입니다:
|CDN|Edge Compute|Key-Value / Tags|Blob Storage|PPR Resuming|
|---|---|---|---|---|
|Cloudflare|Workers|KV|R2|Yes (worker)|
|Akamai|EdgeWorkers|EdgeKV|Object Storage|Yes (worker)|
|Amazon CloudFront|Lambda@Edge|KeyValueStore|S3|Yes (Lambda)|
|Fastly|Compute|KV Store|Object Storage|Yes (WASM)|
|Azure|Functions|Managed Redis|Blob Storage|Yes (server)|
|Google Cloud|Cloud Run|Various KV|Cloud Storage|Yes (server)|

이는 완성된 통합이 아닌 사용 가능한 빌딩 블록입니다. 오늘날 대부분의 커뮤니티 어댑터는 Edge KV 또는 PPR 재개와 같은 CDN 관련 기본 요소를 활용하지 않고 Next.js를 Docker 컨테이너 또는 Node.js 서버로 배포합니다. 현재 어댑터 목록은 [배포](/docs/app/getting-started/deployingad#apters) 페이지를 참조하세요. CDN 관련 캐싱 고려 사항(사용자 정의 헤더의 `Vary`에 대한 알려진 제한 사항 포함)은 [CDN 캐싱](/docs/app/guides/cdn-caching)을 참조하세요.

## Adapters
Next.js는 플랫폼이 인프라에 맞게 Next.js 애플리케이션을 구축하고 배포하는 방법을 사용자 정의할 수 있는 [배포 어댑터 API](/docs/app/api-reference/config/next-config-js/adapterPath)를 제공합니다. 어댑터는 빌드 시 실행되며 표준 Next.js 빌드에서 플랫폼별 출력을 생성합니다. 누구나 특별한 액세스가 필요 없이 공개 API를 사용하여 어댑터를 구축할 수 있습니다.

어댑터 API와 Next.js 캐싱 인터페이스가 완전한 플랫폼 통합 표면을 형성합니다. 어댑터는 빌드 시간 출력을 처리하는 반면 `cacheHandler` 및 `cacheHandlers`는 다양한 런타임 캐싱 경로를 처리합니다. `cacheHandler`(단수)는 ISR, 경로 핸들러, 패치된 `fetch`/`unstable_cache` 및 이미지 최적화와 같은 서버 캐시 경로를 다룹니다. `cacheHandlers`(복수형)는 `'캐시 사용'` 지시문 백엔드를 구성합니다.

### Verified Adapters
**검증된 어댑터**는 두 가지 요구 사항을 충족하는 어댑터입니다:

1. **Open source.** 어댑터 소스 코드는 공개적으로 사용 가능하므로 커뮤니티와 Next.js 팀이 이를 검사하고 기여하고 확인할 수 있습니다.
2. **Runs the compatibility test suite.** 플랫폼은 해당 어댑터에 대해 전체 [Next.js 호환성 테스트 모음](/docs/app/api-reference/adapters/testing-adapters)을 실행하는 방법을 제공합니다. 이를 통해 어떤 기능이 작동하는지, 어떤 기능이 진행 중인지, 어디에 공백이 남아 있는지 확인할 수 있습니다.

검증된 어댑터는 [Next.js GitHub 조직](https://github.com/nextjs)에서 호스팅되고 Next.js 문서에 지원되는 배포 대상으로 나열되며 해당 플랫폼 팀에서 유지관리합니다. 프라이빗 프레임워크 훅이나 통합 경로가 없습니다. Vercel의 어댑터는 다른 모든 어댑터와 동일한 공개 API를 사용합니다.

[생태계 워킹 그룹](https://nextjs.org/ecosystem-working-group)을 통해 검증된 상태를 향해 노력하는 검증된 어댑터 및 플랫폼에 대해 Next.js 팀은 다음을 약속합니다:

* **Coordinated testing.** 주요 릴리스 이전에 우리는 플랫폼 팀과 협력하여 호환성 테스트 제품군을 실행하고 문제를 조기에 표면화합니다.
* **Early access.** 어댑터 작성자는 RFC 및 릴리스 후보 중에 API 변경 사항에 대한 조기 액세스 권한을 받습니다.
* **Direct support.** 어댑터 계약을 업데이트해야 하는 경우 어댑터 팀과 직접 협력합니다.

> **알아두면 좋은 점:** 플랫폼은 동일한 공개 API 및 테스트 도구 모음에서 비공개 소스 어댑터를 구축할 수 있습니다. 그러나 비공개 소스 어댑터는 확인된 항목으로 나열되지 않습니다. 왜냐하면 Next.js 팀은 검사할 수 없는 항목을 확인할 수 없기 때문입니다.

## A Note on Infrastructure Requirements
Next.js의 [렌더링 모델](/docs/app/guides/rendering-philosophy)은 경로 수준이 아닌 구성 요소 수준에 정적/동적 경계를 배치합니다. 더욱 세분화된 경계는 호스팅 플랫폼에 대한 요구 사항이 더 넓어지는 대신 개발자에게 더 많은 유연성을 제공합니다. 이는 의도적인 절충안입니다. 이 페이지의 인프라 요구 사항은 렌더링 모델이 제공하는 것 때문에 존재합니다.