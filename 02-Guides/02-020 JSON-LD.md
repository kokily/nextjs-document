# How to implement JSON-LD in your Next.js application
<span style="color: #808080;">Last updated March 2, 2026</span>

[JSON-LD](https://json-ld.org/)는 검색 엔진과 AI가 순수한 콘텐츠를 넘어 페이지 구조를 이해하는 데 도움을 주는 데 사용할 수 있는 구조화된 데이터 형식입니다. 예를 들어, 사람, 사건, 조직, 영화, 책, 요리법 및 기타 여러 유형의 개체를 설명하는 데 사용할 수 있습니다.

JSON-LD에 대한 현재 권장 사항은 `layout.js` 또는 `page.js` 컴포넌트에서 구조화된 데이터를 `<script>` 태그로 렌더링하는 것입니다.

다음 스니펫은 XSS 주입에 사용되는 악성 문자열을 삭제하지 않는 `JSON.stringify`를 사용합니다. 이러한 유형의 취약점을 방지하려면 예를 들어 문자 `<`를 해당 유니코드 `\u003c`로 대체하여 `JSON-LD` 페이로드에서 `HTML` 태그를 제거할 수 있습니다.

잠재적으로 위험한 문자열을 삭제하기 위해 조직에서 권장하는 접근 방식을 검토하거나 [serialize-javascript](https://www.npmjs.com/package/serialize-javascript)와 같은 `JSON.stringify`에 대해 커뮤니티에서 유지 관리하는 대안을 사용하세요.

app/products/[id]/page.tsx
```tsx
export default async function Page({ params }) {
  const { id } = await params
  const product = await getProduct(id)

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    image: product.image,
    description: product.description,
  }

  return (
    <section>
      {/* 페이지에 JSON-LD 추가 */}
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{
          __html: JSON.stringify(jsonLd).replace(/</g, '\\u003c'),
        }}
      />
      {/* ... */}
    </section>
  )
}
```

Google용 [리치 결과 테스트](https://search.google.com/test/rich-results) 또는 일반 [스키마 마크업 검사기](https://validator.schema.org/)를 사용하여 구조화된 데이터를 검증하고 테스트할 수 있습니다.

[`schema-dts`](https://www.npmjs.com/package/schema-dts)와 같은 커뮤니티 패키지를 사용하여 TypeScript로 JSON-LD를 입력할 수 있습니다.:

```tsx
import { Product, WithContext } from 'schema-dts'

const jsonLd: WithContext<Product> = {
  '@context': 'https://schema.org',
  '@type': 'Product',
  name: 'Next.js Sticker',
  image: 'https://nextjs.org/imgs/sticker.png',
  description: 'Dynamic at the speed of static.',
}
```

> **알아두면 좋은 점**: `next/script` 컴포넌트는 JavaScript 로드 및 실행에 최적화되어 있습니다. JSON-LD는 실행 가능한 코드가 아닌 구조화된 데이터이므로 여기서는 기본 `<script>` 태그가 올바른 선택입니다.