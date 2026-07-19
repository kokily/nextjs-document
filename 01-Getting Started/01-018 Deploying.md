# Deploying
<span style="color: #808080;">Last updated June 23, 2026</span>

Next.js는 Node.js 서버, Docker 컨테이너, 정적 내보내기로 배포하거나 다른 플랫폼에서 실행되도록 조정할 수 있습니다.

|Deployment Option|Feature Support|
|---|---|
|[Node.js server](https://nextjs.org/docs/app/getting-started/deploying#nodejs-server)|All|
|[Docker container](https://nextjs.org/docs/app/getting-started/deploying#docker)|All|
|[Static export](https://nextjs.org/docs/app/getting-started/deploying#static-export)|Limited|
|[Adapters](https://nextjs.org/docs/app/getting-started/deploying#adapters)|Varies([verified](https://nextjs.org/docs/app/getting-started/deploying#verified-adapters) adapters run the test suite)|

---
## Node.js server
Next.js는 Node.js를 지원하는 모든 공급자에 배포될 수 있습니다.
`package.json`에 `"build"` 및 `"start"` 스크립트가 있는지 확인하세요:

package.json
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  }
}
```

그런 다음 `npm run build`를 실행하여 애플리케이션을 빌드하고 `npm run start`를 실행하여 Node.js 서버를 시작하세요. 이 서버는 모든 Next.js 기능을 지원합니다. 필요한 경우 [사용자 정의 서버](https://nextjs.org/docs/app/guides/custom-server)로 꺼낼 수도 있습니다.

Node.js 배포는 모든 Next.js 기능을 지원합니다. 인프라에 맞게 [구성하는 방법](https://nextjs.org/docs/app/guides/self-hosting)을 알아보세요.

### Templates
- [Flightcontrol](https://github.com/nextjs/deploy-flightcontrol)
- [Railway](https://github.com/nextjs/deploy-railway)
- [Replit](https://github.com/nextjs/deploy-replit)
- [Hostinger](https://github.com/hostinger/deploy-nextjs)

## Docker
Next.js는 [Docker](https://www.docker.com/) 컨테이너를 지원하는 모든 공급자에 배포될 수 있습니다. 여기에는 Kubernetes와 같은 컨테이너 조정자 또는 Docker를 실행하는 클라우드 공급자가 포함됩니다. 앱 컨테이너화에 대한 모범 사례는 Docker의 공식 [Next.js](https://docs.docker.com/guides/nextjs) 및 [React.js](https://docs.docker.com/guides/reactjs) 가이드를 참조하세요.

Docker 배포는 모든 Next.js 기능을 지원합니다. 인프라에 맞게 [구성하는 방법](https://nextjs.org/docs/app/guides/self-hosting)을 알아보세요.

> 개발 참고 사항: Docker는 프로덕션 배포에 탁월하지만 성능 향상을 위해 Mac 및 Windows에서 개발하는 동안 Docker 대신 로컬 개발(npm run dev)을 사용하는 것이 좋습니다. [지역 개발 최적화](https://nextjs.org/docs/app/guides/local-development)에 대해 자세히 알아보세요.

### Templates
다음 예는 Next.js 애플리케이션 컨테이너화에 대한 모범 사례를 보여줍니다:

- [Docker 독립 실행형 출력](https://github.com/vercel/next.js/tree/canary/examples/with-docker) - `output: "standalone"`을 사용하여 Next.js 애플리케이션을 배포하여 필요한 런타임 파일과 종속성만 포함된 프로덕션에 즉시 사용 가능한 최소 Docker 이미지를 생성합니다.
- [Docker 내보내기 출력](https://github.com/vercel/next.js/tree/canary/examples/with-docker-export-output) - `output: "export"`를 사용하여 완전 정적 Next.js 애플리케이션을 배포하여 경량 컨테이너 또는 정적 호스팅 환경에서 제공할 수 있는 최적화된 HTML 파일을 생성합니다.
- [Docker 다중 환경](https://github.com/vercel/next.js/tree/canary/examples/with-docker-multi-env) - 다양한 환경 변수를 사용하여 개발, 스테이징 및 프로덕션 환경에 대한 별도의 Docker 구성을 관리합니다.

또한 호스팅 공급자는 Next.js 배포에 대한 지침을 제공합니다:

- [DigitalOcean](https://github.com/nextjs/deploy-digitalocean)
- [Fly.io](https://github.com/nextjs/deploy-fly)
- [Google Cloud Run](https://github.com/nextjs/deploy-google-cloud-run)
- [Render](https://github.com/nextjs/deploy-render)
- [SST](https://github.com/nextjs/deploy-sst)

## Static export
Next.js를 사용하면 정적 사이트 또는 [SPA(단일 페이지 애플리케이션)](https://nextjs.org/docs/app/guides/single-page-applications)로 시작한 다음 나중에 서버가 필요한 기능을 사용하도록 선택적으로 업그레이드할 수 있습니다.

Next.js는 [정적 내보내기](https://nextjs.org/docs/app/guides/static-exports)를 지원하므로 HTML/CSS/JS 정적 자산을 제공할 수 있는 모든 웹 서버에 배포하고 호스팅할 수 있습니다. 여기에는 AWS S3, Nginx 또는 Apache와 같은 도구가 포함됩니다.

[정적 내보내기](https://nextjs.org/docs/app/guides/static-exports)로 실행하면 서버가 필요한 Next.js 기능이 지원되지 않습니다. [자세히 알아보세요.](https://nextjs.org/docs/app/guides/static-exports#unsupported-features)

### Templates
- [GitHub Pages](https://github.com/nextjs/deploy-github-pages)

## Adapters
Next.js는 인프라 기능을 지원하기 위해 다양한 플랫폼에서 실행되도록 조정할 수 있습니다. [배포 어댑터 API](https://nextjs.org/docs/app/api-reference/config/next-config-js/adapterPath)를 사용하면 플랫폼에서 Next.js 애플리케이션을 구축하고 배포하는 방법을 사용자 지정할 수 있습니다.

### verified Adapters
검증된 어댑터는 오픈 소스이며 전체 [Next.js 호환성 테스트 제품군](https://nextjs.org/docs/app/api-reference/adapters/testing-adapters)을 실행하고 [Next.js GitHub 조직](https://github.com/nextjs)에서 호스팅됩니다. Next.js 팀은 주요 릴리스 전에 이러한 플랫폼으로 테스트를 조정합니다. 각 어댑터에 대한 공개 테스트 결과가 곧 공개될 예정입니다. [검증된 어댑터에 대해 자세히 알아보세요.](https://nextjs.org/docs/app/guides/deploying-to-platforms#verified-adapters)

- [Vercel](https://vercel.com/docs/frameworks/nextjs)
- [Bun](https://bun.com/docs/guides/ecosystem/nextjs)

Cloudflare와 Netlify는 Adapter API를 기반으로 구축된 검증된 어댑터에 대해 작업하고 있습니다. 그 동안 자체 Next.js 통합을 제공합니다(아래 참조).

### Other Platforms
다음 플랫폼은 자체 Next.js 통합을 제공합니다. 이는 공개 [Adapter API](https://nextjs.org/docs/app/api-reference/config/next-config-js/adapterPath)를 기반으로 구축되지 않았으며 Next.js 팀에서 확인하지 않았으므로 기능 지원 및 호환성이 다를 수 있습니다. 자세한 내용은 각 공급자의 설명서를 참조하세요.:

- [Appwrite Sites](https://appwrite.io/docs/products/sites/quick-start/nextjs)
- [AWS Amplify Hosting](https://docs.amplify.aws/nextjs/start/quickstart/nextjs-app-router-client-components)
- [Cloudflare](https://developers.cloudflare.com/workers/frameworks/framework-guides/nextjs)
- [Deno Deploy](https://docs.deno.com/examples/next_tutorial)
- [Firebase App Hosting](https://firebase.google.com/docs/app-hosting/get-started)
- [Netlify](https://docs.netlify.com/frameworks/next-js/overview/#next-js-support-on-netlify)

특정 플랫폼 기능이 필요한 Next.js 기능에 대한 자세한 내용은 [플랫폼에 배포](https://nextjs.org/docs/app/guides/deploying-to-platforms)를 참조하세요.