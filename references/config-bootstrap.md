# Application 설정 & 부트스트랩

## 진입점 구조

```typescript
// src/index.ts
import { Application } from '@asapjs/core';
import config from './config';

new Application(__dirname, config).run();
```

### 핵심 포인트

- `@asapjs/sequelize` 사용 시 `reflect-metadata`는 자동 import됨 (별도 import 불필요)
- sequelize 없이 데코레이터를 사용하는 경우에만 `import 'reflect-metadata'`를 첫 줄에 추가
- `__dirname`은 자동 탐색의 루트 경로 (route.ts, *Table.ts, *Dto.ts 스캔 기준)
- `app.run(callback)`의 callback은 서버 시작 후 실행
- 설정은 별도 `config.ts` 파일로 분리 권장

## AsapJSConfig 전체 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `name` | `string` | 아니오 | 애플리케이션 이름 |
| `dirname` | `string` | 아니오 | 자동 탐색 루트 경로 (`__dirname` 전달) |
| `debug` | `boolean` | 아니오 | 디버그 모드 |
| `port` | `number` | 아니오 | 서버 포트 (기본: 3000) |
| `basePath` | `string` | 아니오 | API 기본 경로 (예: `'api'` → `/api/...`, `''` → `/...`) |
| `extensions` | `string[]` | 예 | 로드할 확장 패키지 목록 |
| `auth` | `object` | 아니오 | JWT 관련 설정 |
| `db` | `object` | 아니오 | Sequelize 연결 설정 |
| `router` | `RouterConfig` | 아니오 | `defineRouterConfig()` 결과 |
| `swagger` | `object` | 아니오 | Swagger UI 설정 |
| `socket` | `SocketOption` | 아니오 | Socket.IO 설정 |
| `sentry` | `object` | 아니오 | Sentry DSN + environment |
| `environment` | `object` | 아니오 | 환경 구분 (`is_production`, `web_url` 등) |
| `log` | `object` | 아니오 | 로깅 설정 (`format`, `level`) |

## extensions 배열

사용할 패키지를 명시적으로 나열:

```typescript
extensions: ['@asapjs/router', '@asapjs/sequelize']           // 기본
extensions: ['@asapjs/router', '@asapjs/sequelize', '@asapjs/socket']  // + 소켓
```

Fastify 사용 시:
```typescript
import { FastifyApplication } from '@asapjs/fastify';
// FastifyApplication은 별도 부트스트랩 클래스
```

## 초기화 흐름

```
Application.run()
  ├─ RouterModule(dirname)
  │   ├─ initMiddlewares()         CORS, bodyParser, 커스텀 미들웨어
  │   ├─ require(route.ts)         컨트롤러 로딩
  │   ├─ registerRoutes()          @Get/@Post/@Put/@Delete 처리
  │   └─ Swagger UI 마운트          /docs/swagger-ui.html
  │
  ├─ initSequelizeModule(dirname)  [extensions에 포함 시]
  │   ├─ dbInit()                  DB 연결 (없으면 자동 생성 - MySQL만)
  │   ├─ loadPath()                *Table.ts, *Dto.ts 자동 탐색
  │   ├─ addModelsToSequelize()    모델 등록
  │   └─ addDtosToSwagger()        DTO → Swagger 스키마
  │
  ├─ initSocketModule(server)      [extensions에 포함 시]
  │   ├─ loadPath()                *Socket.ts 자동 탐색
  │   └─ socketInit()              Socket.IO 서버 시작
  │
  ├─ initBeforeStartServer()       사용자 콜백
  │
  └─ server.listen(port)           HTTP 서버 시작
```

## route.ts — 컨트롤러 레지스트리

```typescript
// src/route.ts
import UserController from './user/controller/UserController';
import PostController from './post/controller/PostController';

export default [new UserController(), new PostController()];
```

- `index.ts`와 같은 디렉토리에 위치
- 컨트롤러 인스턴스의 기본 배열을 export
- 새 도메인 추가 시 여기에 컨트롤러 추가

## config.ts — 라우터 설정 분리 패턴

```typescript
// src/config.ts
import { logger } from '@asapjs/core';
import type { InferMiddlewareRouteOptions } from '@asapjs/router';
import { defineRouterConfig } from '@asapjs/router';
import { type JwtUserPayload, jwtMiddleware } from './middleware/jwtMiddleware';

require('dotenv').config({ path: `${__dirname}/../.env` });

const debug = process.env.NODE_ENV === 'development';
const port = parseInt(process.env.PORT || '3000', 10);

export const routerConfig = defineRouterConfig({
  middleware: [jwtMiddleware],
});

export default {
  dirname: __dirname,
  debug,
  name: 'my-api',
  basePath: '',
  extensions: ['@asapjs/router', '@asapjs/sequelize'],
  port,
  router: routerConfig,
  auth: {
    jwt_access_token_secret: process.env.JWT_SECRET || 'secret',
    jwt_access_token_life: process.env.JWT_EXPIRES_IN || '7d',
    jwt_refresh_token_secret: process.env.JWT_SECRET || 'secret',
    jwt_refresh_token_life: '30d',
  },
  db: {
    database: process.env.DB_NAME || 'myapp',
    username: process.env.DB_USERNAME || 'root',
    password: process.env.DB_PASSWORD || '',
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '3306', 10),
    dialect: 'mysql',
    logging: debug ? (query: string) => logger.debug(query) : false,
    timezone: '+09:00',
    pool: {
      max: 10,
      idle: 10000,
      acquire: 30000,
    },
  },
  swagger: {
    name: 'my-api',
    version: '1.0.0',
    scheme: (debug ? 'http' : 'https') as 'http' | 'https',
    host: debug ? `localhost:${port}` : 'api.example.com',
  },
  environment: {
    is_production: !debug,
    web_url: process.env.WEB_URL || 'http://localhost:3000',
  },
  log: process.env.ASAPJS_LOG_FORMAT || process.env.ASAPJS_LOG_LEVEL
    ? {
        format: process.env.ASAPJS_LOG_FORMAT as 'pretty' | 'json' | undefined,
        level: process.env.ASAPJS_LOG_LEVEL,
      }
    : undefined,
};

// 전역 타입 확장 (Module Augmentation)
declare module '@asapjs/router' {
  interface GlobalMiddlewareContext {
    user: JwtUserPayload;
  }
  interface GlobalRouteOptions
    extends InferMiddlewareRouteOptions<typeof routerConfig.middleware> {}
}
```

### 핵심 포인트

- `logger`는 `@asapjs/core` 또는 `@asapjs/common` 모두에서 import 가능
- `require('dotenv').config()` 패턴 사용 (ESM `import dotenv` 대신)
- `dirname` 속성으로 자동 탐색 루트 경로 지정
- `environment` 객체로 환경별 설정 전달
- `log` 설정은 환경변수가 있을 때만 활성화

## 데이터베이스 동기화

```typescript
import { modelsSync, getSequelize } from '@asapjs/sequelize';

app.run(async () => {
  if (process.env.DB_SYNC === 'true') {
    await modelsSync();
    // 내부: sync({ alter: { drop: false } })
    // 컬럼 추가/변경만. 삭제 안 함.
  }
});
```

## 헬스 체크

```typescript
import { healthCheck } from '@asapjs/sequelize';

@Get('/health', { title: 'Health check', auth: false })
async check({}: ExecuteArgs) {
  await healthCheck();  // SELECT 1 (1초 타임아웃)
  return { status: 'ok' };
}
```

## Swagger 설정

Swagger UI는 자동으로 `/docs/swagger-ui.html`에 마운트됩니다.

```typescript
swagger: {
  name: 'My API',
  version: '1.0.0',
  host: 'localhost:3000',
  scheme: 'http',  // 'http' | 'https'
  auth_url: '/auth/login',  // 선택: Swagger UI 인증 URL
  auth: {  // 선택: Swagger UI 접근 제한
    username: 'admin',
    password: 'password',
  },
}
```

## environment 설정

애플리케이션 전역에서 접근 가능한 환경 설정:

```typescript
environment: {
  is_production: process.env.NODE_ENV === 'production',
  web_url: process.env.WEB_URL || 'http://localhost:3000',
  // 커스텀 필드 자유롭게 추가 가능
}
```

## log 설정

Winston 기반 로거 설정:

```typescript
log: {
  format: 'pretty',  // 'pretty' | 'json'
  level: 'debug',     // 'debug' | 'info' | 'warn' | 'error'
}
```

환경변수로도 제어 가능: `ASAPJS_LOG_FORMAT`, `ASAPJS_LOG_LEVEL`

## 환경 변수

```bash
PORT=3000
DB_USER=root
DB_PASSWORD=password
DB_NAME=myapp
DB_HOST=localhost
DB_PORT=3306
DB_SYNC=true          # 개발 환경에서만
JWT_SECRET=your-secret
```

## 로깅

```typescript
import { logger } from '@asapjs/core';
// 또는
import { logger } from '@asapjs/common';

logger.info('서버 시작');
logger.error('에러 발생', { err: error });
logger.debug('디버그 메시지');
```

Winston 기반. `ASAPJS_LOG_LEVEL` (debug/info/warn/error), `ASAPJS_LOG_FORMAT` (pretty/json) 환경변수로 제어.

> ⚠️ `logger`는 `@asapjs/core`와 `@asapjs/common` 모두에서 export됩니다. 어느 쪽에서 import해도 동일하게 동작합니다.

## Fastify 부트스트랩

Fastify 어댑터를 사용하는 경우 `FastifyApplication`을 사용합니다:

```typescript
import { FastifyApplication } from '@asapjs/fastify';
import config from './config';

new FastifyApplication(__dirname, config).run();
```

상세 내용은 `fastify-patterns.md` 참조.

## Application 라이프사이클 메서드

| 메서드 | 반환 타입 | 설명 |
|--------|----------|------|
| `run(initBeforeStartServer?, options?)` | `Promise<express.Application>` | 서버 시작. 콜백으로 시작 전 추가 설정 가능 |
| `getApp()` | `express.Application` | Express 인스턴스 반환 |
| `getServer()` | `http.Server` | 내부 Node.js HTTP 서버 반환 |
| `destroy()` | `Promise<void>` | 플러그인 역순 정리 후 서버 종료 |

### run() 옵션

```typescript
// 기본 사용
new Application(__dirname, config).run();

// 서버 시작 전 콜백
new Application(__dirname, config).run(async () => {
  await modelsSync();
});

// 서버 리스닝 비활성화 (테스트/서버리스 환경)
const app = new Application(__dirname, config);
await app.run(undefined, { disableListenServer: true });
const expressApp = app.getApp();  // Express 인스턴스 직접 사용
```

### destroy() — Graceful Shutdown

```typescript
const app = new Application(__dirname, config);
await app.run();

// 종료 시
process.on('SIGTERM', async () => {
  await app.destroy();  // 플러그인 역순 정리 → 서버 종료
  process.exit(0);
});
```
