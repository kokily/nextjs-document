# Font Optimization
<span style="color: #808080;">Last updated June 23, 2026</span>

`next/font` 모듈은 향상된 개인 정보 보호 및 성능을 위해 자동으로 글꼴을 최적화하고 외부 네트워크 요청을 제거합니다.

여기에는 모든 글꼴 파일에 대한 자체 호스팅이 내장되어 있습니다. 이는 레이아웃을 바꾸지 않고도 웹 글꼴을 최적으로 로드할 수 있음을 의미합니다.

`next/font` 사용을 시작하려면 [next/font/local](https://nextjs.org/docs/app/getting-started/fonts#local-fonts) 또는 [next/font/google](https://nextjs.org/docs/app/getting-started/fonts#google-fonts)에서 가져와서 적절한 옵션을 사용하여 함수로 호출하고 글꼴을 적용하려는 요소의 `className`을 설정하세요. 예를 들어:

app/layout.tsx
```ts
import { Geist } from 'next/font/google'
 
const geist = Geist({
  subsets: ['latin'],
})
 
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={geist.className}>
      <body>{children}</body>
    </html>
  )
}
```

글꼴은 사용되는 컴포넌트에 따라 범위가 지정됩니다. 전체 애플리케이션에 글꼴을 적용하려면 루트 레이아웃에 추가하세요.

---
### Google fonts
모든 Google 글꼴을 자동으로 자체 호스팅할 수 있습니다. 글꼴은 정적 자산으로 포함되며 배포와 동일한 도메인에서 제공됩니다. 즉, 사용자가 사이트를 방문할 때 브라우저에서 Google로 요청이 전송되지 않습니다.

Google 글꼴 사용을 시작하려면 `next/font/google`에서 선택한 글꼴을 가져오세요.:

app/layout.tsx
```ts
import { Geist } from 'next/font/google'
 
const geist = Geist({
  subsets: ['latin'],
})
 
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" className={geist.className}>
      <body>{children}</body>
    </html>
  )
}
```

최고의 성능과 유연성을 위해 [가변 글꼴](https://fonts.google.com/variablefonts)을 사용하는 것이 좋습니다. 하지만 가변 글꼴을 사용할 수 없는 경우에는 가중치를 지정해야 합니다:

app/layout.tsx
```ts
import { Roboto } from 'next/font/google'
 
const roboto = Roboto({
  weight: '400',
  subsets: ['latin'],
})
 
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" className={roboto.className}>
      <body>{children}</body>
    </html>
  )
}
```

---
## Local fonts
로컬 글꼴을 사용하려면 `next/font/local`에서 `localFont` 함수를 가져오고 로컬 글꼴 파일의 [src](https://nextjs.org/docs/app/api-reference/components/font#src)를 지정하세요. 경로는 `localFont`가 호출되는 파일을 기준으로 확인됩니다. 글꼴은 [public](https://nextjs.org/docs/app/api-reference/file-conventions/public-folder) 폴더를 포함하여 프로젝트의 어느 위치에나 저장할 수 있거나 `app` 폴더 내에 공동 위치할 수 있습니다. 예를 들어 `app/fonts/`에 저장된 글꼴을 사용하려면:

app/layout.tsx
```ts
import localFont from 'next/font/local'
 
const myFont = localFont({
  src: './my-font.woff2',
})
 
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" className={myFont.className}>
      <body>{children}</body>
    </html>
  )
}
```

단일 글꼴 모음에 여러 파일을 사용하려는 경우 `src`는 배열일 수 있습니다.:

```ts
const roboto = localFont({
  src: [
    {
      path: './Roboto-Regular.woff2',
      weight: '400',
      style: 'normal',
    },
    {
      path: './Roboto-Italic.woff2',
      weight: '400',
      style: 'italic',
    },
    {
      path: './Roboto-Bold.woff2',
      weight: '700',
      style: 'normal',
    },
    {
      path: './Roboto-BoldItalic.woff2',
      weight: '700',
      style: 'italic',
    },
  ],
})
```