# Entity & DTO 패턴

## Entity — @Table + TypeIs

### 기본 구조

```typescript
// src/{domain}/domain/entity/{Domain}sTable.ts
import { Model } from 'sequelize-typescript';
import { Table } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';

@Table({ tableName: 'users', timestamps: true })
export default class UsersTable extends Model {
  @TypeIs.INT({ primaryKey: true, autoIncrement: true, comment: '사용자 ID' })
  id!: number;

  @TypeIs.STRING({ unique: true, comment: '이메일' })
  email!: string;

  @TypeIs.PASSWORD({ comment: '비밀번호' })
  password!: string;

  @TypeIs.STRING({ comment: '이름' })
  name!: string;

  @TypeIs.BOOLEAN({ comment: '활성 상태', defaultValue: true })
  is_active!: boolean;

  @TypeIs.DATETIME({ comment: '생성일' })
  created_at!: Date;

  @TypeIs.DATETIME({ comment: '수정일' })
  updated_at!: Date;
}
```

> ⚠️ Entity 필드에는 `!` (definite assignment assertion)을 사용합니다. Sequelize가 런타임에 값을 할당하므로 TypeScript의 `strictPropertyInitialization` 경고를 방지합니다.

### @Table 옵션

| 옵션 | 타입 | 설명 |
|------|------|------|
| `tableName` | `string` | 실제 DB 테이블 이름 |
| `timestamps` | `boolean` | `true`이면 `created_at`/`updated_at` 자동 매핑 |

내부적으로 charset `utf8mb4`, collation `utf8mb4_general_ci` 자동 설정.

```typescript
// timestamps 사용 (created_at, updated_at 자동 매핑)
@Table({ tableName: 'users', timestamps: true })

// timestamps 미사용 (created_at, updated_at 수동 관리 또는 미사용)
@Table({ tableName: 'users' })
```

### 관계 정의

```typescript
import UsersTable from '../../../user/domain/entity/UsersTable';

@Table({ tableName: 'posts', timestamps: true })
export default class PostsTable extends Model {
  @TypeIs.INT({ primaryKey: true, autoIncrement: true, comment: '게시글 ID' })
  id!: number;

  @TypeIs.STRING({ comment: '제목' })
  title!: string;

  @TypeIs.TEXT({ comment: '내용' })
  content!: string;

  // 외래 키 — 반드시 함수로 감싸기 (순환 참조 방지)
  @TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
  user_id!: number;

  // 관계 객체 — FOREIGNKEY와 항상 쌍으로
  @TypeIs.BELONGSTO(() => UsersTable, 'user_id')
  user!: UsersTable;

  @TypeIs.DATETIME({ comment: '생성일' })
  created_at!: Date;
}
```

### ENUM 사용

```typescript
export enum UserTypeEnum {
  ADMIN = 'ADMIN',
  USER = 'USER',
}

@TypeIs.ENUM({
  values: Object.keys(UserTypeEnum),  // ['ADMIN', 'USER']
  comment: '유저 유형',
  defaultValue: UserTypeEnum.USER,
})
type: UserTypeEnum;
```

### 파일명 규칙

반드시 `*Table.ts`로 끝나야 자동 탐색됨:
- ✅ `UsersTable.ts`, `PostsTable.ts`, `CommentsTable.ts`
- ❌ `UserModel.ts`, `user.entity.ts`, `Users.ts`

## DTO — @Dto + ExtendableDto

### ⚠️ 핵심 규칙

1. **한 파일에 하나의 DTO만 정의합니다.** `*Dto.ts` 파일명으로 자동 탐색되므로, 하나의 파일에 여러 DTO를 넣으면 Swagger 스키마 등록이 누락될 수 있습니다.

```
✅ 올바른 구조:
  dto/
    UserDto.ts          — export default class UserDto
    CreateUserDto.ts    — export default class CreateUserDto
    UpdateUserDto.ts    — export default class UpdateUserDto

❌ 잘못된 구조:
  dto/
    UserDtos.ts         — UserDto, CreateUserDto, UpdateUserDto 모두 포함
```

2. **`TypeIs.OBJECT`는 존재하지 않습니다.** 중첩 객체가 필요하면 반드시 별도 `*Dto.ts` 파일에 DTO 클래스를 만들고 `TypeIs.DTO()`로 참조하세요.

```typescript
// ❌ TypeIs.OBJECT는 없음 — 런타임 에러
@TypeIs.OBJECT({ comment: '주소' })
address: { city: string; zipCode: string };

// ✅ 별도 DTO 파일 생성 후 TypeIs.DTO로 참조
// dto/AddressDto.ts
@Dto({ name: 'address_dto', defineTable: UsersTable })
export default class AddressDto extends ExtendableDto {
  @TypeIs.STRING({ comment: '도시' })
  city: string;

  @TypeIs.STRING({ comment: '우편번호' })
  zipCode: string;
}

// dto/UserDto.ts
@TypeIs.DTO({ dto: AddressDto, as: 'address' })
address: AddressDto;
```

### 기본 구조

```typescript
// src/{domain}/dto/{Name}Dto.ts
import { ExtendableDto, Dto } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';
import UsersTable from '../domain/entity/UsersTable';

@Dto({ name: 'user_dto', defineTable: UsersTable })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: '아이디' })
  id: number;

  @TypeIs.STRING({ comment: '이메일' })
  email: string;

  @TypeIs.STRING({ comment: '이름' })
  name: string;

  @TypeIs.DATETIME({ comment: '생성일' })
  created_at: Date;
}
```

### @Dto 옵션

| 옵션 | 타입 | 설명 |
|------|------|------|
| `name` | `string` | Swagger 스키마 이름 |
| `defineTable` | `typeof Model` | 연관 Entity 테이블 |
| `timestamps` | `boolean` | `false`면 created_at/updated_at 제외 (기본: `true`) |

### 요청 DTO vs 응답 DTO

**요청 DTO** — 클라이언트가 보내는 필드만:

```typescript
@Dto({ name: 'create_user_dto', defineTable: UsersTable, timestamps: false })
export default class CreateUserDto extends ExtendableDto {
  @TypeIs.STRING({ comment: '이메일' })
  email: string;

  @TypeIs.PASSWORD({ comment: '비밀번호' })
  password: string;

  @TypeIs.STRING({ comment: '이름' })
  name: string;
}
```

**응답 DTO** — 클라이언트에 반환할 필드만 (password 제외):

```typescript
@Dto({ name: 'user_dto', defineTable: UsersTable })
export default class UserDto extends ExtendableDto {
  @TypeIs.INT({ comment: '아이디' })
  id: number;

  @TypeIs.STRING({ comment: '이메일' })
  email: string;

  @TypeIs.STRING({ comment: '이름' })
  name: string;
}
```

### 쿼리 DTO — PaginationQueryDto 확장

```typescript
import { Dto, PaginationQueryDto } from '@asapjs/sequelize';
import { TypeIs } from '@asapjs/schema';

@Dto({ name: 'get_user_list_query_dto', defineTable: UsersTable })
export default class GetUserListQueryDto extends PaginationQueryDto {
  @TypeIs.ENUM({ values: Object.keys(UserTypeEnum), comment: '유저 유형' })
  type: UserTypeEnum;

  @TypeIs.STRING({ comment: '검색어' })
  search: string;
}
```

### 중첩 DTO

```typescript
@Dto({ name: 'post_info_dto', defineTable: PostsTable })
export default class PostInfoDto extends ExtendableDto {
  @TypeIs.INT({ comment: '게시글 ID' })
  id: number;

  @TypeIs.STRING({ comment: '제목' })
  title: string;

  // 중첩 DTO — 작성자 정보 포함
  @TypeIs.DTO({ dto: UserInfoDto, as: 'author' })
  author: UserInfoDto;
}
```

### 파일명 규칙

반드시 `*Dto.ts`로 끝나야 자동 탐색됨:
- ✅ `UserDto.ts`, `CreateUserDto.ts`, `PostInfoDto.ts`
- ❌ `UserResponse.ts`, `user.dto.ts`, `UserSchema.ts`

## ExtendableDto 핵심 메서드

### map(data) — 데이터 변환

```typescript
const dto = new UserDto();
const result = dto.map(rawSequelizeData);
// → DTO에 선언된 필드만 추출 + fixValue 적용
// 반환 타입: DtoData<UserDto> (메서드 제외, 데이터 필드만)
```

- Sequelize Model이면 `dataValues` 자동 추출
- 배열이면 각 요소에 재귀 적용
- 중첩 DTO(`TypeIs.DTO`)는 재귀적으로 `map()` 호출

### pagingMap(data) — 페이지네이션 데이터 변환

```typescript
const dto = new UserDto();
const result = dto.pagingMap(pagingResponse);
// → data 배열의 각 항목에 map() 적용, 메타데이터 유지
// 반환 타입: PagingResponse<DtoData<UserDto>>
```

### middleware() — Sequelize 쿼리 설정 생성

```typescript
const dto = new UserDto();
const queryOptions = dto.middleware();
// → { model: UsersTable, attributes: ['id', 'email', 'name', ...], include: [...] }
```

- 일반 필드 → `attributes` 배열
- `TypeIs.QUERY` → `literal()` SQL로 `attributes`에 추가
- `TypeIs.DTO` → `include` 배열에 중첩 쿼리 추가 (재귀)

## DTO와 라우트 데코레이터 연결

```typescript
@Post('/', {
  body: CreateUserDto,       // → Swagger requestBody + 런타임 타입 변환
  response: UserDto,         // → Swagger 200 response
  errors: [UserErrors.EMAIL_DUPLICATE],  // → Swagger 에러 응답
})
```

DTO가 `body`/`query`/`response`에 전달되면:
1. `generateScheme()` → OpenAPI 스키마 생성
2. Swagger에 `$ref`로 참조
3. `TypeIs.PAGING(Dto)` → 페이지네이션 래퍼 스키마 자동 등록
