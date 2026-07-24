# How to preview content with Draft Mode in Next.js
<span style="color: #808080;">Last updated June 23, 2026</span>

**Draft Mode**를 사용하면 편집자는 재검증을 기다리지 않고도 초안 또는 진행 중인 콘텐츠가 사이트에서 어떻게 렌더링되는지 확인할 수 있습니다. 편집기가 초안 모드에 있는 동안 캐시되거나 사전 렌더링된 콘텐츠는 우회되고 업스트림 소스에서 직접 가져옵니다. 다른 방문자는 페이지의 캐시된 버전이나 사전 렌더링된 버전을 계속해서 볼 수 있습니다.

CMS가 동일한 URL에서 초안 콘텐츠와 게시된 콘텐츠를 제공하는 경우 데이터 가져오기 코드를 변경할 필요가 없습니다. 그렇지 않은 경우에는 [CMS가 별도의 초안 엔드포인트를 사용하는 경우](#when-your-cms-uses-a-separate-draft-endpoint)를 참조하세요.

## What Draft Mode does
요청에 대해 초안 모드가 활성화된 경우:
* `fetch()` 호출은 Next.js 페치 캐시를 건너뛰고 네트워크에 직접 연결됩니다.
* [`use cache`](/docs/app/api-reference/directives/use-cache) 내부의 컴포넌트와 함수는 요청이 있을 때마다 다시 실행되며 해당 결과는 캐시에 저장되지 않습니다.
* [`unstable_cache`](/docs/app/api-reference/functions/unstable_cache) 읽기 및 쓰기는 동일한 방식으로 우회됩니다.
* 해당 페이지는 ISR 응답 캐시에서 제외되며 `Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate`로 제공됩니다.

페이지가 정적으로 생성되거나, 캐시에서 제공되거나, ISR을 통해 재검증되는 경우 효과가 적용됩니다.

## What this guide covers
이 가이드에서는:
* 헤드리스 CMS는 구성 가능한 미리보기 URL을 지원합니다(대부분 지원).
* 편집자가 'Preview'를 클릭하면 CMS는 `/api/draft?secret=XXX&slug=/posts/foo`와 같은 URL을 새 탭에서 엽니다. 비밀은 공유 토큰입니다. 슬러그는 미리 볼 수 있는 경로입니다.
* Next.js 앱은 비밀의 유효성을 검사하고 초안 모드를 활성화한 후 슬러그로 리디렉션합니다.

해당 계약을 염두에 두고 이 가이드의 나머지 부분에서는:
1. 쿠키를 설정하여 초안 모드를 활성화하는 경로 처리기를 만듭니다.
2. CMS의 공유 비밀 및 슬러그를 사용하여 해당 핸들러를 보호합니다.
3. 최신 초안을 읽는 페이지 렌더링.
4. 종료 양식과 함께 미리보기 배너를 표시합니다.

그런 다음 설정에 따라:
* 'use cache' 경계에서 미리보기 상태를 표시하기 위한 [캐시 구성요소가 있는 초안 모드](#draft-mode-with-cache-comComponents).
* [CMS가 별도의 초안 엔드포인트를 사용하는 경우](#when-your-cms-uses-a-separate-draft-endpoint) `isEnabled`에서 가져오기 URL을 분기하기 위한 것입니다.

> **알아두면 좋은 점:** `GET`은 안전한 읽기 전용 방법입니다. 쿠키를 통한 초안 모드 활성화와 같이 향후 요청에 영향을 미치는 작업에는 `POST`를 사용해야 합니다. CMS 미리보기 통합을 가정하기 때문에 항목 핸들러는 `GET`을 사용합니다. CMS는 `GET` 요청인 새 브라우저 탭에서 URL을 엽니다. 4단계의 종료 흐름에서는 `POST`([Server Action](/docs/app/getting-started/mutating-data) 또는 `POST` 경로 처리기를 통해)를 사용합니다.

## Step 1: Create a Route Handler

초안 모드 쿠키를 설정하는 [경로 처리기](/docs/app/api-reference/file-conventions/route)를 만듭니다. 이름은 무엇이든 가질 수 있습니다(예: `app/api/draft/route.ts`).

app/api/draft/route.ts
```ts
import { draftMode } from 'next/headers'

export async function GET(request: Request) {
  const draft = await draftMode()
  draft.enable()
  return new Response('Draft mode is enabled')
}
```

`draft.enable()`은 `__prerender_bypass`라는 쿠키를 설정합니다. 이 쿠키를 전달하는 후속 요청은 위에 나열된 모든 캐시 계층을 건너뜁니다.

`/api/draft`를 방문하고 브라우저의 개발자 도구를 확인하여 수동으로 테스트할 수 있습니다. `Set-Cookie` 응답 헤더를 확인하세요.

작성된 대로 핸들러는 공개됩니다. `/api/draft`를 누르는 사람은 누구나 스스로 초안 모드를 활성화할 수 있습니다. 2단계에서는 CMS만 호출할 수 있도록 공유 비밀을 사용하여 이를 닫습니다.

## Step 2: Access the Route Handler from your headless CMS

> 이 단계에서는 사용 중인 헤드리스 CMS가 **맞춤 초안 URL** 설정을 지원한다고 가정합니다. 그렇지 않은 경우에도 이 방법을 사용하여 초안 URL을 보호할 수 있지만 초안 URL을 수동으로 구성하고 액세스해야 합니다. 구체적인 단계는 사용 중인 헤드리스 CMS에 따라 달라집니다.

헤드리스 CMS에서 경로 핸들러에 안전하게 액세스하려면:

1. 원하는 토큰 생성기를 사용하여 **비밀 토큰 문자열**을 만듭니다. 이 비밀은 Next.js 앱과 헤드리스 CMS에만 알려져 있습니다.
2. 헤드리스 CMS가 사용자 정의 초안 URL 설정을 지원하는 경우 초안 URL을 지정하십시오(이는 경로 핸들러가 `app/api/draft/route.ts`에 있다고 가정함). 예를 들어:

```bash
https://<your-site>/api/draft?secret=<token>&slug=<path>
```

> * `<your-site>` 배포 도메인이어야 합니다.
> * `<token>` 생성한 비밀 토큰으로 바꿔야 합니다.
> * `<path>` 보려는 페이지의 경로여야 합니다. `/posts/one`을 보려면 `&slug=/posts/one`를 사용해야 합니다.
>
> 헤드리스 CMS를 사용하면 `<path>`가 CMS의 데이터를 기반으로 동적으로 설정될 수 있도록 초안 URL에 변수를 포함할 수 있습니다. `&slug=/posts/{entry.fields.slug}`

3. 경로 핸들러에서 비밀이 일치하는지, `slug` 매개변수가 있는지 확인하고(그렇지 않으면 요청이 실패해야 함) `draft.enable()`을 호출하여 쿠키를 설정한 다음 브라우저를 `slug`에 지정된 경로로 리디렉션합니다:

app/api/draft/route.ts
```ts
import { draftMode } from 'next/headers'
import { redirect } from 'next/navigation'

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const secret = searchParams.get('secret')
  const slug = searchParams.get('slug')

  // 이 비밀은 이 경로 핸들러와 CMS에만 알려져야 합니다.
  if (secret !== 'MY_SECRET_TOKEN' || !slug) {
    return new Response('Invalid token', { status: 401 })
  }

  // Verify the slug exists in the CMS before enabling Draft Mode
  const post = await getPostBySlug(slug)
  if (!post) {
    return new Response('Invalid slug', { status: 401 })
  }

  const draft = await draftMode()
  draft.enable()

  // 오픈 리디렉션 취약점을 방지하려면 searchParams가 아닌 가져온 게시물의 경로로
  // 리디렉션하세요.
  redirect(post.slug)
}
```

성공하면 브라우저는 초안 모드 쿠키 세트를 사용하여 대상 경로로 리디렉션됩니다.

## Step 3: Preview the draft content
초안 모드는 자동으로 캐시를 우회하므로 페이지에서 새로운 콘텐츠를 받기 위해 초안 모드가 켜져 있는지 알 필요가 없습니다. 평소대로 가져오기:

app/posts/[slug]/page.tsx
```tsx
async function getPost(slug: string) {
  const res = await fetch(`https://cms.example.com/posts/${slug}`)
  return res.json()
}

export default async function Page({ params }: PageProps<'/posts/[slug]'>) {
  const { slug } = await params
  const post = await getPost(slug)

  return (
    <main>
      <h1>{post.title}</h1>
      <article>{post.content}</article>
    </main>
  )
}
```

초안 모드 쿠키가 있는 경우 위의 `fetch`는 Next.js 가져오기 캐시를 건너뛰고 현재 초안에 대한 CMS에 도달합니다. 그렇지 않은 경우 평소와 같이 캐시에서 동일한 요청을 처리할 수 있습니다.

CMS가 동일한 엔드포인트에서 초안을 제공하지 않고 다른 URL을 사용하는 경우 [CMS가 별도의 초안 엔드포인트를 사용하는 경우](#when-your-cms-uses-a-separate-draft-endpoint)를 참조하세요..

## Step 4: Show a preview indicator
`isEnabled`는 편집자에게 보내는 신호로 가장 유용합니다. 초안 콘텐츠를 보고 있음을 확인하는 배너와 종료 방법을 제공합니다. 모든 미리보기 페이지에 표시되도록 루트 레이아웃에서 표시기를 렌더링합니다.

app/preview-banner.tsx
```tsx
import { draftMode } from 'next/headers'
import { redirect } from 'next/navigation'

async function exitPreview() {
  'use server'
  const draft = await draftMode()
  draft.disable()
  redirect('/')
}

export async function PreviewBanner() {
  const { isEnabled } = await draftMode()
  if (!isEnabled) return null

  return (
    <aside role="status">
      Preview mode is on.{' '}
      <form action={exitPreview}>
        <button type="submit">Exit preview</button>
      </form>
    </aside>
  )
}
```

초안 모드 종료는 `GET` 경로 핸들러에서도 작동하지만 `POST`는 의미상 더 정확합니다. 예를 들어 [Server Action](/docs/app/getting-started/mutating-data) 또는 `POST` 경로 핸들러를 통해 제출된 양식을 통해 말입니다.

`GET` 경로 핸들러를 사용하는 경우 [`<Link>`](/docs/app/api-reference/comComponents/link) 대신 `<form method="GET">`에서 트리거하세요. Next.js는 기본적으로 `<Link>` 컴포넌트를 프리페치하여 편집기가 클릭하기 전에 쿠키를 지웁니다. 방법에 관계없이 양식은 미리 가져오지 않습니다.

## Draft Mode with Cache Components
[`'use 캐시'`](/docs/app/api-reference/directives/use-cache) 범위 내에서 `isEnabled`를 읽어 캐시된 컴포넌트에서 미리보기 표시기를 렌더링할 수 있습니다. 캐시 우회가 계속 적용되므로 컴포넌트는 모든 초안 요청에 대해 새로운 데이터를 사용하여 다시 실행됩니다.

app/posts/[slug]/page.tsx
```tsx
import { draftMode } from 'next/headers'

async function Post({ slug }: { slug: string }) {
  'use cache'

  const post = await fetch(`https://cms.example.com/posts/${slug}`).then((r) =>
    r.json()
  )
  const { isEnabled } = await draftMode()

  return (
    <article>
      {isEnabled && <p role="status">Draft preview</p>}
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  )
}
```

> **알아두면 좋은 점:** `draftMode().enable()` 및 `draftMode().disable()`은 캐싱 지시문 범위 내에서 호출할 수 없습니다. 대신 [Route Handler](/docs/app/api-reference/file-conventions/route) 또는 [Server Action](/docs/app/getting-started/mutating-data)에서 초안 모드를 전환하세요.

## When your CMS uses a separate draft endpoint
CMS가 다른 URL에 초안 콘텐츠를 노출하거나 다른 자격 증명을 요구하는 경우 `isEnabled`에서 가져오기를 분기하세요:

app/posts/[slug]/page.tsx
```tsx
import { draftMode } from 'next/headers'

async function getPost(slug: string) {
  const { isEnabled } = await draftMode()
  const baseUrl = isEnabled
    ? 'https://cms.example.com/preview'
    : 'https://cms.example.com/published'

  const res = await fetch(`${baseUrl}/posts/${slug}`)
  return res.json()
}
```

캐시 우회는 여전히 두 분기 모두에 적용됩니다. 포크는 읽을 위치만 선택합니다.