# TypedMiddleware & 인증

## TypedMiddleware 개념

일반 Express 미들웨어와 달리, TypedMiddleware는 라우트 옵션을 받아 미들웨어를 생성하는 **팩토리** 구조입니다.

```typescript
type TypedMiddleware<
  RouteOptions extends Record<string, any> = {},
  Context extends Record<string, any> = {}
> = ((options: RouteOptions) => (req: any, res: any, next: any) => void) & {
  readonly __contextType?: Context;
  readonly __errors?: any[] | ((options: RouteOptions) => any[]);
};
```

- `RouteOptions`: `@Get`, `@Post` 등 데코레이터 옵션에 추가될 필드 (예: `auth?: boolean`)
- `Context`: 미들웨어가 `req`에 첨부하여 핸들러에 전달할 데이터 타입 (예: `{ user: JwtUserPayload }`)

## defineMiddleware 헬퍼

커스텀 미들웨어 작성 시 사용:

```typescript
import { defineMiddleware } from '@asapjs/router';
import { error } from '@asapjs/error';

const AuthErrors = {
  NO_TOKEN:          error(403, 'NO_TOKEN_PROVIDED', '토큰이 제공되지 않았습니다', {}),
  INVALID_SIGNATURE: error(403, 'INVALID_TOKEN_SIGNATURE', '유효하지 않은 토큰 서명입니다', {}),
  UNAUTHORIZED:      error(401, 'UNAUTHORIZED', '인증이 만료되었거나 유효하지 않습니다', {}),
};

export const jwtMiddleware = defineMiddleware<
  { auth?: boolean },        // RouteOptions
  { user: JwtUserPayload }   // Context
>(
  ({ auth = false } = {}) =>
    (req, res, next) => {
      if (!auth) return next();
      // JWT 검증 로직 ...
      req.user = decoded;
      next();
    },
  {
    // 함수 형태: auth:true인 라우트에만 에러를 Swagger에 포함
    errors: (options) =>
      options.auth !== false
        ? [AuthErrors.NO_TOKEN, AuthErrors.INVALID_SIGNATURE, AuthErrors.UNAUTHORIZED]
        : [],
  }
);
```

### errors 옵션

| 형태 | 설명 |
|------|------|
| `ErrorCreator[]` | 모든 라우트에 항상 포함 |
| `(options) => ErrorCreator[]` | RouteOptions에 따라 동적 결정 |

## Config 등록 및 타입 선언

```typescript
// src/config.ts
import { defineRouterConfig, type InferMiddlewareRouteOptions } from '@asapjs/router';
import { jwtMiddleware, type JwtUserPayload } from './middleware/jwtMiddleware';

export const routerConfig = defineRouterConfig({
  middleware: [jwtMiddleware],
});

export default {
  router: routerConfig,
  auth: {
    jwt_access_token_secret: process.env.JWT_SECRET || 'your-secret-key',
  },
};

// 전역 타입 확장 (Module Augmentation)
declare module '@asapjs/router' {
  interface GlobalMiddlewareContext {
    user: JwtUserPayload;  // ExecuteArgs에서 user 타입 자동 추론
  }
  interface GlobalRouteOptions
    extends InferMiddlewareRouteOptions<typeof routerConfig.middleware> {}
    // = { auth?: boolean }
}
```

### 핵심 포인트

- `defineRouterConfig()`: 미들웨어 배열의 튜플 타입 보존
- `GlobalMiddlewareContext`: `ExecuteArgs`의 기본 컨텍스트 타입 확장
- `GlobalRouteOptions`: `IOptions` 확장. `InferMiddlewareRouteOptions`로 자동 합산
- 여러 미들웨어 등록 시 옵션과 컨텍스트가 자동으로 합쳐짐

## JWT 미들웨어 전체 구현 예시

```typescript
// src/middleware/jwtMiddleware.ts
import { getConfig } from '@asapjs/common';
import type { TypedMiddleware } from '@asapjs/types';
import type { NextFunction, Request, Response } from 'express';
import jwt from 'jsonwebtoken';

export interface JwtUserPayload {
  [key: string]: any;
}

export const jwtMiddleware: TypedMiddleware<{ auth?: boolean }, { user: JwtUserPayload }> =
  ({ auth = false } = {}) =>
  (req: Request, res: Response, next: NextFunction) => {
    const { authorization } = req.headers;

    if (!authorization) {
      if (auth === true) {
        return res.status(403).json({ error: true, message: 'NO Token Provided' });
      }
      return next();
    }

    const token = authorization.split(' ')[1];
    const secret = (getConfig() as any).auth.jwt_access_token_secret;

    jwt.verify(token, secret, (err: any, decoded: any) => {
      if (err && auth === true) {
        if (err.message === 'invalid signature') {
          return res.status(403).json({ error: true, message: 'invalid signature' });
        }
        return res.status(401).json({ error: true, message: 'Unauthorized' });
      }
      req.user = decoded;
      next();
    });
  };
```

## 컨트롤러에서 사용

```typescript
@Get('/me', {
  auth: true,  // ← GlobalRouteOptions에 의해 타입 체크됨
})
public getMe = async ({ user }: ExecuteArgs) => {
  // user는 JwtUserPayload 타입으로 자동 추론됨
  return { user };
};

@Post('/login', {
  auth: false,  // 인증 없이 접근 가능
})
public login = async ({ body }: ExecuteArgs) => {
  return { token: '...' };
};
```

## 커스텀 미들웨어 (라우트별)

```typescript
@Post('/admin/action', {
  auth: true,
  middleware: [requireAdminRole, rateLimiter],  // 라우트별 미들웨어
  body: AdminActionDto,
  response: ActionResultDto,
})
async adminAction({ body, user }: ExecuteArgs) {
  return await this.adminService.performAction(user, body);
}
```

## 동작 원리

TypedMiddleware 팩토리는 **라우트 등록 시점**에 호출됩니다 (매 요청이 아님):

1. `config.router.middleware`에 등록된 팩토리 목록 확인
2. `registerRoutes()` 시 각 라우트의 `IOptions`를 모든 팩토리에 전달
3. 각 팩토리가 옵션에 맞는 실제 Express 미들웨어 반환
4. 반환된 미들웨어들이 라우트 핸들러 앞에 순서대로 삽입

## JWT 헬퍼 유틸리티

```typescript
// src/utils/jwt.ts
import jwt from 'jsonwebtoken';
import { getConfig } from '@asapjs/common';

export function jwtSign(payload: any, expiresIn: string = '7d'): string {
  const secret = (getConfig() as any).auth.jwt_access_token_secret;
  return jwt.sign(payload, secret, { expiresIn });
}
```
