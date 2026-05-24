# Controller 패턴

## Express — RouterController

```typescript
import { RouterController, Get, Post, Put, Delete, ExecuteArgs } from '@asapjs/router';
```

### 기본 구조

```typescript
export default class UserController extends RouterController {
  public basePath = '/users';   // 라우트 접두사: /api/users
  public tag = 'users';         // Swagger 태그 그룹

  private userService: UserApplication;

  constructor() {
    super();
    this.registerRoutes();      // ⚠️ 필수! 누락 시 라우트 미등록
    this.userService = new UserApplication();
  }

  @Get('/', { title: '목록 조회', query: GetUserListQueryDto, response: UserDto })
  public getUserList = async ({ paging, user }: ExecuteArgs<{}, GetUserListQueryDto, {}>) => {
    const result = await this.userService.list(paging, user);
    return { result };
  };
}
```

> **Global Response DTO 사용 시**: `config.response.responseDto`가 설정된 프로젝트에서는
> `return { result }` 대신 raw payload를 직접 반환해야 합니다.
> Wrapper가 자동으로 envelope(`timestamp`, `requestId` 등)을 씌워줍니다.
> ```typescript
> // Global Response DTO 적용 시:
> public getUserList = async ({ paging, user }) => {
>   return await this.userService.list(paging, user);  // raw payload
> };
> ```
> 컨트롤러 단위로 envelope을 비활성화하려면 `public responseDto = null;`을 선언합니다.

### registerRoutes() 호출 순서

```typescript
constructor() {
  super();                    // 1. 부모 생성자
  this.registerRoutes();      // 2. 라우트 등록 (서비스 인스턴스화 전에!)
  this.userService = new UserApplication();  // 3. 서비스 인스턴스화
}
```

## Fastify — FastifyRouterController

```typescript
import { FastifyRouterController, Get, Post, Put, Delete } from '@asapjs/fastify';
import type { FastifyExecuteArgs } from '@asapjs/fastify';
```

데코레이터와 핸들러 구조는 Express와 동일. 차이점:
- `FastifyRouterController` 상속
- `registerRoutes()` 대신 `registerFastifyRoutes()`가 플러그인에 의해 자동 호출
- `FastifyExecuteArgs` 사용 (Proxy로 request 속성 접근)
- Swagger 자동 생성 미지원 (현재)

## IOptions — 데코레이터 옵션

| 옵션 | 타입 | 설명 |
|------|------|------|
| `title` | `string` | Swagger operation 제목 |
| `description` | `string` | Swagger 상세 설명 |
| `summary` | `string` | Swagger operation 요약 |
| `auth` | `boolean` | JWT 인증 미들웨어 적용 (기본: `false`) |
| `body` | `DtoOrTypeIs` | 요청 본문 스키마 |
| `query` | `DtoOrTypeIs` | 쿼리 파라미터 스키마 |
| `response` | `DtoOrTypeIs` | 응답 스키마 |
| `errors` | `ErrorCreator[]` | Swagger 에러 응답 자동 등록 |
| `middleware` | `any[]` | 커스텀 Express 미들웨어 배열 |
| `bodyContentType` | `string` | `'application/json'` 또는 `'multipart/form-data'` |
| `deprecated` | `boolean` | Swagger deprecated 표시 |

## ExecuteArgs 제네릭

```typescript
type ExecuteArgs<P = {}, Q = {}, B = {}, Context = GlobalMiddlewareContext> = {
  req: Request;
  res: Response;
  path?: P;
  query: Q & { [key: string]: any };
  body: B & { [key: string]: any };
  files?: { [key: string]: any };
  paging: PaginationQueryType;
} & Context;
```

### 제네릭 파라미터

| 위치 | 의미 | 예시 |
|------|------|------|
| `P` | path 파라미터 타입 | `{ userId: string }` |
| `Q` | query 파라미터 타입 | `GetUserListQueryDto` |
| `B` | body 타입 | `CreateUserDto` |

```typescript
// path 파라미터 사용
@Get('/:userId', { response: UserDto })
public getUser = async ({ path }: ExecuteArgs<{ userId: string }, {}, {}>) => {
  const result = await this.userService.info(path?.userId);
  return { result };
};

// body 사용
@Post('/', { body: CreateUserDto, response: UserDto })
public createUser = async ({ body, user }: ExecuteArgs<{}, {}, CreateUserDto>) => {
  const result = await this.userService.create(body, user);
  return { result };
};
```

## ExecuteArgs 필드 출처

| 필드 | 출처 | 비고 |
|------|------|------|
| `path` | `req.params` | `/:id` → `{ id: '1' }` |
| `body` | `req.body` | JSON 파싱된 요청 본문 |
| `query` | `req.query` | URL 쿼리 파라미터 |
| `files` | `req.files` | 멀티파트 업로드 |
| `user` | `req.user` | TypedMiddleware가 설정 (GlobalMiddlewareContext) |
| `paging` | `req.query.page/limit` | 항상 존재. 기본값 `{ page: 0, limit: 20 }` |
| `req` | Express Request | 직접 사용 비권장 |
| `res` | Express Response | 직접 사용 비권장 |

## 핸들러 반환값 규칙

- 값 반환 → `Wrapper`가 `res.status(200).json(output)` 자동 호출
- `null`/`undefined` 반환 → 응답 미전송 (수동 제어 시 사용)
- throw → `Wrapper`가 `errorToResponse(err, res)` 호출

## CRUD 컨트롤러 전체 예시

```typescript
import { RouterController, Get, Post, Put, Delete, ExecuteArgs } from '@asapjs/router';
import { PaginationQueryDto } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';
import { PostApplication } from '../application/PostApplication';
import CreatePostDto from '../dto/CreatePostDto';
import UpdatePostDto from '../dto/UpdatePostDto';
import PostInfoDto from '../dto/PostInfoDto';
import { PostErrors } from '../errors/PostErrors';

export default class PostController extends RouterController {
  public tag = 'Post';
  public basePath = '/posts';
  private postService: PostApplication;

  constructor() {
    super();
    this.registerRoutes();
    this.postService = new PostApplication();
  }

  @Get('/', {
    title: '게시글 목록',
    query: PaginationQueryDto,
    response: TypeIs.PAGING(PostInfoDto),
  })
  public getPosts = async ({ paging }: ExecuteArgs) => {
    const result = await this.postService.getPosts(paging);
    return { result };
  };

  @Post('/', {
    title: '게시글 작성',
    auth: true,
    body: CreatePostDto,
    response: PostInfoDto,
  })
  public createPost = async ({ body, user }: ExecuteArgs<{}, {}, CreatePostDto>) => {
    const result = await this.postService.createPost(user, body);
    return { result };
  };

  @Get('/:postId', {
    title: '게시글 상세',
    response: PostInfoDto,
    errors: [PostErrors.NOT_FOUND],
  })
  public getPost = async ({ path }: ExecuteArgs<{ postId: string }>) => {
    const result = await this.postService.getPost(path?.postId);
    return { result };
  };

  @Put('/:postId', {
    title: '게시글 수정',
    auth: true,
    body: UpdatePostDto,
    response: PostInfoDto,
    errors: [PostErrors.NOT_FOUND],
  })
  public updatePost = async ({ path, body, user }: ExecuteArgs<{ postId: string }, {}, UpdatePostDto>) => {
    const result = await this.postService.updatePost(path?.postId, user, body);
    return { result };
  };

  @Delete('/:postId', {
    title: '게시글 삭제',
    auth: true,
    errors: [PostErrors.NOT_FOUND],
  })
  public deletePost = async ({ path, user }: ExecuteArgs<{ postId: string }>) => {
    await this.postService.deletePost(path?.postId, user);
    return { success: true };
  };
}
```

## route.ts 등록

```typescript
// src/route.ts
import UserController from './user/controller/UserController';
import PostController from './post/controller/PostController';

export default [new UserController(), new PostController()];
```
