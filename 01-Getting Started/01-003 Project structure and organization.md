# 프로젝트 구조 및 조직
<span style="color: #808080">Last updated June 23, 2026</span>

이 페이지는 Next.js의 모든 폴더 및 파일 규칙에 대한 개요와 프로젝트 구성에 대한 권장 사항을 제공합니다.

---
## 폴더 및 파일 규칙
### 최상위 폴더
최상위 폴더는 애플리케이션의 코드와 정적 자산을 구성하는 데 사용됩니다.

![](./images/003/top-level-folders.avif)

|||
|---|---|
|`app`|앱 라우터|
|`pages`|페이지 라우터|
|`public`|제공할 정적 자산|
|`src`|선택적 애플리케이션 소스 폴더|

### 최상위 파일들
최상위 파일은 애플리케이션 구성, 종속성 관리, 프록시 실행, 모니터링 도구 통합 및 환경 변수 정의에 사용됩니다.

|Next.js||
|---|---|
|`next.config.js`|Next.js 구성 파일|
|`package.json`|프로젝트 종속성 및 스크립트|
|`instrumentation.ts`|OpenTelemetry 및 계측 파일|
|`proxy.ts`|Next.js 요청 프록시|
|`.env`|환경변수 (버전 관리로 추적하면 안 됩니다.)
|`.env.local`|로컬 환경 변수 (버전 관리로 추적하면 안 됩니다.)|
|`.env.production`|프로덕션 환경 변수 (버전관리로 추적하면 안 됩니다.)|
|`.env.development`|개발 환경 변수 (버전관리로 추적하면 안 됩니다.)|
|`eslint.config.mjs`|ESLint용 구성 파일|
|`.gitignore`|무시할 Git 파일 및 폴더|
|`next-env.d.ts`|Next.js용 TypeScript 선언 파일 (버전관리로 추적하면 안 됩니다.)|
|`tsconfig.json`|TypeScript용 구성 파일|
|`jsconfig.json`|JavaScript용 구성 파일|

### 라우팅 파일
경로를 노출하는 페이지를 추가하고, 헤더, 탐색, 바닥글과 같은 공유 UI의 레이아웃, 스켈레톤 로드, 오류 경계에 대한 오류, API에 대한 경로를 추가합니다.

||||
|---|---|---|
|`layout`|`.js` `.jsx` `.tsx`|Layout|
|`page`|`.js` `.jsx` `.tsx` |Page|
|`loading`|`.js` `.jsx` `.tsx` |Loading UI|
|`not-found`|`.js` `.jsx` `.tsx` |Not found UI|
|`error`|`.js` `.jsx` `.tsx` |Error UI|
|`global-error`|`.js` `.jsx` `.tsx` |Global error UI|
|`route`|`.js` `.ts`|API endpoint|
|`template`|`.js` `.jsx` `.tsx` |Re-rendered layout|
|`default`|`.js` `.jsx` `.tsx` |Parallel route fallback page|

### 중첩된 경로
폴더는 URL 세그먼트를 정의합니다. 중첩 폴더는 세그먼트를 중첩합니다. 모든 수준의 레이아웃은 하위 세그먼트를 래핑합니다. `페이지`나 `경로 파일`이 존재하면 경로가 공개됩니다.

|Path|URL pattern|Notes|
|---|---|---|
|`app/layout.tsx`|-|루트 레이아웃은 모든 경로를 래핑합니다.|
|`app/blog/layout.tsx`|-|`/blog`와 하위 항목을 래핑합니다.|
|`app/page.tsx`|`/`|Public route|
|`app/blog/page.tsx`|`/blog`|Public route|
|`app/blog/authors/page.tsx`|`/blog/authors`|Public route|

### 동적 경로
대괄호를 사용하여 세그먼트를 매개변수화합니다. 단일 매개변수에는 `[segment]`를 사용하고, 모두 포괄하려면 `[...segment]`를 사용하고, 선택적 포괄에는 `[[...segment]]`를 사용합니다. [params](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional) prop을 통해 값에 접근합니다.

|Path|URL Pattern|
|---|---|
|`app/blog/[slug]/page.tsx`|`/blog/my-first-post`|
|`app/shop/[...slug]/page.tsx`|`/shop/clothing`, `/shop/clothing/shirts`|
|`app/docs/[[...slug]]/page.tsx`|`/docs`, `/docs/layouts-and-pages`, `/docs/api-reference/use-router`|

### 경로 그룹 및 개인 폴더
경로 그룹[(그룹)](https://nextjs.org/docs/app/api-reference/file-conventions/route-groups#convention)을 사용하여 URL을 변경하지 않고 코드를 구성하고, 개인 폴더 [_folder](https://nextjs.org/docs/app/getting-started/project-structure#private-folders)를 사용하여 라우팅할 수 없는 파일을 함께 배치합니다.

|Path|URL pattern|Notes|
|---|---|---|
|`app/(marketing)/page.tsx`|`/`|URL에서 그룹이 생략됨|
|`app/(shop)/cart/page.tsx`|`/cart`|`(shop)` 내에서 레이아웃 공유|
|`app/blog/_components/Post.tsx`|`-`|라우팅할 수 없습니다. UI 유틸리티를 위한 안전한 장소|
|`app/blog/_lib/data.ts`|`-`|라우팅할 수 없습니다. UI 유틸리티를 위한 안전한 장소|

### 평행 및 차단 경로
이러한 기능은 슬롯 기반 레이아웃이나 모달 라우팅과 같은 특정 UI 패턴에 적합합니다.
상위 레이아웃에 의해 렌더링된 명명된 슬롯에는 `@slot`을 사용합니다. 인터셉트 패턴을 사용하면 URL을 변경하지 않고 현재 레이아웃 내에서 다른 경로를 렌더링할 수 있습니다. 예를 들어 세부 정보 보기를 목록 위에 모달로 표시합니다.

|Pattern (docs)|의미|일반적인 사용 사례|
|---|---|---|
|`@folder`|Named slot|Sidebar + main content|
|`(.)folder`|Intercept same level|Preview sibling route in a modal|
|`(..)folder`|Intercept parent|Open a child of the parent as an overlay|
|`(..)` `(..)folder`|intercept two levels|Deeply nested overlay|
|`(...)folder`|Intercept from root|Show arbitrary route in current view|

### 메타데이터 파일 규칙
#### App Icons

||||
|---|---|---|
|`favicon`|`.ico`|Favicon file|
|`icon`|`.ico` `.jpg` `.jpeg` `.png` `.svg`|App Icon file|
|`icon`|`.js` `.ts` `.tsx`|Generated App Icon|
|`apple-icon`|`.jpg` `.jpeg` `.png`|Apple App Icon file|
|`apple-icon`|`.js` `.ts` `.tsx`|Generated Apple App Icon|

#### Open Graph and Twitter Images

||||
|---|---|---|
|`opengraph-image`|`.jpg` `.jpeg` `.png` `.gif`|Open Graph Image file|
|`opengraph-image`|`.js` `.ts` `.tsx`|Generated Open Graph Image|
|`twitter-image`|`.jpg` `.jpeg` `.png` `.gif`| Twitter Image file|
|`twitter-image`|`.js` `.ts` `.tsx`|Generated Twitter Image|

#### SEO

||||
|---|---|---|
|`sitemap`|`.xml`|Sitemap file|
|`sitemap`|`.js` `.ts`|Generated Sitemap|
|`robots`|`.txt`|Robots file|
|`robots`|`.js` `.ts`|Generated Robots file|

---
## 프로젝트 구성
Next.js는 프로젝트 파일을 구성하고 배치하는 방법에 대해 의견이 없습니다. 하지만 프로젝트를 구성하는 데 도움이 되는 여러 기능을 제공합니다.

### 구성 요소 계층 구조
특수 파일에 정의된 구성 요소는 특정 계층 구조로 렌더링됩니다:

- layout.js
- template.js
- error.js (React error boundary)
- loading.js (React suspense boundary)
- not-found.js (React error boundary for "not found" UI)
- page.js or nested layout.js

![](./images/003/file-conventions-component-hierarchy.avif)

컴포넌트는 중첩된 경로에서 반복적으로 렌더링됩니다. 즉, 경로 세그먼트의 컴포넌트는 상위 세그먼트의 컴포넌트 내에 중첩됩니다.

![](./images/003/nested-file-conventions-component-hierarchy.avif)

### Colocation
`app` 디렉터리에서 중첩된 폴더는 경로 구조를 정의합니다. 각 폴더는 URL 경로의 해당 세그먼트에 매핑된 경로 세그먼트를 나타냅니다.

그러나 경로 구조가 폴더를 통해 정의되더라도 `page.js` 또는 `route.js` 파일이 경로 세그먼트에 추가될 때까지 경로에 공개적으로 액세스할 수 없습니다.

![](./images/003/project-organization-not-routable.avif)

또한 경로가 공개적으로 액세스 가능하도록 설정된 경우에도 `page.js` 또는 `route.js`에서 반환된 콘텐츠만 클라이언트로 전송됩니다.

![](./images/003/project-organization-routable.avif)

이는 실수로 라우팅되지 않고 프로젝트 파일이 `app` 디렉터리의 경로 세그먼트 내에 안전하게 같은 위치에 배치될 수 있음을 의미합니다.

![](./images/003/project-organization-colocation.avif)

```
알아두면 좋은 점: `app`에서 프로젝트 파일을 같은 위치에 배치할 수 있지만 반드시 그럴 필요는 없습니다. 원하는 경우 [앱 디렉토리 외부에 보관](https://nextjs.org/docs/app/getting-started/project-structure#store-project-files-outside-of-app)할 수 있습니다.
```

### 개인 폴더
개인 폴더는 폴더 앞에 밑줄을 붙여 생성할 수 있습니다: `_folderName`

이는 폴더가 비공개 구현 세부 사항이므로 라우팅 시스템에서 고려해서는 안 됨을 나타냅니다. 따라서 폴더와 해당 하위 폴더는 모두 라우팅에서 제외됩니다.

![](./images/003/project-organization-private-folders.avif)

`app` 디렉터리의 파일은 기본적으로 [안전하게 공동 배치](https://nextjs.org/docs/app/getting-started/project-structure#colocation)될 수 있으므로 공동 배치에는 개인 폴더가 필요하지 않습니다. 그러나 다음과 같은 경우에 유용할 수 있습니다:

- 라우팅 로직에서 UI 로직을 분리합니다.
- 프로젝트와 Next.js 생태계 전반에 걸쳐 내부 파일을 일관되게 구성합니다.
- 코드 편집기에서 파일 정렬 및 그룹화
- 향후 Next.js 파일 규칙과의 잠재적인 이름 지정 충돌을 방지합니다.

```
알아두면 좋은 점:

프레임워크 규칙은 아니지만 동일한 밑줄 패턴을 사용하여 개인 폴더 외부의 파일을 "개인"으로 표시하는 것을 고려할 수도 있습니다.
폴더 이름 앞에 %5F(URL 인코딩된 밑줄 형식): %5FfolderName을 붙여 밑줄로 시작하는 URL 세그먼트를 생성할 수 있습니다.
개인 폴더를 사용하지 않는 경우 예기치 않은 이름 충돌을 방지하기 위해 Next.js [특수 파일 규칙](https://nextjs.org/docs/app/getting-started/project-structure#routing-files)을 아는 것이 도움이 될 것입니다.
```

## 경로 그룹
경로 그룹은 폴더를 괄호로 묶어 생성할 수 있습니다.: (folderName)

이는 폴더가 조직화 목적으로 사용되며 경로의 URL 경로에 포함되어서는 안 됨을 나타냅니다.

![](./images/003/project-organization-route-groups.avif)

경로 그룹은 다음에 유용합니다:

- 사이트 섹션, 의도 또는 팀별로 경로를 구성합니다. 예를 들어 마케팅 페이지, 관리 페이지 등
- 동일한 경로 세그먼트 수준에서 중첩 레이아웃 활성화:
  - [여러 루트 레이아웃을 포함하여 동일한 세그먼트에 여러 중첩 레이아웃 만들기](https://nextjs.org/docs/app/getting-started/project-structure#creating-multiple-root-layouts)
  - [공통 구간의 경로 하위 집합에 레이아웃 추가](https://nextjs.org/docs/app/getting-started/project-structure#opting-specific-segments-into-a-layout)

### `src` 폴더
Next.js는 선택적 [src 폴더](https://nextjs.org/docs/app/api-reference/file-conventions/src-folder) 내에 애플리케이션 코드(`app` 포함) 저장을 지원합니다. 이는 주로 프로젝트 루트에 있는 프로젝트 구성 파일에서 애플리케이션 코드를 분리합니다.

![](./images/003/project-organization-src-directory.avif)

## 예제들
다음 섹션에는 일반적인 전략에 대한 매우 높은 수준의 개요가 나열되어 있습니다. 가장 간단한 요점은 귀하와 귀하의 팀에 적합하고 프로젝트 전반에 걸쳐 일관성을 유지하는 전략을 선택하는 것입니다.

```
알아두면 좋은 점: 아래 예에서는 `components`와 `lib` 폴더를 일반화된 자리 표시자로 사용하고 있으며 이름 지정에는 특별한 프레임워크 의미가 없으며 프로젝트에서 `ui`, `utils`, `hooks`, `styles 등과 같은 다른 폴더를 사용할 수 있습니다.
```

### 앱 외부에 프로젝트 파일 저장
이 전략은 모든 애플리케이션 코드를 프로젝트 루트의 공유 폴더에 저장하고 순전히 라우팅 목적으로만 `app` 디렉터리를 유지합니다.

![](./images/003/project-organization-project-root.avif)

### Store project files in top-level folders inside of app
이 전략은 모든 애플리케이션 코드를 `app` 디렉터리 루트의 공유 폴더에 저장합니다.

![](./images/003/project-organization-app-root.avif)

### 기능이나 경로별로 프로젝트 파일 분할
이 전략은 전역적으로 공유되는 애플리케이션 코드를 루트 `app` 디렉터리에 저장하고 보다 구체적인 애플리케이션 코드를 이를 사용하는 경로 세그먼트로 분할합니다.

![](./images/003/project-organization-app-root-split.avif)

### URL 경로에 영향을 주지 않고 경로 구성
URL에 영향을 주지 않고 경로를 구성하려면 관련 경로를 함께 유지하는 그룹을 만드세요. 괄호 안의 폴더는 URL(예: (marketing) 또는 (shop))에서 생략됩니다.

![](./images/003/route-group-organisation.avif)

(marketing) 및 (shop) 내부 경로가 동일한 URL 계층 구조를 공유하더라도 해당 폴더 내에 `layout.js` 파일을 추가하여 각 그룹마다 다른 레이아웃을 만들 수 있습니다.

![](./images/003/route-group-multiple-layouts.avif)

### 특정 세그먼트를 레이아웃으로 선택
특정 경로를 레이아웃으로 선택하려면 새 경로 그룹(예: (shop))을 만들고 동일한 레이아웃을 공유하는 경로를 그룹(예: account 및 cart)으로 이동합니다. 그룹 외부의 경로는 레이아웃을 공유하지 않습니다(예: checkout).

![](./images/003/route-group-opt-in-layouts.avif)

### 특정 경로에서 스켈레톤 로드 선택
`loading.js` 파일을 통해 [로딩 스켈레톤](https://nextjs.org/docs/app/api-reference/file-conventions/loading)을 특정 경로에 적용하려면 새 경로 그룹(예: `/(overview)`)을 만든 다음 `loading.tsx`를 해당 경로 그룹 내로 이동하세요.

![](./images/003/route-group-loading.avif)

이제 `loading.tsx` 파일은 URL 경로 구조에 영향을 주지 않고 모든 대시보드 페이지가 아닌 대시보드 → 개요 페이지에만 적용됩니다.

### 여러 루트 레이아웃 만들기
여러 [루트 레이아웃](https://nextjs.org/docs/app/api-reference/file-conventions/layout#root-layout)을 생성하려면 최상위 `layout.js` 파일을 제거하고 각 경로 그룹 내에 `layout.js` 파일을 추가합니다. 이는 완전히 다른 UI나 경험을 가진 섹션으로 애플리케이션을 분할하는 데 유용합니다. `<html>` 및 `<body>` 태그를 각 루트 레이아웃에 추가해야 합니다.

![](./images/003/route-group-multiple-root-layouts.avif)

위의 예에서 `(marketing)`과 `(shop)`은 모두 자체 루트 레이아웃을 가지고 있습니다.