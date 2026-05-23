# CLI 도구 (@asapjs/cli)

ASAPJS CLI는 프로젝트 생성, 개발 서버, 빌드, 프로덕션 실행, 코드 생성, 환경 점검을 제공합니다. 모든 명령은 `asapjs` 바이너리로 실행합니다.

## 명령어 목록

| 명령어 | 설명 |
|--------|------|
| `asapjs new <name>` | 새 프로젝트 생성 |
| `asapjs dev` | 개발 서버 (hot-reload) |
| `asapjs build` | 프로덕션 빌드 |
| `asapjs start` | 프로덕션 서버 실행 |
| `asapjs generate <type> <name>` (별칭: `g`) | 코드 생성 |
| `asapjs env:check` | 환경 설정 및 Infisical 연결 점검 |

---

## `asapjs new <name>`

새 ASAPJS 프로젝트를 스캐폴딩합니다. 대화형 프롬프트로 패키지 매니저, DB 사용 여부, DB 종류를 선택합니다.

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `-t, --template <template>` | 프로젝트 템플릿 | `basic` |

### 대화형 프롬프트

| 질문 | 선택지 | 기본값 |
|------|--------|--------|
| 패키지 매니저 | `yarn`, `npm`, `pnpm` | `yarn` |
| DB 사용 여부 | `Y/n` | `true` |
| DB 종류 (DB 사용 시) | `mysql`, `postgres`, `sqlite` | `mysql` |

### 생성되는 구조

```
<name>/
  src/
    index.ts          — Application 엔트리
    config.ts         — 설정 (extensions, db, swagger)
    route.ts          — 빈 라우트 배열
  env/
  .asapjs.json        — CLI 설정
  .env / .env.local / .env.dev / .env.prod
  tsconfig.json
  package.json
  .gitignore
```

### 생성되는 의존성

- `@asapjs/core`, `@asapjs/router`, `@asapjs/common`, `dotenv`, `reflect-metadata`
- DB 선택 시: `@asapjs/sequelize` + DB 드라이버 (`mysql2`, `postgres`, `sqlite`)
- devDependencies: `@asapjs/cli`, `@types/node`, `typescript`, `ts-node`

### 사용 예시

```bash
asapjs new my-api
# → 대화형 프롬프트 → 프로젝트 생성 → 의존성 설치
cd my-api
asapjs dev
```

---

## `asapjs dev`

`ts-node`로 개발 서버를 실행하고, `chokidar`로 파일 변경을 감지하여 자동 재시작합니다.

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `-e, --env <env>` | 환경 이름 | `local` |
| `-p, --port <port>` | 포트 번호 | `.asapjs.json` 설정 또는 `3000` |

### 동작 방식

1. `loadEnv(env, cwd)` — 환경 파일 로드 + Infisical 시크릿 가져오기
2. `loadConfig(cwd)` — `.asapjs.json` 설정 로드
3. 포트 결정: `--port` 옵션 > `.asapjs.json`의 `environments[env].port` > `3000`
4. `npx ts-node -r tsconfig-paths/register src/index.ts` 실행
5. `TS_NODE_TRANSPILE_ONLY=true` — 타입 체크 생략으로 빠른 시작

### 파일 감시 대상

- `src/**/*.ts` — TypeScript 소스
- `.env*` — 환경 파일
- `env/**/*` — env 디렉토리 내 모든 파일

변경 감지 시 200ms 디바운스 후 프로세스 트리를 종료하고 재시작합니다.

### 환경 변수 우선순위

```
호스트 환경 변수 (최우선)
  ↑ 덮어씀
Infisical 시크릿
  ↑ 덮어씀
dotenv 파일
```

`dev` 명령은 호스트 환경 변수 스냅샷을 시작 시점에 저장하고, Infisical 시크릿 위에 다시 적용합니다. 따라서 호스트 환경 변수가 항상 최우선입니다.

### 사용 예시

```bash
asapjs dev                    # local 환경, 기본 포트
asapjs dev -e development     # development 환경
asapjs dev -p 4000            # 포트 4000 지정
```

---

## `asapjs build`

TypeScript를 컴파일하고 프로덕션 배포용 `dist` 디렉토리를 생성합니다.

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `-e, --env <env>` | 환경 이름 | `production` |
| `-o, --output <dir>` | 출력 디렉토리 | `dist` |

### 빌드 단계

1. 환경 파일 로드 (`loadEnv`)
2. `.asapjs.json` 설정 로드
3. `dist` 디렉토리 생성
4. 환경 파일을 `dist/.env`로 복사
5. `.asapjs.json`을 `dist/.asapjs.json`으로 복사
6. 워크스페이스 모노레포 감지 (`findWorkspaceRoot`)
7. TypeScript 컴파일 (sourceMap 설정 반영)
8. 워크스페이스 환경: 중첩된 출력 경로에 대한 엔트리 래퍼 생성
9. `dist/tsconfig.json` 생성 — 런타임 path alias 해석용 (`tsconfig-paths/register`)
10. `package.json` 복사 (`devDependencies`, `scripts` 제거)

### 워크스페이스 모노레포 지원

`tsc`가 모노레포에서 출력을 중첩 구조로 생성할 때, CLI가 자동으로 `dist/index.js` 엔트리 래퍼를 생성합니다:

```javascript
module.exports = require('./packages/my-app/src/index.js');
```

### 사용 예시

```bash
asapjs build                          # production 환경, dist 출력
asapjs build -e development           # development 환경
asapjs build -o build                 # build 디렉토리로 출력
```

---

## `asapjs start`

빌드된 애플리케이션을 프로덕션 모드로 실행합니다.

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `-e, --env <env>` | 환경 이름 | `production` |
| `-d, --dir <dir>` | 애플리케이션 디렉토리 | `dist` |

### 동작 방식

1. `dist` 디렉토리 존재 확인 (없으면 `asapjs build` 실행 안내)
2. 엔트리 파일 탐색: `dist/index.js` → `dist/src/index.js`
3. Infisical 시크릿 로드 (`loadEnv`)
4. `node -r tsconfig-paths/register <entry>` 실행
5. `NODE_ENV=production` 강제 설정

### 환경 변수 우선순위

`dev` 명령과 동일하게 호스트 환경 변수가 최우선입니다.

### 사용 예시

```bash
asapjs start                          # dist에서 production 실행
asapjs start -d build                 # build 디렉토리에서 실행
asapjs start -e staging               # staging 환경으로 실행
```

---

## `asapjs generate <type> <name>` (별칭: `g`)

컨트롤러, 서비스(Application), DTO 파일을 생성합니다.

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `-p, --path <path>` | 생성 대상 경로 | `src` |

### 지원 타입

| 타입 | 별칭 | 생성 파일 | 클래스명 |
|------|------|----------|---------|
| `controller` | — | `{name}.controller.ts` | `{Name}Controller` |
| `service` | `application` | `{name}.application.ts` | `{Name}Application` |
| `dto` | — | `{name}.dto.ts` | `{Name}Dto` |

### 사용 예시

```bash
asapjs generate controller user       # src/user.controller.ts
asapjs g service order                 # src/order.application.ts
asapjs g dto product -p src/dto        # src/dto/product.dto.ts
```

### ⚠️ 생성기 템플릿 버그 (중요)

**CLI 생성기가 만드는 코드는 프레임워크의 실제 컨벤션과 다릅니다.** 생성된 파일을 그대로 사용하지 말고, 아래 문제를 반드시 수정하세요.

#### controller 생성기 버그

| 문제 | 생성기 출력 | 올바른 패턴 |
|------|-----------|------------|
| 빈 문자열 라우트 | `@Get('')` | `@Get('/')` |
| `params` 사용 | `{ params }: ExecuteArgs<{}, { id: string }, {}>` | `{ path }: ExecuteArgs<{}, { id: string }, {}>` |
| `any` 타입 사용 | `ExecuteArgs<any, {}, {}>`, `ExecuteArgs<{}, {}, any>` | 구체적인 DTO 타입 지정 |
| DTO 미연결 | 데코레이터에 `query`, `body`, `response` 없음 | `@Get('/', { query: ListDto, response: ItemDto })` |

**생성기 출력 (버그 포함):**

```typescript
import { Get, Post, Put, Delete, RouterController, ExecuteArgs } from '@asapjs/router';

export default class UserController extends RouterController {
  public basePath = '/users';
  public tag = 'users';

  constructor() {
    super();
    this.registerRoutes();
  }

  @Get('', {                    // ❌ '' 대신 '/' 사용해야 함
    title: 'Get all users',
  })
  public getAll = async ({ query }: ExecuteArgs<any, {}, {}>) => {  // ❌ any 타입
    return { result: [] };
  };

  @Get('/:id', {
    title: 'Get user by ID',
  })
  public getById = async ({ params }: ExecuteArgs<{}, { id: string }, {}>) => {  // ❌ params → path
    return { result: { id: params.id } };  // ❌ params → path
  };

  @Post('', {                   // ❌ '' 대신 '/' 사용해야 함
    title: 'Create new user',
  })
  public create = async ({ body }: ExecuteArgs<{}, {}, any>) => {  // ❌ any 타입
    return { result: body };
  };
}
```

**올바른 패턴:**

```typescript
import { RouterController, Get, Post, Put, Delete, ExecuteArgs } from '@asapjs/router';
import { UserApplication } from '../application/UserApplication';
import UserDto from '../dto/UserDto';
import CreateUserDto from '../dto/CreateUserDto';
import GetUserListQueryDto from '../dto/GetUserListQueryDto';

export default class UserController extends RouterController {
  public basePath = '/users';
  public tag = 'users';
  private userService: UserApplication;

  constructor() {
    super();
    this.registerRoutes();
    this.userService = new UserApplication();
  }

  @Get('/', {
    title: '사용자 목록 조회',
    query: GetUserListQueryDto,
    response: UserDto,
  })
  public getAll = async ({ paging, user }: ExecuteArgs<{}, GetUserListQueryDto, {}>) => {
    return { result: await this.userService.list(paging, user) };
  };

  @Get('/:id', {
    title: '사용자 상세 조회',
    response: UserDto,
  })
  public getById = async ({ path }: ExecuteArgs<{ id: string }, {}, {}>) => {
    return { result: await this.userService.info(Number(path?.id)) };
  };

  @Post('/', {
    title: '사용자 생성',
    body: CreateUserDto,
    response: UserDto,
  })
  public create = async ({ body }: ExecuteArgs<{}, {}, CreateUserDto>) => {
    return { result: await this.userService.create(body) };
  };
}
```

#### dto 생성기 버그

| 문제 | 생성기 출력 | 올바른 패턴 |
|------|-----------|------------|
| TypeIs import 경로 | `import { TypeIs } from '@asapjs/sequelize'` | `import { TypeIs } from '@asapjs/schema'` |
| 존재하지 않는 `required` 옵션 | `@TypeIs.STRING({ required: true })` | `@TypeIs.STRING({ allowNull: false })` 또는 옵션 생략 |
| 한 줄 import | `import { Dto, ExtendableDto, TypeIs } from '@asapjs/sequelize'` | TypeIs만 분리: `import { TypeIs } from '@asapjs/schema'` + `import { Dto, ExtendableDto } from '@asapjs/sequelize'` |

**생성기 출력 (버그 포함):**

```typescript
import { Dto, ExtendableDto, TypeIs } from '@asapjs/sequelize';  // ❌ TypeIs는 @asapjs/schema에서 import해야 함

@Dto({ name: 'user_dto' })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: 'ID' })
  id: number;

  @TypeIs.STRING({ comment: 'Name' })
  name: string;

  @TypeIs.STRING({ comment: 'Description', required: false })  // ❌ required 옵션 없음
  description?: string;

  @TypeIs.BOOLEAN({ comment: 'Active status', defaultValue: true })
  is_active: boolean;
}

@Dto({ name: 'create_user_dto' })
export class CreateUserDto extends ExtendableDto {
  @TypeIs.STRING({ comment: 'Name', required: true })  // ❌ required 옵션 없음
  name: string;
}
```

**올바른 패턴:**

```typescript
import { Dto, ExtendableDto } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';
import UsersTable from '../domain/entity/UsersTable';

@Dto({ name: 'user_dto', defineTable: UsersTable })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: 'ID' })
  id: number;

  @TypeIs.STRING({ comment: '이름' })
  name: string;

  @TypeIs.STRING({ comment: '설명', allowNull: true })
  description?: string;

  @TypeIs.BOOLEAN({ comment: '활성 상태', defaultValue: true })
  is_active: boolean;
}
```

#### service 생성기 버그

| 문제 | 생성기 출력 | 올바른 패턴 |
|------|-----------|------------|
| `HttpException` 직접 사용 | `throw new HttpException(500, 'message')` | `error()` 팩토리 함수 사용 |
| 모든 곳에 `any` 타입 | `query: any`, `data: any`, `Promise<any>` | 구체적인 DTO 타입 지정 |
| 에러 처리 패턴 | `try/catch`로 `HttpException` 래핑 | 도메인 에러 객체 (`*Errors.ts`) 사용 |

**생성기 출력 (버그 포함):**

```typescript
import { HttpException } from '@asapjs/router';  // ❌ error() 팩토리 사용해야 함

export class UserApplication {
  constructor() {
    // Initialize dependencies
  }

  async findAll(query: any): Promise<any[]> {  // ❌ any 타입
    try {
      return [];
    } catch (error) {
      throw new HttpException(500, 'Failed to fetch users');  // ❌ HttpException 직접 사용
    }
  }

  async findById(id: number): Promise<any> {  // ❌ any 타입
    try {
      const item = null;
      if (!item) {
        throw new HttpException(404, 'user not found');  // ❌ HttpException 직접 사용
      }
      return item;
    } catch (error) {
      if (error instanceof HttpException) throw error;
      throw new HttpException(500, 'Failed to fetch user');
    }
  }
}
```

**올바른 패턴:**

```typescript
import UsersTable from '../domain/entity/UsersTable';
import UserDto from '../dto/UserDto';
import { UserErrors } from '../errors/UserErrors';

export class UserApplication {
  private users: typeof UsersTable;

  constructor() {
    this.users = UsersTable;
  }

  public info = async (userId: number) => {
    const raw = await this.users.findByPk(userId);
    if (!raw) throw UserErrors.NOT_FOUND({ userId });
    return new UserDto().map(raw);
  };

  public create = async (body: CreateUserDto) => {
    const data = new CreateUserDto().map(body);
    const raw = await this.users.create(data);
    return new UserDto().map(raw);
  };
}
```

---

## `asapjs env:check`

환경 설정 파일과 Infisical 연결 상태를 점검합니다.

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `-e, --env <env>` | 환경 이름 | `development` |

### 점검 항목

1. `.asapjs.json`에서 해당 환경 설정 존재 여부
2. 환경 파일 존재 여부 및 키 개수
3. Infisical 시크릿 설정 여부
4. Infisical 인증 정보 확인 (clientId, clientSecret)
5. `@infisical/sdk` 설치 여부
6. Infisical 연결 및 인증 테스트
7. 시크릿 목록 가져오기
8. 키 병합 미리보기 — dotenv와 Infisical 키 간 충돌 표시

### 키 병합 미리보기 출력

```
Key Merge Preview:
  DB_HOST                             ← Infisical (overrides dotenv)
  DB_PASSWORD                         ← Infisical
  PORT                                ← dotenv

Summary: 5 from Infisical, 3 dotenv-only, 2 overlapping (Infisical wins)
```

Infisical 시크릿이 dotenv 키와 겹칠 경우 Infisical이 우선합니다.

### 사용 예시

```bash
asapjs env:check                      # development 환경 점검
asapjs env:check -e production        # production 환경 점검
asapjs env:check -e local             # local 환경 점검
```

---

## `.asapjs.json` 설정 스키마

프로젝트 루트에 위치하는 CLI 설정 파일입니다.

```json
{
  "environments": {
    "local": {
      "envFile": ".env.local",
      "port": 3000
    },
    "development": {
      "envFile": ".env.dev",
      "port": 4000,
      "secrets": {
        "provider": "infisical",
        "siteUrl": "https://infisical.example.com",
        "projectId": "project-uuid",
        "environment": "dev",
        "secretPath": "/",
        "auth": {
          "method": "universal",
          "clientIdEnv": "INFISICAL_CLIENT_ID",
          "clientSecretEnv": "INFISICAL_CLIENT_SECRET"
        }
      }
    },
    "production": {
      "envFile": ".env.prod",
      "port": 8080
    }
  },
  "build": {
    "outDir": "dist",
    "sourceMap": true
  }
}
```

### `AsapJSConfig` 인터페이스

| 필드 | 타입 | 설명 |
|------|------|------|
| `environments` | `Record<string, EnvironmentConfig>` | 환경별 설정 |
| `build` | `BuildConfig` | 빌드 설정 |

### `EnvironmentConfig`

| 필드 | 타입 | 설명 |
|------|------|------|
| `envFile` | `string?` | 환경 파일 경로 |
| `port` | `number?` | 포트 번호 |
| `secrets` | `SecretsConfig?` | 외부 시크릿 제공자 설정 |

### `BuildConfig`

| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `outDir` | `string?` | `dist` | 빌드 출력 디렉토리 |
| `sourceMap` | `boolean?` | `true` | 소스맵 생성 여부 |

### 환경 파일 기본 경로

`envFile`을 지정하지 않으면 다음 기본 경로를 사용합니다:

| 환경 | 기본 경로 |
|------|----------|
| `local` | `./env/.env.local` |
| `development` | `./env/.env.dev` |
| `production` | `./env/.env.prod` |
| 기타 | `.env.{환경이름}` |

기본 경로에도 파일이 없으면 프로젝트 루트의 `.env`를 폴백으로 사용합니다.

---

## Infisical 시크릿 통합

`@infisical/sdk`를 통해 외부 시크릿 관리 서비스에서 환경 변수를 가져옵니다.

### `SecretsConfig` 인터페이스

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `provider` | `'infisical'` | ✅ | 시크릿 제공자 (현재 `infisical`만 지원) |
| `siteUrl` | `string?` | — | Infisical 서버 URL (셀프호스팅 시) |
| `projectId` | `string` | ✅ | Infisical 프로젝트 ID |
| `environment` | `string` | ✅ | Infisical 환경 이름 (예: `dev`, `staging`, `prod`) |
| `secretPath` | `string?` | — | 시크릿 경로 (기본값: `/`) |
| `auth.method` | `'universal'` | ✅ | 인증 방식 (현재 `universal`만 지원) |
| `auth.clientId` | `string?` | — | 클라이언트 ID (직접 지정) |
| `auth.clientSecret` | `string?` | — | 클라이언트 시크릿 (직접 지정) |
| `auth.clientIdEnv` | `string?` | — | 클라이언트 ID를 담은 환경 변수 이름 |
| `auth.clientSecretEnv` | `string?` | — | 클라이언트 시크릿을 담은 환경 변수 이름 |

### 인증 방식

인증 정보는 두 가지 방식으로 제공할 수 있습니다:

1. **직접 지정** — `clientId`, `clientSecret`에 값을 직접 입력 (보안 주의)
2. **환경 변수 참조** — `clientIdEnv`, `clientSecretEnv`에 환경 변수 이름 지정 (권장)

```json
{
  "auth": {
    "method": "universal",
    "clientIdEnv": "INFISICAL_CLIENT_ID",
    "clientSecretEnv": "INFISICAL_CLIENT_SECRET"
  }
}
```

### 시크릿 로드 흐름

```
1. .asapjs.json에서 secrets 설정 확인
2. @infisical/sdk 로드 (미설치 시 에러)
3. Universal Auth로 인증
4. listSecrets() — expandSecretReferences: true
5. 시크릿을 process.env에 병합
```

### 필수 의존성

Infisical 시크릿을 사용하려면 `@infisical/sdk`를 설치해야 합니다:

```bash
yarn add @infisical/sdk
```

### NODE_ENV 설정 규칙

`loadEnv` 함수는 환경 이름에 따라 `NODE_ENV`를 자동 설정합니다:

| 환경 이름 | NODE_ENV |
|----------|----------|
| `production` | `production` |
| 그 외 모두 | `development` |
