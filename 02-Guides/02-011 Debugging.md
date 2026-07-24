# How to use debugging tools with Next.js
<span style="color: #808080;">Last updated July 22, 2026</span>

이 문서에서는 [VS 코드 디버거](https://code.visualstudio.com/docs/editor/debugging), [Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools) 또는 [Firefox를 사용하여 전체 소스 맵 지원으로 Next.js 프런트엔드 및 백엔드 코드를 디버깅할 수 있는 방법을 설명합니다. DevTools](https://firefox-source-docs.mozilla.org/devtools-user/).

Node.js에 연결할 수 있는 모든 디버거를 사용하여 Next.js 애플리케이션을 디버깅할 수도 있습니다. Node.js [디버깅 가이드](https://nodejs.org/learn/getting-started/debugging/)에서 자세한 내용을 확인할 수 있습니다.

## Debugging with VS Code
다음 내용을 사용하여 프로젝트 루트에 `.vscode/launch.json`이라는 파일을 만듭니다:

launch.json
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "npm run dev -- --inspect"
    },
    {
      "name": "Next.js: debug client-side",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000"
    },
    {
      "name": "Next.js: debug client-side (Firefox)",
      "type": "firefox",
      "request": "launch",
      "url": "http://localhost:3000",
      "reAttach": true,
      "pathMappings": [
        {
          "url": "webpack://_N_E",
          "path": "${workspaceFolder}"
        }
      ]
    },
    {
      "name": "Next.js: debug full stack",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/node_modules/next/dist/bin/next",
      "runtimeArgs": ["--inspect"],
      "skipFiles": ["<node_internals>/**"],
      "serverReadyAction": {
        "action": "debugWithEdge",
        "killOnServerStop": true,
        "pattern": "- Local:.+(https?://.+)",
        "uriFormat": "%s",
        "webRoot": "${workspaceFolder}"
      }
    }
  ]
}
```

> **Note**: VS Code에서 Firefox 디버깅을 사용하려면 [Firefox 디버거 확장](https://marketplace.visualstudio.com/items?itemName=firefox-devtools.vscode-firefox-debug)을 설치해야 합니다.

Yarn을 사용하는 경우 `npm run dev`를 `yarn dev`로 바꾸거나 pnpm을 사용하는 경우 `pnpm dev`로 바꿀 수 있습니다.

"Next.js: 전체 스택 디버그" 구성에서 `serverReadyAction.action`은 서버가 준비되면 열 브라우저를 지정합니다. `debugWithEdge`는 Edge 브라우저를 시작하는 것을 의미합니다. Chrome을 사용하는 경우 이 값을 `debugWithChrome`으로 변경하세요.

[포트 번호를 변경](/docs/pages/api-reference/cli/next#next-dev-options)하는 경우 애플리케이션이 시작되면 `http://localhost:3000`의 `3000`을 대신 사용 중인 포트로 바꾸세요.

루트가 아닌 디렉터리에서 Next.js를 실행하는 경우(예: Turborepo를 사용하는 경우) 서버 측 및 전체 스택 디버깅 작업에 'cwd'를 추가해야 합니다. 예를 들어 `"cwd": "${workspaceFolder}/apps/web"`입니다.

이제 디버그 패널(Windows/Linux에서는 `Ctrl+Shift+D`, macOS에서는 `⇧+⌘+D`)로 이동하여 실행 구성을 선택한 다음 `F5`를 누르거나 명령 팔레트에서 **디버그: 디버깅 시작**을 선택하여 디버깅 세션을 시작합니다.

## Using the Debugger in Jetbrains WebStorm
런타임 구성이 나열된 드롭다운 메뉴를 클릭하고 `Edit Configurations...`을 클릭합니다. `http://localhost:3000`을 URL로 사용하여 `JavaScript Debug` 디버그 구성을 만듭니다. 원하는 대로 사용자 정의하고(예: 디버깅용 브라우저, 프로젝트 파일로 저장) `OK`를 클릭하세요. 이 디버그 구성을 실행하면 선택한 브라우저가 자동으로 열립니다. 이 시점에서 디버그 모드에는 NextJS 노드 애플리케이션과 클라이언트/브라우저 애플리케이션이라는 2개의 애플리케이션이 있어야 합니다.

## Debugging with Browser DevTools
### Client-side code
`next dev`, `npm run dev` 또는 `yarn dev`를 실행하여 평소처럼 개발 서버를 시작하세요. 서버가 시작되면 원하는 브라우저에서 `http://localhost:3000`(또는 대체 URL)을 엽니다.

크롬의 경우:
* Chrome의 개발자 도구(Windows/Linux에서는 `Ctrl+Shift+J`, macOS에서는 `⌥+⌘+I`)를 엽니다.
* **소스** 탭으로 이동하세요.

파이어폭스의 경우:
* Firefox의 개발자 도구를 엽니다(Windows/Linux에서는 `Ctrl+Shift+I`, macOS에서는 `⌥+⌘+I`).
* **디버거** 탭으로 이동하세요.

어느 브라우저에서든 클라이언트 측 코드가 [`debugger`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Statements/debugger) 문에 도달할 때마다 코드 실행이 일시 중지되고 해당 파일이 디버그 영역에 나타납니다. 중단점을 수동으로 설정하기 위해 파일을 검색할 수도 있습니다:

* 크롬에서: Windows/Linux에서는 `Ctrl+P`를 누르고 macOS에서는 `⌘+P`를 누르세요.
* 파이어폭스에서: Windows/Linux에서는 `Ctrl+P`를, macOS에서는 `⌘+P`를 누르거나 왼쪽 패널의 파일 트리를 사용하세요.

검색할 때 소스 파일의 경로는 `webpack://_N_E/./`로 시작됩니다.

### React Developer Tools

React 관련 디버깅을 위해서는 [React 개발자 도구](https://react.dev/learn/react-developer-tools) 브라우저 확장 프로그램을 설치하세요. 이 필수 도구는 다음과 같은 도움을 줍니다:
* React 컴포넌트 검사
* props 및 state 편집
* 성능 문제 식별

### Server-side code
브라우저 DevTools를 사용하여 서버측 Next.js 코드를 디버그하려면 `--inspect` 플래그를 전달해야 합니다:

```bash package="npm"
npm run dev -- --inspect
```

`--inspect` 값은 기본 Node.js 프로세스에 전달됩니다. [고급 사용 사례에 대한 '--inspect' 문서](https://nodejs.org/api/cli.html#--inspecthostport)를 확인하세요.

> **알아두면 좋은 점**: Docker 컨테이너에서 앱을 실행할 때와 같이 localhost 외부에서 원격 디버깅 액세스를 허용하려면 `--inspect=0.0.0.0`을 사용하세요.

`--inspect` 플래그를 사용하여 Next.js 서버를 시작하면 다음과 같습니다:

```bash filename="Terminal"
Debugger listening on ws://127.0.0.1:9229/0cf90313-350d-4466-a748-cd60f4e47c95
For help, see: https://nodejs.org/learn/getting-started/debugging
ready - started server on 0.0.0.0:3000, url: http://localhost:3000
```

크롬의 경우:
1. 새 탭을 열고 `chrome://inspect`를 방문하세요.
2. **원격 대상** 섹션에서 Next.js 애플리케이션을 찾으세요.
3. **inspect**를 클릭하여 별도의 DevTools 창을 엽니다.
4. **소스** 탭으로 이동합니다.

파이어폭스의 경우:
1. 새 탭을 열고 `about:debugging`을 방문하세요.
2. 왼쪽 사이드바에서 **This Firefox**를 클릭합니다.
3. **원격 대상**에서 Next.js 애플리케이션을 찾으세요.
4. **검사**를 클릭하여 디버거를 엽니다.
5. **디버거** 탭으로 이동합니다.

서버측 코드 디버깅은 클라이언트측 디버깅과 유사하게 작동합니다. 파일을 검색할 때(`Ctrl+P`/`⌘+P`) 소스 파일에는 `webpack://{application-name}/./`으로 시작하는 경로가 있습니다(여기서 `{application-name}`은 `package.json` 파일에 따라 애플리케이션 이름으로 대체됩니다).

`--inspect-brk` 또는 `--inspect-wait`를 사용하려면 대신 `NODE_OPTIONS`를 지정해야 합니다. 예를 들어 `NODE_OPTIONS=--inspect-brk next dev`.

### Inspect Server Errors with Browser DevTools
오류가 발생했을 때 소스 코드를 검사하면 오류의 근본 원인을 추적하는 데 도움이 될 수 있습니다.

Next.js는 오류 오버레이의 Next.js 버전 표시기 아래에 Node.js 아이콘을 표시합니다. 해당 아이콘을 클릭하면 DevTools URL이 클립보드에 복사됩니다. 해당 URL로 새 브라우저 탭을 열어 Next.js 서버 프로세스를 검사할 수 있습니다.

### Debugging on Windows
컴퓨터에서 Windows Defender가 비활성화되어 있는지 확인하세요. 이 외부 서비스는 *읽은 모든 파일*을 검사하며, 이는 `next dev`를 사용하면 빠른 새로 고침 시간이 크게 증가하는 것으로 보고되었습니다. 이는 Next.js와 관련이 없는 알려진 문제이지만 Next.js 개발에 영향을 미칩니다.

## More information
JavaScript 디버거 사용 방법에 대해 자세히 알아보려면 다음 문서를 살펴보세요:

* [Node.js debugging in VS Code: Breakpoints](https://code.visualstudio.com/docs/nodejs/nodejs-debugging#_breakpoints)
* [Chrome DevTools: Debug JavaScript](https://developers.google.com/web/tools/chrome-devtools/javascript)
* [Firefox DevTools: Debugger](https://firefox-source-docs.mozilla.org/devtools-user/debugger/)