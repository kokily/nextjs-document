# Mutating Data
<span style="color: #808080;">Last updated June 23, 2026</span>

[React Server Functions](https://react.dev/reference/rsc/server-functions)를 사용하여 Next.js의 데이터를 변경할 수 있습니다. 이 페이지에서는 서버 기능을 [create](https://nextjs.org/docs/app/getting-started/mutating-data#creating-server-functions)하고 [invoke](https://nextjs.org/docs/app/getting-started/mutating-data#invoking-server-functions)하는 방법을 살펴보겠습니다. Next.js 관련 동작(단일 왕복 응답, 순차적 디스패치, 보안, 배포)에 대해서는 [서버 Action 및 Mutations](https://nextjs.org/docs/app/guides/server-actions)를 참조하세요.

---
## 서버 기능이란 무엇입니까?
서버 기능은 서버에서 실행되는 비동기 기능입니다. 네트워크 요청을 통해 클라이언트에서 호출할 수 있으므로 비동기식이어야 합니다.

작업 또는 변형 컨텍스트에서는 서버 `action`이라고도 합니다.

관례적으로 서버 작업은 [startTransition](https://react.dev/reference/react/startTransition)과 함께 사용되는 비동기 함수입니다. 이는 함수가 다음과 같은 경우 자동으로 발생합니다:

- `action` props를 사용하여 `<form>`에 전달됩니다.
- `formAction` props를 사용하여 `<button>`에 전달됩니다.

작업이 호출되면 Next.js는 단일 서버 왕복으로 업데이트된 UI와 새 데이터를 모두 반환할 수 있습니다.

이면에서는 작업이 `POST` 메서드를 사용하며 이 HTTP 메서드만 해당 작업을 호출할 수 있습니다.

```
서버 기능은 애플리케이션의 UI뿐만 아니라 직접 POST 요청을 통해 접근할 수 있습니다. 항상 모든 서버 기능 내에서 인증 및 승인을 확인하십시오. 권장 패턴은 데이터 보안 가이드를 참조하세요.
```

> 알아두면 좋은 점: 서버 작업은 특정 방식(양식 제출 및 변형 처리)으로 사용되는 서버 기능입니다. 서버 기능은 더 넓은 의미의 용어입니다.

## 서버 기능 생성
서버 기능은 [use server](https://react.dev/reference/rsc/use-server) 지시문을 사용하여 정의할 수 있습니다. 비동기 함수의 상단에 지시문을 배치하여 함수를 서버 기능으로 표시하거나 별도의 파일 상단에 배치하여 해당 파일의 모든 내보내기를 표시할 수 있습니다.

app/lib/actions.ts
```typescript
import { auth } from '@/lib/auth'
 
export async function createPost(formData: FormData) {
  'use server'
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
 
  const title = formData.get('title')
  const content = formData.get('content')
 
  // Mutate data
  // Revalidate cache
}
 
export async function deletePost(formData: FormData) {
  'use server'
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
 
  const id = formData.get('id')
 
  // 삭제하기 전에 사용자가 이 리소스를 소유하고 있는지 확인하세요.
  // Mutate data
  // Revalidate cache
}
```

### 서버 컴포넌트
서버 기능은 함수 본문 상단에 `"use server"` 지시어를 추가하여 서버 구성 요소에 인라인될 수 있습니다:

app/page.tsx
```typescript
export default function Page() {
  // Server Action
  async function createPost(formData: FormData) {
    'use server'
    // ...
  }
 
  return <></>
}
```

> 알아두면 좋은 점: 서버 컴포넌트는 기본적으로 점진적인 향상을 지원합니다. 즉, JavaScript가 아직 로드되지 않았거나 비활성화된 경우에도 서버 작업을 호출하는 양식이 제출됩니다.

### 클라이언트 컴포넌트
클라이언트 컴포넌트에서는 서버 기능을 정의할 수 없습니다. 그러나 맨 위에 `"use server"` 지시어가 있는 파일에서 가져와서 클라이언트 컴포넌트에서 호출할 수 있습니다:

app/actions.ts
```typescript
'use server'
 
export async function createPost() {}
```

app/ui/button.tsx
```typescript
'use client'
 
import { createPost } from '@/app/actions'
 
export function Button() {
  return <button formAction={createPost}>Create</button>
}
```

> 알아두면 좋은 점: 클라이언트 컴포넌트에서 서버 작업을 호출하는 양식은 JavaScript가 아직 로드되지 않은 경우 제출을 대기열에 추가하고 하이드레이션 우선 순위를 지정합니다. 수화 후 양식 제출 시 브라우저가 새로 고쳐지지 않습니다.

### 액션을 props로 전달하기
클라이언트 컴포넌트에 작업을 props로 전달할 수도 있습니다:

```<ClientComponent updateItemAction={updateItem} />```

app/client-component.tsx
```typescript
'use client'
 
export default function ClientComponent({
  updateItemAction,
}: {
  updateItemAction: (formData: FormData) => void
}) {
  return <form action={updateItemAction}>{/* ... */}</form>
}
```

---
## Invoking Server Functions
서버 기능을 호출할 수 있는 두 가지 주요 방법이 있습니다:

1. 서버 및 클라이언트 컴포넌트의 [Forms](https://nextjs.org/docs/app/getting-started/mutating-data#forms)
2. 클라이언트 컴포넌트의 [이벤트 처리기](https://nextjs.org/docs/app/getting-started/mutating-data#event-handlers) 및 [useEffect](https://nextjs.org/docs/app/getting-started/mutating-data#useeffect)

> 알아두면 좋은 점: 서버 기능은 서버측 변형을 위해 설계되었습니다. 클라이언트는 현재 한 번에 하나씩 파견하고 기다리고 있습니다. 이는 구현 세부 사항이며 변경될 수 있습니다. 병렬 데이터 가져오기가 필요한 경우 서버 컴포넌트에서 데이터 가져오기를 사용하거나 단일 서버 기능 또는 경로 처리기 내에서 병렬 작업을 수행하세요.

### Forms
React는 HTML [<form>](https://react.dev/reference/react-dom/components/form) 요소를 확장하여 HTML action prop으로 서버 함수를 호출할 수 있습니다.

양식에서 호출되면 함수는 자동으로 [FormData](https://developer.mozilla.org/docs/Web/API/FormData/FormData) 개체를 받습니다. 기본 [FormData 메서드](https://developer.mozilla.org/en-US/docs/Web/API/FormData#instance_methods)를 사용하여 데이터를 추출할 수 있습니다:

app/ui/form.tsx
```typescript
import { createPost } from '@/app/actions'
 
export function Form() {
  return (
    <form action={createPost}>
      <input type="text" name="title" />
      <input type="text" name="content" />
      <button type="submit">Create</button>
    </form>
  )
}
```

app/actions.ts
```typescript
'use server'
 
import { auth } from '@/lib/auth'
 
export async function createPost(formData: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
 
  const title = formData.get('title')
  const content = formData.get('content')
 
  // Mutate data
  // Revalidate cache
}
```

### Event Handlers
`onClick`과 같은 이벤트 핸들러를 사용하여 클라이언트 컴포넌트에서 서버 기능을 호출할 수 있습니다.

---
## Examples
### 보류(Pending) 상태 표시
서버 기능을 실행하는 동안 React의 [useActionState](https://react.dev/reference/react/useActionState) 후크를 사용하여 로딩 표시기를 표시할 수 있습니다. 이 후크는 `pending` 중인 부울을 반환합니다:

app/ui/button.tsx
```typescript
'use client'
 
import { useActionState, startTransition } from 'react'
import { createPost } from '@/app/actions'
import { LoadingSpinner } from '@/app/ui/loading-spinner'
 
export function Button() {
  const [state, action, pending] = useActionState(createPost, false)
 
  return (
    <button onClick={() => startTransition(action)}>
      {pending ? <LoadingSpinner /> : 'Create Post'}
    </button>
  )
}
```

### Refresh Data
변형 후에는 현재 페이지를 새로 고쳐 최신 데이터를 표시할 수 있습니다. 서버 작업에서 `next/cache`에서 [refresh](https://nextjs.org/docs/app/api-reference/functions/refresh)을 호출하여 이 작업을 수행할 수 있습니다:

app/lib/actions.ts
```typescript
'use server'
 
import { auth } from '@/lib/auth'
import { refresh } from 'next/cache'
 
export async function updatePost(formData: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
  // Mutate data
  // ...
 
  refresh()
}
```

이렇게 하면 클라이언트 라우터가 새로 고쳐져 UI에 최신 상태가 반영됩니다. `refresh()` 함수는 태그가 지정된 데이터의 유효성을 다시 검사하지 않습니다. 태그가 지정된 데이터를 재검증하려면 대신 [updateTag](https://nextjs.org/docs/app/api-reference/functions/updateTag) 또는 [revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)를 사용하세요.

### Revalidate data
변형을 수행한 후 서버 함수 내에서 [revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath) 또는 [revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)를 호출하여 Next.js 캐시를 재검증하고 업데이트된 데이터를 표시할 수 있습니다:

app/lib/actions.ts
```typescript
import { auth } from '@/lib/auth'
import { revalidatePath } from 'next/cache'
 
export async function createPost(formData: FormData) {
  'use server'
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
  // Mutate data
  // ...
 
  revalidatePath('/posts')
}
```

### Redirect after a mutation
mutation 후에 사용자를 다른 페이지로 리디렉션할 수 있습니다. 서버 기능 내에서 [redirect](https://nextjs.org/docs/app/api-reference/functions/redirect)을 호출하여 이를 수행할 수 있습니다.

app/lib/actions.ts
```typescript
'use server'
 
import { auth } from '@/lib/auth'
import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'
 
export async function createPost(formData: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
  // Mutate data
  // ...
 
  revalidatePath('/posts')
  redirect('/posts')
}
```

`redirect` [throws](https://nextjs.org/docs/app/api-reference/functions/redirect#behavior)을 호출하면 프레임워크에서 처리하는 제어 흐름 예외(Exception)가 발생합니다. 그 이후의 코드는 실행되지 않습니다. 새로운 데이터가 필요한 경우 미리 [revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath) 또는 [revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)를 호출하세요.

### Cookies
[cookies](https://nextjs.org/docs/app/api-reference/functions/cookies) API를 사용하여 서버 작업 내에서 쿠키를 `get`, `set`, `delete` 할 수 있습니다.

서버 작업에서 쿠키를 [설정하거나 삭제](https://nextjs.org/docs/app/api-reference/functions/cookies#understanding-cookie-behavior-in-server-functions)하면 Next.js는 UI에 새 쿠키 값이 반영되도록 서버에서 현재 페이지와 해당 레이아웃을 다시 렌더링합니다.

> 알아두면 좋은 점: 서버 업데이트는 필요에 따라 현재 React 트리, 다시 렌더링, 마운트 또는 마운트 해제에 적용됩니다. 다시 렌더링된 컴포넌트에 대해 클라이언트 상태가 유지되며 종속성이 변경되면 효과가 다시 실행됩니다.

app/actions.ts
```typescript
'use server'
 
import { cookies } from 'next/headers'
 
export async function exampleAction() {
  const cookieStore = await cookies()
 
  // Get cookie
  cookieStore.get('name')?.value
 
  // Set cookie
  cookieStore.set('name', 'Delba')
 
  // Delete cookie
  cookieStore.delete('name')
}
```

### useEffect
React [useEffect](https://react.dev/reference/react/useEffect) 후크를 사용하면 구성 요소가 마운트되거나 종속성이 변경될 때 서버 작업을 호출할 수 있습니다. 이는 전역 이벤트에 의존하거나 자동으로 트리거되어야 하는 변형에 유용합니다. 예를 들어 앱 바로가기를 위한 `onKeyDown`, 무한 스크롤을 위한 교차 관찰자 후크 또는 보기 수를 업데이트하기 위해 구성요소가 마운트되는 경우:

app/view-count.tsx
```typescript
'use client'
 
import { incrementViews } from './actions'
import { useState, useEffect, useTransition } from 'react'
 
export default function ViewCount({ initialViews }: { initialViews: number }) {
  const [views, setViews] = useState(initialViews)
  const [isPending, startTransition] = useTransition()
 
  useEffect(() => {
    startTransition(async () => {
      const updatedViews = await incrementViews()
      setViews(updatedViews)
    })
  }, [])
 
  // 'isPending'을 사용하여 사용자에게 피드백을 제공할 수 있습니다.
  return <p>Total Views: {views}</p>
}
```

