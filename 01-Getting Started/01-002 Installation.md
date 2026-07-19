# 설치
<span style="color: #808080">Last updated March 3, 2026</span>

새 Next.js 앱을 만들고 로컬에서 실행하세요.

---
## 빠른 시작
- my-app이라는 새 Next.js 앱을 만듭니다.
- my-app을 CD로 입력하고 개발 서버를 시작하세요.
- http://localhost:3000 로 접속하세요.

  ```terminal
  npx create-next-app@latest my-app --yes
  cd my-app
  npm run dev
  ```

- **--yes**는 저장된 기본 설정 또는 기본값을 사용하여 프롬프트를 건너뜁니다. 기본 설정에서는 가져오기 별칭 @/*를 사용하여 TypeScript, Tailwind CSS, ESLint, App Router 및 Turbopack을 활성화하고 코딩 에이전트가 최신 Next.js 코드를 작성하도록 안내하는 **AGENTS.md**(이를 참조하는 **CLAUDE.md** 포함)를 포함합니다.

---
## 시스템 요구사항
시작하기 전에 개발 환경이 다음 요구 사항을 충족하는지 확인하세요:

- 최소 Node.js 버전: 20.9
- 운영 체제: macOS, Windows(WSL 포함) 및 Linux.

---
## 지원되는 브라우저
Next.js는 구성이 필요 없는 최신 브라우저를 지원합니다.

- Chrome 111+
- Edge 111+
- Firefox 111+
- Safari 16.4+

폴리필을 구성하고 특정 브라우저를 대상으로 지정하는 방법을 포함하여 [브라우저 지원](https://nextjs.org/docs/architecture/supported-browsers)에 대해 자세히 알아보세요.

---
## CLI를 사용하여 생성
새로운 Next.js 앱을 만드는 가장 빠른 방법은 모든 것을 자동으로 설정하는 create-next-app을 사용하는 것입니다. 프로젝트를 생성하려면 다음을 실행하세요:

```Terminal
npx create-next-app@latest
```

설치 시 다음 메시지가 표시됩니다

```Terminal
What is your project named? my-app
Would you like to use the recommended Next.js defaults?
    Yes, use recommended defaults - TypeScript, ESLint, Tailwind CSS, App Router, AGENTS.md
    No, reuse previous settings
    No, customize settings - Choose your own preferences
```

설정을 사용자 정의하도록 선택하면 다음 메시지가 표시됩니다.

```Terminal
Would you like to use TypeScript? No / Yes
Which linter would you like to use? ESLint / Biome / None
Would you like to use React Compiler? No / Yes
Would you like to use Tailwind CSS? No / Yes
Would you like your code inside a `src/` directory? No / Yes
Would you like to use App Router? (recommended) No / Yes
Would you like to customize the import alias (`@/*` by default)? No / Yes
What import alias would you like configured? @/*
Would you like to include AGENTS.md to guide coding agents to write up-to-date Next.js code? No / Yes
```

프롬프트가 표시되면 create-next-app은 프로젝트 이름으로 폴더를 생성하고 필요한 종속성을 설치합니다.

---
## 수동 설치
새 Next.js 앱을 수동으로 생성하려면 필수 패키지를 설치하세요.

```Terminal
pnpm i next@latest react@latest react-dom@latest
```

```
알아두면 좋은 점:

App Router는 모든 안정적인 React 19 변경 사항과 프레임워크에서 검증되는 새로운 기능을 포함하는 내장된 [React Canary 릴리스](https://react.dev/blog/2023/05/03/react-canaries)를 사용하지만 도구 및 생태계 호환성을 위해 package.json에서 react 및 react-dom을 선언해야 합니다.
Pages Router는 package.json의 React 버전을 사용합니다.
```

그런 다음 package.json 파일에 다음 스크립트를 추가합니다:

```json
package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint",
    "lint:fix": "eslint --fix"
  }
}
```

이 스크립트는 애플리케이션 개발의 다양한 단계를 나타냅니다:

- next dev: Turbopack(기본 번들러)을 사용하여 개발 서버를 시작합니다.
- next build: 프로덕션용 애플리케이션을 빌드합니다.
- next start: 프로덕션 서버를 시작합니다.
- eslint: ESLint를 실행합니다.

이제 Turbopack이 기본 번들러입니다. Webpack을 사용하려면 `next dev --webpack` 또는 `next build --webpack`을 실행하세요. 구성 세부사항은 [Turbopack 문서](https://nextjs.org/docs/app/api-reference/turbopack)를 참조하십시오.

### 앱 디렉터리 만들기
Next.js는 파일 시스템 라우팅을 사용합니다. 즉, 파일 구조에 따라 애플리케이션의 경로가 결정됩니다.

앱 폴더를 만듭니다. 그런 다음 앱 내에서 레이아웃.tsx 파일을 만듭니다. 이 파일은 [루트 레이아웃](https://nextjs.org/docs/app/api-reference/file-conventions/layout#root-layout)입니다. 이는 필수이며 `<html>` 및 `<body>` 태그를 포함해야 합니다.

app/layout.tsx
```typescript
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

일부 초기 콘텐츠가 포함된 홈 페이지 app/page.tsx를 만듭니다.

app/page.tsx
```typeScript
export default function Page() {
  return <h1>Hello, Next.js!</h1>
}
```

사용자가 애플리케이션의 루트(/)를 방문하면 layout.tsx와 page.tsx가 모두 렌더링됩니다.
![](./images/002/app-getting-started.avif)

```
알아두면 좋은 점:

루트 레이아웃 생성을 잊어버린 경우 `next dev`로 개발 서버를 실행할 때 Next.js가 자동으로 이 파일을 생성합니다.
선택적으로 프로젝트 루트의 [src 폴더](https://nextjs.org/docs/app/api-reference/file-conventions/src-folder)를 사용하여 구성 파일에서 애플리케이션 코드를 분리할 수 있습니다.
```

### 공용 폴더 생성(선택 사항)
프로젝트 루트에 [공용 폴더](https://nextjs.org/docs/app/api-reference/file-conventions/public-folder)를 생성하여 이미지, 글꼴 등과 같은 정적 자산을 저장합니다.
공용 내의 파일은 기본 URL(/)부터 시작하는 코드에서 참조할 수 있습니다.

그런 다음 루트 경로(`/`)를 사용하여 이러한 자산을 참조할 수 있습니다. 예를 들어 `public/profile.png`는 `/profile.png`로 참조될 수 있습니다.

app/page.tsx
```typeScript
import Image from 'next/image'
 
export default function Page() {
  return <Image src="/profile.png" alt="Profile" width={100} height={100} />
}
```

---
## 개발 서버 실행
- npm run dev를 실행하여 개발 서버를 시작합니다.
- 애플리케이션을 보려면 http://localhost:3000을 방문하세요.
- app/page.tsx 파일을 편집하고 저장하여 브라우저에서 업데이트된 결과를 확인하세요.

---
## TypeScript 설정
`최소 TypeScript 버전: v5.1.0`

Next.js에는 TypeScript 지원이 내장되어 있습니다. 프로젝트에 TypeScript를 추가하려면 파일 이름을 `.ts` / `.tsx`로 바꾸고 `next dev`를 실행하세요. Next.js는 필요한 종속성을 자동으로 설치하고 권장 구성 옵션과 함께 `tsconfig.json` 파일을 추가합니다.

### IDE 플러그인
Next.js에는 VSCode 및 기타 코드 편집기가 고급 유형 검사 및 자동 완성에 사용할 수 있는 사용자 정의 TypeScript 플러그인 및 유형 검사기가 포함되어 있습니다.

다음과 같이 VS Code에서 플러그인을 활성화할 수 있습니다:

- 명령 팔레트 열기 (Ctrl/⌘ + Shift + P)
- "TypeScript: TypeScript 버전 선택" 검색 중
- "Workspace 버전 사용" 선택

![](./images/002/typescript-command-palette.avif)

자세한 내용은 [TypeScript 참조](https://nextjs.org/docs/app/api-reference/config/typescript) 페이지를 참조하세요.

---
## 린팅 설정
Next.js는 ESLint 또는 Biome을 사용한 Linting을 지원합니다. 린터를 선택하고 `package.json` 스크립트를 통해 직접 실행하세요.

- ESLint 사용(포괄적인 규칙):
  ```json
  package.json
  {
    "scripts": {
      "lint": "eslint",
      "lint:fix": "eslint --fix"
    }
  }
  ```

- 또는 Biome(빠른 린터 포맷터)을 사용하세요:
  ```json
  package.json
  {
    "scripts": {
      "lint": "biome check",
      "format": "biome format --write"
    }
  }
  ```

프로젝트에서 이전에 `next lint`를 사용한 경우 codemod를 사용하여 스크립트를 ESLint CLI로 마이그레이션하세요:

```terminal
npx @next/codemod@canary next-lint-to-eslint-cli .
```

ESLint를 사용하는 경우 명시적 구성을 만듭니다(`eslint.config.mjs` 권장). ESLint는 [기존 .eslintrc.*와 최신 eslint.config.mjs 형식](https://eslint.org/docs/latest/use/configure/configuration-files#configuring-eslint)을 모두 지원합니다. 권장 설정은 [ESLint API 참조](https://nextjs.org/docs/app/api-reference/config/eslint#with-core-web-vitals)를 참조하세요.

```
알아두면 좋은 점: Next.js 16부터 다음 빌드에서는 더 이상 Linter를 자동으로 실행하지 않습니다. 대신 NPM 스크립트를 통해 linter를 실행할 수 있습니다.
```

자세한 내용은 [ESLint 플러그인](https://nextjs.org/docs/app/api-reference/config/eslint) 페이지를 참조하세요.

---
## 절대 가져오기 및 모듈 경로 별칭 설정
Next.js에는 tsconfig.json 및 jsconfig.json 파일의 "paths" 및 "baseUrl" 옵션에 대한 지원이 내장되어 있습니다.

이러한 옵션을 사용하면 프로젝트 디렉터리의 별칭을 절대 경로로 지정하여 모듈을 더 쉽고 깔끔하게 가져올 수 있습니다. 예를 들어:

```typescript
// Before
import { Button } from '../../../components/button'
 
// After
import { Button } from '@/components/button'
```

절대 가져오기를 구성하려면 tsconfig.json 또는 jsconfig.json 파일에 baseUrl 구성 옵션을 추가하세요. 예를 들어:

```json
tsconfig.json or jsconfig.json
{
  "compilerOptions": {
    "baseUrl": "src/"
  }
}
```

baseUrl 경로를 구성하는 것 외에도 "경로" 옵션을 사용하여 모듈 경로를 "별칭"으로 설정할 수 있습니다.

예를 들어 다음 구성은 `@/comComponents/*`를 `Components/*`에 매핑합니다:

```json
tsconfig.json or jsconfig.json
{
  "compilerOptions": {
    "baseUrl": "src/",
    "paths": {
      "@/styles/*": ["styles/*"],
      "@/components/*": ["components/*"]
    }
  }
}
```

각 "경로"는 baseUrl 위치를 기준으로 합니다.