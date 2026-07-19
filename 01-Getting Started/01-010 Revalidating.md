# Revalidating
<span style="color: #808080;">Last updated June 23, 2026</span>

> 이 페이지에서는 next.config.ts 파일에서 [cacheComponents=true](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)로 설정하여 활성화되는 [캐시 컴포넌트](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)를 사용한 재검증을 다룹니다. 캐시 컴포넌트를 사용하지 않는 경우 [캐싱 및 유효성 재검사(이전 모델) 가이드](https://nextjs.org/docs/app/guides/caching-without-cache-components)를 참조하세요.

재검증은 캐시된 데이터를 업데이트하는 프로세스입니다. 이를 통해 콘텐츠를 최신 상태로 유지하면서 빠르고 캐시된 응답을 계속 제공할 수 있습니다. 두 가지 전략이 있습니다:

- 시간 기반 재검증: 캐시라이프([cacheLife](https://nextjs.org/docs/app/getting-started/revalidating#cachelife))를 사용하여 설정된 기간이 지나면 캐시된 데이터를 자동으로 새로 고칩니다.
- 주문형 재검증: [revalidateTag](https://nextjs.org/docs/app/getting-started/revalidating#revalidatetag), [updateTag](https://nextjs.org/docs/app/getting-started/revalidating#updatetag) 또는 [revalidatePath](https://nextjs.org/docs/app/getting-started/revalidating#revalidatepath)를 사용하여 변형 후 캐시된 데이터를 수동으로 무효화합니다.

---
`cacheLife`
캐시라이프([cacheLife](https://nextjs.org/docs/app/api-reference/functions/cacheLife))는 캐시된 데이터가 유효한 상태로 유지되는 기간을 제어합니다. 캐시 수명을 설정하려면 [use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache) 범위 내에서 사용하세요.

app/lib/data.ts
```typescript
import { cacheLife } from 'next/cache'

export async function getProducts() {
  'use cache'
  cacheLife('hours')
  return db.query('SELECT * FROM products')
}
```

`cacheLife`는 프로필 이름이나 맞춤 구성 개체를 허용합니다:

|Profile|`stale`|`revlidate`|`expire`
|---|---|---|---|
|`defulat`|5m|15m|never|
|`seconds`|30s|1s|60s|
|`minutes`|5m|1m|1h|
|`hours`|5m|1h|1d|
|`days`|5m|1d|1w|
|`weeks`|5m|1w|30d|
|`max`|5m|30d|1y|

세밀하게 제어하려면 객체를 전달하세요:

```typescript
'use cache'
cacheLife({
  stale: 3600,  // 오래된 것으로 간주될 때까지 1시간
  relivdate: 7200,  // 재확인까지 2시간 남았습니다
  expire: 86400,  // 만료까지 1일
})
```

> 알아두면 좋은 점: `seconds` 프로필을 사용하거나, `revalidate: 0`을 수행하거나, 5분 이내에 `expire`되는 경우 캐시는 "단기 수명"으로 간주됩니다. 수명이 짧은 캐시는 사전 렌더링에서 자동으로 제외되고 대신 동적 구멍이 됩니다. 자세한 내용은 [사전 렌더링 동작](https://nextjs.org/docs/app/api-reference/functions/cacheLife#prerendering-behavior)을 참조하세요.

모든 프로필 및 사용자 정의 구성 옵션에 대해서는 [캐시라이프 API 참조](https://nextjs.org/docs/app/api-reference/functions/cacheLife)를 확인하세요.

---
`cacheTag`
[cacheTag](https://nextjs.org/docs/app/api-reference/functions/cacheTag)를 사용하면 캐시된 데이터에 태그를 지정하여 필요할 때 무효화할 수 있습니다. [use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache) 범위 내에서 사용하세요:

app/lib/data.ts
```typescript
import { cacheTag } from 'next/cache'

export async function getProducts() {
  'use cache'
  cacheTag('products')
  return db.query('SELECT * FROM products')
}
```

태그가 지정되면 [revalidateTag](https://nextjs.org/docs/app/getting-started/revalidating#revalidatetag) 또는 [updateTag](https://nextjs.org/docs/app/getting-started/revalidating#updatetag)를 사용하여 캐시를 무효화합니다.

자세한 내용은 [캐시태그 API 참조](https://nextjs.org/docs/app/api-reference/functions/cacheTag)를 확인하세요.

---
`revalidateTag`
`revalidateTag`는 재검증 중 오래된 의미 체계를 사용하여 태그별로 캐시 항목을 무효화합니다. 즉, 오래된 콘텐츠는 백그라운드에서 새로운 콘텐츠가 로드되는 동안 즉시 제공됩니다. 이는 블로그 게시물이나 제품 카탈로그와 같이 약간의 업데이트 지연이 허용되는 콘텐츠에 이상적입니다.

app/lib/actions.ts
```typescript
import { revalidateTag } from 'next/cache'

export async function updateUser(id: string) {
  // Mutate data
  revalidateTag('user', 'max')  // 권장사항: 재검증하는 동안 오래된
}
```

여러 함수에서 동일한 태그를 재사용하여 한 번에 모두 재검증할 수 있습니다. [Server Action](https://nextjs.org/docs/app/getting-started/mutating-data) 또는 [Route handler](https://nextjs.org/docs/app/api-reference/file-conventions/route)에서 `revalidateTag`를 호출합니다.

> 알아두면 좋은 점: 두 번째 인수는 백그라운드에서 새로운 콘텐츠가 생성되는 동안 오래된 콘텐츠를 제공할 수 있는 기간을 설정합니다. 만료되면 새로운 콘텐츠가 준비될 때까지 후속 요청이 차단됩니다. `max`를 사용하면 가장 긴 오래된 기간이 제공됩니다.

자세한 내용은 [revalidateTag API 참조](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)를 확인하세요.

---
`updateTag`
updateTag immediately expires cached data for read-your-own-writes scenarios — the user sees their change right away instead of stale content. Unlike revalidateTag, it can only be used in Server Actions.

app/lib/actions.ts
```typescript
```

| |`updateTag`|`revalidateTag`|
|---|---|---|
|Where|Server Action only|Server Action and Route handlers|
|Behavior|Immediately expires cache|Stale-while-revlidate|
|Use case|Read-your-own-writes (user sees their change)|Background refresh (slight delay OK)|

자세한 내용은 [updateTag API 참조](https://nextjs.org/docs/app/api-reference/functions/updateTag)를 확인하세요.

---
`revalidatePath`
`revalidatePath`는 특정 경로 경로에 대해 캐시된 모든 데이터를 무효화합니다. 어떤 태그가 연결되어 있는지 알지 못한 채 경로를 재검증하고 싶을 때 사용하세요.

app/lib/actions.ts
```typescript
import { revalidatePath } from 'next/cache'

export async function updateUser(id: string) {
  // Mutate data
  revalidatePath('/profile')
}
```

> 알아두면 좋은 점: 가능한 경우 경로 기반보다 태그 기반 재검증(`revalidateTag`/`updateTag`)을 선호합니다. 이는 더 정확하고 과도한 무효화를 방지합니다.

자세한 내용은 [revalidatePath API 참조](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)를 참조하세요.

---
### 무엇을 캐시해야 하나요?
[런타임 데이터](https://nextjs.org/docs/app/getting-started/caching#working-with-runtime-apis)에 의존하지 않고 일정 기간 동안 캐시에서 제공해도 괜찮은 캐시 데이터입니다. 해당 동작을 설명하려면 `use cache`와 함께 `cacheLife`를 사용하세요.

업데이트 메커니즘이 있는 콘텐츠 관리 시스템의 경우 캐시 기간이 더 긴 태그를 사용하고 캐시를 미리 만료하는 대신 `revalidateTag`를 사용하여 콘텐츠가 실제로 변경될 때 콘텐츠를 새로 고칩니다.

> 알아두면 좋은 점: 서버리스 환경에서는 인메모리 캐시 항목이 재검증 후에도 유지되지 않을 수 있습니다. 자세한 내용은 [런타임 캐싱 고려 사항](https://nextjs.org/docs/app/api-reference/directives/use-cache#runtime-caching-considerations)을 참조하세요.