# Error Handling
<span style="color: #808080;">Last updated March 25, 2026</span>

오류는 [예상되는 오류](https://nextjs.org/docs/app/getting-started/error-handling#handling-expected-errors)와 [포착되지 않은 예외](https://nextjs.org/docs/app/getting-started/error-handling#handling-uncaught-exceptions)라는 두 가지 범주로 나눌 수 있습니다. 이 페이지에서는 Next.js 애플리케이션에서 이러한 오류를 처리하는 방법을 안내합니다.

---
## 예상되는 오류 처리
예상되는 오류는 [서버 측 양식 유효성 검사](https://nextjs.org/docs/app/guides/forms)나 실패한 요청 등 애플리케이션의 정상적인 작동 중에 발생할 수 있는 오류입니다. 이러한 오류는 명시적으로 처리되어 클라이언트에 반환되어야 합니다.

### Server Functions
[useActionState](https://react.dev/reference/react/useActionState) 훅을 사용하여 [서버 기능](https://react.dev/reference/rsc/server-functions)에서 예상되는 오류를 처리할 수 있습니다.

이러한 오류의 경우 `try`/`catch` 블록을 사용하지 말고 오류를 발생시키세요. 대신 예상되는 오류를 반환 값으로 모델링하세요.

app/actions.ts
```typescript
'use server'
 
export async function createPost(prevState: any, formData: FormData) {
  const title = formData.get('title')
  const content = formData.get('content')
 
  const res = await fetch('https://api.vercel.app/posts', {
    method: 'POST',
    body: { title, content },
  })
  const json = await res.json()
 
  if (!res.ok) {
    return { message: 'Failed to create post' }
  }
}
```

`useActionState` 훅에 작업을 전달하고 반환된 `상태`를 사용하여 오류 메시지를 표시할 수 있습니다.

app/ui/form.tsx
```typescript
'use client'
 
import { useActionState } from 'react'
import { createPost } from '@/app/actions'
 
const initialState = {
  message: '',
}
 
export function Form() {
  const [state, formAction, pending] = useActionState(createPost, initialState)
 
  return (
    <form action={formAction}>
      <label htmlFor="title">Title</label>
      <input type="text" id="title" name="title" required />
      <label htmlFor="content">Content</label>
      <textarea id="content" name="content" required />
      {state?.message && <p aria-live="polite">{state.message}</p>}
      <button disabled={pending}>Create Post</button>
    </form>
  )
}
```

### Server Components
서버 컴포넌트 내부에서 데이터를 가져올 때 응답을 사용하여 조건부로 오류 메시지를 렌더링하거나 [리디렉션](https://nextjs.org/docs/app/api-reference/functions/redirect)할 수 있습니다.

app/page.tsx
```typescript
export default async function Page() {
  const res = await fetch(`https://...`)
  const data = await res.json()
 
  if (!res.ok) {
    return 'There was an error.'
  }
 
  return '...'
}
```

### Not found
경로 세그먼트 내에서 [notFound](https://nextjs.org/docs/app/api-reference/functions/not-found) 함수를 호출하고 [not-found.js](https://nextjs.org/docs/app/api-reference/file-conventions/not-found) 파일을 사용하여 404 UI를 표시할 수 있습니다.

app/blog/[slug]/page.tsx
```typescript
import { notFound } from 'next/navigation'
import { getPostBySlug } from '@/lib/posts'
 
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = getPostBySlug(slug)
 
  if (!post) {
    notFound()
  }
 
  return <div>{post.title}</div>
}
```

app/blog/[slug]/not-found.tsx
```typescript
export default function NotFound() {
  return <div>404 - Page Not Found</div>
}
```

---
## Handling uncaught exceptions
포착되지 않은 예외는 애플리케이션의 정상적인 흐름 중에 발생해서는 안 되는 버그나 문제를 나타내는 예기치 않은 오류입니다. 이는 오류를 발생시켜 처리해야 하며 오류 경계에 의해 포착됩니다.

### Nested error boundaries
Next.js는 오류 경계를 사용하여 포착되지 않은 예외를 처리합니다. 오류 경계는 하위 컴포넌트의 오류를 포착하고 충돌이 발생한 컴포넌트 트리 대신 대체 UI를 표시합니다.

경로 세그먼트 내에 [error.js](https://nextjs.org/docs/app/api-reference/file-conventions/error) 파일을 추가하고 React 컴포넌트를 내보내 오류 경계를 만듭니다:

app/dashboard/error.tsx
```typescript
'use client' // 오류 경계는 클라이언트 컴포넌트여야 합니다.
 
import { useEffect } from 'react'
 
export default function ErrorPage({
  error,
  unstable_retry,
}: {
  error: Error & { digest?: string }
  unstable_retry: () => void
}) {
  useEffect(() => {
    // 오류 보고 서비스에 오류 기록
    console.error(error)
  }, [error])
 
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button
        onClick={
          // 세그먼트를 다시 가져오고 다시 렌더링하여 복구를 시도합니다.
          () => unstable_retry()
        }
      >
        Try again
      </button>
    </div>
  )
}
```

오류는 가장 가까운 상위 오류 경계까지 표시됩니다. 이를 통해 [경로 계층 구조](https://nextjs.org/docs/app/getting-started/project-structure#component-hierarchy)의 다양한 수준에 `error.tsx` 파일을 배치하여 세부적인 오류 처리가 가능합니다.

![](./images/011/nested-error-component-hierarchy.avif)

컴포넌트 수준 오류 복구의 경우 [unstable_catchError](https://nextjs.org/docs/app/api-reference/functions/catchError) 함수를 사용하면 컴포넌트 트리의 모든 부분을 래핑할 수 있는 오류 경계를 만들 수 있습니다:

app/custom-error-boundary.tsx
```typescript
'use client'
 
import { unstable_catchError as catchError, type ErrorInfo } from 'next/error'
 
function ErrorFallback(
  props: { title: string },
  { error, unstable_retry }: ErrorInfo
) {
  return (
    <div>
      <h2>{props.title}</h2>
      <p>{error.message}</p>
      <button onClick={() => unstable_retry()}>Try again</button>
    </div>
  )
}
 
export default catchError(ErrorFallback)
```

그런 다음 반환된 컴포넌트를 레이아웃이나 페이지의 래퍼로 사용합니다:

app/some-component.tsx
```typescript
import ErrorBoundary from './custom-error-boundary'
 
export default function Component({ children }: { children: React.ReactNode }) {
  return <ErrorBoundary title="Dashboard Error">{children}</ErrorBoundary>
}
```

오류 경계는 이벤트 핸들러 내부의 오류를 포착하지 않습니다. 전체 앱이 충돌하는 대신 대체 UI를 표시하기 위해 [렌더링 중](https://react.dev/reference/react/Component#static-getderivedstatefromerror)에 오류를 포착하도록 설계되었습니다.

일반적으로 이벤트 핸들러나 비동기 코드의 오류는 렌더링 후에 실행되기 때문에 오류 경계에 의해 처리되지 않습니다.

이러한 경우를 처리하려면 오류를 수동으로 포착하고 `useState` 또는 `useReducer`를 사용하여 저장한 다음 UI를 업데이트하여 사용자에게 알립니다.

```typescript
'use client'
 
import { useState } from 'react'
 
export function Button() {
  const [error, setError] = useState(null)
 
  const handleClick = () => {
    try {
      // 실패할 수도 있는 일을 해라
      throw new Error('Exception')
    } catch (reason) {
      setError(reason)
    }
  }
 
  if (error) {
    /* render fallback UI */
  }
 
  return (
    <button type="button" onClick={handleClick}>
      Click me
    </button>
  )
}
```

`useTransition`의 `startTransition` 내부에서 처리되지 않은 오류는 가장 가까운 오류 경계까지 버블링됩니다.

```typescript
'use client'
 
import { useTransition } from 'react'
 
export function Button() {
  const [pending, startTransition] = useTransition()
 
  const handleClick = () =>
    startTransition(() => {
      throw new Error('Exception')
    })
 
  return (
    <button type="button" onClick={handleClick}>
      Click me
    </button>
  )
}
```

### Global errors
흔하지는 않지만 [국제화](https://nextjs.org/docs/app/guides/internationalization)를 활용하는 경우에도 루트 앱 디렉터리에 있는 [global-error.js](https://nextjs.org/docs/app/api-reference/file-conventions/error#global-error) 파일을 사용하여 루트 레이아웃의 오류를 처리할 수 있습니다. 전역 오류 UI는 활성화되면 루트 레이아웃이나 템플릿을 대체하므로 자체 `<html>` 및 `<body>` 태그를 정의해야 합니다.

app/global-error.tsx
```typescript
'use client' // Error boundaries must be Client Components
 
export default function GlobalError({
  error,
  unstable_retry,
}: {
  error: Error & { digest?: string }
  unstable_retry: () => void
}) {
  return (
    // global-error must include html and body tags
    <html>
      <body>
        <h2>Something went wrong!</h2>
        <button onClick={() => unstable_retry()}>Try again</button>
      </body>
    </html>
  )
}
```