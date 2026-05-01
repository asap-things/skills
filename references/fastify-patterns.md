# Fastify 어댑터

`@asapjs/fastify`는 Express 기반 `@asapjs/router`의 대안으로, Fastify 5 위에서 동작하는 ASAPJS 어댑터입니다.

## 의존성

```json
{
  "dependencies": {
    "@asapjs/common": "^1.0.0-alpha.33",
    "@asapjs/error": "^1.0.0-alpha.33",
    "@asapjs/types": "^1.0.0-alpha.33",
    "@fastify/cors": "^11.0.0",
    "fastify": "^5.2.0"
  },
  "peerDependencies": {
    "@asapjs/schema": "*",
    "@asapjs/sequelize": "*"
  }
}
```

`@asapjs/schema`와 `@asapjs/sequelize`는 선택적 피어 의존성입니다.

## FastifyApplication 부트스트랩

`FastifyApplication`은 Express의 `Application` 클래스에 대응합니다. Fastify 인스턴스 생성, 플러그인 초기화, 서버 시작을 담당합니다.

```typescript
import FastifyApplication from '@asapjs/fastify';
import type { AsapJSConfig } from '@asapjs/types';

const config: AsapJSConfig = {
  name: 'my-service',
  port: 3000,
  debug: false,
  basePath: 'api',
  extensions: ['@asapjs/fastify', '@asapjs/sequelize'],
  router: {
    middleware: [],
  },
};

const app = new FastifyApplication(__dirname, config);

app.run(async (fastifyInstance) => {
  // 서버 시작 전 추가 설정 (선택)
  // fastifyInstance.register(...)
});
```

### 부트스트랩 순서

1. `constructor` — `setConfig()`, `configureLogger()` 호출
2. `run()` 호출 시 `initModules()` 실행:
   - `Fastify({ logger, trustProxy: true })` 인스턴스 생성
   - `config.extensions`에 `@asapjs/fastify` 포함 시 → `FastifyRouterPlugin.init()` 실행
   - `config.extensions`에 `@asapjs/sequelize` 포함 시 → `SequelizePlugin.init()` 실행
3. `initBeforeStartServer` 콜백 실행 (선택)
4. `app.listen({ port, host: '0.0.0.0' })`

### 주요 메서드

| 메서드 | 반환 타입 | 설명 |
|--------|----------|------|
| `run(initBeforeStartServer?)` | `Promise<FastifyInstance>` | 서버 시작. 콜백으로 시작 전 추가 설정 가능 |
| `getApp()` | `FastifyInstance` | Fastify 인스턴스 반환 |
| `getServer()` | `http.Server` | 내부 Node.js HTTP 서버 반환 |
| `destroy()` | `Promise<void>` | 플러그인 역순 정리 후 `app.close()` |

## FastifyRouterPlugin

`FastifyRouterPlugin`은 `AsapJSPlugin` 인터페이스를 구현하며, 라우터 초기화를 담당합니다.

### init() 처리 순서

1. CORS 등록 (`@fastify/cors`)
2. `config.router.middleware` 설정 시 → `setRegisteredMiddlewares()` 호출
3. `{dirname}/route.ts`에서 컨트롤러 배열 로드
4. 각 컨트롤러를 `fastify.register()`로 등록 (prefix 적용)
5. `setErrorHandler(fastifyErrorHandler)` 설정
6. health-check 엔드포인트 자동 등록 (`/{basePath}/health-check`)

```typescript
// route.ts 예시
import UserController from './user/controller/UserController';
import PostController from './post/controller/PostController';

export default [new UserController(), new PostController()];
```

> Express와 달리 `registerRoutes()`를 컨트롤러 생성자에서 호출하지 않습니다. `FastifyRouterPlugin`이 `registerFastifyRoutes()`를 직접 호출합니다.

## FastifyRouterController

`FastifyRouterController`는 Express의 `RouterController`에 대응하는 기본 클래스입니다.

```typescript
import { FastifyRouterController, Get, Post, Put, Delete } from '@asapjs/fastify';
import type { FastifyExecuteArgs } from '@asapjs/fastify';

export default class UserController extends FastifyRouterController {
  public basePath = '/users';
  public tag = 'users';
  private userService: UserApplication;

  constructor() {
    super();
    // registerRoutes() 호출 불필요!
    this.userService = new UserApplication();
  }

  @Get('/', {
    title: '사용자 목록 조회',
    query: GetUserListQueryDto,
    response: UserDto,
  })
  public getUserList = async ({ paging, query }: FastifyExecuteArgs<{}, GetUserListQueryDto>) => {
    const result = await this.userService.list(paging);
    return { result };
  };

  @Post('/', {
    title: '사용자 생성',
    body: CreateUserDto,
    response: UserDto,
  })
  public createUser = async ({ body }: FastifyExecuteArgs<{}, {}, CreateUserDto>) => {
    const result = await this.userService.create(body);
    return { result };
  };
}
```

### registerFastifyRoutes() 내부 동작

`registerFastifyRoutes(fastify)` 메서드는 플러그인에 의해 호출되며:

1. 데코레이터가 수집한 `routes` 배열을 순회
2. 각 라우트에 대해:
   - 글로벌 미들웨어(`getRegisteredMiddlewares()`)를 `preHandler` 훅으로 변환
   - 라우트별 미들웨어(`route.middleware`)를 `preHandler` 훅으로 변환
   - `fastify[method](path, { preHandler, handler: FastifyWrapper(handler) })` 등록

## FastifyExecuteArgs

`FastifyExecuteArgs`는 Express의 `ExecuteArgs`에 대응하는 타입입니다.

```typescript
type FastifyExecuteArgs<P = {}, Q = {}, B = {}, Context = {}> = {
  req: FastifyRequest;
  res: FastifyReply;
  path: P & Record<string, any>;
  query: Q & Record<string, any>;
  body: B & Record<string, any>;
  paging: { page: number; limit: number };
} & Context;
```

### Proxy 기반 속성 접근

`FastifyWrapper`는 `Proxy`를 사용하여 `args` 객체를 생성합니다. `args`에 없는 속성에 접근하면 `FastifyRequest` 객체에서 자동으로 조회합니다.

```typescript
// args.user 접근 시:
// 1. base 객체에 user가 있으면 → base.user 반환
// 2. 없으면 → request.user 반환 (Proxy fallback)
```

이 동작은 Express `Wrapper`와 동일합니다. 미들웨어에서 `request`에 추가한 속성(예: `request.user`)을 핸들러에서 `args.user`로 접근할 수 있습니다.

### Express ExecuteArgs와의 차이

| 항목 | Express `ExecuteArgs` | Fastify `FastifyExecuteArgs` |
|------|----------------------|------------------------------|
| `req` 타입 | `express.Request` | `FastifyRequest` |
| `res` 타입 | `express.Response` | `FastifyReply` |
| `path` | `req.params` (선택적) | `request.params` (필수) |
| `files` | `req.files` 포함 | 미포함 |
| `paging` 타입 | `PaginationQueryType` (`@asapjs/sequelize`) | `{ page: number; limit: number }` (인라인) |
| Context 기본값 | `GlobalMiddlewareContext` | `{}` |

## 데코레이터

`@asapjs/fastify`는 Express 어댑터와 동일한 이름의 데코레이터를 제공합니다. 단, import 경로가 다릅니다.

```typescript
import { Get, Post, Put, Delete } from '@asapjs/fastify';
```

### IOptions 인터페이스

```typescript
interface IOptions {
  title?: string;
  description?: string;
  deprecated?: boolean;
  body?: unknown;
  bodyContentType?: 'application/json' | 'multipart/form-data';
  query?: unknown;
  response?: unknown;
  errors?: ErrorCreator[];
  middleware?: unknown[];
}
```

데코레이터는 메서드 메타데이터를 `routes` 배열에 수집합니다. `registerFastifyRoutes()` 호출 시 이 배열을 순회하며 Fastify 라우트로 등록합니다.

```typescript
@Get('/search', {
  title: '검색',
  query: SearchQueryDto,
  response: SearchResultDto,
  middleware: [authMiddleware],
  errors: [SearchErrors.INVALID_QUERY],
})
public search = async ({ query }: FastifyExecuteArgs<{}, SearchQueryDto>) => {
  return await this.searchService.search(query);
};
```

> Express 데코레이터의 `options` 파라미터는 필수이지만, Fastify 데코레이터는 `options`가 선택적입니다 (기본값 `{}`).

## 에러 처리

### fastifyErrorHandler

`fastifyErrorHandler`는 `setErrorHandler()`로 등록되는 전역 에러 핸들러입니다.

```typescript
import { fastifyErrorHandler } from '@asapjs/fastify';
```

처리 흐름:

1. 에러 발생 시 스택 트레이스에서 파일명, 라인, 함수명 추출
2. 요청 정보(requestId, method, url, ip)와 함께 로깅
3. `resolveErrorBody(error)`로 에러 본문 생성
4. `reply.code(status).send(errorBody)` 응답

```typescript
// resolveErrorBody는 @asapjs/error에서 제공
// HttpError, 일반 Error, 알 수 없는 에러를 통일된 형식으로 변환
```

### FastifyWrapper 내부 에러 처리

`FastifyWrapper`는 핸들러 실행 중 발생하는 에러를 자체적으로 처리합니다:

| 에러 타입 | 처리 방식 |
|----------|----------|
| `HttpError` 인스턴스 | `reply.code(err.status).send(err.toJSON())` |
| `{ status, message }` 객체 | `HttpError`로 래핑 후 응답 |
| 기타 에러 | 500 `INTERNAL_SERVER_ERROR`로 응답 |
| `reply.sent === true` | 무시 (이미 응답 완료) |

> Express `Wrapper`와 달리 Sentry 연동이 없습니다. Fastify 어댑터에서는 Sentry 캡처가 포함되어 있지 않습니다.

### 에러 핸들러 이중 구조

Fastify 어댑터는 두 단계의 에러 처리를 가집니다:

1. `FastifyWrapper` — 핸들러 내부 try/catch (라우트 레벨)
2. `fastifyErrorHandler` — `setErrorHandler()` (앱 레벨, 미들웨어/preHandler 에러 포함)

## 미들웨어 변환

Express 스타일의 `TypedMiddleware`는 Fastify `preHandler` 훅으로 자동 변환됩니다.

### 변환 메커니즘

```typescript
// Express 미들웨어 시그니처
(req: Request, res: Response, next: NextFunction) => void

// Fastify preHandler 훅으로 변환
async (request: FastifyRequest, reply: FastifyReply) => Promise<void>
```

`adaptMiddlewareToHook()` 함수가 변환을 수행합니다:

```typescript
function adaptMiddlewareToHook(expressMw: (req: any, res: any, next: any) => void) {
  return async (request: any, reply: any) => {
    return new Promise<void>((resolve, reject) => {
      expressMw(request, reply, (err?: unknown) => {
        if (err) reject(err);
        else resolve();
      });
    });
  };
}
```

### 미들웨어 적용 순서

1. 글로벌 미들웨어 (`config.router.middleware`로 등록, `setRegisteredMiddlewares()`)
   - 각 미들웨어 팩토리에 라우트 옵션(`routeOptions`)을 전달하여 실행
2. 라우트별 미들웨어 (`IOptions.middleware`로 지정)
3. 핸들러 (`FastifyWrapper(handler)`)

```typescript
// 글로벌 미들웨어 등록 예시
import type { AsapJSConfig } from '@asapjs/types';
import { authMiddlewareFactory } from './middleware/auth';

const config: AsapJSConfig = {
  // ...
  router: {
    middleware: [authMiddlewareFactory],
  },
};
```

> Express에서는 `app.use(middleware)`로 글로벌 미들웨어를 등록하지만, Fastify에서는 `setRegisteredMiddlewares()`로 저장 후 각 라우트의 `preHandler`에 개별 적용됩니다.

## Express와의 차이점

| 항목 | Express (`@asapjs/router`) | Fastify (`@asapjs/fastify`) |
|------|---------------------------|----------------------------|
| 패키지 | `@asapjs/router` | `@asapjs/fastify` |
| 기본 클래스 | `RouterController` | `FastifyRouterController` |
| 라우트 등록 | 생성자에서 `this.registerRoutes()` 필수 | 플러그인이 `registerFastifyRoutes()` 자동 호출 |
| 라우터 인스턴스 | `this.expressRouter` (Express Router) | Fastify 인스턴스 직접 사용 |
| ExecuteArgs | `ExecuteArgs` | `FastifyExecuteArgs` |
| 미들웨어 | Express 미들웨어 직접 사용 | `adaptMiddlewareToHook()`으로 preHandler 변환 |
| 에러 핸들러 | `app.use(errorHandler)` (Express 미들웨어) | `app.setErrorHandler(fastifyErrorHandler)` |
| bodyParser | `bodyParser.json()` / `bodyParser.urlencoded()` 수동 설정 | Fastify 내장 (설정 불필요) |
| Sentry 연동 | `Wrapper`에서 500 에러 시 `Sentry.captureException()` | 미지원 |
| Swagger 자동 생성 | `addPaths()`, `addScheme()` 호출로 자동 생성 | 미지원 |
| Effect 트레이싱 | `wrapWithEffect()` 적용 | 미적용 |
| 데코레이터 options | 필수 파라미터 | 선택적 파라미터 (기본값 `{}`) |
| CORS | `cors()` Express 미들웨어 | `@fastify/cors` 플러그인 |
| trust proxy | `app.set('trust proxy', true)` | `Fastify({ trustProxy: true })` |

## Swagger 지원

현재 `@asapjs/fastify`는 Swagger 자동 생성을 지원하지 않습니다.

Express 어댑터(`@asapjs/router`)에서는 `RouterController.excute()` 내부에서 데코레이터 메타데이터(body, query, response, errors)를 분석하여 `addPaths()`, `addScheme()`을 호출하고 Swagger UI를 자동 제공합니다.

Fastify 어댑터의 `FastifyRouterController.registerFastifyRoutes()`에는 이 로직이 없습니다. 데코레이터의 `body`, `query`, `response`, `errors` 옵션은 메타데이터로 저장되지만, Swagger 문서 생성에는 사용되지 않습니다.

Swagger가 필요한 프로젝트에서는 Express 어댑터를 사용하거나, 별도의 Fastify Swagger 플러그인(`@fastify/swagger`)을 수동으로 설정해야 합니다.

## 요청 라이프사이클

```
Client → Fastify Instance
  [1] @fastify/cors 플러그인
  [2] preHandler 훅: 글로벌 TypedMiddleware (adaptMiddlewareToHook 변환)
  [3] preHandler 훅: 라우트별 middleware (adaptMiddlewareToHook 변환)
  [4] FastifyWrapper → FastifyExecuteArgs 추출 (Proxy) → handler 호출 → reply.send(output)
  [5] 핸들러 에러 시: FastifyWrapper 내부 catch → reply.code(status).send(errorBody)
  [6] preHandler 에러 시: fastifyErrorHandler → resolveErrorBody → reply.code(status).send(errorBody)
```
