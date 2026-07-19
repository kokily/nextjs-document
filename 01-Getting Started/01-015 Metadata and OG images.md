# Metadata and OG images
<span style="color: #808080;">Last updated June 23, 2026</span>

메타데이터 API는 향상된 SEO 및 웹 공유성을 위해 애플리케이션 메타데이터를 정의하고 다음을 포함하는 데 사용할 수 있습니다:

1. [정적 `metadata` 개체](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#static-metadata)
2. [동적 `generateMetadata` 함수](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#generated-metadata)
3. 정적 또는 동적으로 생성된 [파비콘](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#favicons) 및 [OG 이미지](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#static-open-graph-images)를 추가하는 데 사용할 수 있는 특수 [파일 규칙](https://nextjs.org/docs/app/api-reference/file-conventions/metadata)입니다.

위의 모든 옵션을 사용하여 Next.js는 페이지에 대한 관련 `<head>` 태그를 자동으로 생성하며, 이는 브라우저의 개발자 도구에서 검사할 수 있습니다.

`metadata` 개체 및 `generateMetadata` 함수 내보내기는 서버 컴포넌트에서만 지원됩니다.

---
## Default fields
경로가 메타데이터를 정의하지 않는 경우에도 항상 추가되는 두 개의 기본 `메타` 태그가 있습니다:
- [메타 문자 집합 태그](https://developer.mozilla.org/docs/Web/HTML/Element/meta#attr-charset)는 웹사이트의 문자 인코딩을 설정합니다.
- [메타 뷰포트 태그](https://developer.mozilla.org/docs/Web/HTML/Viewport_meta_tag)는 웹 사이트의 뷰포트 너비와 배율을 설정하여 다양한 장치에 맞게 조정합니다.

```html
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

다른 메타데이터 필드는 `Metadata` 개체([정적 메타데이터](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#static-metadata)의 경우) 또는 `generateMetadata` 함수([생성된 메타데이터](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#generated-metadata)의 경우)를 사용하여 정의할 수 있습니다.

---
## Static metadata
정적 메타데이터를 정의하려면 정적 [layout.js](https://nextjs.org/docs/app/api-reference/file-conventions/layout) 또는 [page.js](https://nextjs.org/docs/app/api-reference/file-conventions/page) 파일에서 [메타데이터 개체](https://nextjs.org/docs/app/api-reference/functions/generate-metadata#metadata-object)를 내보냅니다. 예를 들어 블로그 경로에 제목과 설명을 추가하려면:

app/blog/layout.tsx
```ts
import type { Metadata } from 'next'
 
export const metadata: Metadata = {
  title: 'My Blog',
  description: '...',
}
 
export default function Layout() {}
```

[`generateMetadata` 문서](https://nextjs.org/docs/app/api-reference/functions/generate-metadata#metadata-fields)에서 사용 가능한 옵션의 전체 목록을 볼 수 있습니다.

---
## Generated metadata
[generateMetadata](https://nextjs.org/docs/app/api-reference/functions/generate-metadata) 함수를 사용하여 데이터에 의존하는 메타데이터를 `가져올 수 있습니다`. 예를 들어 특정 블로그 게시물의 제목과 설명을 가져오려면:

app/blog/[slug]/page.tsx
```ts
import type { Metadata, ResolvingMetadata } from 'next'
 
type Props = {
  params: Promise<{ slug: string }>
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}
 
export async function generateMetadata(
  { params, searchParams }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const slug = (await params).slug
 
  // fetch post information
  const post = await fetch(`https://api.vercel.app/blog/${slug}`).then((res) =>
    res.json()
  )
 
  return {
    title: post.title,
    description: post.description,
  }
}
 
export default function Page({ params, searchParams }: Props) {}
```

### Streaming metadata
동적으로 렌더링된 페이지의 경우 Next.js는 메타데이터를 별도로 스트리밍하여 UI 렌더링을 차단하지 않고 `generateMetadata`가 확인되면 이를 HTML에 삽입합니다.

스트리밍 메타데이터는 시각적 콘텐츠가 먼저 스트리밍되도록 하여 인지 성능을 향상시킵니다.

메타데이터가 `<head>` 태그에 있을 것으로 예상하는 봇 및 크롤러(예: `Twitterbot`, `Slackbot`, `Bingbot`)에 대해서는 스트리밍 메타데이터가 비활성화됩니다. 이는 들어오는 요청의 사용자 에이전트 헤더를 사용하여 감지됩니다.

Next.js 구성 파일의 [`htmlLimitedBots`](https://nextjs.org/docs/app/api-reference/config/next-config-js/htmlLimitedBots#disabling) 옵션을 사용하여 스트리밍 메타데이터를 완전히 사용자 정의하거나 비활성화할 수 있습니다.

메타데이터는 빌드 시 확인되므로 사전 렌더링된 페이지는 스트리밍을 사용하지 않습니다.

[스트리밍 메타데이터](https://nextjs.org/docs/app/api-reference/functions/generate-metadata#streaming-metadata)에 대해 자세히 알아보세요.

### Memozing data requests
메타데이터와 페이지 자체에 대해 동일한 데이터를 가져와야 하는 경우가 있을 수 있습니다. 중복 요청을 피하기 위해 React의 [캐시 기능](https://react.dev/reference/react/cache)을 사용하여 반환 값을 메모하고 데이터를 한 번만 가져올 수 있습니다. 예를 들어 메타데이터와 페이지 모두에 대한 블로그 게시물 정보를 가져오려면:

app/lib/data.ts
```ts
import { cache } from 'react'
import { db } from '@/app/lib/db'
 
// getPost will be used twice, but execute only once
export const getPost = cache(async (slug: string) => {
  const res = await db.query.posts.findFirst({ where: eq(posts.slug, slug) })
  return res
})
```

app/blog/[slug]/page.tsx
```ts
import { getPost } from '@/app/lib/data'
 
export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug)
  return {
    title: post.title,
    description: post.description,
  }
}
 
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug)
  return <div>{post.title}</div>
}
```

---
## File-based metadata
메타데이터에는 다음과 같은 특수 파일을 사용할 수 있습니다:

- favicon.ico, apple-icon.jpg, and icon.jpg
- opengraph-image.jpg and twitter-image.jpg
- robots.txt
- sitemap.xml

이를 정적 메타데이터에 사용하거나 코드를 사용하여 프로그래밍 방식으로 이러한 파일을 생성할 수 있습니다.

---
## Favicons
파비콘은 북마크와 검색 결과에서 사이트를 나타내는 작은 아이콘입니다. 애플리케이션에 파비콘을 추가하려면 `favicon.ico`를 생성하고 앱 폴더의 루트에 추가하세요.

![](./images/015/favicon-ico.avif)

> 코드를 사용하여 프로그래밍 방식으로 파비콘을 생성할 수도 있습니다. 자세한 내용은 [파비콘 문서](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/app-icons)를 참조하세요.

---
## Static Open Graph images
오픈 그래프(OG) 이미지는 소셜 미디어에서 사이트를 나타내는 이미지입니다. 애플리케이션에 정적 OG 이미지를 추가하려면 앱 폴더의 루트에 `opengraph-image.jpg` 파일을 생성하세요.

![](./images/015/opengraph-image.avif)

폴더 구조 아래에 `opengraph-image.jpg`를 생성하여 특정 경로에 대한 OG 이미지를 추가할 수도 있습니다. 예를 들어, `/blog` 경로와 관련된 OG 이미지를 생성하려면 블로그 폴더 내에 `opengraph-image.jpg` 파일을 추가하세요.

![](./images/015/opengraph-image-blog.avif)

더 구체적인 이미지는 폴더 구조에서 그 위에 있는 OG 이미지보다 우선합니다.

> `JPEG`, `png`, `gif` 등의 다른 이미지 형식도 지원됩니다. 자세한 내용은 [오픈 그래프 이미지 문서](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/opengraph-image)를 참조하세요.

---
## Generated Open Graph images
[`ImageResponse` 생성자](https://nextjs.org/docs/app/api-reference/functions/image-response)를 사용하면 JSX 및 CSS를 사용하여 동적 이미지를 생성할 수 있습니다. 이는 데이터에 의존하는 OG 이미지에 유용합니다.

예를 들어, 각 블로그 게시물에 대해 고유한 OG 이미지를 생성하려면 `blog` 폴더 내에 ``opengraph-image.tsx`` 파일을 추가하고 `next/og`에서 `ImageResponse` 생성자를 가져옵니다:

app/blog/[slug]/opengraph-image.tsx
```ts
import { ImageResponse } from 'next/og'
import { getPost } from '@/app/lib/data'
 
// Image metadata
export const size = {
  width: 1200,
  height: 630,
}
 
export const contentType = 'image/png'
 
// Image generation
export default async function Image({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug)
 
  return new ImageResponse(
    (
      // ImageResponse JSX element
      <div
        style={{
          fontSize: 128,
          background: 'white',
          width: '100%',
          height: '100%',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
        }}
      >
        {post.title}
      </div>
    )
  )
}
```

`ImageResponse`는 가변상자 및 절대 위치 지정, 사용자 정의 글꼴, 텍스트 줄 바꿈, 가운데 맞춤 및 중첩된 이미지를 포함한 일반적인 CSS 속성을 지원합니다. [지원되는 CSS 속성의 전체 목록을 확인하세요.](https://nextjs.org/docs/app/api-reference/functions/image-response)

> 알아두면 좋은 점:
> - 예제는 [Vercel OG Playground](https://og-playground.vercel.app/)에서 확인할 수 있습니다.
> - `ImageResponse`는 [@vercel/og](https://vercel.com/docs/og-image-generation), [satori](https://github.com/vercel/satori) 및 `resvg`를 사용하여 HTML과 CSS를 PNG로 변환합니다.
> - Flexbox와 CSS 속성의 하위 집합만 지원됩니다. 고급 레이아웃(예: `display: grid`)은 작동하지 않습니다.