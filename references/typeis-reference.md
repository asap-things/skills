# TypeIs 타입 시스템 레퍼런스

`TypeIs`는 ASAPJS의 핵심. 하나의 데코레이터가 Sequelize 컬럼 + Swagger 스키마 + 런타임 변환을 동시에 생성합니다.

## Import

```typescript
// 항상 @asapjs/schema에서 import
import { TypeIs } from '@asapjs/schema';

// ❌ 절대 하지 말 것
import { DataTypes } from 'sequelize';  // TypeIs를 사용하세요
```

## 기본 타입 (Primitives)

| TypeIs | Sequelize | Swagger | fixValue | 비고 |
|--------|-----------|---------|----------|------|
| `TypeIs.INT(opts)` | `INTEGER` | `integer (int32)` | `parseInt(String(o), 10)` | |
| `TypeIs.BIGINT(opts)` | `BIGINT` | `integer (int64)` | `parseInt(String(o), 10)` | |
| `TypeIs.LONG(opts)` | `BIGINT` | `integer (int64)` | `parseInt(String(o), 10)` | BIGINT 별칭 |
| `TypeIs.FLOAT(opts)` | `FLOAT` | `number (float)` | `parseFloat(String(o))` | |
| `TypeIs.DOUBLE(opts)` | `DOUBLE` | `number (double)` | `parseFloat(String(o))` | |
| `TypeIs.DECIMAL(opts)` | `DECIMAL` | `number` | `parseFloat(String(o))` | |
| `TypeIs.STRING(opts)` | `STRING` | `string` | `String(o)` | |
| `TypeIs.TEXT(opts)` | `TEXT` | `string` | `String(o)` | 장문 텍스트 |
| `TypeIs.PASSWORD(opts)` | `STRING(512)` | `string (password)` | `String(o)` | Swagger에서 마스킹 |
| `TypeIs.BOOLEAN(opts)` | `BOOLEAN` | `boolean` | `'true'/'false' → boolean` | |
| `TypeIs.DATEONLY(opts)` | `DATEONLY` | `string (date)` | 그대로 반환 | 날짜만 |
| `TypeIs.DATETIME(opts)` | `DATE` | `string (date-time)` | 그대로 반환 | 날짜+시간 |
| `TypeIs.ENUM(opts)` | `ENUM(values)` | `string (enum)` | `String(o)` | `values` 필수 |
| `TypeIs.JSON(opts)` | `JSON` | `object` | 그대로 반환 | |
| `TypeIs.BASE64(opts)` | `TEXT` | `string (byte)` | `String(o)` | |
| `TypeIs.BINARY(opts)` | `BLOB` | `string (binary)` | 그대로 반환 | |

## 공통 옵션

모든 TypeIs 타입에 적용 가능한 옵션:

| 옵션 | 타입 | 설명 |
|------|------|------|
| `comment` | `string` | Swagger description + DB 컬럼 코멘트 |
| `allowNull` | `boolean` | NULL 허용 여부 (기본: false) |
| `defaultValue` | `any` | 기본값 |
| `primaryKey` | `boolean` | 기본키 |
| `autoIncrement` | `boolean` | 자동 증가 |
| `unique` | `boolean` | 유니크 제약 |

## 관계 타입 (Entity 전용)

Entity(`*Table.ts`)에서만 사용. DTO에서 사용 불가.

### TypeIs.FOREIGNKEY

```typescript
@TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
user_id: number;
```

- `table`: 참조 테이블을 **함수로 감싸서** 전달 (순환 참조 방지)
- 기본 타입: `INTEGER`
- Swagger 출력 없음

### TypeIs.BELONGSTO

```typescript
@TypeIs.BELONGSTO(() => UsersTable, 'user_id')
user: UsersTable;
```

- 첫 번째 인자: 연관 테이블 반환 함수
- 두 번째 인자: 외래 키 컬럼명
- FOREIGNKEY와 항상 쌍으로 사용

```typescript
// 올바른 패턴: 항상 쌍으로
@TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
user_id: number;

@TypeIs.BELONGSTO(() => UsersTable, 'user_id')
user: UsersTable;
```

## 조합 타입 (DTO/라우터 전용)

DTO 또는 라우트 데코레이터 옵션에서만 사용. Entity에서 사용 불가.

### TypeIs.DTO — 중첩 객체

```typescript
@TypeIs.DTO({ dto: UserInfoDto, as: 'author' })
author: UserInfoDto;
```

- `map()` 시 중첩 DTO의 `map()` 재귀 호출
- `middleware()` 시 Sequelize `include`에 추가
- Swagger에서 `$ref`로 참조

### TypeIs.QUERY — 계산된 필드

```typescript
@TypeIs.QUERY({
  query: ({ association }) =>
    `(SELECT COUNT(*) FROM comments WHERE comments.post_id = ${association}.id)`,
  type: () => TypeIs.INT(),
})
commentCount: number;
```

- `middleware()` 시 `literal()` SQL로 `attributes`에 추가
- `query` 함수는 `{ association?, user? }` 객체를 받음
- `type`은 getter `() => TypeIs.INT()` 또는 직접 값

### TypeIs.ARRAY — 배열 래퍼

```typescript
// 라우트 데코레이터에서
response: TypeIs.ARRAY(PostInfoDto)
// Swagger: { type: 'array', items: { $ref: '#/components/schemas/PostInfoDto' } }
```

### TypeIs.PAGING — 페이지네이션 래퍼

```typescript
// 라우트 데코레이터에서
response: TypeIs.PAGING(PostInfoDto)
// Swagger: { data: PostInfoDto[], page, page_size, max_page, has_prev, has_next, total_elements }
```

## 사용 컨텍스트 요약

| 타입 | Entity | DTO | 라우트 옵션 |
|------|--------|-----|------------|
| 기본 타입 (INT, STRING 등) | ✅ | ✅ | ✅ (인라인) |
| FOREIGNKEY, BELONGSTO | ✅ | ❌ | ❌ |
| DTO, QUERY | ❌ | ✅ | ❌ |
| ARRAY, PAGING | ❌ | ❌ | ✅ |
| ENUM | ✅ | ✅ | ✅ |

## ENUM 사용 패턴

```typescript
// Entity에서
export enum UserTypeEnum {
  ADMIN = 'ADMIN',
  USER = 'USER',
}

@TypeIs.ENUM({ values: Object.keys(UserTypeEnum), comment: '유저 유형', defaultValue: UserTypeEnum.USER })
type: UserTypeEnum;
```

`values`에는 반드시 `Object.keys(Enum)` 사용. 직접 배열 리터럴도 가능하지만 enum과 동기화 유지를 위해 `Object.keys` 권장.

---

## 스키마 플러그인 시스템

TypeIs 타입은 플러그인을 통해 다양한 대상(Swagger, Sequelize 등)으로 변환됩니다. 플러그인은 `SchemaPlugin` 인터페이스를 구현하고, `registry`에 등록하여 사용합니다.

### 핵심 인터페이스

```typescript
import { SchemaType, SchemaOptions, SchemaPlugin } from '@asapjs/schema';

// 스키마 타입 인터페이스 — 모든 TypeIs 타입이 구현
interface SchemaType<T = any> {
  __name: string;
  __options: SchemaOptions;
  to(target: string): any;
  validate(value: any): boolean;
  parse(value: any): T;
  toSwagger?(): any;
  toSequelize?(): any;
}

// 스키마 옵션 — 모든 타입의 공통 옵션
interface SchemaOptions {
  comment?: string;
  optional?: boolean;
  default?: any;
  example?: any;
  [key: string]: any;  // 타입별 추가 옵션 허용
}

// 플러그인 인터페이스
interface SchemaPlugin {
  name: string;
  transform(type: SchemaType, options: SchemaOptions): any;
}
```

### BaseSchemaType 추상 클래스

모든 커스텀 타입의 기반 클래스. `to()`, `toSwagger()`, `toSequelize()` 편의 메서드를 제공합니다.

```typescript
import { BaseSchemaType, SchemaOptions } from '@asapjs/schema';

abstract class BaseSchemaType<T = any> implements SchemaType<T> {
  abstract __name: string;
  __options: SchemaOptions;

  constructor(options: SchemaOptions = {}) {
    this.__options = options;
  }

  // 등록된 플러그인으로 변환
  to(target: string): any {
    // registry에서 플러그인을 찾아 transform() 호출
  }

  abstract validate(value: any): boolean;
  abstract parse(value: any): T;

  // 편의 메서드
  toSwagger(): any {
    return this.to('swagger');
  }

  toSequelize(): any {
    return this.to('sequelize');
  }
}
```

### 플러그인 레지스트리 (registry)

`registry`는 싱글턴으로, 플러그인 등록/조회/삭제를 관리합니다.

```typescript
import { registry } from '@asapjs/schema';

// 플러그인 등록
registry.register(myPlugin);    // 이미 등록된 이름이면 Error throw

// 플러그인 조회
registry.get('swagger');         // SchemaPlugin | undefined
registry.has('swagger');         // boolean

// 등록된 플러그인 목록
registry.list();                 // string[] — 예: ['swagger', 'sequelize']

// 플러그인 해제
registry.unregister('swagger');
```

### 내장 플러그인

`@asapjs/schema`를 import하면 아래 두 플러그인이 자동 등록됩니다:

| 플러그인 | 이름 | 설명 |
|----------|------|------|
| `swaggerPlugin` | `'swagger'` | TypeIs → OpenAPI 스키마 객체 변환 |
| `sequelizePlugin` | `'sequelize'` | TypeIs → Sequelize 컬럼 정의 변환 (sequelize 미설치 시 무시) |

```typescript
import { swaggerPlugin, sequelizePlugin } from '@asapjs/schema';

// 이미 자동 등록되므로 수동 등록 불필요
// registry.register(swaggerPlugin);   — 자동 등록됨
// registry.register(sequelizePlugin); — 자동 등록됨 (sequelize 미설치 시 무시)
```

### to() 메서드 사용

모든 TypeIs 타입에서 `to(target)` 메서드로 플러그인 변환을 호출할 수 있습니다.

```typescript
import { TypeIs } from '@asapjs/schema';

const intType = TypeIs.INT({ comment: '사용자 ID', min: 1 });

// 플러그인 이름으로 변환
intType.to('swagger');
// → { type: 'integer', format: 'int32', description: '사용자 ID', minimum: 1 }

intType.to('sequelize');
// → { type: DataTypes.INTEGER, comment: '사용자 ID', allowNull: false }

// 편의 메서드 (동일한 결과)
intType.toSwagger();
intType.toSequelize();
```

### 커스텀 플러그인 작성 예시

```typescript
import { SchemaPlugin, SchemaType, SchemaOptions, registry } from '@asapjs/schema';

const jsonSchemaPlugin: SchemaPlugin = {
  name: 'json-schema',

  transform(type: SchemaType, options: SchemaOptions): Record<string, unknown> {
    switch (type.__name) {
      case 'int':
        return { type: 'integer' };
      case 'string':
        return { type: 'string', maxLength: options.maxLength };
      default:
        return { type: 'string' };
    }
  },
};

registry.register(jsonSchemaPlugin);

// 사용
TypeIs.STRING({ maxLength: 100 }).to('json-schema');
// → { type: 'string', maxLength: 100 }
```

---

## 커스텀 타입 확장 (extendTypeIs)

`extendTypeIs()`를 사용하면 `TypeIs` 객체에 새로운 타입 팩토리를 추가할 수 있습니다. 추가된 타입은 기존 타입과 동일하게 데코레이터로도 사용 가능합니다.

### 함수 시그니처

```typescript
import { extendTypeIs } from '@asapjs/schema';

// extensions 객체의 각 값이 함수이면, 반환값이 SchemaType일 때 자동으로 DecoratableSchemaType으로 래핑
function extendTypeIs<T extends Record<string, any>>(extensions: T): void;
```

### TypeIsInterface 모듈 확장

TypeScript에서 타입 안전성을 유지하려면 `declare module`로 `TypeIsInterface`를 확장해야 합니다.

```typescript
import { SchemaFactory, SchemaOptions, DecoratableSchemaType } from '@asapjs/schema';

// 커스텀 옵션 정의
interface PhoneOptions extends SchemaOptions {
  countryCode?: string;
}

// TypeIs 인터페이스 확장 선언
declare module '@asapjs/schema' {
  interface TypeIsInterface {
    PHONE: SchemaFactory<PhoneOptions>;
  }
}
```

### 전체 예시: 커스텀 타입 생성 및 등록

```typescript
import {
  BaseSchemaType,
  SchemaOptions,
  extendTypeIs,
} from '@asapjs/schema';

// 1. 옵션 인터페이스 정의
interface PhoneOptions extends SchemaOptions {
  countryCode?: string;
}

// 2. 타입 구현
class PhoneType extends BaseSchemaType<string> {
  __name = 'phone';

  validate(value: unknown): boolean {
    return typeof value === 'string' && /^\+?[\d\-\s]+$/.test(value);
  }

  parse(value: unknown): string {
    return String(value).replace(/[\s\-]/g, '');
  }
}

// 3. TypeIs 인터페이스 확장 (타입 안전성)
declare module '@asapjs/schema' {
  interface TypeIsInterface {
    PHONE: (options?: PhoneOptions) => import('@asapjs/schema').DecoratableSchemaType<string>;
  }
}

// 4. TypeIs에 등록
extendTypeIs({
  PHONE: (options?: PhoneOptions) => new PhoneType(options ?? {}),
});

// 5. 사용
import { TypeIs } from '@asapjs/schema';

const phoneField = TypeIs.PHONE({ comment: '연락처', countryCode: 'KR' });
phoneField.validate('+82-10-1234-5678'); // true
phoneField.parse('+82-10-1234-5678');    // '+821012345678'
phoneField.toSwagger();                  // swagger 플러그인 변환 결과
```

---

## 스키마 유효성 검사 유틸리티

`@asapjs/schema`는 스키마 기반 데이터 유효성 검사 및 변환 유틸리티를 제공합니다.

### validateDataWithSchema

스키마 기준으로 데이터 전체를 검사합니다. 필수 필드 누락이나 타입 위반 시 `Error`를 throw합니다.

```typescript
import { validateDataWithSchema, TypeIs, SchemaType } from '@asapjs/schema';

const schema: Record<string, SchemaType | (() => SchemaType)> = {
  name: TypeIs.STRING({ comment: '이름' }),
  age: TypeIs.INT({ comment: '나이' }),
  email: TypeIs.STRING({ optional: true, comment: '이메일' }),
};

// 정상 — 통과
validateDataWithSchema({ name: '홍길동', age: 30 }, schema);

// Error: Missing required field: name
validateDataWithSchema({ age: 30 }, schema);

// Error: Field 'age' has invalid type for int
validateDataWithSchema({ name: '홍길동', age: 'abc' }, schema);
```

- `optional: true`인 필드는 `undefined`여도 에러 없음
- 스키마 값이 함수(`() => SchemaType`)이면 호출하여 resolve

### validateFieldType

단일 필드의 타입/제약을 검사합니다. `schemaType.validate(value)`가 `false`이면 `Error`를 throw합니다.

```typescript
import { validateFieldType, TypeIs } from '@asapjs/schema';

const intType = TypeIs.INT({ comment: 'ID' });

// 정상 — 통과
validateFieldType('id', 42, intType);

// Error: Field 'id' has invalid type for int
validateFieldType('id', 'not-a-number', intType);
```

### mapDataWithSchema

스키마에 맞춰 데이터를 변환합니다 (유효성 검사 없음). 각 필드에 대해 `schemaType.parse(value)`를 호출합니다.

```typescript
import { mapDataWithSchema, TypeIs, SchemaType } from '@asapjs/schema';

const schema: Record<string, SchemaType | (() => SchemaType)> = {
  id: TypeIs.INT({ comment: 'ID' }),
  name: TypeIs.STRING({ comment: '이름' }),
  active: TypeIs.BOOLEAN({ comment: '활성 여부' }),
};

const raw = { id: '123', name: 456, active: 'true' };
const result = mapDataWithSchema(raw, schema);
// → { id: 123, name: '456', active: true }
```

- `parse()` 실패 시 원본 값을 그대로 유지 (에러 throw 안 함)
- `undefined` 값은 `undefined`로 유지

---

## Sequelize 전용 타입 확장

`@asapjs/sequelize` 패키지는 `extendTypeIs()`를 통해 Sequelize 전용 타입(`FOREIGNKEY`, `BELONGSTO`, `QUERY`, `PAGING`)을 `TypeIs`에 추가합니다. `registerSequelizeTypes()` 호출 시 등록됩니다.

### 모듈 확장 선언

`@asapjs/sequelize`는 아래와 같이 `TypeIsInterface`를 확장합니다:

```typescript
// @asapjs/sequelize 내부의 declare module
import { SchemaFactory, SchemaOptions, DecoratableSchemaType } from '@asapjs/schema';

declare module '@asapjs/schema' {
  interface TypeIsInterface {
    QUERY: SchemaFactory<QueryOptions>;
    FOREIGNKEY: (
      relatedClassGetterOrOptions: (() => any) | ForeignKeyOptions,
      extraData?: Record<string, any>
    ) => DecoratableSchemaType<number | string>;
    BELONGSTO: (
      associatedClassGetterOrOptions: (() => any) | BelongsToOptions,
      optionsOrForeignKey?: string | { foreignKey?: string; as?: string }
    ) => DecoratableSchemaType<any>;
    PAGING: (options: DtoOrTypeIs) => any;
  }
}
```

### TypeIs.FOREIGNKEY

외래 키 컬럼을 정의합니다. 레거시 형식과 신규 형식 모두 지원합니다.

#### ForeignKeyOptions 인터페이스

```typescript
import { SchemaOptions } from '@asapjs/schema';

interface ForeignKeyOptions extends SchemaOptions {
  // 레거시 호환
  table?: () => any;
  extraData?: Record<string, any>;

  // 신규 옵션
  references?: {
    model: string;
    key: string;
  };
  onDelete?: 'CASCADE' | 'SET NULL' | 'NO ACTION' | 'RESTRICT';
  onUpdate?: 'CASCADE' | 'SET NULL' | 'NO ACTION' | 'RESTRICT';
}
```

#### 레거시 형식 (함수 시그니처)

```typescript
import { TypeIs } from '@asapjs/schema';

// TypeIs.FOREIGNKEY(() => Model, extraData?)
@TypeIs.FOREIGNKEY(() => UsersTable, { comment: '작성자 ID' })
user_id: number;

// table 옵션 객체 형태도 가능
@TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
user_id: number;
```

#### 신규 형식 (references 옵션)

```typescript
import { TypeIs } from '@asapjs/schema';

@TypeIs.FOREIGNKEY({
  references: { model: 'users', key: 'id' },
  onDelete: 'CASCADE',
  onUpdate: 'CASCADE',
  comment: '작성자 ID',
})
user_id: number;
```

- 내부적으로 `ForeignKeyType` 클래스(`BaseSchemaType` 상속)로 구현
- `toSequelize()`는 `{ table, extraData }` 형식을 반환 (레거시 호환)
- 기본 타입은 `TypeIs.INT`

### TypeIs.BELONGSTO

BelongsTo 관계를 정의합니다. `FOREIGNKEY`와 항상 쌍으로 사용합니다.

#### BelongsToOptions 인터페이스

```typescript
import { SchemaOptions } from '@asapjs/schema';

interface BelongsToOptions extends SchemaOptions {
  // 레거시 호환
  associatedClassGetter?: () => any;
  optionsOrForeignKey?: string | { foreignKey?: string; as?: string };

  // 신규 옵션
  target?: any;
  foreignKey?: string;
  as?: string;
}
```

#### 레거시 형식 (함수 시그니처)

```typescript
import { TypeIs } from '@asapjs/schema';

// TypeIs.BELONGSTO(() => Model, 'foreignKey')
@TypeIs.BELONGSTO(() => UsersTable, 'user_id')
user: UsersTable;

// 옵션 객체로도 가능
@TypeIs.BELONGSTO(() => UsersTable, { foreignKey: 'user_id', as: 'author' })
user: UsersTable;
```

#### 신규 형식 (옵션 객체)

```typescript
import { TypeIs } from '@asapjs/schema';

@TypeIs.BELONGSTO({
  target: UsersTable,
  foreignKey: 'user_id',
  as: 'author',
})
user: UsersTable;
```

- 내부적으로 `BelongsToType` 클래스(`BaseSchemaType` 상속)로 구현
- `toSequelize()`는 `{ associatedClassGetter, optionsOrForeignKey }` 형식을 반환

#### FOREIGNKEY + BELONGSTO 쌍 사용 패턴

```typescript
import { TypeIs } from '@asapjs/schema';

// 레거시 형식
@TypeIs.FOREIGNKEY({ table: () => UsersTable, comment: '작성자 ID' })
user_id: number;

@TypeIs.BELONGSTO(() => UsersTable, 'user_id')
user: UsersTable;

// 신규 형식
@TypeIs.FOREIGNKEY({
  references: { model: 'users', key: 'id' },
  onDelete: 'CASCADE',
  comment: '작성자 ID',
})
user_id: number;

@TypeIs.BELONGSTO({
  target: UsersTable,
  foreignKey: 'user_id',
  as: 'author',
})
user: UsersTable;
```

### TypeIs.QUERY

계산된 필드(서브쿼리)를 정의합니다. DTO에서만 사용합니다.

#### QueryOptions 인터페이스

```typescript
import { SchemaOptions } from '@asapjs/schema';

interface QueryOptions extends SchemaOptions {
  /** SQL 서브쿼리. association은 현재 테이블 별칭, ${association}.id 등으로 참조 */
  query: (props: { association?: string; user?: any }) => string;
  /** 결과 타입. TypeIs.JSON() 등 반환값 또는 () => TypeIs.JSON() getter */
  type?: (() => any) | any;
}
```

#### 사용 예시

```typescript
import { TypeIs } from '@asapjs/schema';

@TypeIs.QUERY({
  query: ({ association }) =>
    `(SELECT COUNT(*) FROM comments WHERE comments.post_id = ${association}.id)`,
  type: () => TypeIs.INT(),
  comment: '댓글 수',
})
commentCount: number;
```

- 내부적으로 `QueryType` 클래스(`BaseSchemaType` 상속)로 구현
- `getType()`: `type` 옵션이 함수이면 호출, 아니면 그대로 반환
- `getQuery(props)`: SQL 서브쿼리 문자열 반환
- `toSwagger()`: 내부 타입의 `toSwagger()` 결과를 위임
- `toSequelize()`: `{ query, type }` 형식을 반환 (레거시 호환)

### TypeIs.PAGING

페이지네이션 래퍼입니다. DTO 클래스 또는 TypeIs 팩토리를 받아 페이지네이션 응답 구조를 생성합니다.

```typescript
import { TypeIs } from '@asapjs/schema';

// DTO 클래스 전달
TypeIs.PAGING(PostInfoDto)

// TypeIs 팩토리 전달 (함수)
TypeIs.PAGING(() => TypeIs.INT())
```

- 파라미터 타입: `DtoOrTypeIs` = `typeof ExtendableDto | (() => TypeIsData)`
- Swagger 출력: `{ data: T[], page, page_size, max_page, has_prev, has_next, total_elements }` 구조

### Sequelize 타입 등록 과정

`@asapjs/sequelize`는 내부적으로 `registerSequelizeTypes()`를 호출하여 타입을 등록합니다:

```typescript
import { extendTypeIs } from '@asapjs/schema';
import { QueryType, ForeignKeyType, BelongsToType } from '@asapjs/sequelize';

// @asapjs/sequelize 내부에서 자동 호출
extendTypeIs({
  QUERY: (options: QueryOptions) => new QueryType(options),
  FOREIGNKEY: (relatedClassGetterOrOptions, extraData?) => {
    if (typeof relatedClassGetterOrOptions === 'function') {
      // 레거시: TypeIs.FOREIGNKEY(() => Model, extraData?)
      return new ForeignKeyType({ table: relatedClassGetterOrOptions, extraData });
    }
    // 신규: TypeIs.FOREIGNKEY({ references: ... })
    return new ForeignKeyType(relatedClassGetterOrOptions);
  },
  BELONGSTO: (associatedClassGetterOrOptions, optionsOrForeignKey?) => {
    if (typeof associatedClassGetterOrOptions === 'function') {
      // 레거시: TypeIs.BELONGSTO(() => Model, 'foreignKey')
      return new BelongsToType({ associatedClassGetter: associatedClassGetterOrOptions, optionsOrForeignKey });
    }
    // 신규: TypeIs.BELONGSTO({ target, foreignKey, as })
    return new BelongsToType(associatedClassGetterOrOptions);
  },
  PAGING: TypeIsPaging,
});
```
