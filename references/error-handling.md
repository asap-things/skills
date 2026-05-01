# 에러 처리

ASAPJS는 두 가지 에러 클래스를 제공합니다.

| 클래스 | 패키지 | 필드 | 용도 |
|--------|--------|------|------|
| `HttpError` | `@asapjs/error` | `status`, `errorCode`, `message`, `data?` | 타입 안전한 에러 (권장) |
| `HttpException` | `@asapjs/router` | `status`, `message` | 간단한 에러 / 레거시 |

## error() 팩토리 (권장)

```typescript
import { error } from '@asapjs/error';
import { TypeIs } from '@asapjs/schema';

// 에러 생성자 정의
const UserNotFound = error(
  404,                                    // HTTP 상태 코드
  'USER_NOT_FOUND',                       // 에러 코드 (문자열)
  '사용자 {userId}를 찾을 수 없습니다',      // 메시지 템플릿 ({key}로 보간)
  { userId: TypeIs.INT() }                // data 스키마 (타입 + Swagger)
);

// 에러 던지기 — data가 타입 검사됨
throw UserNotFound({ userId: 42 });
// → HTTP 404 { status: 404, errorCode: 'USER_NOT_FOUND',
//              message: '사용자 42를 찾을 수 없습니다', data: { userId: 42 } }
```

### error() 파라미터

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `status` | `number` | HTTP 상태 코드 |
| `code` | `string` | 에러 식별 코드. Swagger 스키마 이름으로도 사용 |
| `message` | `string` | 메시지 템플릿. `{key}` 형식으로 data 값 보간 |
| `schema` | `Record<string, SchemaType>` | data 필드의 TypeIs 스키마 |

## 에러 정의 패턴

도메인별 에러 클래스에 static 필드로 정의:

```typescript
// src/user/errors/UserErrors.ts
import { error } from '@asapjs/error';
import { TypeIs } from '@asapjs/schema';

export class UserErrors {
  static NOT_FOUND = error(
    404, 'USER_NOT_FOUND',
    '사용자를 찾을 수 없습니다. ID: {userId}',
    { userId: TypeIs.INT({ comment: '사용자 ID' }) }
  );

  static EMAIL_DUPLICATE = error(
    409, 'USER_EMAIL_DUPLICATE',
    '이미 사용 중인 이메일입니다: {email}',
    {
      email: TypeIs.STRING({ comment: '중복된 이메일' }),
      existingUserId: TypeIs.INT({ comment: '기존 사용자 ID' }),
    }
  );

  static INVALID_DATA = error(
    400, 'USER_INVALID_DATA',
    '유효하지 않은 데이터입니다',
    {}
  );
}
```

## Swagger 에러 자동 등록

데코레이터의 `errors` 옵션에 전달하면 Swagger에 에러 응답 스키마 자동 등록:

```typescript
@Get('/:userId', {
  title: '사용자 상세 조회',
  response: UserDto,
  errors: [UserErrors.NOT_FOUND],  // Swagger에 404 에러 응답 자동 등록
})
public getUserById = async ({ path }: ExecuteArgs<{ userId: string }>) => {
  const result = await this.userService.info(path?.userId);
  return { result };
};

@Post('/', {
  title: '사용자 생성',
  body: CreateUserDto,
  response: UserDto,
  errors: [UserErrors.EMAIL_DUPLICATE, UserErrors.INVALID_DATA],  // 409, 400 자동 등록
})
public createUser = async ({ body, user }: ExecuteArgs<{}, {}, CreateUserDto>) => {
  const result = await this.userService.create(body, user);
  return { result };
};
```

## HttpException (간단한 경우)

```typescript
import { HttpException } from '@asapjs/router';

throw new HttpException(404, 'Post not found');
// → HTTP 404 { status: 404, errorCode: 'LEGACY_HTTP_EXCEPTION', message: 'Post not found' }
```

## 에러 응답 형식

### HttpError (error() 팩토리)
```json
{ "status": 404, "errorCode": "USER_NOT_FOUND", "message": "사용자를 찾을 수 없습니다. ID: 42", "data": { "userId": 42 } }
```

### HttpException (레거시)
```json
{ "status": 404, "errorCode": "LEGACY_HTTP_EXCEPTION", "message": "Post not found" }
```

### 일반 Error (500)
```json
{ "status": 500, "errorCode": "INTERNAL_SERVER_ERROR", "message": "알 수 없는 서버 오류가 발생했습니다." }
```

## 에러 처리 흐름

```
핸들러에서 throw
  ↓
Wrapper의 try-catch
  ↓
isServerError 판별 (status === 500 || undefined)
  ↓ (서버 에러인 경우)
logger.error + Sentry.captureException (설정 시)
  ↓
errorToResponse(err, res) → resolveErrorBody(err)
  ↓
HTTP 응답 전송
```

### resolveErrorBody 분기

| 에러 유형 | 조건 | 응답 형식 |
|----------|------|----------|
| `HttpError` | `instanceof HttpError` | `{ status, errorCode, message, data? }` |
| `HttpException` | `status` + `message` 있고 `errorCode` 없음 | `{ status, errorCode: 'LEGACY_HTTP_EXCEPTION', message }` |
| 일반 에러 | 위 조건 해당 없음 | `{ status: 500, errorCode: 'INTERNAL_SERVER_ERROR', message }` |

## 미들웨어에서 에러 선언

`defineMiddleware`의 `errors` 옵션으로 미들웨어가 throw할 수 있는 에러를 선언하면, 해당 미들웨어가 등록된 모든 라우트의 Swagger에 자동 포함됩니다.

→ 상세: `middleware-auth.md` 참조

## ⚠️ 주의사항

- `new Error('message')`를 throw하면 항상 HTTP 500으로 처리됨. 적절한 상태 코드가 필요하면 `error()` 또는 `HttpException` 사용
- Sentry 연동은 `config.sentry` 설정 시 자동. 500 에러만 캡처
- Fastify의 `fastifyErrorHandler`도 동일한 `resolveErrorBody()`를 사용하므로 Express/Fastify 에러 응답 형식 동일
