# Upgrading
<span style="color: #808080;">Last updated February 24, 2026</span>

---
## Latest version
Next.js의 최신 버전으로 업데이트하려면 `upgrade` 명령을 사용할 수 있습니다.:

```terminal
npx next upgrade
```

Next.js 16.1.0 이전 버전은 `upgrade` 명령을 지원하지 않으며 대신 별도의 패키지를 사용해야 합니다:

```terminal
npx @next/codemod@canary upgrdae latest
```

수동으로 업그레이드하려면 최신 Next.js 및 React 버전을 설치하세요:

```terminal
npm i next@latest react@latest react-dom@latest eslint-config-next@latest
```

---
## Carnary version
최신 Canary로 업데이트하려면 최신 버전의 Next.js를 사용하고 있고 모든 것이 예상대로 작동하는지 확인하세요. 그런 다음 다음 명령을 실행하십시오:

```terminal
npm i next@canary
```

### Features available in canary
현재 카나리에서는 다음 기능을 사용할 수 있습니다:

Authentication:
- [forbidden](https://nextjs.org/docs/app/api-reference/functions/forbidden)
- [unauthorized](https://nextjs.org/docs/app/api-reference/functions/unauthorized)
- [forbidden.js](https://nextjs.org/docs/app/api-reference/file-conventions/forbidden)
- [unauthorized.js](https://nextjs.org/docs/app/api-reference/file-conventions/unauthorized)
- [authInterrupts](https://nextjs.org/docs/app/api-reference/config/next-config-js/authInterrupts)