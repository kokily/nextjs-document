# How to set up instrumentation

<span style="color: #808080;">Last updated February 16, 2026</span>

계측은 코드를 사용하여 모니터링 및 로깅 도구를 애플리케이션에 통합하는 프로세스입니다. 이를 통해 애플리케이션의 성능과 동작을 추적하고 프로덕션에서 문제를 디버깅할 수 있습니다.

## Convention
계측을 설정하려면 프로젝트의 **루트 디렉터리**(또는 사용하는 경우 [`src`](/docs/app/api-reference/file-conventions/src-folder) 폴더 내부)에 `instrumentation.ts|js` 파일을 만듭니다.

그런 다음 파일에 `register` 함수를 내보냅니다. 이 함수는 새로운 Next.js 서버 인스턴스가 시작될 때 **한 번** 호출되며, 서버가 요청을 처리할 준비가 되기 전에 완료되어야 합니다.

예를 들어 [OpenTelemetry](https://opentelemetry.io/) 및 [@vercel/otel](https://vercel.com/docs/observability/otel-overview)과 함께 Next.js를 사용하려면:

instrumentation.ts
```ts
import { registerOTel } from '@vercel/otel'

export function register() {
  registerOTel('next-app')
}
```

전체 구현을 보려면 [OpenTelemetry를 사용한 Next.js 예시](https://github.com/vercel/next.js/tree/canary/examples/with-opentelemetry)를 참조하세요.

> **알아두면 좋은 점**:
>
> * `instrumentation` 파일은 `app` 또는 `pages` 디렉터리가 아닌 프로젝트 루트에 있어야 합니다. `src` 폴더를 사용하는 경우 `pages` 및 `app`과 함께 `src` 안에 파일을 배치하세요.
> * [`pageExtensions` 구성 옵션](/docs/app/api-reference/config/next-config-js/pageExtensions)을 사용하여 접미사를 추가하는 경우 일치하도록 `instrumentation` 파일 이름도 업데이트해야 합니다.

## Examples
### Importing files with side effects
때로는 발생할 수 있는 부작용 때문에 코드에서 파일을 가져오는 것이 유용할 수 있습니다. 예를 들어, 전역 변수 세트를 정의하는 파일을 가져올 수 있지만 가져온 파일을 코드에서 명시적으로 사용하지 않을 수 있습니다. 패키지가 선언한 전역 변수에는 계속 액세스할 수 있습니다.

`register` 기능 내에서 JavaScript `import` 구문을 사용하여 파일을 가져오는 것이 좋습니다. 다음 예는 `register` 함수에서 `import`의 기본 사용법을 보여줍니다:

instrumentation.ts
```ts
export async function register() {
  await import('package-with-side-effect')
}
```

> **알아두면 좋은 점:**
>
> 파일 상단보다는 `register` 기능 내에서 파일을 가져오는 것이 좋습니다. 이렇게 하면 모든 부작용을 코드의 한 위치에 배치할 수 있으며 파일 상단에서 전체적으로 가져오는 데 따른 의도하지 않은 결과를 방지할 수 있습니다.

### Importing runtime-specific code
Next.js는 모든 환경에서 `register`를 호출하므로 특정 런타임을 지원하지 않는 코드(예: [Edge 또는 Node.js](/docs/app/api-reference/edge))를 조건부로 가져오는 것이 중요합니다. `NEXT_RUNTIME` 환경 변수를 사용하여 현재 환경을 가져올 수 있습니다:

instrumentation.ts
```ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./instrumentation-node')
  }

  if (process.env.NEXT_RUNTIME === 'edge') {
    await import('./instrumentation-edge')
  }
}
```

## Learn more about Instrumentation- [instrumentation.js](/docs/app/api-reference/file-conventions/instrumentation)
  - Instrumentation.js 파일에 대한 API 참조입니다.