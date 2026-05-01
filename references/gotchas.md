# 흔한 실수와 안티패턴

AI 에이전트가 asapjs 코드를 생성할 때 가장 빈번하게 발생하는 실수 목록입니다.

## 🔴 치명적 (런타임 에러)

### registerRoutes() 누락

```typescript
// ❌ 라우트가 등록되지 않음 — 모든 엔드포인트 404
constructor() {
  super();
  this.userService = new UserApplication();
}

// ✅ 반드시 super() 직후 호출
constructor() {
  super();
  this.registerRoutes();  // 서비스 인스턴스화 전에!
  this.userService = new UserApplication();
}
```

### 파일명 규칙 위반

```
❌ UserModel.ts      → 자동 탐색 안됨, Sequelize 모델 미등록
❌ user.entity.ts    → 자동 탐색 안됨
❌ UserResponse.ts   → 자동 탐색 안됨, Swagger 스키마 미등록
❌ chatHandler.ts    → 자동 탐색 안됨, 소켓 핸들러 미등록

✅ UsersTable.ts     → Sequelize 모델로 자동 등록
✅ UserDto.ts        → Swagger 스키마로 자동 등록
✅ ChatSocket.ts     → Socket.IO 핸들러로 자동 등록
```

### @Dto 데코레이터 누락

```typescript
// ❌ Swagger 스키마 미등록, middleware() 동작 안함
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: '아이디' })
  id: number;
}

// ✅ @Dto 데코레이터 필수
@Dto({ name: 'user_dto', defineTable: UsersTable })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: '아이디' })
  id: number;
}
```

### FOREIGNKEY에서 table 직접 참조

```typescript
// ❌ 순환 참조 위험 — 런타임 에러 가능
@TypeIs.FOREIGNKEY({ table: UsersTable, comment: '작성자 ID' })
user_id: number;

// ✅ 함수로 감싸기 (lazy evaluation)
@TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
user_id: number;
```

### reflect-metadata 미임포트

```typescript
// ❌ 데코레이터 메타데이터 동작 안함
import { Application } from '@asapjs/core';

// ✅ 반드시 첫 번째 줄에
import 'reflect-metadata';
import { Application } from '@asapjs/core';
```

## 🟡 논리 오류 (잘못된 동작)

### Sequelize DataTypes 직접 사용

```typescript
// ❌ TypeIs 시스템 우회 — Swagger 스키마 미생성, fixValue 미적용
import { DataTypes } from 'sequelize';
@Column({ type: DataTypes.STRING })
email: string;

// ✅ TypeIs 사용 — Sequelize + Swagger + fixValue 동시 생성
import { TypeIs } from '@asapjs/schema';
@TypeIs.STRING({ comment: '이메일' })
email: string;
```

### new Error() 사용

```typescript
// ❌ 항상 HTTP 500으로 처리됨
throw new Error('User not found');

// ✅ 적절한 상태 코드 + 구조화된 응답
throw UserErrors.NOT_FOUND({ userId: 42 });

// ✅ 간단한 경우
throw new HttpException(404, 'User not found');
```

### page가 1-based라고 가정

```typescript
// ❌ 첫 페이지를 page: 1로 가정
const offset = (paging.page - 1) * paging.limit;

// ✅ page는 0-based. 첫 페이지 = 0
// Repository 내부: offset = limit * page
```

### Controller에서 DB 직접 접근

```typescript
// ❌ 레이어 경계 위반
@Get('/', { title: '목록' })
async getUsers({}: ExecuteArgs) {
  return await UsersTable.findAll();  // Controller에서 직접 DB 접근
}

// ✅ Application에 위임
@Get('/', { title: '목록' })
async getUsers({ paging }: ExecuteArgs) {
  return await this.userService.list(paging);
}
```

### Application에서 req/res 사용

```typescript
// ❌ HTTP 의존성 — 테스트 불가
import { Request, Response } from 'express';
async register(req: Request, res: Response) {
  const body = req.body;
  res.json({ success: true });
}

// ✅ 순수 TypeScript — HTTP 무의존
async register(dto: CreateUserDto) {
  const user = await this.users.create(dto);
  return new UserDto().map(user);
}
```

## 🟠 패턴 오류 (비효율/비관례)

### NestJS 패턴 혼용

```typescript
// ❌ ASAPJS에 없는 패턴
@Controller('users')           // 존재하지 않음
@Injectable()                  // DI 컨테이너 없음
@Module({ controllers: [] })   // 모듈 시스템 없음

// ✅ ASAPJS 패턴
export default class UserController extends RouterController {
  public basePath = '/users';
  constructor() {
    super();
    this.registerRoutes();
    this.userService = new UserApplication();  // 직접 인스턴스화
  }
}
```

### TypeIs import 경로 오류

```typescript
// ❌ 잘못된 import
import { TypeIs } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/router';
import { TypeIs } from '@asapjs/types';

// ✅ 항상 @asapjs/schema에서
import { TypeIs } from '@asapjs/schema';
```

### DTO에서 password 노출

```typescript
// ❌ 응답 DTO에 password 포함
@Dto({ name: 'user_dto', defineTable: UsersTable })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: 'ID' })
  id: number;
  @TypeIs.PASSWORD({ comment: '비밀번호' })
  password: string;  // 응답에 비밀번호 노출!
}

// ✅ 응답 DTO에서 password 제외
@Dto({ name: 'user_dto', defineTable: UsersTable })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: 'ID' })
  id: number;
  @TypeIs.STRING({ comment: '이메일' })
  email: string;
  // password 필드 없음
}
```

### BELONGSTO 없이 FOREIGNKEY만 사용

```typescript
// ❌ 관계 객체 접근 불가
@TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
user_id: number;
// post.user 접근 불가

// ✅ 항상 쌍으로
@TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
user_id: number;

@TypeIs.BELONGSTO(() => UsersTable, 'user_id')
user: UsersTable;
```

### Express/Fastify 혼용

```typescript
// ❌ Express 컨트롤러에서 Fastify import
import { FastifyRouterController } from '@asapjs/fastify';
import { Get } from '@asapjs/router';  // Express용 데코레이터

// ✅ 하나만 선택
// Express: @asapjs/router의 RouterController + Get/Post/Put/Delete
// Fastify: @asapjs/fastify의 FastifyRouterController + Get/Post/Put/Delete
```

### TypeIs.OBJECT 사용 시도

```typescript
// ❌ TypeIs.OBJECT는 존재하지 않음 — 런타임 에러
@TypeIs.OBJECT({ comment: '설정' })
settings: { theme: string; language: string };

// ✅ 별도 *Dto.ts 파일에 DTO 클래스 생성 후 TypeIs.DTO로 참조
// dto/SettingsDto.ts 파일 생성
@Dto({ name: 'settings_dto', defineTable: UsersTable })
export default class SettingsDto extends ExtendableDto {
  @TypeIs.STRING({ comment: '테마' })
  theme: string;
  @TypeIs.STRING({ comment: '언어' })
  language: string;
}

// 사용하는 DTO에서 참조
@TypeIs.DTO({ dto: SettingsDto, as: 'settings' })
settings: SettingsDto;
```

### 한 파일에 여러 DTO 정의

```typescript
// ❌ 하나의 파일에 여러 DTO — Swagger 스키마 등록 누락 위험
// dto/UserDtos.ts
export class UserDto extends ExtendableDto { ... }
export class CreateUserDto extends ExtendableDto { ... }

// ✅ 한 파일에 하나의 DTO — 자동 탐색 보장
// dto/UserDto.ts
export default class UserDto extends ExtendableDto { ... }
// dto/CreateUserDto.ts
export default class CreateUserDto extends ExtendableDto { ... }
```

## 🟣 프레임워크 알려진 이슈

### `extensionInlcude` 오타 (sequelize 패키지)

`@asapjs/sequelize` 내부에 `extensionInlcude`라는 함수명 오타가 있습니다 (`extensionInclude`가 올바른 이름). 프레임워크 내부 함수이므로 직접 호출할 일은 없지만, 소스 코드를 읽을 때 혼동하지 마세요.

### `ExcuteArgs` 오타 (types 패키지)

`@asapjs/types`에 `ExcuteArgs`라는 인터페이스가 `ExecuteArgs`와 함께 존재합니다. 레거시 호환용이므로 항상 `ExecuteArgs`를 사용하세요.

```typescript
// ❌ 오타 인터페이스 — 사용하지 말 것
import type { ExcuteArgs } from '@asapjs/types';

// ✅ 올바른 인터페이스
import { ExecuteArgs } from '@asapjs/router';
```

### `logger` import 경로 혼동

`logger`는 `@asapjs/core`와 `@asapjs/common` 모두에서 export됩니다. 어느 쪽에서 import해도 동일하게 동작하지만, 프로젝트 내에서 하나로 통일하세요.

```typescript
// 둘 다 동작함 — 프로젝트 내 하나로 통일
import { logger } from '@asapjs/core';
import { logger } from '@asapjs/common';
```

### `dirname` 설정 속성

`config.ts`에서 `dirname: __dirname`을 설정하면 자동 탐색 루트 경로를 명시적으로 지정할 수 있습니다. `Application` 생성자의 첫 번째 인자로도 전달하지만, config 객체에도 포함하는 것이 관례입니다.

```typescript
// src/config.ts
export default {
  dirname: __dirname,  // 자동 탐색 루트 경로
  // ...
};

// src/index.ts
new Application(__dirname, config).run();  // 여기서도 전달
```

### CLI 생성기 템플릿 버그

`asapjs generate` 명령으로 생성되는 코드에 여러 버그가 있습니다:

| 생성기 | 버그 | 올바른 패턴 |
|--------|------|-------------|
| controller | `params` 사용 | `path` 사용 |
| controller | `''` 빈 문자열 라우트 | `'/'` 사용 |
| controller | `any` 타입 | 적절한 DTO 타입 사용 |
| dto | `@asapjs/sequelize`에서 TypeIs import | `@asapjs/schema`에서 import |
| dto | `required` 옵션 사용 | `allowNull` 옵션 또는 생략 |
| service | `HttpException` 직접 사용 | `error()` 팩토리 사용 |
| service | `any` 타입 | 적절한 DTO/Entity 타입 사용 |

생성 후 반드시 수정하세요. 상세 내용: `cli-reference.md` 참조.

### 동적 `require()` 패턴

프레임워크 내부에서 `route.ts`, `*Table.ts`, `*Dto.ts`, `*Socket.ts` 파일을 동적 `require()`로 로드합니다. 이로 인해:

- 파일명 규칙을 어기면 **조용히 무시**됩니다 (에러 없이 로드 안 됨)
- 순환 참조 시 런타임 에러가 발생할 수 있습니다
- FOREIGNKEY의 `table` 옵션을 함수로 감싸야 하는 이유가 이것입니다

## 📋 체크리스트

새 도메인 추가 시 확인:

- [ ] Entity 파일명이 `*Table.ts`로 끝나는가?
- [ ] DTO 파일명이 `*Dto.ts`로 끝나는가?
- [ ] Entity에 `@Table` 데코레이터가 있는가?
- [ ] DTO에 `@Dto` 데코레이터가 있는가?
- [ ] Controller에서 `this.registerRoutes()` 호출하는가?
- [ ] Controller 생성자에서 `super()` → `registerRoutes()` → 서비스 인스턴스화 순서인가?
- [ ] `route.ts`에 새 Controller를 등록했는가?
- [ ] TypeIs는 `@asapjs/schema`에서 import하는가?
- [ ] FOREIGNKEY에 `table: () => Table` (함수)로 전달하는가?
- [ ] 응답 DTO에 민감한 필드(password 등)가 없는가?
- [ ] 한 파일에 하나의 DTO만 정의했는가?
- [ ] 중첩 객체에 `TypeIs.OBJECT` 대신 별도 DTO + `TypeIs.DTO()`를 사용했는가?
