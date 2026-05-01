# 실시간 소켓 통신

`@asapjs/socket` 패키지는 Socket.IO를 래핑합니다. `*Socket.ts` 파일에서 `createSocket()`으로 이벤트 핸들러를 선언하면 자동 탐색됩니다.

## 설정

```typescript
// src/index.ts
const config = {
  extensions: ['@asapjs/sequelize', '@asapjs/socket'],
  socket: {},  // 단일 서버 모드 (Redis 없음)
};

// Redis 어댑터 (다중 서버)
const config = {
  extensions: ['@asapjs/sequelize', '@asapjs/socket'],
  socket: {
    redis: {
      socket: {
        host: process.env.REDIS_HOST || 'localhost',
        port: parseInt(process.env.REDIS_PORT || '6379', 10),
      },
    },
  },
};
```

## createSocket() — 이벤트 핸들러 등록

```typescript
import { createSocket } from '@asapjs/socket';

createSocket(
  { path: 'chat:message' },  // 수신할 이벤트 이름
  async (socket, { body, user }) => {
    // socket: Socket.IO 인스턴스
    // body: 클라이언트가 보낸 페이로드
    // user: JWT 디코딩된 사용자 (socket.request.user)
  }
);
```

### 타입 정의

```typescript
interface RouteRequest {
  path: string;       // 소켓 이벤트 이름
  roles?: string[];   // 역할 기반 필터링 (예약)
}

type ExecuteFunction = (
  socket: Socket,
  args: { body: any; user: any }
) => Promise<unknown> | unknown | void;
```

## 메시지 전송 API

### socketSendAll — 전체 브로드캐스트

```typescript
import { socketSendAll } from '@asapjs/socket';

socketSendAll('system:announcement', { message: '점검 예정' });
```

### socketSendTo — 특정 대상

```typescript
import { socketSendTo } from '@asapjs/socket';

// 특정 클라이언트
socketSendTo(socket.id, 'notification', { message: '주문 발송' });

// 방의 모든 클라이언트
socketSendTo('room:lobby', 'chat:message', { text: '안녕하세요' });
```

### getSocketIO — 서버 인스턴스 접근

```typescript
import { getSocketIO } from '@asapjs/socket';

const io = getSocketIO();  // Socket.IO Server | undefined
if (io) {
  const sockets = await io.fetchSockets();
  console.log(`연결된 클라이언트: ${sockets.length}`);
}
```

## 전체 핸들러 예시

```typescript
// src/chat/ChatSocket.ts
import { createSocket, socketSendAll, socketSendTo } from '@asapjs/socket';

// 브로드캐스트 메시지
createSocket(
  { path: 'chat:message' },
  async (socket, { body, user }) => {
    socketSendAll('chat:message', {
      from: user?.id ?? 'anonymous',
      text: body.text,
      timestamp: new Date().toISOString(),
    });
  }
);

// 귓속말 (특정 클라이언트)
createSocket(
  { path: 'chat:whisper' },
  async (socket, { body, user }) => {
    socketSendTo(body.targetSocketId, 'chat:whisper', {
      from: user?.id,
      text: body.text,
    });
  }
);

// 방 참가
createSocket(
  { path: 'room:join' },
  async (socket, { body, user }) => {
    await socket.join(body.roomId);
    socketSendTo(body.roomId, 'room:member_joined', { userId: user?.id });
    socket.emit('room:joined', { roomId: body.roomId });
  }
);
```

## REST 컨트롤러에서 소켓 사용

```typescript
import { socketSendAll } from '@asapjs/socket';

@Post('/', { title: '게시글 작성', auth: true })
async createPost({ body, user }: ExecuteArgs) {
  const post = await this.postService.createPost(user, body);
  socketSendAll('post:created', { id: post.id, title: post.title });
  return post;
}
```

## 파일 명명 규칙

| 올바른 예 | 잘못된 예 |
|-----------|-----------|
| `ChatSocket.ts` | `chat.ts` |
| `NotificationSocket.ts` | `socketHandler.ts` |
| `deep/path/OrderSocket.ts` | `OrderEvents.ts` |

반드시 `*Socket.ts`로 끝나야 자동 탐색됩니다.

## 연결 이벤트

클라이언트 연결 시 서버가 자동으로:
1. `socket.request.user`에서 인증 사용자 정보 추출
2. `success` 이벤트 emit: `{ message: 'success connected' }`
3. 등록된 모든 `createSocket()` 핸들러를 해당 소켓에 바인딩

## SocketOption

```typescript
interface SocketOption {
  adapter?: unknown;                    // 커스텀 어댑터
  listener?: (socket: unknown) => void; // 연결 시 원시 훅
  redis?: RedisClientOptions;           // Redis pub/sub 설정
}
```
