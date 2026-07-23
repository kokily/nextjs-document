# AI 코딩 에이전트를 위해 Next.js 프로젝트를 설정하는 방법
<span style="color: #808080;">Last updated March 20, 2026</span>

Next.js는 버전이 일치하는 문서를 `next` 패키지 내에 제공하므로 AI 코딩 에이전트가 정확한 최신 API 및 패턴을 참조할 수 있습니다. 프로젝트 루트에 있는 `AGENTS.md` 파일은 에이전트를 교육 데이터 대신 이러한 번들 문서로 안내합니다.

---
## How it works
`next`를 설치하면 Next.js 문서가 `node_modules/next/dist/docs/`에 번들로 제공됩니다. 번들 문서는 [Next.js 문서 사이트](https://nextjs.org/docs)의 구조를 반영합니다:

> node_modules/next/dist/docs/  
> ├── 01-app/  
> │   ├── 01-getting-started/  
> │   ├── 02-guides/  
> │   └── 03-api-reference/  
> ├── 02-pages/  
> ├── 03-architecture/  
> └── index.mdx

즉, 상담원은 설치된 버전과 일치하는 문서에 항상 액세스할 수 있으며 네트워크 요청이나 외부 조회가 필요하지 않습니다.

프로젝트 루트에 있는 `AGENTS.md` 파일은 에이전트에게 코드를 작성하기 전에 이러한 번들 문서를 읽도록 지시합니다. Claude Code, Cursor, GitHub Copilot 등을 포함한 대부분의 AI 코딩 에이전트는 세션을 시작할 때 자동으로 `AGENTS.md`를 읽습니다.

## Getting started
### New projects
[create-next-app](https://nextjs.org/docs/app/api-reference/cli/create-next-app)은 `AGENTS.md` 및 `CLAUDE.md`를 자동으로 생성합니다. 추가 설정이 필요하지 않습니다:

```npm
npx create-next-app@canary
```

에이전트 파일을 원하지 않으면 `--no-agents-md`를 전달하십시오:

```npm
npx create-next-app@canary --no-agents-md
```

### Existing projects
Next.js `v16.2.0-canary.37` 이상인지 확인한 후 프로젝트 루트에 다음 파일을 추가하세요.

`AGENTS.md`에는 에이전트가 읽을 지침이 포함되어 있습니다:

AGENTS.md
```
<!-- BEGIN:nextjs-agent-rules -->
 
# Next.js: 코딩하기 전에 항상 문서를 읽으세요
 
Next.js가 작동하기 전에 `node_modules/next/dist/docs/`에서 관련 문서를 찾아서 읽어보세요. 훈련 데이터가 오래되었습니다. 문서가 진실의 원천입니다.
 
<!-- END:nextjs-agent-rules -->
```

`CLAUDE.md`는 `@` import 구문을 사용하여 `AGENTS.md`를 포함하므로 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 사용자는 콘텐츠를 복제하지 않고도 동일한 지침을 얻을 수 있습니다:

CLAUDE.md
```
@AGENTS.md
```

## Understanding AGENTS.md
기본 `AGENTS.md`에는 단일 집중 지침이 포함되어 있습니다. 코드를 작성하기 전에 번들 문서를 읽으십시오. 이는 의도적으로 최소화된 것입니다. 목표는 에이전트를 오래된 훈련 데이터에서 `node_modules/next/dist/docs/`에 있는 정확하고 버전이 일치하는 문서로 리디렉션하는 것입니다.

`<!-- BEGIN:nextjs-agent-rules -->` 및 `<!-- END:nextjs-agent-rules -->` 주석 표시는 Next.js 관리 섹션을 구분합니다. 향후 업데이트로 인해 덮어쓰여질 염려 없이 이러한 마커 외부에 프로젝트별 지침을 추가할 수 있습니다.

번들 문서에는 앱 라우터 및 페이지 라우터에 대한 가이드, API 참조, 파일 규칙이 포함되어 있습니다. 에이전트가 라우팅, 데이터 가져오기 또는 기타 Next.js 기능과 관련된 작업에 직면하면 잠재적으로 오래된 교육 데이터에 의존하는 대신 번들 문서에서 올바른 API를 찾을 수 있습니다.

> 알아두면 좋은 점: 번들 문서와 `AGENTS.md`가 실제 Next.js 작업에서 에이전트 성능을 어떻게 향상시키는지 확인하려면 [벤치마크 결과](https://nextjs.org/evals)를 방문하세요.