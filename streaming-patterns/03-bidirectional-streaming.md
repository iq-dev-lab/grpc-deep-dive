# Bidirectional Streaming — 채팅과 게임 상태 동기화

---

## 🎯 핵심 질문

- Bidirectional Streaming에서 클라이언트와 서버가 동시에 메시지를 보내는 원리는?
- Half-close란 무엇이고, 한쪽이 종료해도 반대쪽이 계속 보낼 수 있는 이유는?
- 채팅 서비스를 Bidirectional Streaming으로 구현하는 방법은?
- responseObserver.onNext()가 Thread-safe하지 않은 이유와 해결 방법은?
- WebSocket과 비교했을 때 Bidirectional gRPC의 장단점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Bidirectional Streaming은 실시간 양방향 통신(채팅, 게임, 협업 도구)의 핵심 패턴입니다. REST에서는 채팅을 위해 POST(메시지 전송)와 SSE(메시지 수신) 2개의 연결이 필요했지만, gRPC는 단일 연결에서 양방향 메시지를 동시에 처리합니다. Half-close 메커니즘으로 클라이언트는 전송을 종료하고도 계속 수신할 수 있어, "채팅 방 나가기" 같은 순차적 동작을 우아하게 표현합니다.

---

## 😱 흔한 실수 (Before — 단순하게 생각하는 접근)

```java
// 실수 1: 여러 스레드에서 synchronized 없이 responseObserver.onNext() 호출
public void broadcastMessage(ChatMessage message) {
    for (StreamObserver<ChatMessage> observer : roomSubscribers) {
        observer.onNext(message);  // Race condition!
    }
}

// 문제:
// - StreamObserver가 Thread-safe하지 않음
// - 100개 스레드가 동시에 onNext()를 호출하면:
//   * HTTP/2 Frame 순서 뒤바뀜
//   * 부분 write (일부만 전송)
//   * 연결 손상


// 실수 2: 채팅을 REST POST(전송) + Server Streaming(수신)으로 구현
@PostMapping("/api/chat/send")
public ResponseEntity<Void> sendMessage(@RequestBody ChatMessage msg) {
    // 1. 메시지 전송 (POST)
    chatService.saveMessage(msg);
    return ResponseEntity.ok().build();
}

@GetMapping("/api/chat/receive")
public ResponseEntity<SseEmitter> subscribeMessages() {
    // 2. 메시지 수신 (Server-Sent Events)
    SseEmitter emitter = new SseEmitter();
    chatService.subscribe(emitter);
    return ResponseEntity.ok(emitter);
}

// 문제:
// - 2개의 연결 필요 (POST, SSE)
// - 메시지 순서 불일치 가능 (A가 전송, B가 수신 간의 경쟁)
// - 브라우저에서 동시 요청 제한 (같은 도메인 6개 연결)


// 실수 3: onCompleted() 후에도 onNext() 호출 시도
public void handleClientDisconnect(String userId) {
    StreamObserver<ChatMessage> observer = userObservers.get(userId);
    observer.onCompleted();  // 스트림 종료
    
    // 잠시 후, 다른 메시지가 도착하면:
    observer.onNext(newMessage);  // IllegalStateException!
    // → "이미 완료된 스트림에 메시지 전송 불가"
}
```

---

## ✨ 올바른 접근 (After — gRPC 양방향으로 우아하게)

```java
// 서비스 정의 (chat.proto)
service ChatService {
    rpc ChatRoom(stream ChatMessage) returns (stream ChatMessage) {}
}

message ChatMessage {
    string user_id = 1;
    string room_id = 2;
    string content = 3;
    int64 timestamp_ms = 4;
    enum MessageType {
        TEXT = 0;
        USER_JOINED = 1;
        USER_LEFT = 2;
    }
    MessageType type = 5;
}

// 서버 구현
public class ChatServiceImpl extends ChatServiceGrpc.ChatServiceImplBase {
    
    private final ConcurrentHashMap<String, List<StreamObserver<ChatMessage>>> rooms;
    
    public ChatServiceImpl() {
        this.rooms = new ConcurrentHashMap<>();
    }
    
    @Override
    public StreamObserver<ChatMessage> chatRoom(
            StreamObserver<ChatMessage> responseObserver) {
        
        return new StreamObserver<ChatMessage>() {
            private String userId = null;
            private String roomId = null;
            
            @Override
            public void onNext(ChatMessage message) {
                try {
                    // 첫 메시지에서 사용자와 방 식별
                    if (userId == null) {
                        userId = message.getUserId();
                        roomId = message.getRoomId();
                        
                        // 이 클라이언트를 방 구독자에 추가
                        List<StreamObserver<ChatMessage>> roomSubscribers = 
                            rooms.computeIfAbsent(roomId, 
                                k -> new CopyOnWriteArrayList<>());
                        roomSubscribers.add(responseObserver);
                        
                        // 입장 알림 브로드캐스트
                        ChatMessage joinMsg = ChatMessage.newBuilder()
                            .setUserId(userId)
                            .setRoomId(roomId)
                            .setType(ChatMessage.MessageType.USER_JOINED)
                            .setTimestampMs(System.currentTimeMillis())
                            .build();
                        broadcastToRoom(roomId, joinMsg);
                        
                        log.info("User {} joined room {}", userId, roomId);
                    }
                    
                    // 메시지 브로드캐스트
                    broadcastToRoom(roomId, message);
                    
                } catch (Exception e) {
                    log.error("Error processing message from {}", userId, e);
                    responseObserver.onError(Status.INTERNAL.asException());
                }
            }
            
            @Override
            public void onError(Throwable t) {
                log.error("Client error: {}", t.getMessage());
                if (userId != null && roomId != null) {
                    handleUserDisconnect(userId, roomId);
                }
            }
            
            @Override
            public void onCompleted() {
                log.info("Client completed: {}", userId);
                if (userId != null && roomId != null) {
                    handleUserDisconnect(userId, roomId);
                }
                responseObserver.onCompleted();
            }
            
            private void handleUserDisconnect(String userId, String roomId) {
                List<StreamObserver<ChatMessage>> roomSubscribers = 
                    rooms.get(roomId);
                
                if (roomSubscribers != null) {
                    roomSubscribers.remove(responseObserver);
                    
                    // 퇴장 알림
                    ChatMessage leaveMsg = ChatMessage.newBuilder()
                        .setUserId(userId)
                        .setRoomId(roomId)
                        .setType(ChatMessage.MessageType.USER_LEFT)
                        .setTimestampMs(System.currentTimeMillis())
                        .build();
                    broadcastToRoom(roomId, leaveMsg);
                    
                    log.info("User {} left room {}", userId, roomId);
                }
            }
        };
    }
    
    /**
     * 특정 방의 모든 구독자에게 메시지 브로드캐스트
     * Thread-safe 처리 필수 (CopyOnWriteArrayList + synchronized)
     */
    private void broadcastToRoom(String roomId, ChatMessage message) {
        List<StreamObserver<ChatMessage>> roomSubscribers = rooms.get(roomId);
        
        if (roomSubscribers != null && !roomSubscribers.isEmpty()) {
            for (StreamObserver<ChatMessage> observer : roomSubscribers) {
                // synchronized 블록으로 각 observer 보호
                synchronized (observer) {
                    try {
                        observer.onNext(message);
                    } catch (Exception e) {
                        log.debug("Failed to broadcast to subscriber: {}", 
                            e.getMessage());
                        roomSubscribers.remove(observer);
                        try {
                            observer.onError(Status.UNAVAILABLE.asException());
                        } catch (Exception ignored) {}
                    }
                }
            }
        }
    }
}

// 클라이언트 구현
public class ChatClient {
    private final ManagedChannel channel;
    private final ChatServiceGrpc.ChatServiceStub asyncStub;
    
    public ChatClient(String host, int port) {
        this.channel = ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext().build();
        this.asyncStub = ChatServiceGrpc.newStub(channel);
    }
    
    public void joinChatRoom(String userId, String roomId) {
        // requestObserver: 우리가 서버에 메시지를 보내는 통로
        // responseObserver: 서버에서 메시지를 받는 통로
        
        StreamObserver<ChatMessage> responseObserver = 
            new StreamObserver<ChatMessage>() {
                @Override
                public void onNext(ChatMessage message) {
                    log.info("[{}] {}: {}",
                        message.getType(),
                        message.getUserId(),
                        message.getContent());
                }
                
                @Override
                public void onError(Throwable t) {
                    log.error("Chat error: {}", t.getMessage());
                }
                
                @Override
                public void onCompleted() {
                    log.info("Chat completed");
                }
            };
        
        // 양방향 스트림 시작
        StreamObserver<ChatMessage> requestObserver = 
            asyncStub.chatRoom(responseObserver);
        
        // 입장 메시지 전송
        ChatMessage joinMsg = ChatMessage.newBuilder()
            .setUserId(userId)
            .setRoomId(roomId)
            .setContent(userId + " joined the room")
            .setTimestampMs(System.currentTimeMillis())
            .build();
        requestObserver.onNext(joinMsg);
        
        // 사용자 입력을 받아서 계속 전송
        new Thread(() -> {
            try (Scanner scanner = new Scanner(System.in)) {
                while (scanner.hasNextLine()) {
                    String line = scanner.nextLine();
                    
                    if ("exit".equals(line)) {
                        // Half-close: 전송 종료, 수신은 계속
                        requestObserver.onCompleted();
                        break;
                    }
                    
                    ChatMessage msg = ChatMessage.newBuilder()
                        .setUserId(userId)
                        .setRoomId(roomId)
                        .setContent(line)
                        .setType(ChatMessage.MessageType.TEXT)
                        .setTimestampMs(System.currentTimeMillis())
                        .build();
                    requestObserver.onNext(msg);
                }
            }
        }).start();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. HTTP/2 Frame 흐름 — 양방향 독립 스트림

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP/2 Connection (TCP)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                         Stream 1
                      (Bidirectional)
                         │
    CLIENT               │                SERVER
     │                   │                   │
     │──► HEADERS #1 ───►│ (ChatRoom)       │
     │  Stream ID=1       │  REQUEST_STREAM │
     │  (request)         │                   │
     │                    │                   │
     │──► DATA #1 ───────►│                   │
     │  ChatMessage       │  User A: "hello" │
     │  {userId:"A", ...} │                   │
     │                    │  [Server processes]
     │                    │  [Broadcasts to all users]
     │                    │
     │ ◄─── DATA #1 ─────│                   │
     │  ChatMessage       │  User A: "hello" │
     │  (broadcast from   │  (sent back)      │
     │   server to A)     │                   │
     │                    │
     │ ◄─── DATA #2 ─────│                   │
     │  ChatMessage       │  User B: "hi A"  │
     │  (from other       │  (server broadcast)
     │   client: User B)  │                   │
     │                    │
     │──► DATA #2 ───────►│                   │
     │  ChatMessage       │  User A: "hi B"  │
     │  {userId:"A", ...} │                   │
     │                    │
     │ ◄─── DATA #3 ─────│                   │
     │  ChatMessage       │  User A: "hi B"  │
     │  (A's message      │  (echo back)      │
     │   echoed back)     │                   │
     │                    │
     │ ◄─── DATA #4 ─────│                   │
     │  ChatMessage       │  User B: "bye"   │
     │  (B's message      │                   │
     │   from server)     │                   │
     │                    │
     │──► HEADERS ───────►│                   │
     │  END_STREAM        │  (half-close)    │
     │  (A finishes send, │  A stopped sending
     │   but still        │  but still listening
     │   receiving)       │                   │
     │                    │
     │ ◄─── DATA #5 ─────│                   │
     │  ChatMessage       │  User C: "hi A"  │
     │  {userId:"C", ...} │  (A still receives!)
     │                    │
     │ ◄─── HEADERS ─────│                   │
     │  END_STREAM        │  (server closes)
     │                    │  All streaming done


KEY POINTS:

1. Single HTTP/2 Stream (ID=1)
   - 양방향 메시지가 같은 Stream에서 처리됨
   - Stream ID로 클라이언트/서버 메시지 구분 (X)
   - 모든 메시지가 섞여서 전달됨

2. CLIENT → SERVER (requestObserver.onNext())
   - DATA Frame + HEADERS with STREAM_DEPENDENCY
   - End 신호: requestObserver.onCompleted() = Half-close

3. SERVER → CLIENT (responseObserver.onNext())
   - DATA Frame (순수 응답용)
   - End 신호: responseObserver.onCompleted()

4. Half-close (한쪽만 종료)
   - CLIENT가 onCompleted() → 더 이상 전송 안 함
   - SERVER는 계속 onNext() 호출 가능
   - SERVER가 onCompleted() → 양쪽 모두 종료
```

### 2. Half-close 동작 원리 — END_STREAM 플래그

```
Normal Request-Response:
┌──────────────────────────────────────────┐
│ CLIENT                SERVER               │
│  │                    │                    │
│  ├─ HEADERS ─────────►│                   │
│  ├─ DATA ────────────►│  [Process]        │
│  │  END_STREAM        │                   │
│  │  (전송 완료 신호)   │                   │
│  │                    │                   │
│  │◄─ HEADERS ─────────┤                   │
│  │◄─ DATA ────────────┤  (응답)           │
│  │  END_STREAM        │                   │
│  │                    │                   │
│  ▼ Connection closed  ▼                   │
└──────────────────────────────────────────┘


Half-close (Bidirectional):
┌──────────────────────────────────────────┐
│ CLIENT                SERVER               │
│  │                    │                    │
│  ├─ DATA ────────────►│  [Process Msg1]   │
│  │                    │  [Broadcast]      │
│  │◄─ DATA ────────────┤  (응답)           │
│  │                    │                   │
│  ├─ DATA ────────────►│  [Process Msg2]   │
│  │                    │  [Broadcast]      │
│  │◄─ DATA ────────────┤  (응답)           │
│  │                    │                   │
│  ├─ DATA ────────────►│  (사용자 입력)    │
│  │ END_STREAM flag    │  "exit" 명령      │
│  │ (더 이상 보내지 않음)│                   │
│  │                    │                   │
│  │◄─ DATA ────────────┤  [다른 사용자      │
│  │  (계속 수신)        │   메시지 전송]   │
│  │                    │                   │
│  │◄─ HEADERS ─────────┤                   │
│  │  END_STREAM        │  [서버도 완료]    │
│  │  (서버 응답 완료)   │                   │
│  │                    │                   │
│  ▼ Stream closed      ▼                   │
└──────────────────────────────────────────┘

Timeline (Half-close):
T=0s:   CLIENT ──► END_STREAM (전송 종료 신호)
T=0ms:  SERVER 감지: "클라이언트가 더 이상 보내지 않음"
T=5ms:  SERVER가 onNext() 호출 → 여전히 데이터 전송 가능
T=10ms: SERVER ──► END_STREAM (모든 처리 완료)
T=15ms: Stream 양쪽 모두 완료, 연결 해제 가능
```

### 3. 채팅 서비스 구현 (서버 Java 코드: 방 참여, 메시지 브로드캐스트, 나가기)

```java
/**
 * 채팅 서버 구현
 * Bidirectional Streaming 패턴 + Thread-safe 브로드캐스트
 */
public class ChatServiceImpl extends ChatServiceGrpc.ChatServiceImplBase {
    
    private final ConcurrentHashMap<String, ChatRoom> rooms;
    private final ScheduledExecutorService executor;
    
    public ChatServiceImpl() {
        this.rooms = new ConcurrentHashMap<>();
        this.executor = Executors.newScheduledThreadPool(4);
    }
    
    @Override
    public StreamObserver<ChatMessage> chatRoom(
            StreamObserver<ChatMessage> responseObserver) {
        
        return new ClientChatHandler(responseObserver);
    }
    
    /**
     * 각 클라이언트 연결을 처리하는 핸들러
     */
    private class ClientChatHandler implements StreamObserver<ChatMessage> {
        private final StreamObserver<ChatMessage> responseObserver;
        private String userId;
        private String roomId;
        private ChatRoom chatRoom;
        private volatile boolean isActive = true;
        
        public ClientChatHandler(StreamObserver<ChatMessage> responseObserver) {
            this.responseObserver = responseObserver;
        }
        
        @Override
        public void onNext(ChatMessage message) {
            if (!isActive) return;
            
            try {
                // 첫 메시지: 사용자 식별
                if (userId == null) {
                    userId = message.getUserId();
                    roomId = message.getRoomId();
                    
                    // 채팅방 생성 또는 기존 방 가져오기
                    chatRoom = rooms.computeIfAbsent(roomId, 
                        key -> new ChatRoom(roomId));
                    
                    // 이 클라이언트를 방에 등록
                    chatRoom.addClient(userId, this);
                    
                    log.info("User {} joined room {}", userId, roomId);
                    
                    // 입장 메시지 브로드캐스트
                    ChatMessage joinNotification = ChatMessage.newBuilder()
                        .setUserId("SYSTEM")
                        .setRoomId(roomId)
                        .setContent(userId + " joined the room")
                        .setType(ChatMessage.MessageType.USER_JOINED)
                        .setTimestampMs(System.currentTimeMillis())
                        .build();
                    chatRoom.broadcast(joinNotification);
                }
                
                // 일반 메시지 처리
                if (message.getType() == ChatMessage.MessageType.TEXT) {
                    // 메시지 유효성 검증
                    if (message.getContent() == null || 
                        message.getContent().isEmpty()) {
                        log.warn("Empty message from {}", userId);
                        return;
                    }
                    
                    // 메시지 타임스탬프 설정
                    ChatMessage processedMsg = ChatMessage.newBuilder(message)
                        .setTimestampMs(System.currentTimeMillis())
                        .setUserId(userId)
                        .setRoomId(roomId)
                        .build();
                    
                    // 모든 구독자에게 브로드캐스트
                    chatRoom.broadcast(processedMsg);
                    
                    log.debug("Message from {}: {}", userId, 
                        message.getContent());
                }
                
            } catch (Exception e) {
                log.error("Error processing message from {}", userId, e);
                onError(e);
            }
        }
        
        @Override
        public void onError(Throwable t) {
            log.error("Client error for user {}: {}", userId, t.getMessage());
            handleDisconnect();
        }
        
        @Override
        public void onCompleted() {
            log.info("Client {} completed (half-close or graceful close)", userId);
            // Half-close: 클라이언트가 전송 종료했지만 수신은 계속 가능
            // 우리는 계속 메시지를 보낼 수 있음
            
            // 실제로 모두 종료되려면 onCompleted() 또는 onError() 호출 필요
            // 일정 시간 후 자동 종료
            executor.schedule(this::handleDisconnect, 30, TimeUnit.SECONDS);
        }
        
        private void handleDisconnect() {
            if (!isActive) return;
            isActive = false;
            
            try {
                if (chatRoom != null && userId != null) {
                    chatRoom.removeClient(userId);
                    
                    // 퇴장 메시지 브로드캐스트
                    ChatMessage leaveNotification = ChatMessage.newBuilder()
                        .setUserId("SYSTEM")
                        .setRoomId(roomId)
                        .setContent(userId + " left the room")
                        .setType(ChatMessage.MessageType.USER_LEFT)
                        .setTimestampMs(System.currentTimeMillis())
                        .build();
                    chatRoom.broadcast(leaveNotification);
                    
                    // 비어있는 방 정리
                    if (chatRoom.isEmpty()) {
                        rooms.remove(roomId);
                        log.info("Room {} removed (empty)", roomId);
                    }
                }
                
                // ResponseObserver 종료
                try {
                    responseObserver.onCompleted();
                } catch (Exception ignored) {}
                
                log.info("User {} disconnected from room {}", userId, roomId);
                
            } catch (Exception e) {
                log.error("Error during disconnect", e);
            }
        }
        
        /**
         * 이 클라이언트에게 메시지 전송 (thread-safe)
         */
        public void sendMessage(ChatMessage message) {
            if (!isActive) return;
            
            synchronized (responseObserver) {
                try {
                    responseObserver.onNext(message);
                } catch (Exception e) {
                    log.debug("Failed to send message to {}: {}", 
                        userId, e.getMessage());
                    handleDisconnect();
                }
            }
        }
    }
    
    /**
     * 채팅방 관리
     */
    private static class ChatRoom {
        private final String roomId;
        private final ConcurrentHashMap<String, ClientChatHandler> clients;
        
        public ChatRoom(String roomId) {
            this.roomId = roomId;
            this.clients = new ConcurrentHashMap<>();
        }
        
        public void addClient(String userId, ClientChatHandler handler) {
            clients.put(userId, handler);
        }
        
        public void removeClient(String userId) {
            clients.remove(userId);
        }
        
        public boolean isEmpty() {
            return clients.isEmpty();
        }
        
        /**
         * 모든 클라이언트에게 메시지 브로드캐스트
         * CopyOnWriteArrayList + synchronized로 Thread-safe
         */
        public void broadcast(ChatMessage message) {
            if (clients.isEmpty()) return;
            
            // Iterator 사용하여 동시 수정 방지
            for (ClientChatHandler handler : clients.values()) {
                handler.sendMessage(message);
            }
        }
        
        public int getClientCount() {
            return clients.size();
        }
    }
    
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }
}
```

### 4. Thread-safe onNext() 패턴 — synchronized vs ConcurrentLinkedQueue

```java
/**
 * 패턴 1: Synchronized (간단하지만 성능 저하)
 */
private class SynchronizedBroadcast {
    public void broadcast(ChatMessage message) {
        for (StreamObserver<ChatMessage> observer : subscribers) {
            synchronized (observer) {
                try {
                    observer.onNext(message);
                } catch (Exception e) {
                    log.error("Send failed", e);
                    subscribers.remove(observer);
                }
            }
        }
    }
}


/**
 * 패턴 2: ConcurrentLinkedQueue (병렬성 높음)
 */
private class QueueBasedBroadcast {
    private final ConcurrentLinkedQueue<ChatMessage> outgoingQueue;
    private final ScheduledExecutorService executor;
    
    public QueueBasedBroadcast(StreamObserver<ChatMessage> observer) {
        this.outgoingQueue = new ConcurrentLinkedQueue<>();
        this.executor = Executors.newSingleThreadScheduledExecutor();
        
        // 단일 스레드로 큐에서 꺼내서 onNext() 호출
        executor.scheduleAtFixedRate(
            this::flushQueue, 0, 10, TimeUnit.MILLISECONDS);
    }
    
    public void enqueue(ChatMessage message) {
        outgoingQueue.offer(message);  // Non-blocking
    }
    
    private void flushQueue() {
        ChatMessage message;
        while ((message = outgoingQueue.poll()) != null) {
            try {
                observer.onNext(message);
            } catch (Exception e) {
                log.error("Send failed", e);
                // Observer 제거 로직
            }
        }
    }
}


/**
 * 비교:
 * 
 * Synchronized:
 * - 장점: 간단한 구현
 * - 단점: Lock contention (많은 클라이언트 = 느림)
 * - 사용: 클라이언트 수 < 100
 * 
 * ConcurrentLinkedQueue:
 * - 장점: Lock-free, 높은 병렬성
 * - 단점: 지연 증가 (큐 플러시 간격)
 * - 사용: 클라이언트 수 > 1000
 * 
 * 권장: Synchronized + CopyOnWriteArrayList (대부분의 경우)
 */
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# Bidirectional Streaming 채팅 테스트

# 1. 서버 시작
java -cp grpc-all.jar:. ChatServiceServer &
SERVER_PID=$!

# 2. 여러 클라이언트로 채팅
for i in {1..5}; do
  java -cp grpc-all.jar:. ChatClient \
    -user "User$i" -room "general" &
  sleep 1
done

# 3. 클라이언트에서 메시지 입력
# [Terminal 1] User1: hello everyone
# [Terminal 2] User2: hi user1
# [Terminal 3] User3: how are you all?
# ...

# 4. 종료 테스트
# 한 클라이언트에서 "exit" 입력 → 퇴장 메시지 브로드캐스트 확인

# 5. 부하 테스트 (100개 동시 클라이언트)
for i in {1..100}; do
  java -cp grpc-all.jar:. ChatClient \
    -user "LoadTest$i" -room "load_test" &
done
wait

# 6. 메모리 모니터링
jstat -gc $SERVER_PID 1000

kill $SERVER_PID
```

---

## 📊 성능/비용 비교

```
REST (POST + SSE) vs Bidirectional gRPC (100명 채팅)

┌────────────────┬──────────────────┬──────────────────────┐
│ 메트릭          │ REST             │ Bidirectional gRPC   │
├────────────────┼──────────────────┼──────────────────────┤
│ 연결 수         │ 200 (2×100)     │ 100 (1×100)         │
│ (POST + SSE)   │                  │                      │
├────────────────┼──────────────────┼──────────────────────┤
│ 메시지 지연     │ 50ms (POST)      │ 10ms (양방향)        │
│                │ + 50ms (SSE)     │                      │
│                │ = 100ms          │                      │
├────────────────┼──────────────────┼──────────────────────┤
│ 메모리 (서버)  │ 100×1MB = 100MB │ 100×256KB = 25MB    │
├────────────────┼──────────────────┼──────────────────────┤
│ 네트워크 대역폭│ 1MB/s (헤더)     │ 100KB/s (데이터)     │
├────────────────┼──────────────────┼──────────────────────┤
│ Half-close 지원│ 없음             │ 있음                 │
└────────────────┴──────────────────┴──────────────────────┘
```

---

## ⚖️ 트레이드오프

```
✅ 장점
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 단일 연결 (REST 대비 연결 수 50% 감소)
2. 낮은 지연 (양방향 동시 처리)
3. Half-close (한쪽만 종료 가능)
4. 메모리 효율 (공유 연결)

❌ 제약사항
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Thread-safe 구현 복잡 (synchronized 필수)
2. 디버깅 어려움 (양방향 메시지 흐름)
3. 부분 메시지 순서 보장 어려움
```

---

## 📌 핵심 정리

```
Bidirectional Streaming: 양방향 메시지 동시 처리

1. HTTP/2 단일 Stream에서 양방향 메시지
2. Half-close로 한쪽만 종료 가능
3. Thread-safe: synchronized + CopyOnWriteArrayList
4. 연결 수 50% 감소
5. REST 대비 지연 5배 단축
```

---

## 🤔 생각해볼 문제

### Q1: 여러 스레드에서 동시에 observer.onNext()를 호출하면 정확히 무슨 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

gRPC의 StreamObserver는 Thread-safe하지 않으므로, 여러 스레드의 onNext() 호출이 HTTP/2 Frame을 뒤섞을 수 있습니다.

```
Thread 1: onNext(Message A) → Frame A 송신 시작
Thread 2: onNext(Message B) → Frame B 송신 시작 (A 중간에 끼어듦)
Thread 3: onNext(Message C) → Frame C 송신 시작

결과 Frame:
[A Header][B Header][A Data][C Data][B Data]...

클라이언트가 받는 데이터:
[A Header][B Header] → 둘 다 헤더?
[A Data] → A의 데이터는?
[C Data] → C의 헤더는 없는데?
[B Data] → B와 C가 섞임!

HTTP/2 디코더가 프레임을 복구할 수 없음 → 연결 손상!
```

해결: synchronized 블록으로 observer.onNext() 호출을 보호합니다.
</details>

### Q2: Half-close 상태에서 클라이언트가 requestObserver.onNext()를 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Half-close는 "전송 종료 신호"일 뿐, 즉시 전송을 막는 것이 아닙니다. END_STREAM 플래그는 "더 이상 메시지를 보내지 않겠다"는 신호이므로, 그 후 onNext()를 호출하면:

```
Timeline:
T=0s: requestObserver.onCompleted() → END_STREAM 플래그 전송
T=5ms: 클라이언트 코드에서 requestObserver.onNext() 호출
Result: StreamObserver Dead → onError() 발생

Error: "Cannot send on a closed stream"
Status.CANCELLED 또는 Status.FAILED_PRECONDITION
```

Best Practice: onCompleted() 후에는 더 이상 onNext()를 호출하지 않아야 합니다.
</details>

### Q3: 100명이 같은 채팅방에 있을 때, 한 사람이 메시지를 보내면 HTTP/2 Frame 관점에서 무슨 일이 일어나는가?

<details>
<summary>해설 보기</summary>

```
Client A (Stream ID=1)
┌─────────────────────────────────────┐
│ onNext(Message A) → Frame A 송신   │
└─────────────────────────────────────┘
         ↓ (Netty Buffer)
┌─────────────────────────────────────┐
│ TCP 송신 버퍼                       │
│ [DATA Frame(A)] → 약 100 bytes     │
└─────────────────────────────────────┘
         ↓ (네트워크)
┌─────────────────────────────────────┐
│ Server 수신                         │
│ Frame A 디코딩 → Message A 추출    │
└─────────────────────────────────────┘
         ↓ (Broadcast Loop)
┌─────────────────────────────────────┐
│ for (Client B~CZ) {                 │
│   synchronized (observer) {         │
│     observer.onNext(Message A)      │
│   }                                 │
│ }                                   │
└─────────────────────────────────────┘

결과: 100개의 동시 Frame이 클라이언트로 전송
- 각 클라이언트 Stream은 독립적 (Stream ID마다 다름)
- TCP 입장에서는 1개의 연결 (Multiplexing)
- 네트워크에서는 약 5-10개의 TCP Segment로 집계

성능:
- 메모리: 100 × 256KB = 25MB (각 스트림 버퍼)
- 처리량: 모든 클라이언트가 100ms 이내에 수신
- CPU: 브로드캐스트 1회 = 100 synchronized 블록
```
</details>

---

<div align="center">

**[⬅️ 이전: Client Streaming](./02-client-streaming.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스트리밍 흐름 제어 ➡️](./04-flow-control.md)**

</div>
