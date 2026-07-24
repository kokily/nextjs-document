# How to set up a custom server in Next.js
<span style="color: #808080;">Last updated June 23, 2026</span>

Next.js에는 기본적으로 `next start` 기능이 있는 자체 서버가 포함되어 있습니다. 기존 백엔드가 있는 경우 계속해서 Next.js와 함께 사용할 수 있습니다(이것은 사용자 정의 서버가 아닙니다). 사용자 정의 Next.js 서버를 사용하면 사용자 정의 패턴에 대한 서버를 프로그래밍 방식으로 시작할 수 있습니다. 대부분의 경우 이 접근 방식이 필요하지 않습니다. 그러나 `eject` 하는 경우에는 사용할 수 있습니다.

> **알아두면 좋은 점**:
>
> * 사용자 정의 서버를 사용하기로 결정하기 전에 Next.js의 통합 라우터가 앱 요구 사항을 충족할 수 없는 경우에만 사용해야 한다는 점을 명심하세요.
> * 독립형 출력 모드를 사용하는 경우 사용자 정의 서버 파일을 추적하지 않습니다. 대신 이 모드는 별도의 최소 `server.js` 파일을 출력합니다. 이들은 함께 사용할 수 없습니다.

커스텀 서버의 [다음 예시](https://github.com/vercel/next.js/tree/canary/examples/custom-server)를 살펴보세요:

server.ts
```ts
import { createServer } from 'http'
import next from 'next'

const port = parseInt(process.env.PORT || '3000', 10)
const dev = process.env.NODE_ENV !== 'production'
const app = next({ dev })
const handle = app.getRequestHandler()

app.prepare().then(() => {
  createServer((req, res) => {
    handle(req, res)
  }).listen(port)

  console.log(
    `> Server listening at http://localhost:${port} as ${
      dev ? 'development' : process.env.NODE_ENV
    }`
  )
})
```

> `server.js`는 Next.js 컴파일러 또는 번들링 프로세스를 통해 실행되지 않습니다. 이 파일에 필요한 구문과 소스 코드가 현재 사용 중인 Node.js 버전과 호환되는지 확인하세요. [예시 보기](https://github.com/vercel/next.js/tree/canary/examples/custom-server)

커스텀 서버를 실행하려면 `package.json`의 `scripts`를 다음과 같이 업데이트해야 합니다:

package.json
```json
{
  "scripts": {
    "dev": "node server.js",
    "build": "next build",
    "start": "NODE_ENV=production node server.js"
  }
}
```

또는 `nodemon`([예제](https://github.com/vercel/next.js/tree/canary/examples/custom-server))을 설정할 수 있습니다. 사용자 정의 서버는 다음 가져오기를 사용하여 서버를 Next.js 애플리케이션과 연결합니다:

```js
import next from 'next'

const app = next({})
```

위의 `next` 가져오기는 다음 옵션을 사용하여 객체를 받는 함수입니다.:

|Option|Type|Description|
|---|---|---|
|`conf`|`Object`|`next.config.js`에서 사용하는 것과 동일한 객체입니다. 기본값은 `{}`입니다.|
|`dev`|`Boolean`|(*선택적*)개발자 모드에서 Next.js를 시작할지 여부입니다. 기본값은 `false`입니다.|
|`dir`|`String`|(*선택적*)Next.js 프로젝트의 위치입니다. 기본값은 `'.'`입니다.|
|`quiet`|`Boolean`|(*선택적*)서버 정보가 포함된 오류 메시지를 숨깁니다. 기본값은 'false'입니다.|
|`hostname`|`String`|(*선택적*)서버가 실행되고 있는 호스트 이름|
|`port`|`Number`|(*선택적*)서버가 실행되는 포트|
|`httpServer`|`node:http#Server`|(*선택적*)Next.js가 실행되고 있는 HTTP 서버|
|`turbopack`|`Boolean`|(*선택적*)Turbopack 활성화(기본적으로 활성화됨)|
|`webpack`|`Boolean`|(*선택적*)웹팩 활성화|

그런 다음 반환된 `app`을 사용하여 Next.js가 필요에 따라 요청을 처리하도록 할 수 있습니다.