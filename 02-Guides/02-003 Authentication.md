# Next.js에서 인증을 구현하는 방법
<span style="color: #808080;">Last updated June 23, 2026</span>

애플리케이션 데이터를 보호하려면 인증을 이해하는 것이 중요합니다. 이 페이지에서는 인증을 구현하는 데 사용할 React 및 Next.js 기능을 안내합니다.

시작하기 전에 프로세스를 세 가지 개념으로 나누는 것이 도움이 됩니다:

1. **[Authentication](#authentication)**: 사용자가 자신이 말하는 사람인지 확인합니다. 사용자는 사용자 이름, 비밀번호 등 자신이 가지고 있는 정보를 사용하여 자신의 신원을 증명해야 합니다.
2. **[Session Management](#session-management)**: 요청 전반에 걸쳐 사용자의 인증 상태를 추적합니다.
3. **[Authorization](#authorization)**: 사용자가 액세스할 수 있는 경로와 데이터를 결정합니다.

이 다이어그램은 React 및 Next.js 기능을 사용한 인증 흐름을 보여줍니다:

![Diagram showing the authentication flow with React and Next.js features](https://h8DxKfmAPhn8O0p3.public.blob.vercel-storage.com/docs/light/authentication-overview.png)

이 페이지의 예에서는 교육 목적의 기본 사용자 이름 및 비밀번호 인증을 안내합니다. 사용자 정의 인증 솔루션을 구현할 수도 있지만 보안과 단순성을 높이기 위해 인증 라이브러리를 사용하는 것이 좋습니다. 이는 인증, 세션 관리 및 권한 부여를 위한 내장 솔루션은 물론 소셜 로그인, 다단계 인증, 역할 기반 액세스 제어와 같은 추가 기능을 제공합니다. [인증 라이브러리](#auth-libraries) 섹션에서 목록을 찾을 수 있습니다.

## Authentication

### Sign-up and login functionality

React의 [서버 작업](/docs/app/getting-started/mutating-data) 및 `useActionState`와 함께 [`<form>`](https://react.dev/reference/react-dom/comComponents/form) 요소를 사용하여 사용자 자격 증명을 캡처하고, 양식 필드의 유효성을 검사하고, 인증 공급자의 API 또는 데이터베이스를 호출할 수 있습니다.

서버 작업은 항상 서버에서 실행되므로 인증 논리 처리를 위한 보안 환경을 제공합니다.

가입/로그인 기능을 구현하는 단계는 다음과 같습니다:

#### 1. Capture user credentials

사용자 자격 증명을 캡처하려면 제출 시 서버 작업을 호출하는 양식을 만듭니다. 예를 들어 사용자 이름, 이메일, 비밀번호를 허용하는 가입 양식:

app/ui/signup-form.tsx
```tsx
import { signup } from '@/app/actions/auth'

export function SignupForm() {
  return (
    <form action={signup}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" placeholder="Name" />
      </div>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" placeholder="Email" />
      </div>
      <div>
        <label htmlFor="password">Password</label>
        <input id="password" name="password" type="password" />
      </div>
      <button type="submit">Sign Up</button>
    </form>
  )
}
```

app/ui/signup-form.js
```jsx
import { signup } from '@/app/actions/auth'

export function SignupForm() {
  return (
    <form action={signup}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" placeholder="Name" />
      </div>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" placeholder="Email" />
      </div>
      <div>
        <label htmlFor="password">Password</label>
        <input id="password" name="password" type="password" />
      </div>
      <button type="submit">Sign Up</button>
    </form>
  )
}
```

app/actions/auth.ts
```tsx
export async function signup(formData: FormData) {}
```

app/actions/auth.js
```jsx
export async function signup(formData) {}
```

#### 2. Validate form fields on the server

서버 작업을 사용하여 서버의 양식 필드를 확인하세요. 인증 공급자가 양식 유효성 검사를 제공하지 않는 경우 [Zod](https://zod.dev/) 또는 [Yup](https://github.com/jqueense/yup)과 같은 스키마 유효성 검사 라이브러리를 사용할 수 있습니다.

Zod를 예로 사용하면 적절한 오류 메시지가 포함된 양식 스키마를 정의할 수 있습니다:

app/lib/definitions.ts
```ts
import * as z from 'zod'

export const SignupFormSchema = z.object({
  name: z
    .string()
    .min(2, { error: 'Name must be at least 2 characters long.' })
    .trim(),
  email: z.email({ error: 'Please enter a valid email.' }).trim(),
  password: z
    .string()
    .min(8, { error: 'Be at least 8 characters long' })
    .regex(/[a-zA-Z]/, { error: 'Contain at least one letter.' })
    .regex(/[0-9]/, { error: 'Contain at least one number.' })
    .regex(/[^a-zA-Z0-9]/, {
      error: 'Contain at least one special character.',
    })
    .trim(),
})

export type FormState =
  | {
      errors?: {
        name?: string[]
        email?: string[]
        password?: string[]
      }
      message?: string
    }
  | undefined
```

app/lib/definitions.js
```js
import * as z from 'zod'

export const SignupFormSchema = z.object({
  name: z
    .string()
    .min(2, { error: 'Name must be at least 2 characters long.' })
    .trim(),
  email: z.email({ error: 'Please enter a valid email.' }).trim(),
  password: z
    .string()
    .min(8, { error: 'Be at least 8 characters long' })
    .regex(/[a-zA-Z]/, { error: 'Contain at least one letter.' })
    .regex(/[0-9]/, { error: 'Contain at least one number.' })
    .regex(/[^a-zA-Z0-9]/, {
      error: 'Contain at least one special character.',
    })
    .trim(),
})
```

인증 공급자의 API 또는 데이터베이스에 대한 불필요한 호출을 방지하려면 양식 필드가 정의된 스키마와 일치하지 않는 경우 서버 작업 초기에 `반환`할 수 있습니다.

```ts filename="app/actions/auth.ts" switcher
import { SignupFormSchema, FormState } from '@/app/lib/definitions'

export async function signup(state: FormState, formData: FormData) {
  // 양식 필드 유효성 검사
  const validatedFields = SignupFormSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    password: formData.get('password'),
  })

  // 잘못된 양식 필드가 있으면 일찍 반환하세요.
  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors,
    }
  }

  // 사용자를 생성하려면 공급자나 DB에 전화하세요...
}
```

app/actions/auth.js
```js
import { SignupFormSchema } from '@/app/lib/definitions'

export async function signup(state, formData) {
  // 양식 필드 유효성 검사
  const validatedFields = SignupFormSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    password: formData.get('password'),
  })

  // 잘못된 양식 필드가 있으면 일찍 반환하세요.
  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors,
    }
  }

  // 사용자를 생성하려면 공급자나 DB에 전화하세요...
}
```

`<SignupForm />`으로 돌아가서 React의 `useActionState` 훅을 사용하여 양식을 제출하는 동안 유효성 검사 오류를 표시할 수 있습니다:

app/ui/signup-form.tsx
```tsx
'use client'

import { signup } from '@/app/actions/auth'
import { useActionState } from 'react'

export default function SignupForm() {
  const [state, action, pending] = useActionState(signup, undefined)

  return (
    <form action={action}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" placeholder="Name" />
      </div>
      {state?.errors?.name && <p>{state.errors.name}</p>}

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" placeholder="Email" />
      </div>
      {state?.errors?.email && <p>{state.errors.email}</p>}

      <div>
        <label htmlFor="password">Password</label>
        <input id="password" name="password" type="password" />
      </div>
      {state?.errors?.password && (
        <div>
          <p>Password must:</p>
          <ul>
            {state.errors.password.map((error) => (
              <li key={error}>- {error}</li>
            ))}
          </ul>
        </div>
      )}
      <button disabled={pending} type="submit">
        Sign Up
      </button>
    </form>
  )
}
```

app/ui/signup-form.js
```jsx
'use client'

import { signup } from '@/app/actions/auth'
import { useActionState } from 'react'

export default function SignupForm() {
  const [state, action, pending] = useActionState(signup, undefined)

  return (
    <form action={action}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" placeholder="Name" />
      </div>
      {state?.errors?.name && <p>{state.errors.name}</p>}

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" placeholder="Email" />
      </div>
      {state?.errors?.email && <p>{state.errors.email}</p>}

      <div>
        <label htmlFor="password">Password</label>
        <input id="password" name="password" type="password" />
      </div>
      {state?.errors?.password && (
        <div>
          <p>Password must:</p>
          <ul>
            {state.errors.password.map((error) => (
              <li key={error}>- {error}</li>
            ))}
          </ul>
        </div>
      )}
      <button disabled={pending} type="submit">
        Sign Up
      </button>
    </form>
  )
}
```

> **알아두면 좋은 점:**
>
> * React 19에서 `useFormStatus`에는 데이터, 메서드, 작업과 같은 반환된 개체에 대한 추가 키가 포함됩니다. React 19를 사용하지 않는 경우 `pending` 키만 사용할 수 있습니다.
> * 데이터를 변경하기 전에 항상 사용자에게 작업을 수행할 권한이 있는지 확인해야 합니다. [인증 및 승인](#authorization)을 참조하세요.

#### 3. Create a user or check user credentials

양식 필드를 확인한 후 새 사용자 계정을 생성하거나 인증 공급자의 API 또는 데이터베이스를 호출하여 사용자가 존재하는지 확인할 수 있습니다.

이전 예에 이어서:

app/actions/auth.tsx
```tsx
export async function signup(state: FormState, formData: FormData) {
  // 1. 양식 필드 유효성 검사
  // ...

  // 2. 데이터베이스에 삽입할 데이터 준비
  const { name, email, password } = validatedFields.data
  // 예를 들어 사용자의 비밀번호를 저장하기 전에 해시하세요.
  const hashedPassword = await bcrypt.hash(password, 10)

  // 3. 사용자를 데이터베이스에 삽입하거나 인증 라이브러리의 API를 호출하세요.
  const data = await db
    .insert(users)
    .values({
      name,
      email,
      password: hashedPassword,
    })
    .returning({ id: users.id })

  const user = data[0]

  if (!user) {
    return {
      message: 'An error occurred while creating your account.',
    }
  }

  // TODO:
  // 4. 사용자 세션 생성
  // 5. 사용자 리디렉션
}
```

app/actions/auth.js
```jsx
export async function signup(state, formData) {
  // 1. 양식 필드 유효성 검사
  // ...

  // 2. 데이터베이스에 삽입할 데이터 준비
  const { name, email, password } = validatedFields.data
  // 예를 들어 사용자의 비밀번호를 저장하기 전에 해시하세요.
  const hashedPassword = await bcrypt.hash(password, 10)

  // 3. 사용자를 데이터베이스에 삽입하거나 라이브러리 API를 호출합니다.
  const data = await db
    .insert(users)
    .values({
      name,
      email,
      password: hashedPassword,
    })
    .returning({ id: users.id })

  const user = data[0]

  if (!user) {
    return {
      message: 'An error occurred while creating your account.',
    }
  }

  // TODO:
  // 4. 사용자 세션 생성
  // 5. 사용자 리디렉션
}
```

사용자 계정을 성공적으로 생성하거나 사용자 자격 증명을 확인한 후 세션을 생성하여 사용자의 인증 상태를 관리할 수 있습니다. 세션 관리 전략에 따라 세션은 쿠키나 데이터베이스 또는 둘 다에 저장될 수 있습니다. 자세히 알아보려면 [세션 관리](#session-management) 섹션을 계속 진행하세요.

> **Tips:**
>
> * 위의 예는 교육 목적으로 인증 단계를 세분화했기 때문에 장황합니다. 이는 자체 보안 솔루션을 구현하는 것이 빠르게 복잡해질 수 있다는 점을 강조합니다. 프로세스를 단순화하려면 [인증 라이브러리](#auth-libraries)를 사용하는 것이 좋습니다.
> * 사용자 경험을 개선하려면 등록 절차 초기에 중복된 이메일이나 사용자 이름이 있는지 확인하는 것이 좋습니다. 예를 들어, 사용자가 사용자 이름을 입력하거나 입력 필드가 포커스를 잃습니다. 이를 통해 불필요한 양식 제출을 방지하고 사용자에게 즉각적인 피드백을 제공할 수 있습니다. [use-debounce](https://www.npmjs.com/package/use-debounce)와 같은 라이브러리를 사용하여 요청을 디바운스하여 이러한 확인 빈도를 관리할 수 있습니다.

## Session Management

세션 관리는 사용자의 인증된 상태가 요청 전반에 걸쳐 보존되도록 보장합니다. 여기에는 세션 또는 토큰 생성, 저장, 새로 고침 및 삭제가 포함됩니다.

세션에는 두 가지 유형이 있습니다:

1. [**Stateless**](#stateless-sessions): 세션 데이터(또는 토큰)는 브라우저의 쿠키에 저장됩니다. 쿠키는 각 요청과 함께 전송되므로 서버에서 세션을 확인할 수 있습니다. 이 방법은 더 간단하지만 올바르게 구현되지 않으면 보안이 떨어질 수 있습니다.
2. [**Database**](#database-sessions): 세션 데이터는 데이터베이스에 저장되며 사용자의 브라우저는 암호화된 세션 ID만 수신합니다. 이 방법은 더 안전하지만 복잡할 수 있고 더 많은 서버 리소스를 사용할 수 있습니다.

> **알아두면 좋은 점:** 두 가지 방법 중 하나 또는 둘 다를 사용할 수 있지만 [iron-session](https://github.com/vvo/iron-session) 또는 [Jose](https://github.com/panva/jose)와 같은 세션 관리 라이브러리를 사용하는 것이 좋습니다.

### Stateless Sessions

상태 비저장 세션을 생성하고 관리하려면 따라야 할 몇 가지 단계가 있습니다:

1. 세션 서명에 사용할 비밀 키를 생성하고 이를 [환경 변수](/docs/app/guides/environment-variables)로 저장합니다.
2. 세션 관리 라이브러리를 사용하여 세션 데이터를 암호화/해독하는 논리를 작성합니다.
3. Next.js [`cookies`](/docs/app/api-reference/functions/cookies) API를 사용하여 쿠키를 관리하세요.

위의 내용 외에도 사용자가 애플리케이션으로 돌아올 때 세션을 [update(or refresh)](https://nextjs.org/docs/app/guides/authentication#updating-or-refreshing-sessions)하고, 사용자가 로그아웃할 때 세션을 [delete](https://nextjs.org/docs/app/guides/authentication#deleting-the-session)하는 기능을 추가하는 것을 고려하세요.

> **알아두면 좋은 점:** [인증 라이브러리](#auth-libraries)에 세션 관리가 포함되어 있는지 확인하세요.

#### 1. Generating a secret key

세션에 서명하기 위해 비밀 키를 생성할 수 있는 몇 가지 방법이 있습니다. 예를 들어, 터미널에서 `openssl` 명령을 사용하도록 선택할 수 있습니다:

terminal
```bash
openssl rand -base64 32
```

이 명령어는 비밀 키로 사용하고 [환경 변수 파일](/docs/app/guides/environment-variables)에 저장할 수 있는 32자 임의 문자열을 생성합니다:

.env
```bash
SESSION_SECRET=your_secret_key
```

그런 다음 세션 관리 로직에서 이 키를 참조할 수 있습니다:

app/lib/session.js
```js
const secretKey = process.env.SESSION_SECRET
```

#### 2. Encrypting and decrypting sessions

다음으로, 선호하는 [세션 관리 라이브러리](https://nextjs.org/docs/app/guides/authentication#session-management-libraries)를 사용하여 세션을 암호화하고 해독할 수 있습니다. 이전 예에 이어 [Jose](https://www.npmjs.com/package/jose)([Edge Runtime](https://nextjs.org/docs/app/api-reference/edge)과 호환 가능) 및 React의 [`서버 전용`](https://www.npmjs.com/package/server-only) 패키지를 사용하여 세션 관리 로직이 서버에서만 실행되도록 하겠습니다.

app/lib/session.ts
```tsx
import 'server-only'
import { SignJWT, jwtVerify } from 'jose'
import { SessionPayload } from '@/app/lib/definitions'

const secretKey = process.env.SESSION_SECRET
const encodedKey = new TextEncoder().encode(secretKey)

export async function encrypt(payload: SessionPayload) {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d')
    .sign(encodedKey)
}

export async function decrypt(session: string | undefined = '') {
  try {
    const { payload } = await jwtVerify(session, encodedKey, {
      algorithms: ['HS256'],
    })
    return payload
  } catch (error) {
    console.log('Failed to verify session')
  }
}
```

app/lib/session.js
```jsx
import 'server-only'
import { SignJWT, jwtVerify } from 'jose'

const secretKey = process.env.SESSION_SECRET
const encodedKey = new TextEncoder().encode(secretKey)

export async function encrypt(payload) {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d')
    .sign(encodedKey)
}

export async function decrypt(session) {
  try {
    const { payload } = await jwtVerify(session, encodedKey, {
      algorithms: ['HS256'],
    })
    return payload
  } catch (error) {
    console.log('Failed to verify session')
  }
}
```

> **Tips**:
>
> * 페이로드에는 사용자 ID, 역할 등 후속 요청에 사용되는 **최소** 고유 사용자 데이터가 포함되어야 합니다. 전화번호, 이메일 주소, 신용 카드 정보 등과 같은 개인 식별 정보나 비밀번호와 같은 민감한 데이터는 포함되어서는 안 됩니다.

#### 3. Setting cookies (recommended options)

세션을 쿠키에 저장하려면 Next.js [`cookies`](/docs/app/api-reference/functions/cookies) API를 사용하세요. 쿠키는 서버에 설정되어야 하며 권장 옵션을 포함해야 합니다:

* **HttpOnly**: 클라이언트 측 JavaScript가 쿠키에 액세스하는 것을 방지합니다.
* **Secure**: 쿠키를 보내려면 https를 사용하십시오.
* **SameSite**: 크로스 사이트 요청과 함께 쿠키를 보낼 수 있는지 여부를 지정합니다.
* **Max-Age or Expires**: 일정 기간이 지나면 쿠키를 삭제합니다.
* **Path**: 쿠키의 URL 경로를 정의합니다.

각 옵션에 대한 자세한 내용은 [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)을 참조하세요.

app/lib/session.ts
```ts
import 'server-only'
import { cookies } from 'next/headers'

export async function createSession(userId: string) {
  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  const session = await encrypt({ userId, expiresAt })
  const cookieStore = await cookies()

  cookieStore.set('session', session, {
    httpOnly: true,
    secure: true,
    expires: expiresAt,
    sameSite: 'lax',
    path: '/',
  })
}
```

서버 작업으로 돌아가서 `createSession()` 함수를 호출하고 [`redirect()`](/docs/app/guides/redirecting) API를 사용하여 사용자를 적절한 페이지로 리디렉션할 수 있습니다:

app/actions/auth.ts
```ts
import { createSession } from '@/app/lib/session'

export async function signup(state: FormState, formData: FormData) {
  // 이전 단계: 
  // 1. 양식 필드 유효성 검사
  // 2. 데이터베이스에 삽입할 데이터 준비
  // 3. 사용자를 데이터베이스에 삽입하거나 라이브러리 API를 호출합니다.

  // 현재 단계:
  // 4. 사용자 세션 생성
  await createSession(user.id)
  // 5. 사용자 리디렉션
  redirect('/profile')
}
```

> **Tips**:
>
> * **쿠키는 클라이언트 측 변조를 방지하기 위해 서버에 설정되어야 합니다**.
> * 🎥 시청: Next.js를 사용한 무상태 세션 및 인증에 대해 자세히 알아보세요. → [YouTube (11 minutes)](https://www.youtube.com/watch?v=DJvM2lSPn6w).

#### Updating (or refreshing) sessions

세션 만료 시간을 연장할 수도 있습니다. 이는 사용자가 애플리케이션에 다시 액세스한 후에도 로그인 상태를 유지하는 데 유용합니다. 예를 들어:

app/lib/session.ts
```ts
import 'server-only'
import { cookies } from 'next/headers'
import { decrypt } from '@/app/lib/session'

export async function updateSession() {
  const session = (await cookies()).get('session')?.value
  const payload = await decrypt(session)

  if (!session || !payload) {
    return null
  }

  const expires = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)

  const cookieStore = await cookies()
  cookieStore.set('session', session, {
    httpOnly: true,
    secure: true,
    expires: expires,
    sameSite: 'lax',
    path: '/',
  })
}
```

> **Tip:** 인증 라이브러리가 사용자 세션을 확장하는 데 사용할 수 있는 새로 고침 토큰을 지원하는지 확인하세요.

#### Deleting the session

세션을 삭제하려면 쿠키를 삭제하면 됩니다:

app/lib/session.ts
```ts
import 'server-only'
import { cookies } from 'next/headers'

export async function deleteSession() {
  const cookieStore = await cookies()
  cookieStore.delete('session')
}
```

그런 다음 애플리케이션에서 `deleteSession()` 함수를 재사용할 수 있습니다(예: 로그아웃 시):

app/actions/auth.ts
```ts
import { cookies } from 'next/headers'
import { deleteSession } from '@/app/lib/session'

export async function logout() {
  await deleteSession()
  redirect('/login')
}
```

### Database Sessions
데이터베이스 세션을 생성하고 관리하려면 다음 단계를 따라야 합니다:

1. 세션과 데이터를 저장하기 위해 데이터베이스에 테이블을 만듭니다(또는 인증 라이브러리가 이를 처리하는지 확인하세요)
2. 세션을 삽입, 업데이트, 삭제하는 기능 구현
3. 세션 ID를 사용자 브라우저에 저장하기 전에 암호화하고 데이터베이스와 쿠키가 동기화 상태를 유지하는지 확인하세요(이는 선택 사항이지만 [프록시](#optimistic-checks-with-proxy-선택 사항)의 낙관적 인증 확인에 권장됩니다).

예를 들어:

app/lib/session.ts
```ts
import cookies from 'next/headers'
import { db } from '@/app/lib/db'
import { encrypt } from '@/app/lib/session'

export async function createSession(id: number) {
  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)

  // 1. 데이터베이스에 세션 생성
  const data = await db
    .insert(sessions)
    .values({
      userId: id,
      expiresAt,
    })
    // 세션 ID를 반환합니다.
    .returning({ id: sessions.id })

  const sessionId = data[0].id

  // 2. 세션 ID 암호화
  const session = await encrypt({ sessionId, expiresAt })

  // 3. 낙관적 인증 확인을 위해 세션을 쿠키에 저장합니다.
  const cookieStore = await cookies()
  cookieStore.set('session', session, {
    httpOnly: true,
    secure: true,
    expires: expiresAt,
    sameSite: 'lax',
    path: '/',
  })
}
```

> **Tips**:
>
> * 더 빠른 액세스를 위해 세션 수명 동안 서버 캐싱을 추가하는 것을 고려할 수 있습니다. 세션 데이터를 기본 데이터베이스에 보관하고 데이터 요청을 결합하여 쿼리 수를 줄일 수도 있습니다.
> * 사용자가 마지막으로 로그인한 시간이나 활성 장치 수를 추적하거나 사용자에게 모든 장치에서 로그아웃할 수 있는 기능을 제공하는 등 고급 사용 사례에 대해 데이터베이스 세션을 사용하도록 선택할 수 있습니다.

세션 관리를 구현한 후에는 사용자가 애플리케이션 내에서 액세스하고 수행할 수 있는 작업을 제어하는 ​​권한 부여 논리를 추가해야 합니다. 자세히 알아보려면 [Authorization](#authorization) 섹션을 계속 진행하세요.

## Authorization

사용자가 인증되고 세션이 생성되면 사용자가 애플리케이션 내에서 액세스하고 수행할 수 있는 작업을 제어하는 ​​인증을 구현할 수 있습니다.

승인 확인에는 두 가지 주요 유형이 있습니다:

1. **Optimistic**: 사용자가 쿠키에 저장된 세션 데이터를 사용하여 경로에 액세스하거나 작업을 수행할 수 있는 권한이 있는지 확인합니다. 이러한 검사는 UI 요소 표시/숨기기 또는 권한이나 역할에 따라 사용자 리디렉션과 같은 빠른 작업에 유용합니다.
2. **Secure**: 사용자가 데이터베이스에 저장된 세션 데이터를 사용하여 경로에 액세스하거나 작업을 수행할 수 있는 권한이 있는지 확인합니다. 이러한 검사는 더욱 안전하며 중요한 데이터나 작업에 대한 액세스가 필요한 작업에 사용됩니다.

두 경우 모두에 권장됩니다:

* 인증 로직을 중앙 집중화하기 위한 [데이터 액세스 레이어](#creating-a-data-access-layer-dal) 생성
* [데이터 전송 개체(DTO)](#using-data-transfer-objects-dto)를 사용하여 필요한 데이터만 반환
* 선택적으로 [프록시](#optimistic-checks-with-proxy-ional)를 사용하여 낙관적 검사를 수행합니다.

### Optimistic checks with Proxy (Optional)

[프록시](/docs/app/api-reference/file-conventions/proxy)를 사용하고 권한에 따라 사용자를 리디렉션하려는 경우가 있습니다.:

* 낙관적인 검사를 수행합니다. 프록시는 모든 경로에서 실행되므로 리디렉션 논리를 중앙 집중화하고 승인되지 않은 사용자를 사전 필터링하는 좋은 방법입니다.
* 사용자 간에 데이터를 공유하는 정적 경로(예: 페이월 뒤의 콘텐츠)를 보호합니다.

그러나 프록시는 [prefetched](/docs/app/getting-started/linking-and-navigating#prefetching) 경로를 포함한 모든 경로에서 실행되므로 쿠키에서 세션만 읽고(낙관적 검사) 성능 문제를 방지하려면 데이터베이스 검사를 피하는 것이 중요합니다.

예를 들어:

proxy.ts
```tsx
import { NextRequest, NextResponse } from 'next/server'
import { decrypt } from '@/app/lib/session'
import { cookies } from 'next/headers'

// 1. 보호된 경로와 공개 경로 지정
const protectedRoutes = ['/dashboard']
const publicRoutes = ['/login', '/signup', '/']

export default async function proxy(req: NextRequest) {
  // 2. 현재 경로가 보호되어 있는지 공개되어 있는지 확인하세요.
  const path = req.nextUrl.pathname
  const isProtectedRoute = protectedRoutes.includes(path)
  const isPublicRoute = publicRoutes.includes(path)

  // 3. 쿠키에서 세션을 해독합니다.
  const cookie = (await cookies()).get('session')?.value
  const session = await decrypt(cookie)

  // 4. 사용자가 인증되지 않은 경우 /login으로 리디렉션
  if (isProtectedRoute && !session?.userId) {
    return NextResponse.redirect(new URL('/login', req.nextUrl))
  }

  // 5. 사용자가 인증되면 /dashboard로 리디렉션됩니다.
  if (
    isPublicRoute &&
    session?.userId &&
    !req.nextUrl.pathname.startsWith('/dashboard')
  ) {
    return NextResponse.redirect(new URL('/dashboard', req.nextUrl))
  }

  return NextResponse.next()
}

// 경로 프록시는 다음에서 실행되어서는 안 됩니다.
export const config = {
  matcher: ['/((?!api|_next/static|_next/image|.*\\.png$).*)'],
}
```

프록시는 초기 확인에 유용할 수 있지만 데이터를 보호하는 유일한 방어선이 되어서는 안 됩니다. 대부분의 보안 검사는 데이터 소스에 최대한 가깝게 수행되어야 합니다. 자세한 내용은 [데이터 액세스 레이어](#creating-a-data-access-layer-dal)를 참조하세요.
> **Tips**:
>
> * 프록시에서는 `req.cookies.get('session')?.value`를 사용하여 쿠키를 읽을 수도 있습니다.
> * 프록시는 Node.js 런타임을 사용합니다. 인증 라이브러리와 세션 관리 라이브러리가 호환되는지 확인하세요. 인증 라이브러리가 [Edge 런타임](/docs/app/api-reference/edge)만 지원하는 경우 [미들웨어](https://github.com/vercel/next.js/blob/v15.5.6/docs/01-app/03-api-reference/03-file-conventions/middleware.mdx)를 사용해야 할 수도 있습니다.
> * 프록시의 `matcher` 속성을 사용하여 프록시가 실행되어야 하는 경로를 지정할 수 있습니다. 인증의 경우 모든 경로에서 프록시를 실행하는 것이 좋습니다.

### Creating a Data Access Layer (DAL)

데이터 요청 및 권한 부여 논리를 중앙 집중화하려면 DAL을 만드는 것이 좋습니다.

DAL에는 사용자가 애플리케이션과 상호 작용할 때 사용자 세션을 확인하는 기능이 포함되어야 합니다. 최소한 이 함수는 세션이 유효한지 확인한 다음 추가 요청에 필요한 사용자 정보를 리디렉션하거나 반환해야 합니다.

예를 들어 'verifySession()' 함수를 포함하는 DAL용 별도 파일을 만듭니다. 그런 다음 React의 [cache](https://react.dev/reference/react/cache) API를 사용하여 React 렌더 패스 중 함수의 반환 값을 기억합니다:

app/lib/dal.ts
```tsx
import 'server-only'

import { cookies } from 'next/headers'
import { decrypt } from '@/app/lib/session'

export const verifySession = cache(async () => {
  const cookie = (await cookies()).get('session')?.value
  const session = await decrypt(cookie)

  if (!session?.userId) {
    redirect('/login')
  }

  return { isAuth: true, userId: session.userId }
})
```

그런 다음 데이터 요청, 서버 작업, 경로 처리기에서 `verifySession()` 함수를 호출할 수 있습니다:

app/lib/dal.ts
```tsx
export const getUser = cache(async () => {
  const session = await verifySession()
  if (!session) return null

  try {
    const data = await db.query.users.findMany({
      where: eq(users.id, session.userId),
      // 전체 사용자 개체가 아닌 필요한 열을 명시적으로 반환합니다.
      columns: {
        id: true,
        name: true,
        email: true,
      },
    })

    const user = data[0]

    return user
  } catch (error) {
    console.log('Failed to fetch user')
    return null
  }
})
```

> **Tip**:
>
> * DAL을 사용하면 요청 시 가져온 데이터를 보호할 수 있습니다. 그러나 사용자 간에 데이터를 공유하는 정적 경로의 경우 요청 시간이 아닌 빌드 시간에 데이터를 가져옵니다. 정적 경로를 보호하려면 [프록시](#optimistic-checks-with-proxy-Optional)를 사용하세요.
> * 보안 점검을 위해 세션 ID를 데이터베이스와 비교하여 세션이 유효한지 확인할 수 있습니다. 렌더 패스 중에 데이터베이스에 대한 불필요한 중복 요청을 방지하려면 React의 [캐시](https://react.dev/reference/react/cache) 기능을 사용하세요.
> * 메소드보다 먼저 'verifySession()'을 실행하는 JavaScript 클래스에 관련 데이터 요청을 통합할 수 있습니다.

### Using Data Transfer Objects (DTO)
데이터를 검색할 때 전체 개체가 아닌 애플리케이션에서 사용될 필수 데이터만 반환하는 것이 좋습니다. 예를 들어 사용자 데이터를 가져오는 경우 비밀번호, 전화번호 등이 포함될 수 있는 전체 사용자 개체가 아닌 사용자 ID와 이름만 반환할 수 있습니다.

그러나 반환된 데이터 구조를 제어할 수 없거나 전체 개체가 클라이언트에 전달되는 것을 방지하려는 팀에서 작업하는 경우 클라이언트에 노출해도 안전한 필드를 지정하는 등의 전략을 사용할 수 있습니다.

app/lib/dto.ts
```tsx
import 'server-only'
import { getUser } from '@/app/lib/dal'

function canSeeUsername(viewer: User) {
  return true
}

function canSeePhoneNumber(viewer: User, team: string) {
  return viewer.isAdmin || team === viewer.team
}

export async function getProfileDTO(slug: string) {
  const data = await db.query.users.findMany({
    where: eq(users.slug, slug),
    // 여기에 특정 열을 반환하세요.
  })
  const user = data[0]

  const currentUser = await getUser(user.id)

  // 또는 여기에 쿼리와 관련된 내용만 반환합니다.
  return {
    username: canSeeUsername(currentUser) ? user.username : null,
    phonenumber: canSeePhoneNumber(currentUser, user.team)
      ? user.phonenumber
      : null,
  }
}
```

데이터 요청 및 권한 부여 논리를 DAL에 중앙 집중화하고 DTO를 사용하면 모든 데이터 요청이 안전하고 일관되게 유지되므로 애플리케이션이 확장됨에 따라 유지 관리, 감사 및 디버깅이 더 쉬워집니다.

> **알아두면 좋은 점**:
>
> * `toJSON()`을 사용하는 것부터 위의 예와 같은 개별 함수 또는 JS 클래스에 이르기까지 DTO를 정의할 수 있는 몇 가지 방법이 있습니다. 이는 React 또는 Next.js 기능이 아닌 JavaScript 패턴이므로 애플리케이션에 가장 적합한 패턴을 찾기 위해 몇 가지 조사를 수행하는 것이 좋습니다.
> * [Next.js의 보안 문서](/blog/security-nextjs-server-comComponents-actions)에서 보안 모범 사례에 대해 자세히 알아보세요.

### Server Components

[서버 구성 요소](/docs/app/getting-started/server-and-client-comComponents)의 인증 확인은 역할 기반 액세스에 유용합니다. 예를 들어 사용자의 역할에 따라 구성 요소를 조건부로 렌더링하려면:

app/dashboard/page.tsx
```tsx
import { verifySession } from '@/app/lib/dal'

export default async function Dashboard() {
  const session = await verifySession()
  const userRole = session?.user?.role // '역할'이 세션 개체의 일부라고 가정

  if (userRole === 'admin') {
    return <AdminDashboard />
  } else if (userRole === 'user') {
    return <UserDashboard />
  } else {
    redirect('/login')
  }
}
```

이 예에서는 DAL의 `verifySession()` 함수를 사용하여 '관리자', '사용자' 및 승인되지 않은 역할을 확인합니다. 이 패턴을 사용하면 각 사용자가 자신의 역할에 적합한 구성 요소와만 상호 작용할 수 있습니다.

### Layouts and auth checks

[부분 렌더링](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions)으로 인해 [레이아웃](https://nextjs.org/docs/app/api-reference/file-conventions/layout)에서 확인할 때는 탐색 시 다시 렌더링되지 않으므로 주의하세요. 즉, 모든 경로 변경 시 사용자 세션이 확인되지는 않습니다.

대신 조건부로 렌더링될 컴포넌트나 데이터 소스에 가까운 검사를 수행해야 합니다.

예를 들어, 사용자 데이터를 가져오고 탐색에 사용자 이미지를 표시하는 공유 레이아웃을 생각해 보세요. 레이아웃에서 인증 확인을 수행하는 대신 레이아웃에서 사용자 데이터(`getUser()`)를 가져와 DAL에서 인증 확인을 수행해야 합니다.

이렇게 하면 애플리케이션 내에서 `getUser()`가 호출될 때마다 인증 확인이 수행되고 개발자가 사용자에게 데이터 액세스 권한이 있는지 확인하는 것을 잊어버리는 것을 방지할 수 있습니다.

#### Auth checks in page components

예를 들어 대시보드 페이지에서 사용자 세션을 확인하고 사용자 데이터를 가져올 수 있습니다:

app/dashboard/page.tsx
```tsx
import { verifySession } from '@/app/lib/dal'

export default async function DashboardPage() {
  const session = await verifySession()

  // 데이터베이스 또는 데이터 소스에서 사용자별 데이터를 가져옵니다.
  const user = await getUserData(session.userId)

  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      {/* Dashboard content */}
    </div>
  )
}
```

#### Auth checks in leaf components

사용자 권한에 따라 UI 요소를 조건부로 렌더링하는 리프 컴포넌트에서 인증 검사를 수행할 수도 있습니다. 예를 들어 관리자 전용 작업을 표시하는 컴포넌트:

app/ui/admin-actions.tsx
```tsx
import { verifySession } from '@/app/lib/dal'

export default async function AdminActions() {
  const session = await verifySession()
  const userRole = session?.user?.role

  if (userRole !== 'admin') {
    return null
  }

  return (
    <div>
      <button>Delete User</button>
      <button>Edit Settings</button>
    </div>
  )
}
```

이 패턴을 사용하면 사용자 권한에 따라 UI 요소를 표시하거나 숨기면서 각 컴포넌트의 렌더링 시 인증 확인이 발생하도록 할 수 있습니다.

> **알아두면 좋은 점:**
>
> * SPA의 일반적인 패턴은 사용자가 인증되지 않은 경우 레이아웃이나 최상위 구성 요소에서 'null'을 반환하는 것입니다. Next.js 애플리케이션에는 여러 개의 진입점이 있어 중첩된 경로 세그먼트 및 서버 작업에 액세스하는 것을 방지하지 않으므로 이 패턴은 **권장되지 않습니다**.
> * 클라이언트측 UI 제한만으로는 보안에 충분하지 않으므로 이러한 컴포넌트에서 호출된 모든 서버 작업도 자체 인증 확인을 수행하는지 확인하세요.

### Server Actions

[서버 작업](/docs/app/getting-started/mutating-data)을 공개 API 엔드포인트와 동일한 보안 고려 사항으로 처리하고 사용자가 변형을 수행할 수 있는지 확인하세요.

아래 예에서는 작업 진행을 허용하기 전에 사용자의 역할을 확인합니다:

app/lib/actions.ts
```ts
'use server'
import { verifySession } from '@/app/lib/dal'

export async function serverAction(formData: FormData) {
  const session = await verifySession()
  const userRole = session?.user?.role

  // 사용자가 작업을 수행할 권한이 없는 경우 일찍 반환
  if (userRole !== 'admin') {
    return null
  }

  // 승인된 사용자에 대한 작업을 진행합니다.
}
```

### Route Handlers

[경로 처리기](/docs/app/api-reference/file-conventions/route)를 공개 API 엔드포인트와 동일한 보안 고려 사항으로 처리하고 사용자가 경로 처리기에 액세스할 수 있는지 확인하세요.

예를 들어:

app/api/route.ts
```ts
import { verifySession } from '@/app/lib/dal'

export async function GET() {
  // 사용자 인증 및 역할 확인
  const session = await verifySession()

  // 사용자가 인증되었는지 확인
  if (!session) {
    // User is not authenticated
    return new Response(null, { status: 401 })
  }

  // 사용자에게 '관리자' 역할이 있는지 확인하세요.
  if (session.user.role !== 'admin') {
    // User is authenticated but does not have the right permissions
    return new Response(null, { status: 403 })
  }

  // 승인된 사용자에 대해 계속
}
```

위의 예는 2계층 보안 검사가 포함된 경로 핸들러를 보여줍니다. 먼저 활성 세션을 확인한 다음 로그인한 사용자가 '관리자'인지 확인합니다.

## Context Providers

[인터리빙](/docs/app/getting-started/server-and-client-components#interleaving-server-and-client-components)으로 인해 인증에 컨텍스트 공급자를 사용하는 것이 작동합니다. 그러나 React 'context'는 서버 컴포넌트에서 지원되지 않으므로 클라이언트 컴포넌트에만 적용할 수 있습니다.

이는 작동하지만 모든 하위 서버 컴포넌트는 먼저 서버에서 렌더링되며 컨텍스트 공급자의 세션 데이터에 액세스할 수 없습니다:

app/layout.ts
```tsx
import { ContextProvider } from 'auth-lib'

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <ContextProvider>{children}</ContextProvider>
      </body>
    </html>
  )
}
```

app/ui/profile.ts
```tsx
'use client';

import { useSession } from "auth-lib";

export default function Profile() {
  const { userId } = useSession();
  const { data } = useSWR(`/api/user/${userId}`, fetcher)

  return (
    // ...
  );
}
```

클라이언트 컴포넌트에 세션 데이터가 필요한 경우(예: 클라이언트 측 데이터 가져오기) React의 [`taintUniqueValue`](https://react.dev/reference/react/experimental_taintUniqueValue) API를 사용하여 민감한 세션 데이터가 클라이언트에 노출되는 것을 방지하세요.

## Resources
Next.js의 인증에 대해 배웠으므로 이제 보안 인증 및 세션 관리를 구현하는 데 도움이 되는 Next.js 호환 라이브러리와 리소스가 있습니다.:

### Auth Libraries
* [Auth0](https://auth0.com/docs/quickstart/webapp/nextjs)
* [Better Auth](https://www.better-auth.com/docs/integrations/next)
* [Clerk](https://clerk.com/docs/quickstarts/nextjs)
* [Descope](https://docs.descope.com/getting-started/nextjs)
* [Kinde](https://kinde.com/docs/developer-tools/nextjs-sdk)
* [Logto](https://docs.logto.io/quick-starts/next-app-router)
* [NextAuth.js](https://authjs.dev/getting-started/installation?framework=next.js)
* [Ory](https://www.ory.sh/docs/getting-started/integrate-auth/nextjs)
* [Stack Auth](https://docs.stack-auth.com/getting-started/setup)
* [Supabase](https://supabase.com/docs/guides/getting-started/quickstarts/nextjs)
* [Stytch](https://stytch.com/docs/guides/quickstarts/nextjs)
* [WorkOS](https://workos.com/docs/user-management/nextjs)

### Session Management Libraries

* [Iron Session](https://github.com/vvo/iron-session)
* [Jose](https://github.com/panva/jose)

## Further Reading
인증 및 보안에 대해 계속 알아보려면 다음 리소스를 확인하세요:
* [How to think about security in Next.js](/blog/security-nextjs-server-components-actions)
* [Understanding XSS Attacks](https://vercel.com/guides/understanding-xss-attacks)
* [Understanding CSRF Attacks](https://vercel.com/guides/understanding-csrf-attacks)
* [The Copenhagen Book](https://thecopenhagenbook.com/)