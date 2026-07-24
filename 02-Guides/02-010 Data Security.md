# How to think about data security in Next.js
<span style="color: #808080;">Last updated June 23, 2026</span>

[React 서버 컴포넌트](https://react.dev/reference/rsc/server-comComponents)는 성능을 향상하고 데이터 가져오기를 단순화할 뿐만 아니라 데이터에 액세스하는 위치와 방법을 변경하여 프런트엔드 앱에서 데이터를 처리하기 위한 기존 보안 가정 중 일부를 변경합니다.

이 가이드는 Next.js의 데이터 보안에 대해 생각하는 방법과 모범 사례를 구현하는 방법을 이해하는 데 도움이 됩니다.

## Data fetching approaches
프로젝트의 규모와 기간에 따라 Next.js에서 데이터를 가져오는 데 권장되는 세 가지 주요 접근 방식이 있습니다:
* [HTTP APIs](#external-http-apis): 기존 대규모 애플리케이션 및 조직의 경우.
* [Data Access Layer](#data-access-layer): 새로운 프로젝트를 위해.
* [Component-Level Data Access](#component-level-data-access): 프로토타입과 학습을 위해.

하나의 데이터 가져오기 접근 방식을 선택하고 혼합을 피하는 것이 좋습니다. 이를 통해 코드 베이스에서 작업하는 개발자와 보안 감사자 모두에게 무엇을 기대할 수 있는지 명확해집니다.

### External HTTP APIs
기존 프로젝트에 서버 구성 요소를 채택할 때는 **제로 트러스트** 모델을 따라야 합니다. 클라이언트 컴포넌트에서와 마찬가지로 [`fetch`](/docs/app/api-reference/functions/fetch)를 사용하여 서버 컴포넌트에서 REST 또는 GraphQL과 같은 기존 API 엔드포인트를 계속 호출할 수 있습니다.

app/page.tsx
```tsx
import { cookies } from 'next/headers'

export default async function Page() {
  const cookieStore = await cookies()
  const token = cookieStore.get('AUTH_TOKEN')?.value

  const res = await fetch('https://api.example.com/profile', {
    headers: {
      Cookie: `AUTH_TOKEN=${token}`,
      // Other headers
    },
  })

  // ....
}
```

이 접근 방식은 다음과 같은 경우에 효과적입니다:
* 이미 보안 관행이 마련되어 있습니다.
* 별도의 백엔드 팀이 다른 언어를 사용하거나 독립적으로 API를 관리합니다.

### Data Access Layer
새 프로젝트의 경우 전용 **DAL(데이터 액세스 계층)**을 만드는 것이 좋습니다. 이는 데이터를 가져오는 방법과 시기, 렌더링 컨텍스트에 전달되는 항목을 제어하는 ​​내부 라이브러리입니다.

데이터 액세스 계층은 다음을 수행해야 합니다:
* 서버에서만 실행됩니다.
* 승인 확인을 수행합니다.
* 안전하고 최소한의 **DTO(데이터 전송 개체)**를 반환합니다.

이 접근 방식은 모든 데이터 액세스 논리를 중앙 집중화하여 일관된 데이터 액세스를 보다 쉽게 ​​적용하고 승인 버그의 위험을 줄입니다. 또한 요청의 여러 부분에서 메모리 내 캐시를 공유하는 이점도 있습니다.

data/auth.ts
```ts
import { cache } from 'react'
import { cookies } from 'next/headers'

// 캐시된 도우미 메서드를 사용하면 수동으로 전달하지 않고도 여러 위치에서 동일한 값을
// 쉽게 얻을 수 있습니다. 이렇게 하면 서버 컴포넌트에서 서버 컴포넌트로 전달하는 것을
// 방지하여 클라이언트 컴포넌트로 전달하는 위험을 최소화합니다.
export const getCurrentUser = cache(async () => {
  const cookieStore = await cookies()
  const token = cookieStore.get('AUTH_TOKEN')
  const decodedToken = await decryptAndValidate(token)
  // 비밀 토큰이나 개인 정보를 공개 필드로 포함하지 마세요.
  // 실수로 전체 개체를 클라이언트에 전달하는 것을 방지하려면 클래스를 사용하세요.
  return new User(decodedToken.id)
})
```

data/user-dto.tsx
```tsx
import 'server-only'
import { getCurrentUser } from './auth'

function canSeeUsername(viewer: User) {
  // 현재 공개 정보이지만 변경될 수 있음
  return true
}

function canSeePhoneNumber(viewer: User, team: string) {
  // 개인 정보 보호 규칙
  return viewer.isAdmin || team === viewer.team
}

export async function getProfileDTO(slug: string) {
  // 값을 전달하지 않고, 캐시된 값을 다시 읽고, 컨텍스트를 해결하고 더 쉽게 게으르게 만들 수 있습니다.

  // 안전한 쿼리 템플릿을 지원하는 데이터베이스 API를 사용하세요.
  const [rows] = await sql`SELECT * FROM user WHERE slug = ${slug}`
  const userData = rows[0]

  const currentUser = await getCurrentUser()

  // 모든 데이터가 아닌 이 쿼리와 관련된 데이터만 반환합니다.
  // <https://www.w3.org/2001/tag/doc/APIMinimization>
  return {
    username: canSeeUsername(currentUser) ? userData.username : null,
    phonenumber: canSeePhoneNumber(currentUser, userData.team)
      ? userData.phonenumber
      : null,
  }
}
```

app/page.tsx
```tsx
import { getProfileDTO } from '../../data/user-dto'

export default async function Page({ params }) {
  const { slug } = await params
  // 이제 이 페이지에는 민감한 내용이 포함되어서는 안 된다는 사실을 알고
  // 이 프로필을 안전하게 전달할 수 있습니다.
  const profile = await getProfileDTO(slug)
  ...
}
```

> **알아두면 좋은 점:** 비밀 키는 환경 변수에 저장되어야 하지만 데이터 액세스 레이어만 `process.env`에 액세스해야 합니다. 이렇게 하면 비밀이 애플리케이션의 다른 부분에 노출되는 것을 방지할 수 있습니다.

### Component-level data access
빠른 프로토타입 및 반복을 위해 데이터베이스 쿼리를 서버 컴포넌트에 직접 배치할 수 있습니다.

그러나 이 접근 방식을 사용하면 개인 데이터를 실수로 클라이언트에 노출하는 것이 더 쉬워집니다:

app/page.tsx
```tsx
import Profile from './components/profile.tsx'

export default async function Page({ params }) {
  const { slug } = await params
  const [rows] = await sql`SELECT * FROM user WHERE slug = ${slug}`
  const userData = rows[0]
  // EXPOSED: 서버 컴포넌트에서 클라이언트로 데이터를 전달하기 때문에 userData의 모든 필드가 클라이언트에 노출됩니다.
  return <Profile user={userData} />
}
```

app/ui/profile.tsx
```tsx
'use client'

// BAD: 이는 클라이언트 컴포넌트에 필요한 것보다 훨씬 더 많은 데이터를 허용하고
//      서버 컴포넌트가 해당 데이터를 모두 전달하도록 장려하기 때문에 나쁜 소품
//      인터페이스입니다.
//      더 나은 해결책은 프로필을 렌더링하는 데 필요한 필드만으로 제한된 개체를
//      허용하는 것입니다.
export default async function Profile({ user }: { user: User }) {
  return (
    <div>
      <h1>{user.name}</h1>
      ...
    </div>
  )
}
```

클라이언트 컴포넌트에 데이터를 전달하기 전에 데이터를 삭제해야 합니다:

data/user.ts
```ts
import { sql } from './db'

export async function getUser(slug: string) {
  const [rows] = await sql`SELECT * FROM user WHERE slug = ${slug}`
  const user = rows[0]

  // 공개 필드만 반환
  return {
    name: user.name,
  }
}
```

app/page.tsx
```tsx
import { getUser } from '../data/user'
import Profile from './ui/profile'

export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const publicProfile = await getUser(slug)
  return <Profile user={publicProfile} />
}
```

## Reading data

### Passing data from server to client
초기 로드 시 서버와 클라이언트 컴포넌트 모두 서버에서 실행되어 HTML을 생성합니다. 그러나 격리된 모듈 시스템에서 실행됩니다. 이렇게 하면 서버 컴포넌트는 개인 데이터 및 API에 액세스할 수 있지만 클라이언트 컴포넌트는 액세스할 수 없습니다.

**Server Components:**
* 서버에서만 실행하세요.
* 환경 변수, 비밀, 데이터베이스, 내부 API에 안전하게 액세스할 수 있습니다.

**Client Components:**
* 사전 렌더링 중에 서버에서 실행되지만 브라우저에서 실행되는 코드와 동일한 보안 가정을 ​​따라야 합니다.
* 권한 있는 데이터 또는 서버 전용 모듈에 액세스해서는 안 됩니다.

이렇게 하면 기본적으로 앱이 안전하지만 데이터를 가져오거나 구성 요소에 전달하는 방식을 통해 실수로 개인 데이터가 노출될 수 있습니다.

### Tainting
개인 데이터가 실수로 클라이언트에 노출되는 것을 방지하기 위해 React Taint API를 사용할 수 있습니다:

* 데이터 객체용 [`experimental_taintObjectReference`](https://react.dev/reference/react/experimental_taintObjectReference).
* 특정 값의 경우 [`experimental_taintUniqueValue`](https://react.dev/reference/react/experimental_taintUniqueValue).

`next.config.js`의 [`experimental.taint`](/docs/app/api-reference/config/next-config-js/taint) 옵션을 사용하여 Next.js 앱에서 사용을 활성화할 수 있습니다:

next.config.js
```js
module.exports = {
  experimental: {
    taint: true,
  },
}
```

이렇게 하면 오염된 개체나 값이 클라이언트에 전달되는 것을 방지할 수 있습니다. 그러나 이는 추가 보호 계층이므로 React의 렌더링 컨텍스트에 전달하기 전에 [DAL](#data-access-layer)의 데이터를 필터링하고 삭제해야 합니다.

> **알아두면 좋은 점:**
>
> * 기본적으로 환경 변수는 서버에서만 사용할 수 있습니다. Next.js는 `NEXT_PUBLIC_` 접두사가 붙은 모든 환경 변수를 클라이언트에 노출합니다. [자세히 알아보기](/docs/app/guides/environment-variables)
> * 함수와 클래스는 기본적으로 클라이언트 컴포넌트에 전달되지 않도록 이미 차단되어 있습니다.

### Preventing client-side execution of server-only code
서버 전용 코드가 클라이언트에서 실행되는 것을 방지하려면 [`서버 전용`](https://www.npmjs.com/package/server-only) 패키지로 모듈을 표시하면 됩니다.:

lib/data.ts
```ts
import 'server-only'

//...
```

이렇게 하면 클라이언트 환경에서 모듈을 가져오는 경우 빌드 오류가 발생하여 독점 코드 또는 내부 비즈니스 논리가 서버에 유지됩니다.

Next.js는 `서버 전용` 가져오기를 내부적으로 처리합니다. NPM의 이러한 패키지 내용은 사용되지 않습니다. 그러나 Linting 규칙이 외부 종속성을 표시하는 경우 문제를 방지하기 위해 해당 규칙을 설치할 수 있습니다.

```bash
npm install server-only
```

[환경 오염 방지](/docs/app/getting-started/server-and-client-comComponents#preventing-environment-poisoning) 섹션에서 `서버 전용`에 대해 자세히 알아보세요.

## Mutating Data
Next.js는 [Server Action](https://react.dev/reference/rsc/server-functions)을 사용하여 변형을 처리합니다.

### Built-in Server Actions Security features
기본적으로 Server Action을 만들고 내보낼 때 애플리케이션의 UI뿐만 아니라 직접 POST 요청을 통해 연결할 수 있습니다. 즉, Server Action이나 유틸리티 기능을 코드의 다른 곳으로 가져오지 않더라도 외부에서 계속 호출할 수 있습니다.

보안을 강화하기 위해 Next.js에는 다음과 같은 내장 기능이 있습니다:
* **보안 작업 ID:** Next.js는 클라이언트가 Server Action을 참조하고 호출할 수 있도록 암호화된 비결정적 ID를 생성합니다. 이러한 ID는 보안 강화를 위해 빌드 간에 주기적으로 다시 계산됩니다.
* **데드 코드 제거:** 공개 액세스를 방지하기 위해 사용되지 않는 Server Action(ID로 참조됨)이 클라이언트 번들에서 제거됩니다.

> **알아두면 좋은 점**:
>
> ID는 컴파일 중에 생성되며 최대 14일 동안 캐시됩니다. 새 빌드가 시작되거나 빌드 캐시가 무효화되면 다시 생성됩니다.
> 이러한 보안 개선은 인증 레이어가 누락된 경우의 위험을 줄여줍니다. 그러나 여전히 Server Action을 직접 POST 요청을 통해 도달할 수 있는 것으로 처리하고 각 작업 내에서 인증 및 권한 부여를 확인해야 합니다.

```jsx
// app/actions.js
'use server'

// 이 작업이 애플리케이션에서 사용되는 경우 Next.js는 클라이언트가 Server Action을
// 참조하고 호출할 수 있도록 보안 ID를 생성합니다.
export async function updateUserAction(formData) {}

// 이 작업이 애플리케이션에서 사용되지 **않으면** Next.js는 `next build` 중에
// 이 코드를 자동으로 제거하고 공개 엔드포인트를 생성하지 않습니다.
export async function deleteUserAction(formData) {}
```

### Validating client input
클라이언트의 입력은 쉽게 수정될 수 있으므로 항상 클라이언트의 입력을 검증해야 합니다. 예를 들어 form data, URL 매개변수, headers, searchParams 등이 있습니다:

app/page.tsx
```tsx
// BAD: searchParams를 직접 신뢰하기
export default async function Page({ searchParams }) {
  const isAdmin = (await searchParams).isAdmin
  if (isAdmin === 'true') {
    // 취약함: 신뢰할 수 없는 클라이언트 데이터에 의존
    return <AdminPanel />
  }
}

// GOOD: 매번 다시 확인하세요
import { cookies } from 'next/headers'
import { verifyAdmin } from './auth'

export default async function Page() {
  const cookieStore = await cookies()
  const token = cookieStore.get('AUTH_TOKEN')
  const isAdmin = await verifyAdmin(token)

  if (isAdmin) {
    return <AdminPanel />
  }
}
```

### Authentication and authorization
페이지 수준 인증 확인은 그 안에 정의된 Server Action으로 확장되지 않습니다. 항상 작업 내부에서 다시 확인하세요:

```tsx highlight={13,14,15,16}
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'

export default async function AdminPage() {
  const session = await auth()
  if (!session?.user?.isAdmin) {
    redirect('/login')
  }

  return (
    <form
      action={async () => {
        'use server'
        const session = await auth()
        if (!session?.user?.isAdmin) {
          throw new Error('Unauthorized')
        }
        await db.record.deleteMany()
      }}
    >
      <button>Delete Records</button>
    </form>
  )
}
```

작업 내부에 강조 표시된 `auth()` 확인이 중요합니다. 6행의 페이지 수준 리디렉션은 렌더링되는 UI를 제어하지만 Server Action은 별도의 진입점이므로 호출자를 자체적으로 확인해야 합니다.

인증(사용자가 로그인 여부) 외에도 **authorization**(이 사용자가 이 특정 리소스에 대해 작업할 권한이 있는지)을 확인하는 것을 잊지 마십시오. 이렇게 하면 [안전하지 않은 IDOR(직접 개체 참조)](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) 취약성을 방지할 수 있습니다.:

app/actions.ts
```tsx
'use server'

import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

export async function deletePost(postId: string) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }

  const post = await db.post.findUnique({ where: { id: postId } })

  // 사용자가 이 리소스를 소유하고 있는지 확인하세요.
  if (post.authorId !== session.user.id) {
    throw new Error('Forbidden')
  }

  await db.post.delete({ where: { id: postId } })
}
```

Next.js의 [Authentication](/docs/app/guides/authentication)에 대해 자세히 알아보세요.

### Using a Data Access Layer for mutations
Just as we recommend a [Data Access Layer](#data-access-layer) for reading data, you can apply the same pattern to mutations. This keeps authentication, authorization, and database logic in a dedicated `server-only` module, while `"use server"` actions stay thin.

data/posts.ts
```ts
import 'server-only'

import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

export async function deletePost(postId: string) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }

  const post = await db.post.findUnique({ where: { id: postId } })

  if (post.authorId !== session.user.id) {
    throw new Error('Forbidden')
  }

  await db.post.delete({ where: { id: postId } })
}
```

그런 다음 `"user server"` 작업이 DAL에 위임됩니다:

app/actions.ts
```ts
'use server'

import { deletePost } from '@/data/posts'
import { revalidatePath } from 'next/cache'

export async function deletePostAction(postId: string) {
  await deletePost(postId) // Auth + authz 인증은 DAL 내부에서 발생합니다.
  revalidatePath('/posts')
}
```

> **알아두면 좋은 점:** 데이터 액세스 계층과 `"use server"` 파일 자체 모두에서 `import 'server-only'`를 사용할 수 있습니다. 둘 다 작업을 클라이언트 컴포넌트로 가져올 때(예: `useActionState`에 전달하기 위해) 작동합니다. `"use server'` 모듈은 서버 전용 웹팩 레이어에서 확인되기 때문입니다.

### Controlling return values
Server Action 반환 값은 직렬화되어 클라이언트로 전송됩니다. 원시 데이터베이스 레코드가 아닌 UI에 필요한 것만 반환합니다.

app/actions.ts
```tsx
'use server'

import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

// BAD: 클라이언트가 볼 수 없는 내부 필드를 포함할 수 있는 전체 데이터베이스 레코드를 반환합니다.
export async function updateUser(data: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
  return db.user.update({
    where: { id: session.user.id },
    data: { name: data.get('name') as string },
  })
}

// GOOD: 클라이언트에 필요한 것만 반환합니다.
export async function updateUserSafe(data: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }
  await db.user.update({
    where: { id: session.user.id },
    data: { name: data.get('name') as string },
  })
  return { success: true }
}
```

### Rate limiting
비용이 많이 드는 작업(이메일 보내기, 데이터베이스에 쓰기)의 경우 남용을 방지하기 위해 속도 제한을 추가하는 것이 좋습니다. 프런트엔드용 백엔드 가이드의 [속도 제한](/docs/app/guides/backend-for-frontend#rate-limiting) 예시를 참조하세요.

### Closures and encryption
구성 요소 내부에 서버 작업을 정의하면 작업이 외부 함수 범위에 액세스할 수 있는 [클로저](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)가 생성됩니다. 예를 들어 `publish` 작업은 `publishVersion` 변수에 액세스할 수 있습니다.:

app/page.tsx
```tsx
export default async function Page() {
  const publishVersion = await getLatestVersion();

  async function publish() {
    "use server";
    if (publishVersion !== await getLatestVersion()) {
      throw new Error('The version has changed since pressing publish');
    }
    ...
  }

  return (
    <form>
      <button formAction={publish}>Publish</button>
    </form>
  );
}
```

클로저는 나중에 작업이 호출될 때 사용할 수 있도록 렌더링 시 데이터의 *스냅샷*(예: `publishVersion`)을 캡처해야 할 때 유용합니다.

그러나 이를 위해서는 작업이 호출될 때 캡처된 변수가 클라이언트로 전송되었다가 다시 서버로 전송됩니다. 민감한 데이터가 클라이언트에 노출되는 것을 방지하기 위해 Next.js는 폐쇄형 변수를 자동으로 암호화합니다. Next.js 애플리케이션이 구축될 때마다 각 작업에 대해 새로운 개인 키가 생성됩니다. 이는 특정 빌드에 대해서만 작업을 호출할 수 있음을 의미합니다.

> **알아두면 좋은 점:** 중요한 값이 클라이언트에 노출되는 것을 방지하기 위해 암호화에만 의존하지 않는 것이 좋습니다.

### Overwriting encryption keys (advanced)
여러 서버에 걸쳐 Next.js 애플리케이션을 **자체 호스팅**할 때 각 서버 인스턴스는 서로 다른 암호화 키로 종료되어 잠재적인 불일치가 발생할 수 있습니다.

이를 완화하려면 `process.env.NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` 환경 변수를 사용하여 암호화 키를 덮어쓸 수 있습니다. 이 변수를 지정하면 암호화 키가 빌드 전반에 걸쳐 지속되고 모든 서버 인스턴스가 동일한 키를 사용하게 됩니다.

키는 디코딩된 길이가 유효한 AES 키 크기(16, 24 또는 32바이트)와 일치하는 base64로 인코딩된 값이어야 합니다. Next.js는 기본적으로 32바이트 키를 생성합니다. 플랫폼의 암호화 도구를 사용하여 호환 가능한 키를 생성할 수 있습니다. 예를 들면 다음과 같습니다:

```bash
openssl rand -base64 32
```

이는 여러 배포에서 일관된 암호화 동작이 애플리케이션에 중요한 고급 사용 사례입니다. 키 순환 및 서명과 같은 표준 보안 관행을 따르십시오. 배포별 고려 사항은 [셀프 호스팅 가이드](/docs/app/guides/self-hosting#server-functions-encryption-key)를 참조하세요.

### Allowed origins (advanced)
Server Action은 `<form>` 요소에서 호출될 수 있으므로 [CSRF 공격](https://developer.mozilla.org/en-US/docs/Glossary/CSRF)에 노출됩니다.

뒤에서 Server Action은 `POST` 메서드를 사용하며 이 HTTP 메서드만 호출할 수 있습니다. 이는 특히 [SameSite 쿠키](https://web.dev/articles/samesite-cookies-explained)가 기본값인 최신 브라우저에서 대부분의 CSRF 취약성을 방지합니다.

추가 보호로서 Next.js의 Server Action은 [Origin 헤더](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin)를 [호스트 헤더](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Host)(또는 `X-Forwarded-Host`)와 비교합니다. 일치하지 않으면 요청이 중단됩니다. 즉, Server Action은 이를 호스팅하는 페이지와 동일한 호스트에서만 호출될 수 있습니다.

역방향 프록시 또는 다중 계층 백엔드 아키텍처(서버 API가 프로덕션 도메인과 다름)를 사용하는 대규모 애플리케이션의 경우 구성 옵션 [`serverActions.allowedOrigins`](/docs/app/api-reference/config/next-config-js/serverActions) 옵션을 사용하여 안전한 원본 목록을 지정하는 것이 좋습니다. 이 옵션은 문자열 배열을 허용합니다.

next.config.js
```js
/** @type {import('next').NextConfig} */
module.exports = {
  experimental: {
    serverActions: {
      allowedOrigins: ['my-proxy.com', '*.my-proxy.com'],
    },
  },
}
```

[보안 및 서버 작업](https://nextjs.org/blog/security-nextjs-server-comComponents-actions)에 대해 자세히 알아보세요.

### Avoiding side-effects during rendering
변형(예: 사용자 로그아웃, 데이터베이스 업데이트, 캐시 무효화)은 서버 또는 클라이언트 컴포넌트에서 결코 부작용이 되어서는 안 됩니다. Next.js는 의도하지 않은 부작용을 피하기 위해 렌더링 메서드 내에서 쿠키 설정이나 캐시 재검증 트리거를 명시적으로 방지합니다.

app/page.tsx
```tsx
// BAD: 렌더링 중 Mutation 유발
export default async function Page({ searchParams }) {
  if ((await searchParams).logout) {
    const cookieStore = await cookies()
    cookieStore.delete('AUTH_TOKEN')
  }

  return <UserProfile />
}
```

대신 Server Action을 사용하여 Mutation을 처리해야 합니다.

app/page.tsx
```tsx
// GOOD: 서버 작업을 사용하여 변형 처리
import { logout } from './actions'

export default function Page() {
  return (
    <>
      <UserProfile />
      <form action={logout}>
        <button type="submit">Logout</button>
      </form>
    </>
  )
}
```

> **알아두면 좋은 점:** Next.js는 `POST` 요청을 사용하여 변형을 처리합니다. 이를 통해 GET 요청으로 인한 우발적인 부작용을 방지하고 CSRF(Cross-Site Request Forgery) 위험을 줄일 수 있습니다.

## Auditing
Next.js 프로젝트를 감사하는 경우 추가로 살펴보는 것이 좋습니다:
* **Data Access Layer:** 격리된 데이터 액세스 계층에 대한 확립된 관행이 있습니까? 데이터베이스 패키지와 환경 변수를 데이터 액세스 계층 외부로 가져오지 않았는지 확인하세요.
* **`"use client"` files:** 컴포넌트 props가 개인 데이터를 기대합니까? 유형 서명이 지나치게 광범위합니까?
* **`"use server"` files:** 작업 인수가 작업 내에서 또는 데이터 액세스 계층 내에서 검증됩니까? 사용자가 작업 내에서 다시 인증을 받았나요? 작업이 리소스의 소유권(인증뿐만 아니라 승인)을 확인합니까? 반환 값은 클라이언트에 필요한 것만 필터링됩니까? 데이터베이스 액세스가 `server-only` 데이터 액세스 레이어에 위임됩니까?
* **`/[param]/.`** 대괄호가 있는 폴더는 사용자 입력입니다. 매개변수가 검증되었나요?
* **`proxy.ts` and `route.ts`:** 많은 힘을 가지세요. 전통적인 기술을 사용하여 이를 감사하는 데 추가 시간을 할애하십시오. 정기적으로 또는 팀의 소프트웨어 개발 수명주기에 맞춰 침투 테스트 또는 취약점 검색을 수행합니다.