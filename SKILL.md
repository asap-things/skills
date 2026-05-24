---
name: asapjs
description: >
  ASAPJS TypeScript backend framework skill. Covers Controller/Application/Repository/Entity
  layered architecture, TypeIs decorator-based schema definitions, DTO system, auto Swagger
  generation, TypedMiddleware, error() factory, Repository pagination, and Socket.IO integration.
  Supports both Express and Fastify adapters.
license: MIT
metadata:
  author: smlee
  version: "1.0.0"
  asapjs-version: "1.0.0-alpha.33"
  domain: backend
  triggers: >
    asapjs, @asapjs/core, @asapjs/router, @asapjs/schema, @asapjs/sequelize,
    @asapjs/error, @asapjs/fastify, @asapjs/socket, @asapjs/common, @asapjs/cli,
    @asapjs/types, RouterController, FastifyRouterController, TypeIs, ExtendableDto,
    ExecuteArgs, defineMiddleware, extendTypeIs, BaseSchemaType,
    asapjs new, asapjs dev, asapjs build, asapjs start, asapjs generate
  role: specialist
  scope: implementation
  output-format: code
  related-skills: typescript, sequelize, express, fastify
---

# ASAPJS Framework

"As Simple As Possible" — a TypeScript web framework that eliminates boilerplate.
One definition generates DB schema + Swagger docs + runtime coercion simultaneously.

## Absolute Rules (violations cause runtime errors)

1. **File naming is mandatory for auto-discovery:**
   - Entity: `*Table.ts` (e.g., `UsersTable.ts`)
   - DTO: `*Dto.ts` (e.g., `UserDto.ts`)
   - Socket: `*Socket.ts` (e.g., `ChatSocket.ts`)
   - Violation → file not discovered, model/schema not registered

2. **`this.registerRoutes()` must be called in Controller constructor:**
   ```typescript
   constructor() {
     super();
     this.registerRoutes();  // BEFORE service instantiation
     this.userService = new UserApplication();
   }
   ```
   Omission → all routes return 404

3. **TypeIs must be imported from `@asapjs/schema`:**
   ```typescript
   import { TypeIs } from '@asapjs/schema';  // ✅ Always
   ```

4. **Entity = `Model` from `sequelize-typescript` + `@Table` from `@asapjs/sequelize`:**
   ```typescript
   import { Model } from 'sequelize-typescript';
   import { Table } from '@asapjs/sequelize';
   ```

5. **DTO = `ExtendableDto` + `@Dto` decorator from `@asapjs/sequelize`:**
   ```typescript
   import { ExtendableDto, Dto } from '@asapjs/sequelize';
   ```

6. **Layer boundaries are strict:**
   - Controller: NO direct DB access, NO business logic
   - Application: NO Express/Fastify imports (`req`, `res`, `next`)
   - Entity: NOT imported by Controller (only Application/Repository)

7. **FOREIGNKEY table must be wrapped in a function (lazy evaluation):**
   ```typescript
   @TypeIs.FOREIGNKEY({ table: () => UsersTable })  // ✅
   @TypeIs.FOREIGNKEY({ table: UsersTable })         // ❌ circular ref risk
   ```

8. **`reflect-metadata`는 `@asapjs/sequelize` 사용 시 자동 import됨:**
   ```typescript
   // @asapjs/sequelize가 내부적으로 import 'reflect-metadata'를 수행
   // 따라서 사용자가 직접 import하지 않아도 동작하지만,
   // sequelize 없이 데코레이터를 사용하는 경우 직접 import 필요:
   import 'reflect-metadata';  // sequelize 미사용 시에만 필수
   ```

## Import Map

| Purpose | Import From |
|---------|-------------|
| Application class, logger | `@asapjs/core` |
| Get, Post, Put, Delete, RouterController, ExecuteArgs | `@asapjs/router` |
| defineMiddleware, defineRouterConfig, defineConfig, HttpException | `@asapjs/router` |
| CreateExecuteArgs, GlobalMiddlewareContext, GlobalRouteOptions, IOptions | `@asapjs/router` |
| AutoInject, ResponsePayload, ErrorPayload | `@asapjs/router` |
| addPaths, addScheme, getSwaggerData, generateSchemeRefWithName | `@asapjs/router` |
| FastifyRouterController, FastifyApplication, FastifyExecuteArgs | `@asapjs/fastify` |
| TypeIs (all types), extendTypeIs, TypeIsInterface | `@asapjs/schema` |
| validateDataWithSchema, validateFieldType, mapDataWithSchema | `@asapjs/schema` |
| BaseSchemaType, registry, SCHEMA_TYPES_METADATA_KEY | `@asapjs/schema` |
| swaggerPlugin, sequelizePlugin | `@asapjs/schema` |
| Table, ExtendableDto, Dto, Repository, PaginationQueryDto | `@asapjs/sequelize` |
| PaginationQueryType, PagingResponse, DtoData, DtoOrTypeIs | `@asapjs/sequelize` |
| modelsSync, getSequelize, healthCheck, generateDBML | `@asapjs/sequelize` |
| getDBMLData, getConsoleData, getUserIdInQuery | `@asapjs/sequelize` |
| registerSequelizeTypes, getData, TypeIsData | `@asapjs/sequelize` |
| error() factory, createErrorFactory() | `@asapjs/error` |
| getConfig, logger | `@asapjs/common` |
| createSocket, socketSendTo, socketSendAll, getSocketIO | `@asapjs/socket` |
| TypedMiddleware, InferMiddlewareRouteOptions | `@asapjs/types` / `@asapjs/router` |

> Note: `logger` is available from both `@asapjs/core` and `@asapjs/common`. Either works.

## Reference Guide

Load the appropriate reference file based on the task at hand:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Architecture | `references/architecture.md` | Adding new domain, understanding layer responsibilities, request flow |
| TypeIs Types | `references/typeis-reference.md` | Defining fields/columns, choosing type decorators, type options, schema plugin system |
| Controllers | `references/controller-patterns.md` | Writing route handlers, IOptions, ExecuteArgs generics |
| Entity & DTO | `references/dto-entity-patterns.md` | Defining tables, DTOs, @Table/@Dto decorators, map/pagingMap |
| Repository | `references/repository-pagination.md` | Pagination, findAll/findOne, PagingResponse, DtoData |
| Middleware | `references/middleware-auth.md` | TypedMiddleware, defineMiddleware, JWT auth, module augmentation |
| Errors | `references/error-handling.md` | error() factory, HttpError, HttpException, Swagger error schemas |
| Sockets | `references/socket-patterns.md` | createSocket, socketSendTo, socketSendAll, Redis adapter |
| Config | `references/config-bootstrap.md` | Application setup, extensions, DB config, initialization flow |
| Fastify | `references/fastify-patterns.md` | FastifyApplication, FastifyRouterController, Fastify adapter |
| CLI | `references/cli-reference.md` | asapjs CLI commands, project scaffolding, code generation, build/deploy |
| Gotchas | `references/gotchas.md` | Debugging, common mistakes, anti-patterns checklist |
| Response DTO | (inline in SKILL.md) | Global Response DTO, @AutoInject, @ResponsePayload, config.response |

## Project Structure Template

```
src/
  common/
    GlobalResponseDto.ts                # Project-wide response envelope (optional)
  {domain}/
    controller/{Domain}Controller.ts    # RouterController + @Get/@Post
    application/{Domain}Application.ts  # Pure TypeScript business logic
    domain/entity/{Domain}sTable.ts     # Model + @Table + TypeIs.*
    dto/
      Create{Domain}Dto.ts              # Request body shape
      {Domain}Dto.ts                    # Response shape
    errors/{Domain}Errors.ts            # error() factory definitions
    infra/{Domain}TableRepository.ts    # Repository pattern (optional)
  index.ts                              # Entry point: Application(__dirname, config).run()
  config.ts                             # Router config + module augmentation
  route.ts                              # Controller registry: export default [...]
```

## Key Patterns (inline for quick reference)

### Minimal Controller

```typescript
import { RouterController, Get, ExecuteArgs } from '@asapjs/router';
import { PaginationQueryDto } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';
import { UserApplication } from '../application/UserApplication';
import UserDto from '../dto/UserDto';

export default class UserController extends RouterController {
  public basePath = '/users';
  public tag = 'User';
  private svc = new UserApplication();

  constructor() { super(); this.registerRoutes(); }

  @Get('/', { title: 'List users', query: PaginationQueryDto, response: TypeIs.PAGING(UserDto) })
  public list = async ({ paging }: ExecuteArgs) => await this.svc.list(paging);
}
```

### Minimal Entity

```typescript
import { Model } from 'sequelize-typescript';
import { Table } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';

@Table({ tableName: 'users', timestamps: true })
export default class UsersTable extends Model {
  @TypeIs.INT({ primaryKey: true, autoIncrement: true, comment: 'ID' })
  id: number;

  @TypeIs.STRING({ unique: true, comment: 'Email' })
  email: string;
}
```

### Minimal DTO

```typescript
import { ExtendableDto, Dto } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';
import UsersTable from '../domain/entity/UsersTable';

@Dto({ name: 'user_dto', defineTable: UsersTable })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: 'ID' })
  id: number;

  @TypeIs.STRING({ comment: 'Email' })
  email: string;
}
```

### Minimal Error Definition

```typescript
import { error } from '@asapjs/error';
import { TypeIs } from '@asapjs/schema';

export class UserErrors {
  static NOT_FOUND = error(404, 'USER_NOT_FOUND', 'User {userId} not found', {
    userId: TypeIs.INT(),
  });
}

// Usage: throw UserErrors.NOT_FOUND({ userId: 42 });
```

### Entry Point

```typescript
import { Application } from '@asapjs/core';
import config from './config';

new Application(__dirname, config).run();
```

### Global Response DTO (Express only)

Wraps all route responses with a common envelope (timestamp, requestId, etc.) automatically.

```typescript
// src/common/GlobalResponseDto.ts
import { TypeIs } from '@asapjs/schema';
import { ExtendableDto } from '@asapjs/sequelize';
import { AutoInject, ResponsePayload } from '@asapjs/router';

export default class GlobalResponseDto extends ExtendableDto {
  @TypeIs.INT({ comment: '응답 시각' })
  @AutoInject(() => Date.now())
  timestamp: number;

  @TypeIs.STRING({ comment: '요청 ID' })
  @AutoInject((req) => req.headers['x-request-id'] || crypto.randomUUID())
  requestId: string;

  @TypeIs.BOOLEAN({ comment: '성공 여부' })
  @AutoInject(() => true)
  success: boolean;

  @ResponsePayload()  // route DTO goes here
  result: any;
}
```

```typescript
// config.ts — wire it
export default {
  response: { responseDto: GlobalResponseDto },
  // ...
};
```

```typescript
// Controller — return raw payload (NOT { result: data })
@Get('/:id', { response: UserDto })
public getUser = async ({ path }) => {
  return await this.userService.info(path.id);
};
// Response: { timestamp: 1716..., requestId: "uuid", success: true, result: { id, name, ... } }
```

Modes:
- **Designated**: `@ResponsePayload()` present → output wrapped in that key
- **Spread**: no `@ResponsePayload()` → output spread with auto-inject fields (Swagger uses `allOf`)
- **Override**: `public responseDto = CustomDto` on controller
- **Disable**: `public responseDto = null` on controller → raw output, no envelope

### Error Envelope (@ErrorPayload)

Add `@ErrorPayload()` to Global DTO to wrap error responses in the same envelope:

```typescript
import { AutoInject, ErrorPayload, ResponsePayload } from '@asapjs/router';

export default class GlobalResponseDto extends ExtendableDto {
  @AutoInject(() => Date.now()) timestamp: number;
  @AutoInject((req) => req.headers['x-request-id'] || crypto.randomUUID()) requestId: string;
  @AutoInject(() => true) success: boolean;

  @ResponsePayload() result: any;

  @TypeIs.JSON({ comment: '에러 정보', optional: true })
  @ErrorPayload()
  error: any;
}
// Error response: { timestamp, requestId, success: false, error: { status, errorCode, message, data } }
// Without @ErrorPayload(): error returned raw { status, errorCode, message, data }
```

### Custom Error Factory (createErrorFactory)

Create error factories with custom serialization and DTO for Swagger:

```typescript
import { createErrorFactory } from '@asapjs/error';

class ErrorResponseDto extends ExtendableDto {
  @TypeIs.INT({ comment: 'HTTP 상태 코드' }) status: number;
  @TypeIs.STRING({ comment: '에러 코드' }) errorCode: string;
  @TypeIs.STRING({ comment: '에러 메시지' }) message: string;
  @TypeIs.JSON({ comment: '에러 상세', optional: true }) data: any;
}

const error = createErrorFactory({
  dto: ErrorResponseDto,
  serialize: ({ status, code, message, data }) => ({
    status,
    errorCode: code,
    message,
    data,
  }),
});

// Usage (same as built-in error() factory):
export class UserErrors {
  static NOT_FOUND = error(404, 'USER_NOT_FOUND', 'User {userId} not found', {
    userId: TypeIs.INT(),
  });
}
// throw UserErrors.NOT_FOUND({ userId: 42 })
```

## Do NOT (common AI hallucinations)

- Do NOT use NestJS patterns (`@Controller`, `@Injectable`, `@Module`) — they don't exist
- Do NOT import `DataTypes` from Sequelize — use `TypeIs` from `@asapjs/schema`
- Do NOT use `@Column` decorator — use `TypeIs.*` decorators
- Do NOT create DI containers — instantiate services directly in constructors
- Do NOT call `res.json()` in handlers — return a value and Wrapper handles it
- Do NOT wrap return values in `{ result: data }` when Global Response DTO is configured — return raw payload directly
- Do NOT assume `page` is 1-based — it's 0-based (first page = 0)
- Do NOT throw plain `new Error()` for client errors — use `error()` factory or `HttpException`
- Do NOT mix Express and Fastify imports in the same controller
- Do NOT trust `asapjs generate` templates blindly — they have known bugs (see `references/cli-reference.md`)
- Do NOT import `TypeIs` from `@asapjs/sequelize` — always use `@asapjs/schema`
- Do NOT use `TypeIs.OBJECT` — it does not exist. For nested objects, create a separate `*Dto.ts` file with `@Dto` decorator and use `TypeIs.DTO({ dto: MyDto })` instead
- Do NOT put multiple DTOs in one file — each DTO must be in its own `*Dto.ts` file for auto-discovery to work
