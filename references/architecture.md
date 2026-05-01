# 레이어드 아키텍처

ASAPJS는 모든 도메인에 동일한 4계층 구조를 강제합니다. 이 구조에서 벗어나면 프레임워크와 싸우게 됩니다.

## 계층 구조

```
HTTP Request
     ↓
[ Controller ]   — HTTP 경계: 데코레이터, ExecuteArgs, 반환값
     ↓
[ Application ]  — 비즈니스 로직: 유효성 검사, 오케스트레이션
     ↓
[ Repository ]   — 데이터 접근: 쿼리 추상화, DTO 매핑 (선택)
     ↓
[   Entity    ]  — 데이터 경계: @Table 모델, TypeIs 컬럼
```

## 각 레이어 책임

### Controller (`*Controller.ts`)

- `RouterController` (Express) 또는 `FastifyRouterController` (Fastify) 상속
- `@Get`, `@Post`, `@Put`, `@Delete` 데코레이터로 라우트 선언
- `ExecuteArgs` 하나만 받아서 Application 호출 후 결과 반환
- 핸들러 본문은 1~2줄이 이상적

```typescript
import { RouterController, Get, Post, ExecuteArgs } from '@asapjs/router';

export default class UserController extends RouterController {
  public basePath = '/users';
  public tag = 'users';
  private userService: UserApplication;

  constructor() {
    super();
    this.registerRoutes();  // 필수! 누락 시 라우트 미등록
    this.userService = new UserApplication();
  }

  @Get('/', {
    title: '사용자 목록 조회',
    query: GetUserListQueryDto,
    response: UserDto,
  })
  public getUserList = async ({ paging, user }: ExecuteArgs<{}, GetUserListQueryDto, {}>) => {
    const result = await this.userService.list(paging, user);
    return { result };
  };
}
```

**금지사항:**
- DB 직접 접근 (`UsersTable.findAll()` 등)
- `req`/`res` 직접 조작 (Wrapper가 처리)
- 비즈니스 로직 (검증, 해싱 등)

### Application (`*Application.ts`)

- 순수 TypeScript 클래스 (프레임워크 기본 클래스 불필요)
- Express/Fastify import 금지
- 비즈니스 규칙, 검증, 트랜잭션 관리
- Repository 또는 Entity 정적 API 호출

```typescript
import type { PaginationQueryType } from '@asapjs/sequelize';
import UsersTable from '../domain/entity/UsersTable';
import UserDto from '../dto/UserDto';
import { UserErrors } from '../errors/UserErrors';

export class UserApplication {
  private users: typeof UsersTable;

  constructor() {
    this.users = UsersTable;
  }

  public info = async (userId: number) => {
    const raw = await this.usersRepository.info(userId);
    if (!raw) throw UserErrors.NOT_FOUND({ userId });
    return raw;
  };

  public create = async (body: CreateUserDto, user: UserDto) => {
    const data = new CreateUserDto().map(body);
    const raw = await this.users.create(data);
    return new UserDto().map(raw);
  };
}
```

**금지사항:**
- `Request`, `Response`, `next` import
- HTTP 상태 코드 직접 설정
- `res.json()` 호출

### Repository (`*Repository.ts`) — 선택적 레이어

- `@asapjs/sequelize`의 `Repository` 클래스 상속
- `this.repository.findAll` / `this.repository.findOne` 사용
- 페이지네이션, DTO 변환 내장

```typescript
import { Repository } from '@asapjs/sequelize';
import type { PaginationQueryType } from '@asapjs/sequelize';
import UsersTable from '../domain/entity/UsersTable';
import UserDto from '../dto/UserDto';

export default class UserTableRepository extends Repository {
  private users: typeof UsersTable;

  constructor() {
    super();
    this.users = UsersTable;
  }

  public list = async (paging: PaginationQueryType, user: UserDto) => {
    const users = await this.repository.findAll(this.users, {
      exportTo: UserDto,
      user,
      paging,
    });
    return new UserDto().pagingMap(users);
  };
}
```

### Entity (`*Table.ts`)

- `sequelize-typescript`의 `Model` 상속
- `@asapjs/sequelize`의 `@Table` 데코레이터
- `@asapjs/schema`의 `TypeIs.*` 데코레이터로 컬럼 정의
- Controller에서 직접 import 금지 — Application/Repository만 접근

```typescript
import { Model } from 'sequelize-typescript';
import { Table } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';

@Table({ tableName: 'users', timestamps: true })
export default class UsersTable extends Model {
  @TypeIs.INT({ primaryKey: true, autoIncrement: true, comment: '사용자 ID' })
  id: number;

  @TypeIs.STRING({ unique: true, comment: '이메일' })
  email: string;

  @TypeIs.PASSWORD({ comment: '비밀번호' })
  password: string;
}
```

## 요청 라이프사이클 (Express)

```
Client → Express App
  [1] CORS middleware
  [2] bodyParser (JSON / multipart)
  [3] TypedMiddleware 팩토리 (config.router.middleware)
  [4] 커스텀 middleware (IOptions.middleware)
  [5] wrapWithEffect (Effect 기반 트레이싱)
  [6] Wrapper → ExecuteArgs 추출 → handler 호출 → res.json(output)
  [7] 에러 시: errorToResponse(err, res)
```

### Fastify 파이프라인 차이점

- bodyParser 불필요 (Fastify 내장)
- 미들웨어는 `preHandler` 훅으로 자동 변환
- 에러 핸들러: `setErrorHandler()` 사용
- `FastifyWrapper`는 `Proxy`로 request 속성 접근

## 도메인 폴더 구조

```
src/{domain}/
  controller/{Domain}Controller.ts
  application/{Domain}Application.ts
  domain/entity/{Domain}sTable.ts
  dto/
    Create{Domain}Dto.ts
    {Domain}Dto.ts
  errors/{Domain}Errors.ts
  infra/{Domain}TableRepository.ts  (선택)
```

## 자동 탐색 규칙

| 파일 패턴 | 발견 시점 | 등록 대상 |
|-----------|----------|----------|
| `*Table.ts` | `initSequelizeModule()` | Sequelize 모델 |
| `*Dto.ts` | `initSequelizeModule()` | Swagger 스키마 |
| `*Socket.ts` | `initSocketModule()` | Socket.IO 핸들러 |
| `route.ts` | `RouterModule()` | 컨트롤러 라우트 |

---

## 고급 패턴

### DBML 스키마 생성

`@asapjs/sequelize`는 등록된 모든 테이블 모델로부터 [DBML](https://dbml.dbdiagram.io/) 형식의 ERD 스키마를 자동 생성합니다. 데이터베이스 문서화나 [dbdiagram.io](https://dbdiagram.io) 같은 시각화 도구에 활용할 수 있습니다.

#### `generateDBML()`

Sequelize 모듈 초기화 시 내부적으로 호출되며, 모든 `@Table` 모델의 컬럼·관계·Enum 정보를 수집하여 DBML 데이터를 생성합니다.

```typescript
import { generateDBML } from '@asapjs/sequelize';

// 일반적으로 직접 호출할 필요 없음 — initSequelizeModule()이 자동 호출
await generateDBML();
```

#### `getDBMLData()`

생성된 DBML 문자열을 반환하고, `./src/dbml/database.dbml` 파일로 저장합니다. DBML 데이터가 아직 생성되지 않았으면 내부적으로 `generateDBMLData()`를 먼저 실행합니다.

```typescript
import { getDBMLData } from '@asapjs/sequelize';

const dbmlString = await getDBMLData();
// → ./src/dbml/database.dbml 파일이 자동 생성됨
```

#### 생성되는 DBML 구조

- 프로젝트 정보: `config.projectName`, `config.db.dialect`에서 추출
- 테이블: 각 `@Table` 모델의 컬럼 (id PK 자동 포함)
- Enum: `TypeIs.ENUM` 컬럼에서 자동 추출 (`{tableName}_{columnName}_enum` 형식)
- 관계: `foreignkey` 타입 컬럼에서 `> 참조테이블.id` 관계 자동 생성

#### 사용 시점

- DB 스키마 문서화가 필요할 때
- ERD 시각화 도구에 입력할 DBML 파일이 필요할 때
- 테이블 간 관계를 한눈에 파악하고 싶을 때

---

### 콘솔 데이터 유틸리티

`@asapjs/sequelize`의 `getConsoleData()`는 등록된 모든 테이블과 DTO의 메타데이터를 구조화된 형태로 반환합니다. 관리 콘솔이나 디버깅 도구에서 모델 구조를 조회할 때 사용합니다.

```typescript
import { getConsoleData } from '@asapjs/sequelize';

const consoleData = await getConsoleData();
// consoleData = { tables: [...], dtos: [...] }
```

#### 반환 데이터 구조

```typescript
{
  tables: [
    {
      name: "users",           // 테이블명
      data: [
        {
          name: "email",       // 컬럼명
          type: "string",      // 타입 (__name)
          note: "이메일",       // Swagger description
          enum?: ["A", "B"],   // ENUM인 경우 값 목록
          data?: [...]         // DTO 타입인 경우 중첩 컬럼 정보
        }
      ]
    }
  ],
  dtos: [
    // 동일한 구조
  ]
}
```

#### 특징

- `belongsto` 타입 컬럼은 자동 제외됨
- `dto` 타입 컬럼은 재귀적으로 중첩 DTO의 컬럼 정보까지 포함
- Enum 컬럼은 `type: "enum"`으로 표시되며 가능한 값 목록 포함

---

### Swagger 커스터마이징

`@asapjs/router`는 Swagger(OpenAPI) 스펙을 프로그래밍 방식으로 확장할 수 있는 API를 제공합니다.

#### `addPaths(data)`

Swagger 스펙에 커스텀 경로를 추가합니다. 데코레이터로 자동 등록되지 않는 경로를 수동으로 추가할 때 사용합니다.

```typescript
import { addPaths } from '@asapjs/router';

await addPaths({
  path: '/custom/endpoint',
  method: 'get',
  tags: ['custom'],
  summary: '커스텀 엔드포인트',
  responses: {
    200: { description: '성공' },
  },
});
```

#### `addScheme(data)`

Swagger 스펙의 `components.schemas`에 커스텀 스키마를 추가합니다. `@asapjs/error`의 `error()` 팩토리가 내부적으로 이 함수를 사용하여 에러 스키마를 자동 등록합니다.

```typescript
import { addScheme } from '@asapjs/router';

await addScheme({
  name: 'CustomResponse',
  data: {
    type: 'object',
    properties: {
      id: { type: 'integer' },
      name: { type: 'string' },
    },
  },
});
```

#### `getSwaggerData(req)`

최종 Swagger 스펙 객체를 반환합니다. 최초 호출 시 내부적으로 `generateSwaggerData(req)`를 실행하여 등록된 모든 경로와 스키마를 조합합니다.

```typescript
import { getSwaggerData } from '@asapjs/router';

// Express 핸들러 내에서
const swaggerSpec = getSwaggerData(req);
```

스펙 생성 시 다음 설정값이 사용됩니다:

- `config.swagger.name` → `info.title`
- `config.swagger.version` → `info.version`
- `config.swagger.description` → `info.description`
- `config.swagger.auth_url` → OAuth2 토큰 URL
- `config.swagger.scheme`, `config.swagger.host` → 서버 URL

#### `generateSchemeRefWithName(name)`

스키마 이름으로 `$ref` 문자열을 생성합니다. Swagger 스펙에서 스키마를 참조할 때 사용합니다.

```typescript
import { generateSchemeRefWithName } from '@asapjs/router';

const ref = generateSchemeRefWithName('UserDto');
// → "#/components/schemas/UserDto"
```

---

### 스키마 유효성 검사

`@asapjs/schema`는 TypeIs 스키마를 기반으로 데이터 검증과 변환을 수행하는 유틸리티를 제공합니다.

#### `validateDataWithSchema(data, schema)`

스키마 기준으로 데이터의 유효성을 검사합니다. 필수 필드 누락이나 타입 위반 시 에러를 throw합니다.

```typescript
import { validateDataWithSchema } from '@asapjs/schema';
import { TypeIs } from '@asapjs/schema';

const schema = {
  userId: TypeIs.INT({ comment: '사용자 ID' }),
  email: TypeIs.STRING({ comment: '이메일' }),
  nickname: TypeIs.STRING({ comment: '닉네임', optional: true }),
};

// 필수 필드 누락 시 throw
validateDataWithSchema({ userId: 1 }, schema);
// → Error: "Missing required field: email"

// 타입 위반 시 throw
validateDataWithSchema({ userId: 'abc', email: 'test@test.com' }, schema);
// → Error: "Field 'userId' has invalid type for int"
```

#### `mapDataWithSchema(data, schema)`

유효성 검사 없이 스키마에 맞춰 데이터를 변환만 수행합니다. `error()` 팩토리가 내부적으로 이 함수를 사용하여 에러 데이터를 매핑합니다.

```typescript
import { mapDataWithSchema } from '@asapjs/schema';
import { TypeIs } from '@asapjs/schema';

const schema = {
  userId: TypeIs.INT(),
  name: TypeIs.STRING(),
};

const mapped = mapDataWithSchema({ userId: '123', name: 'Alice' }, schema);
// → { userId: 123, name: 'Alice' }  ("123" → 123 자동 변환)
```

#### 특징

- `optional: true` 옵션이 있는 필드는 `validateDataWithSchema`에서 누락을 허용
- 스키마 타입이 함수로 전달된 경우 (`() => TypeIs.STRING()`) 자동으로 실행하여 해석
- `mapDataWithSchema`는 변환 실패 시 원본 값을 그대로 유지 (throw하지 않음)

---

### Effect 기반 에러 미들웨어

`@asapjs/error`는 [Effect](https://effect.website/) 라이브러리를 활용하여 구조화된 에러 처리와 트레이싱을 제공합니다.

#### `wrapWithEffect(handler)`

라우트 핸들러를 Effect 기반 트레이싱으로 감싸는 래퍼 함수입니다. 모든 컨트롤러 핸들러에 적용하는 것을 권장합니다.

```typescript
import { wrapWithEffect } from '@asapjs/error';
import { RouterController, Get, ExecuteArgs } from '@asapjs/router';

export default class UserController extends RouterController {
  @Get('/:id', { title: '사용자 조회' })
  public getUser = wrapWithEffect(async ({ path }: ExecuteArgs) => {
    const user = await findUser(path.id);
    return { result: user };
  });
}
```

#### 동작 방식

1. 핸들러를 `Effect.promise()`로 감싸서 Effect 컨텍스트에서 실행
2. `Effect.withSpan()`으로 HTTP 메서드·경로·요청 ID를 포함한 트레이싱 스팬 생성
3. `Effect.runPromiseExit()`로 실행하여 성공/실패를 `Exit` 타입으로 분리
4. 실패 시 `causeToError()`로 원본 에러를 추출하여 `next(error)` 또는 `throw`

#### `effectErrorHandler`

Express 에러 미들웨어로, 모든 에러를 통일된 형식으로 응답합니다.

```typescript
import { effectErrorHandler } from '@asapjs/error';

// Express 앱에 등록 (프레임워크가 자동 등록)
app.use(effectErrorHandler);
```

에러 처리 시 다음 정보를 구조화된 로그로 기록합니다:

- `requestId`: 요청 고유 ID
- `method`, `url`, `ip`, `userAgent`: HTTP 요청 정보
- `file`, `line`, `function`: 에러 발생 위치 (스택 트레이스에서 추출)

#### 에러 전파 흐름

```
핸들러에서 throw
  → wrapWithEffect가 Effect Exit로 캡처
    → causeToError()로 원본 에러 추출
      → next(error) 호출 (Express 미들웨어 체인)
        → effectErrorHandler 도달
          → resolveErrorBody()로 정규화
            → HttpError → { status, errorCode, message, data }
            → Legacy HttpException → { status, errorCode: "LEGACY_HTTP_EXCEPTION", message }
            → 기타 Error → { status: 500, errorCode: "INTERNAL_SERVER_ERROR", message }
          → res.status(status).json(errorBody)
```

#### 헬퍼 함수

- `errorToResponse(error, res)`: 에러를 정규화하여 Express 응답으로 전송
- `resolveErrorBody(error)`: 다양한 에러 타입을 `HttpErrorBody` 형식으로 정규화
- `causeToError(cause)`: Effect의 `Cause` 타입에서 원본 에러를 추출 (failure → die → interrupt 순서)
- `runEffectAsPromise(effect)`: Effect를 Promise로 실행하고 실패 시 원본 에러를 throw

---

### Sentry 통합

ASAPJS는 `@sentry/node`를 통한 에러 모니터링을 지원합니다. 설정만 추가하면 자동으로 초기화됩니다.

#### 설정

`AsapJSConfig`에 `sentry` 객체를 추가합니다:

```typescript
import Application from '@asapjs/core';

const app = new Application(__dirname, {
  port: 3000,
  extensions: ['@asapjs/router', '@asapjs/sequelize'],
  sentry: {
    dsn: 'https://examplePublicKey@o0.ingest.sentry.io/0',
    environment: 'production',  // 기본값: 'development'
  },
  // ... 기타 설정
});
```

#### 초기화 시점

`Application.run()` → `initModules()` 내부에서 `config.sentry`가 존재하면 자동 초기화됩니다:

```typescript
// @asapjs/core 내부 동작 (직접 호출 불필요)
if (this.config.sentry !== undefined) {
  const Sentry = require('@sentry/node');
  Sentry.init({
    dsn: this.config.sentry.dsn,
    environment: this.config.sentry.environment || 'development',
    tracesSampleRate: 1.0,
  });
}
```

#### 설정 옵션

| 옵션 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `dsn` | `string` | ✅ | Sentry 프로젝트 DSN |
| `environment` | `string` | ❌ | 환경 이름 (기본값: `'development'`) |

#### 에러 핸들러와의 연동

- `effectErrorHandler`가 처리하는 모든 에러는 Sentry가 자동으로 캡처합니다
- `tracesSampleRate: 1.0`으로 설정되어 모든 트랜잭션이 추적됩니다
- `config.sentry`가 `undefined`이면 Sentry는 초기화되지 않으며 성능에 영향 없음
