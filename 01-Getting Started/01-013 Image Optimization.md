# Image Optimization
<span style="color: #808080;">Last updated April 2, 2026</span>

Next.js [<Image>](https://nextjs.org/docs/app/api-reference/components/image) 컴포넌트는 HTML `<img>` 요소를 확장하여 다음을 제공합니다:

- 크기 최적화: WebP와 같은 최신 이미지 형식을 사용하여 각 장치에 올바른 크기의 이미지를 자동으로 제공합니다.
- 시각적 안정성: 이미지가 로드될 때 [레이아웃이 자동으로 전환](https://web.dev/articles/cls)되는 것을 방지합니다.
- 더 빠른 페이지 로드: 선택적 흐림 자리 표시자와 함께 기본 브라우저 지연 로딩을 사용하여 뷰포트에 들어갈 때만 이미지를 로드합니다.
- 자산 유연성: 필요에 따라 이미지 크기를 조정하고 원격 서버에 저장된 이미지도 조정합니다.

`<Image>` 사용을 시작하려면 `next/image`에서 가져와서 컴포넌트 내에서 렌더링하세요.

app/page.tsx

```ts
import Image from 'next/image'
 
export default function Page() {
  return <Image src="" alt="" />
}
```

`src` 속성은 [로컬](https://nextjs.org/docs/app/getting-started/images#local-images) 또는 [원격](https://nextjs.org/docs/app/getting-started/images#remote-images) 이미지일 수 있습니다.

> 🎥 시청 : `next/image` 사용법에 대해 자세히 알아보세요 → [YouTube(9분)](https://youtu.be/IU_qq_c_lKA).

## Local Images

루트 디렉터리의 [public](https://nextjs.org/docs/app/api-reference/file-conventions/public-folder) 폴더에 이미지, 글꼴 등의 정적 파일을 저장할 수 있습니다. 그런 다음 기본 URL(/)부터 시작하는 코드에서 `public` 내부의 파일을 참조할 수 있습니다.

![](./images/013/public-folder.avif)

app/page.tsx
```ts
import Image from 'next/image'
 
export default function Page() {
  return (
    <Image
      src="/profile.png"
      alt="Picture of the author"
      width={500}
      height={500}
    />
  )
}
```

이미지를 정적으로 가져온 경우 Next.js는 자동으로 고유 [너비](https://nextjs.org/docs/app/api-reference/components/image#width-and-height)와 [높이](https://nextjs.org/docs/app/api-reference/components/image#width-and-height)를 결정합니다. 이 값은 이미지 비율을 결정하고 이미지가 로드되는 동안 [누적 레이아웃 이동](https://web.dev/articles/cls)을 방지하는 데 사용됩니다.

app/page.tsx
```ts
import Image from 'next/image'
import ProfileImage from './profile.png'
 
export default function Page() {
  return (
    <Image
      src={ProfileImage}
      alt="Picture of the author"
      // width={500} automatically provided
      // height={500} automatically provided
      // blurDataURL="data:..." automatically provided
      // placeholder="blur" // Optional blur-up while loading
    />
  )
}
```

### Images without static imports
이미지에 대해 정적 `import`를 사용할 수 없는 경우 서버 컴포넌트에서 동적 `import()`를 사용하여 자동 `width`, `height` 및 `blurDataURL`을 얻을 수 있습니다:

app/blog/[slug]/page.tsx
```ts
import Image from 'next/image'
 
async function PostImage({
  imageFilename,
  alt,
}: {
  imageFilename: string
  alt: string
}) {
  const { default: image } = await import(
    `../content/blog/images/${imageFilename}`
  )
  // image contains width, height, and blurDataURL
  return <Image src={image} alt={alt} />
}
```

[경로 별칭](https://www.typescriptlang.org/tsconfig/#paths)을 구성한 경우(예: @/) 상대 경로 대신 이를 사용할 수 있습니다:

```ts
const { default: image } = await import(
  `@/content/blog/images/${imageFilename}`
)
```

경로에는 정적 접두사(예: ../content/blog/images/)가 포함되어야 합니다. 해당 접두사와 일치하는 모든 파일이 번들로 제공되므로 최대한 구체적으로 작성하세요. 지정된 디렉터리의 파일만 포함되므로 외부 입력은 해당 디렉터리 외부에 도달할 수 없습니다.

---
## Remote images
원격 이미지를 사용하려면 `src` 속성에 URL 문자열을 제공하면 됩니다.

app/page.tsx
```ts
import Image from 'next/image'
 
export default function Page() {
  return (
    <Image
      src="https://s3.amazonaws.com/my-bucket/profile.png"
      alt="Picture of the author"
      width={500}
      height={500}
    />
  )
}
```

Next.js는 빌드 프로세스 중에 원격 파일에 액세스할 수 없으므로 [너비](https://nextjs.org/docs/app/api-reference/components/image#width-and-height), [높이](https://nextjs.org/docs/app/api-reference/components/image#width-and-height) 및 선택적 [blurDataURL](https://nextjs.org/docs/app/api-reference/components/image#blurdataurl) props를 수동으로 제공해야 합니다. `너비`와 `높이`는 이미지의 올바른 종횡비를 추론하고 이미지 로드 시 레이아웃 변경을 방지하는 데 사용됩니다. 또는 [fill 속성](https://nextjs.org/docs/app/api-reference/components/image#fill)을 사용하여 이미지가 상위 요소의 크기에 맞게 채워지도록 할 수 있습니다.

원격 서버의 이미지를 안전하게 허용하려면 [next.config.js](https://nextjs.org/docs/app/api-reference/config/next-config-js)에서 지원되는 URL 패턴 목록을 정의해야 합니다. 악의적인 사용을 방지하려면 최대한 구체적으로 설명하세요. 예를 들어 다음 구성은 특정 AWS S3 버킷의 이미지만 허용합니다:

next.config.ts
```ts
import type { NextConfig } from 'next'
 
const config: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 's3.amazonaws.com',
        port: '',
        pathname: '/my-bucket/**',
        search: '',
      },
    ],
  },
}
 
export default config
```