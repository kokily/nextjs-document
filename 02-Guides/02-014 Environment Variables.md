# How to use environment variables in Next.js
<span style="color: #808080;">Last updated March 3, 2026</span>

Next.js에는 환경 변수에 대한 지원이 내장되어 있어 다음을 수행할 수 있습니다:

* [`.env`를 사용하여 환경 변수 로드](#loading-environment-variables)
* [`NEXT_PUBLIC_` 접두어를 붙여 브라우저용 환경 변수를 번들링합니다.](#bundling-environment-variables-for-the-browser)

> **경고:** 기본 `create-next-app` 템플릿은 모든 `.env` 파일이 `.gitignore`에 추가되도록 보장합니다. 이러한 파일을 저장소에 커밋하고 싶지는 않을 것입니다.

## Loading Environment Variables

Next.js에는 `.env*` 파일에서 `process.env`로 환경 변수를 로드하는 기능이 내장되어 있습니다.

```txt
DB_HOST=localhost
DB_USER=myuser
DB_PASS=mypassword
```

> **메모**: Next.js는 `.env*` 파일 내에서 여러 줄 변수도 지원합니다:
>
> ```bash
> # .env
>
> # 줄 바꿈으로 쓸 수 있습니다
> PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
> ...
> Kh9NV...
> ...
> -----END DSA PRIVATE KEY-----"
>
> # 또는 큰따옴표 안에 `\n`을 사용합니다.
> PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nKh9NV...\n-----END DSA PRIVATE KEY-----\n"
> ```

> **메모**: `/src` 폴더를 사용하는 경우 Next.js는 상위 폴더에서만 .env 파일을 로드하고 `/src` 폴더에서는 로드하지 **않습니다**.
> 그러면 `process.env.DB_HOST`, `process.env.DB_USER` 및 `process.env.DB_PASS`가 Node.js 환경에 자동으로 로드되어 [경로 핸들러](/docs/app/api-reference/file-conventions/route)에서 사용할 수 있습니다.

예를 들어:

app/api/route.js
```js
export async function GET() {
  const db = await myDB.connect({
    host: process.env.DB_HOST,
    username: process.env.DB_USER,
    password: process.env.DB_PASS,
  })
  // ...
}
```

### Loading Environment Variables with `@next/env`
ORM 또는 테스트 실행기의 루트 구성 파일과 같이 Next.js 런타임 외부에서 환경 변수를 로드해야 하는 경우 `@next/env` 패키지를 사용할 수 있습니다.

이 패키지는 `.env*` 파일에서 환경 변수를 로드하기 위해 Next.js에서 내부적으로 사용됩니다.

이를 사용하려면 패키지를 설치하고 `loadEnvConfig` 함수를 사용하여 환경 변수를 로드합니다.:

```bash package="npm"
npm install @next/env
```

envConfig.ts
```tsx
import { loadEnvConfig } from '@next/env'

const projectDir = process.cwd()
loadEnvConfig(projectDir)
```

그런 다음 필요한 경우 구성을 가져올 수 있습니다. 예를 들어:

orm.config.ts
```tsx
import './envConfig.ts'

export default defineConfig({
  dbCredentials: {
    connectionString: process.env.DATABASE_URL!,
  },
})
```

### Referencing Other Variables
Next.js는 `$`를 사용하는 변수를 자동으로 확장하여 다른 변수를 참조합니다. `.env*` 파일 내부의 `$VARIABLE`. 이를 통해 다른 비밀을 참조할 수 있습니다. 예를 들어:

```txt
TWITTER_USER=nextjs
TWITTER_URL=https://x.com/$TWITTER_USER
```

위의 예에서 `process.env.TWITTER_URL`은 `https://x.com/nextjs`로 설정됩니다.

> **알아두면 좋은 점**: 실제 값에 `$`가 포함된 변수를 사용해야 하는 경우 이스케이프해야 합니다. `\$`.

## Bundling Environment Variables for the Browser

Non-`NEXT_PUBLIC_` 환경 변수는 Node.js 환경에서만 사용할 수 있습니다. 즉, 브라우저에서 액세스할 수 없습니다(클라이언트는 다른 *환경*에서 실행됨).

브라우저에서 환경 변수 값에 액세스할 수 있도록 Next.js는 빌드 시 클라이언트에 전달되는 js 번들에 값을 "인라인"하여 `process.env.[변수]`에 대한 모든 참조를 하드 코딩된 값으로 바꿀 수 있습니다. 이렇게 하려면 변수 앞에 `NEXT_PUBLIC_`을 붙이면 됩니다. 예를 들어:

```txt
NEXT_PUBLIC_ANALYTICS_ID=abcdefghijk
```

이렇게 하면 Next.js가 Node.js 환경의 `process.env.NEXT_PUBLIC_ANALYTICS_ID`에 대한 모든 참조를 `next build`를 실행하는 환경의 값으로 대체하여 코드 어디에서나 사용할 수 있게 됩니다. 이는 브라우저로 전송되는 모든 JavaScript에 인라인됩니다.

> **Note**: 빌드된 후에는 앱이 더 이상 이러한 환경 변수의 변경에 응답하지 않습니다. 예를 들어 Heroku 파이프라인을 사용하여 한 환경에서 빌드된 슬러그를 다른 환경으로 승격하거나 단일 Docker 이미지를 빌드하여 여러 환경에 배포하는 경우 모든 `NEXT_PUBLIC_` 변수는 빌드 시 평가된 값으로 고정되므로 프로젝트가 빌드될 때 이러한 값을 적절하게 설정해야 합니다. 런타임 환경 값에 액세스해야 하는 경우 이를 클라이언트에 제공하기 위해 자체 API를 설정해야 합니다(요청 시 또는 초기화 중에).

pages/index.js
```js
import setupAnalyticsService from '../lib/my-analytics-service'

// 'NEXT_PUBLIC_ANALYTICS_ID'는 'NEXT_PUBLIC_' 접두사가 붙어 있으므로
// 여기서 사용할 수 있습니다.
// 빌드 시 `setupAnalyticsService('abcdefghijk')`로 변환됩니다.
setupAnalyticsService(process.env.NEXT_PUBLIC_ANALYTICS_ID)

function HomePage() {
  return <h1>Hello World</h1>
}

export default HomePage
```

동적 조회는 인라인되지 *않습니다*:

```js
// 변수를 사용하기 때문에 인라인되지 않습니다.
const varName = 'NEXT_PUBLIC_ANALYTICS_ID'
setupAnalyticsService(process.env[varName])

// 변수를 사용하기 때문에 인라인되지 않습니다.
const env = process.env
setupAnalyticsService(env.NEXT_PUBLIC_ANALYTICS_ID)
```

### Runtime Environment Variables

Next.js는 빌드 시간과 런타임 환경 변수를 모두 지원할 수 있습니다.

**기본적으로 환경 변수는 서버에서만 사용할 수 있습니다**. 환경 변수를 브라우저에 노출하려면 `NEXT_PUBLIC_` 접두사가 붙어야 합니다. 그러나 이러한 공개 환경 변수는 `다음 빌드` 중에 JavaScript 번들에 인라인됩니다.

동적 렌더링 중에 서버의 환경 변수를 안전하게 읽을 수 있습니다:

app/page.ts
```tsx
import { connection } from 'next/server'

export default async function Component() {
  await connection()
  // 쿠키, 헤더 및 기타 요청 시간 API도 동적 렌더링을 선택합니다.
  // 즉, 이 환경 변수는 런타임에 평가됩니다.
  const value = process.env.MY_VALUE
  // ...
}
```

이를 통해 서로 다른 값을 가진 여러 환경을 통해 승격될 수 있는 단일 Docker 이미지를 사용할 수 있습니다.

**알아두면 좋은 점:**
* [`register` 함수](/docs/app/guides/instrumentation)를 사용하여 서버 시작 시 코드를 실행할 수 있습니다.

## Test Environment Variables
`개발` 및 `프로덕션` 환경 외에 세 번째 옵션인 `테스트`를 사용할 수 있습니다. 개발 또는 프로덕션 환경에 대한 기본값을 설정할 수 있는 것과 같은 방법으로 `testing` 환경에 대한 `.env.test` 파일을 사용하여 동일한 작업을 수행할 수 있습니다(이것은 이전 두 환경만큼 일반적이지는 않습니다). Next.js는 `testing` 환경의 `.env.development` 또는 `.env.production`에서 환경 변수를 로드하지 않습니다.

이는 테스트 목적으로만 특정 환경 변수를 설정해야 하는 `jest` 또는 `cypress`와 같은 도구를 사용하여 테스트를 실행할 때 유용합니다. `NODE_ENV`가 `test`로 설정된 경우 테스트 기본값이 로드되지만, 일반적으로 테스트 도구가 이를 해결하므로 이 작업을 수동으로 수행할 필요는 없습니다.

`테스트` 환경과 `개발` 및 `프로덕션` 둘 다 염두에 두어야 할 작은 차이가 있습니다. 테스트가 모든 사람에게 동일한 결과를 생성할 것으로 기대하므로 `.env.local`은 로드되지 않습니다. 이렇게 하면 모든 테스트 실행에서 `.env.local`(기본 설정을 재정의하기 위한 것임)을 무시하여 다양한 실행에서 동일한 env 기본값을 사용하게 됩니다.

> **알아두면 좋은 점**: 기본 환경 변수와 유사하게 `.env.test` 파일은 저장소에 포함되어야 하지만 `.env.test.local`은 포함되어서는 안 됩니다. `.env*.local`은 `.gitignore`를 통해 무시되도록 되어 있기 때문입니다.

단위 테스트를 실행하는 동안 `@next/env` 패키지의 `loadEnvConfig` 함수를 활용하여 Next.js와 동일한 방식으로 환경 변수를 로드할 수 있습니다.

```js
// 아래는 Jest 전역 설정 파일이나 테스트 설정과 유사한 파일에서 사용할 수 있습니다.
import { loadEnvConfig } from '@next/env'

export default async () => {
  const projectDir = process.cwd()
  loadEnvConfig(projectDir)
}
```

## Environment Variable Load Order
환경변수는 다음과 같은 곳에서 순서대로 조회되며, 해당 변수가 발견되면 중지됩니다.
1. `process.env`
2. `.env.$(NODE_ENV).local`
3. `.env.local` (`NODE_ENV`가 `test`인 경우 확인되지 않습니다.)
4. `.env.$(NODE_ENV)`
5. `.env`

예를 들어 `NODE_ENV`가 `development`이고 `.env.development.local`과 `.env` 모두에 변수를 정의하는 경우 `.env.development.local`의 값이 사용됩니다.

> **알아두면 좋은 점**: `NODE_ENV`에 허용되는 값은 `production`, `development`, `test`입니다.

## Good to know
* [`/src` 디렉터리](/docs/app/api-reference/file-conventions/src-folder)를 사용하는 경우 `.env.*` 파일은 프로젝트 루트에 남아 있어야 합니다.
* 환경 변수 `NODE_ENV`가 할당되지 않은 경우 Next.js는 `next dev` 명령을 실행할 때 `development`를 자동으로 할당하거나 다른 모든 명령에 대해 `production`을 자동으로 할당합니다.

## Version History

| Version  | Changes                                       |
| -------- | --------------------------------------------- |
| `v9.4.0` | Support `.env` and `NEXT_PUBLIC_` introduced. |