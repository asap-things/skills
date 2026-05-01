# Repository & 페이지네이션

## Repository 클래스

`@asapjs/sequelize`의 `Repository`를 상속하면 `this.repository`를 통해 페이지네이션, DTO 변환이 내장된 쿼리 메서드를 사용할 수 있습니다.

```typescript
import { Repository } from '@asapjs/sequelize';
import type { PaginationQueryType } from '@asapjs/sequelize';
```

### 기본 구조

```typescript
// src/{domain}/infra/{Domain}TableRepository.ts
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

  public info = async (userId: number) => {
    const user = await this.repository.findOne(this.users, {
      exportTo: UserDto,
      where: { id: userId },
    });
    if (!user) return null;
    return new UserDto().map(user);
  };
}
```

## IArgs — 쿼리 옵션

```typescript
interface IArgs<T extends Model, D extends ExtendableDto> extends FindOptions<T> {
  exportTo: new (...args: any[]) => D;  // 필수: DTO 클래스
  user?: any;                             // 선택: DTO middleware()에 전달
  paging?: { page: number; limit: number }; // 선택: 페이지네이션
}
```

| 필드 | 필수 | 설명 |
|------|------|------|
| `exportTo` | ✅ | DTO 클래스. 쿼리 옵션 + 반환 타입 추론 |
| `user` | ❌ | 인증된 사용자 객체. DTO의 `middleware()`에 전달 |
| `paging` | ❌ | 제공 시 `findAndCountAll` 활성화 |

표준 Sequelize `FindOptions` (`where`, `include`, `order`, `attributes` 등)도 모두 사용 가능.

## findAll()

### paging 없을 때

```typescript
const users = await this.repository.findAll(UsersTable, {
  exportTo: UserDto,
  where: { is_active: true },
  order: [['created_at', 'DESC']],
});
// 반환: Model[] (Sequelize 모델 배열)
```

### paging 있을 때

```typescript
const users = await this.repository.findAll(UsersTable, {
  exportTo: UserDto,
  user,
  paging,  // { page: 0, limit: 20 }
});
// 반환: PagingResponse<DtoData<UserDto>>
```

## findOne()

```typescript
const user = await this.repository.findOne(UsersTable, {
  exportTo: UserDto,
  where: { id: userId },
});
// 반환: DtoData<UserDto> | null
```

## PagingResponse 구조

```typescript
{
  data: DtoData<D>[];     // 현재 페이지 항목 배열
  page: number;           // 현재 페이지 (0-based)
  page_size: number;      // 페이지당 항목 수
  max_page: number;       // 마지막 페이지 인덱스: Math.ceil(total / limit) - 1
  has_prev: boolean;      // page > 0이면 true
  has_next: boolean;      // max_page > page이면 true
  total_elements: number; // 전체 레코드 수
}
```

⚠️ `page`는 **0-based**. 첫 페이지 = `page: 0`.

## DtoData 타입

`DtoData<D>`는 DTO 클래스에서 메서드(init, map, pagingMap 등)를 제외한 **데이터 필드만** 포함하는 타입입니다.

```typescript
// UserDto가 { id, email, name, init(), map(), pagingMap() }이면
// DtoData<UserDto>는 { id, email, name }만 포함
```

## PaginationQueryDto & PaginationQueryType

### PaginationQueryDto — 내장 DTO

```typescript
// @asapjs/sequelize 제공
export default class PaginationQueryDto extends ExtendableDto {
  @TypeIs.INT({ comment: '페이지' })
  page: number;

  @TypeIs.INT({ comment: '한 페이지당 표시 개수' })
  limit: number;
}
```

### PaginationQueryType — 타입 별칭

```typescript
export type PaginationQueryType = Pick<PaginationQueryDto, 'page' | 'limit'>;
```

`ExecuteArgs.paging`의 타입. Application/Repository 레이어에서 paging 인자 타입으로 사용.

### Wrapper 자동 파싱

모든 요청에서 `?page=` & `?limit=`을 자동 파싱:

```typescript
// Wrapper 내부
const { page: pageProp = 0, limit: limitProp = 20 } = req.query;
const page = parseInt(String(pageProp), 10);
const limit = parseInt(String(limitProp), 10);
const paging: PaginationQueryType = { page, limit };
```

- 기본값: `page: 0`, `limit: 20`
- `query` 옵션에 PaginationQueryDto를 지정하지 않아도 `paging`은 항상 사용 가능

### PaginationQueryDto 확장

추가 필터가 필요하면 상속:

```typescript
@Dto({ name: 'get_user_list_query_dto', defineTable: UsersTable })
export default class GetUserListQueryDto extends PaginationQueryDto {
  @TypeIs.ENUM({ values: Object.keys(UserTypeEnum), comment: '유저 유형' })
  type: UserTypeEnum;

  @TypeIs.STRING({ comment: '검색어' })
  search: string;
}
```

## 전체 흐름 예시

### Controller

```typescript
@Get('/', {
  title: '사용자 목록',
  query: GetUserListQueryDto,
  response: UserDto,
})
public getUserList = async ({ paging, user }: ExecuteArgs<{}, GetUserListQueryDto, {}>) => {
  const result = await this.userService.list(paging, user);
  return { result };
};
```

### Application

```typescript
public list = async (paging: PaginationQueryType, user: UserDto) => {
  const raws = await this.usersRepository.list(paging, user);
  return raws;
};
```

### Repository

```typescript
public list = async (paging: PaginationQueryType, user: UserDto) => {
  const users = await this.repository.findAll(this.users, {
    exportTo: UserDto,
    user,
    paging,
  });
  return new UserDto().pagingMap(users);
};
```

### Swagger 응답 스키마

```typescript
// 라우트 데코레이터에서
response: TypeIs.PAGING(UserDto)
// → Swagger: { data: UserDto[], page, page_size, max_page, has_prev, has_next, total_elements }
```

## Repository는 선택적

단순한 도메인에서는 Application이 Entity를 직접 호출할 수 있습니다:

```typescript
// Application에서 직접 Sequelize API 사용
export class UserApplication {
  private users = UsersTable;

  async create(body: CreateUserDto) {
    const data = new CreateUserDto().map(body);
    const raw = await this.users.create(data);
    return new UserDto().map(raw);
  }
}
```

복잡한 쿼리 로직이 있는 경우에만 Repository로 분리 권장.
