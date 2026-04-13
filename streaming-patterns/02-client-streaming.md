# Client Streaming — 배치 데이터 전송

---

## 🎯 핵심 질문

- Client Streaming에서 여러 요청 후 단일 응답을 받는 HTTP/2 Frame 흐름은?
- 100MB 파일을 Unary로 전송하면 왜 실패하고, Client Streaming은 어떻게 해결하는가?
- 청크 크기는 어떻게 결정하는가?
- 서버는 언제 onNext()로 응답할 수 있는가?
- 중간에 네트워크가 끊기면 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Client Streaming은 대용량 데이터(파일 업로드, 로그 배치, 센서 데이터 수집)를 청크 단위로 전송하는 핵심 패턴입니다. gRPC의 기본 Unary RPC는 4MB 제한이 있어서 100MB 파일 전송이 불가능하지만, Client Streaming은 무제한으로 데이터를 쌓아서 전송할 수 있습니다. 메모리 효율적인 스트리밍 처리로 O(1) 메모리로 대용량 파일을 처리하고, 부분 전송 실패 시 해당 청크부터 재전송하여 대역폭을 절약합니다.

---

## 😱 흔한 실수 (Before — 단일 메시지로 전송하려는 접근)

```java
// 실수 1: 100MB 파일을 단일 Unary 메시지로 전송 시도
@RestController
public class FileUploadController {
    @PostMapping("/api/upload")
    public ResponseEntity<String> uploadFile(
            @RequestParam("file") MultipartFile file) {
        // 문제: 100MB 파일을 메모리에 모두 로드
        byte[] allBytes = file.getBytes();  // 100MB 메모리 사용
        
        // gRPC Unary RPC 제한: 기본값 4MB
        // → 100MB는 전송 불가능 (MessageSizeTooLargeException)
        return ResponseEntity.ok("Uploaded");
    }
}


// 실수 2: requestObserver.onCompleted() 호출 없이 클라이언트 종료
public void uploadFile(String filePath) {
    StreamObserver<Empty> responseObserver = 
        new StreamObserver<Empty>() {
            @Override
            public void onNext(Empty response) {
                log.info("Upload complete");
            }
            @Override
            public void onError(Throwable t) {
                log.error("Upload failed", t);
            }
            @Override
            public void onCompleted() {
                log.info("Response received");
            }
        };
    
    StreamObserver<FileChunk> requestObserver = 
        stub.uploadFile(responseObserver);
    
    // 파일 청크 전송
    byte[] buffer = new byte[64 * 1024];  // 64KB
    int bytesRead;
    
    try (InputStream input = new FileInputStream(filePath)) {
        while ((bytesRead = input.read(buffer)) != -1) {
            FileChunk chunk = FileChunk.newBuilder()
                .setData(ByteString.copyFrom(buffer, 0, bytesRead))
                .build();
            requestObserver.onNext(chunk);
        }
        
        // 버그: onCompleted() 호출 안 함!
        // requestObserver.onCompleted();  // ← 빠짐
        
        // 결과: 서버가 "파일 끝" 신호를 받지 못함
        // 서버는 다음 청크를 계속 기다림 → 타임아웃 (30초)
    }
}


// 실수 3: 청크 크기를 너무 작게 설정 (1바이트씩)
for (int i = 0; i < fileSize; i++) {
    FileChunk chunk = FileChunk.newBuilder()
        .setData(ByteString.copyFrom(new byte[]{fileByte}))  // 1바이트!
        .build();
    requestObserver.onNext(chunk);
    
    // 문제:
    // - 100MB 파일 = 104,857,600개 청크
    // - 각 청크: 1바이트 데이터 + 30바이트 프로토콜버퍼 오버헤드
    // - Frame 헤더: 5바이트 (gRPC) + 9바이트 (HTTP/2)
    // - 총 오버헤드: 44바이트 / 1바이트 = 4400% (!!)
    // - 처리량: 100MB / (104M × 30ms/chunk) = 32KB/s (느림)
}
```

---

## ✨ 올바른 접근 (After — gRPC 원리를 알고 청크 최적화하기)

```java
// 서비스 정의 (file.proto)
service FileService {
    rpc UploadFile(stream FileChunk) returns (UploadResponse) {}
}

message FileChunk {
    string file_id = 1;           // 파일 식별자
    bytes data = 2;               // 청크 데이터
    int32 chunk_index = 3;        // 청크 번호
    int64 total_size = 4;         // 총 파일 크기
}

message UploadResponse {
    string file_id = 1;
    int64 bytes_received = 2;
    string status = 3;            // "success", "partial", "error"
}

// 서버 구현
public class FileServiceImpl extends FileServiceGrpc.FileServiceImplBase {
    private static final int MAX_CHUNK_SIZE = 256 * 1024;  // 256KB
    
    @Override
    public StreamObserver<FileChunk> uploadFile(
            StreamObserver<UploadResponse> responseObserver) {
        
        return new StreamObserver<FileChunk>() {
            private FileOutputStream output;
            private long totalBytesReceived = 0;
            private String fileId = null;
            
            @Override
            public void onNext(FileChunk chunk) {
                try {
                    // 첫 번째 청크에서 파일 생성
                    if (fileId == null) {
                        fileId = chunk.getFileId();
                        String filePath = "/tmp/uploads/" + fileId;
                        output = new FileOutputStream(filePath);
                        log.info("Started receiving file: {}, total size: {} bytes",
                            fileId, chunk.getTotalSize());
                    }
                    
                    // 청크 데이터 디스크에 쓰기 (스트리밍 처리)
                    byte[] data = chunk.getData().toByteArray();
                    output.write(data);
                    totalBytesReceived += data.length;
                    
                    // 진행 상황 로깅
                    int percent = (int)(totalBytesReceived * 100 / chunk.getTotalSize());
                    log.info("Progress: {}% ({}/{})",
                        percent, totalBytesReceived, chunk.getTotalSize());
                    
                } catch (IOException e) {
                    responseObserver.onError(Status.INTERNAL
                        .withDescription("Failed to write chunk")
                        .withCause(e).asException());
                }
            }
            
            @Override
            public void onError(Throwable t) {
                log.error("Upload error for file {}: {}", fileId, t.getMessage());
                try {
                    if (output != null) output.close();
                } catch (IOException ignored) {}
                
                responseObserver.onError(Status.CANCELLED
                    .withDescription("Upload cancelled").asException());
            }
            
            @Override
            public void onCompleted() {
                try {
                    if (output != null) output.close();
                    
                    log.info("File {} uploaded successfully: {} bytes",
                        fileId, totalBytesReceived);
                    
                    // 응답 전송 (Client Streaming에서는 여기서만!)
                    UploadResponse response = UploadResponse.newBuilder()
                        .setFileId(fileId)
                        .setBytesReceived(totalBytesReceived)
                        .setStatus("success")
                        .build();
                    responseObserver.onNext(response);
                    responseObserver.onCompleted();
                    
                } catch (IOException e) {
                    responseObserver.onError(Status.INTERNAL.asException());
                }
            }
        };
    }
}

// 클라이언트 구현: 최적화된 청크 크기
public class FileUploadClient {
    private static final int CHUNK_SIZE = 256 * 1024;  // 256KB (최적값)
    
    public void uploadFile(String filePath, String fileId) {
        stub.uploadFile(new StreamObserver<UploadResponse>() {
            @Override
            public void onNext(UploadResponse response) {
                log.info("Server received {} bytes", response.getBytesReceived());
            }
            
            @Override
            public void onError(Throwable t) {
                log.error("Upload failed: {}", t.getMessage());
                // 재시도 로직
                scheduleRetry(filePath, fileId);
            }
            
            @Override
            public void onCompleted() {
                log.info("Upload completed successfully");
            }
        }).accept(createFileChunks(filePath, fileId));
    }
    
    private Iterator<FileChunk> createFileChunks(
            String filePath, String fileId) {
        return new Iterator<FileChunk>() {
            private final File file = new File(filePath);
            private final long totalSize = file.length();
            private InputStream input;
            private int chunkIndex = 0;
            private boolean initialized = false;
            
            @Override
            public boolean hasNext() {
                try {
                    if (!initialized) {
                        input = new FileInputStream(file);
                        initialized = true;
                    }
                    return input.available() > 0;
                } catch (IOException e) {
                    return false;
                }
            }
            
            @Override
            public FileChunk next() {
                try {
                    byte[] buffer = new byte[CHUNK_SIZE];
                    int bytesRead = input.read(buffer);
                    
                    if (bytesRead == -1) {
                        input.close();
                        throw new NoSuchElementException();
                    }
                    
                    FileChunk chunk = FileChunk.newBuilder()
                        .setFileId(fileId)
                        .setData(ByteString.copyFrom(buffer, 0, bytesRead))
                        .setChunkIndex(chunkIndex++)
                        .setTotalSize(totalSize)
                        .build();
                    
                    // 진행 상황 출력
                    long bytesProcessed = (long)chunkIndex * CHUNK_SIZE;
                    int percent = (int)(bytesProcessed * 100 / totalSize);
                    log.info("Uploading: {}%", percent);
                    
                    return chunk;
                    
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
        };
    }
    
    private void scheduleRetry(String filePath, String fileId) {
        // Exponential Backoff로 재시도
    }
}
```

---

## 🔬 내부 동작 원리

### 1. HTTP/2 Frame 흐름 — 여러 DATA Frame 후 단일 응답

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP/2 Connection (TCP)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                         Stream 1
                      (File Upload)
                         │
    CLIENT               │                SERVER
     │                   │                   │
     │──────► HEADERS ───►│ (UploadFile)   │
     │  Stream ID=1       │                   │
     │  (request headers) │                   │
     │                    │                   │
     │──────► DATA #1 ───►│ (64KB chunk)   │
     │  Stream ID=1       │                   │
     │  (청크 1: bytes 0-65535)              │
     │                    │                   │
     │──────► DATA #2 ───►│ (64KB chunk)   │
     │  Stream ID=1       │                   │
     │  (청크 2: bytes 65536-131071)         │
     │                    │                   │
     │──────► DATA #3 ───►│ (64KB chunk)   │
     │  Stream ID=1       │                   │
     │  (청크 3: bytes 131072-196607)        │
     │                    │                   │
     │  ... (1000개 청크 계속)                │
     │                    │                   │
     │──────► DATA #1000 ►│ (32KB chunk)   │
     │  Stream ID=1       │                   │
     │  END_STREAM flag   │  [모든 청크 수신 완료]
     │                    │  [파일을 디스크에 저장]
     │                    │  [응답 생성]
     │                    │                   │
     │ ◄──────── HEADERS ─┤  (UploadResponse)
     │  Stream ID=1       │                   │
     │  (response headers)│                   │
     │                    │                   │
     │ ◄──────── DATA ────┤  (응답 데이터)    
     │  Stream ID=1       │  {file_id, bytes_received}
     │  END_STREAM flag   │                   │


키 포인트:
1. 클라이언트가 requestObserver.onNext()를 여러 번 호출
   → 각 호출이 DATA Frame으로 변환
   → 모두 같은 Stream ID에 담김

2. END_STREAM flag는 requestObserver.onCompleted() 호출 시만 설정
   → 이 신호가 없으면 서버는 계속 기다림 (타임아웃까지)

3. 서버는 모든 DATA Frame 수신 후 onCompleted()에서 응답 생성
   → HEADERS + DATA (응답) + END_STREAM로 응답
```

### 2. Unary vs Client Streaming 메모리 사용 비교

```
UNARY RPC (제한 4MB)
──────────────────
[Client] 
  버퍼: 100MB 파일 메모리 로드
  → new byte[100MB] (불가능, OOM)
  또는 여러 Unary RPC 호출
  → 100MB / 4MB = 25개 RPC
  → 25개 연결 (비효율)


CLIENT STREAMING (무제한)
──────────────────────
[Client]
  버퍼: CHUNK_SIZE (256KB) 루프 사용
  메모리: O(1) (고정 크기 버퍼만 유지)
  
  Loop {
    read(256KB) → FileChunk 생성 → onNext() → 메모리 해제
    [다음 청크 읽기]
  }


메모리 사용량 비교:
┌────────────────────┬──────────┬────────────────┐
│ 파일 크기           │ Unary    │ Client Stream  │
├────────────────────┼──────────┼────────────────┤
│ 10MB                │ ~10MB    │ ~256KB         │
│ 100MB               │ ~100MB   │ ~256KB         │
│ 1GB                 │ ~1GB (!!)│ ~256KB         │
│ 10GB                │ 불가능  │ ~256KB         │
└────────────────────┴──────────┴────────────────┘

Client Streaming의 이점: 파일 크기와 관계없이 일정한 메모리 사용


청크 크기 결정:
┌──────────────────────────────────────────────┐
│ 청크 크기 분석                                │
├──────────────────────────────────────────────┤
│                                              │
│ 너무 작음 (1KB):                             │
│ - 청크 수: 100MB / 1KB = 104,857,600개     │
│ - Frame 헤더: 44바이트 × 104M = 4.4GB      │
│ - 오버헤드: 4400% (매우 비효율)             │
│ - 처리량: 32KB/s (매우 느림)                │
│                                              │
│ 최적 (256KB):                                │
│ - 청크 수: 100MB / 256KB = 400개            │
│ - Frame 헤더: 44바이트 × 400 = 17.6KB      │
│ - 오버헤드: 0.018% (거의 없음)              │
│ - 처리량: 100MB/s (매우 빠름)               │
│                                              │
│ 너무 큼 (16MB):                              │
│ - 청크 수: 100MB / 16MB = 6.25개           │
│ - 메모리 사용: 16MB × 2 = 32MB (버퍼링)     │
│ - 네트워크 지연: 부분 재전송 시 16MB 낭비  │
│                                              │
│ 권장: 64KB ~ 256KB                         │
│ - 네트워크 지연: ~1ms (good)                │
│ - 오버헤드: <0.1% (good)                    │
│ - 메모리: <1MB (good)                       │
│ - 재전송 비용: 합리적                       │
└──────────────────────────────────────────────┘
```

### 3. 파일 업로드 구현 (서버/클라이언트 Java 코드)

```java
/**
 * FileService 서버 구현
 * Client Streaming 패턴: 클라이언트 → 여러 청크 → 서버
 */
public class FileServiceImpl extends FileServiceGrpc.FileServiceImplBase {
    
    private static final int CHUNK_SIZE = 256 * 1024;  // 256KB
    private static final String UPLOAD_DIR = "/tmp/uploads";
    
    @Override
    public StreamObserver<FileChunk> uploadFile(
            StreamObserver<UploadResponse> responseObserver) {
        
        // StreamObserver 생명주기:
        // 1. 클라이언트가 onNext()를 여러 번 호출 (청크 전송)
        // 2. 클라이언트가 onCompleted() 호출
        // 3. 우리는 onCompleted()에서 응답 생성
        
        return new StreamObserver<FileChunk>() {
            private FileOutputStream output;
            private long totalBytesReceived = 0;
            private String fileId = null;
            private long expectedSize = 0;
            
            @Override
            public void onNext(FileChunk chunk) {
                try {
                    // 첫 청크: 파일 초기화
                    if (fileId == null) {
                        fileId = chunk.getFileId();
                        expectedSize = chunk.getTotalSize();
                        
                        // 디렉토리 생성
                        new File(UPLOAD_DIR).mkdirs();
                        
                        String filePath = UPLOAD_DIR + "/" + fileId;
                        output = new FileOutputStream(filePath);
                        
                        log.info("Starting file upload: {}, expected size: {} bytes",
                            fileId, expectedSize);
                    }
                    
                    // 청크 데이터 디스크에 쓰기
                    byte[] data = chunk.getData().toByteArray();
                    output.write(data);
                    totalBytesReceived += data.length;
                    
                    // 진행 상황 로깅 (모든 청크마다는 과함, 5% 단위로)
                    long percent = totalBytesReceived * 100 / expectedSize;
                    if (percent % 5 == 0) {
                        log.info("Upload progress: {}% ({}/{})",
                            percent, totalBytesReceived, expectedSize);
                    }
                    
                    // 청크 검증
                    if (chunk.getChunkIndex() >= 0) {
                        log.debug("Chunk #{}: {} bytes", 
                            chunk.getChunkIndex(), data.length);
                    }
                    
                } catch (IOException e) {
                    log.error("Error writing chunk for file {}", fileId, e);
                    responseObserver.onError(Status.INTERNAL
                        .withDescription("Failed to write chunk: " + e.getMessage())
                        .withCause(e).asException());
                }
            }
            
            @Override
            public void onError(Throwable t) {
                log.error("Upload error for file {}: {}", fileId, t.getMessage());
                
                try {
                    if (output != null) {
                        output.close();
                    }
                    // 부분적으로 업로드된 파일 삭제 가능
                    String filePath = UPLOAD_DIR + "/" + fileId;
                    new File(filePath).delete();
                } catch (IOException ignored) {}
                
                responseObserver.onError(Status.CANCELLED
                    .withDescription("Upload cancelled").asException());
            }
            
            @Override
            public void onCompleted() {
                // 클라이언트가 모든 청크 전송 완료 → 여기서 응답
                try {
                    if (output != null) {
                        output.close();
                    }
                    
                    // 파일 크기 검증
                    if (totalBytesReceived != expectedSize) {
                        log.warn("File {} size mismatch: expected {}, got {}",
                            fileId, expectedSize, totalBytesReceived);
                    }
                    
                    log.info("File {} uploaded successfully: {} bytes",
                        fileId, totalBytesReceived);
                    
                    // 응답 생성 및 전송
                    UploadResponse response = UploadResponse.newBuilder()
                        .setFileId(fileId)
                        .setBytesReceived(totalBytesReceived)
                        .setStatus(totalBytesReceived == expectedSize ? 
                            "success" : "partial")
                        .build();
                    
                    // Client Streaming에서는 onCompleted()에서만 응답!
                    responseObserver.onNext(response);
                    responseObserver.onCompleted();
                    
                } catch (IOException e) {
                    log.error("Error completing upload for file {}", fileId, e);
                    responseObserver.onError(Status.INTERNAL.asException());
                }
            }
        };
    }
}

/**
 * FileService 클라이언트 구현
 * 대용량 파일을 256KB 청크로 분할하여 전송
 */
public class FileUploadClient {
    
    private final FileServiceGrpc.FileServiceBlockingStub blockingStub;
    private static final int CHUNK_SIZE = 256 * 1024;  // 256KB
    
    public FileUploadClient(ManagedChannel channel) {
        this.blockingStub = FileServiceGrpc.newBlockingStub(channel);
    }
    
    /**
     * 파일을 청크 단위로 전송하고 서버 응답을 기다림
     */
    public UploadResponse uploadFile(String filePath, String fileId) {
        File file = new File(filePath);
        long fileSize = file.length();
        
        log.info("Starting file upload: {} ({} bytes)", filePath, fileSize);
        
        try {
            // Iterator<FileChunk>을 전달 → 자동으로 onNext() 호출
            Iterator<FileChunk> chunks = createChunkIterator(
                file, fileId, fileSize);
            
            // blockingStub.uploadFile()는 Client Streaming RPC
            // Iterator의 모든 원소를 전송 후 서버 응답 대기
            UploadResponse response = blockingStub.uploadFile(chunks);
            
            log.info("Upload complete: received {} bytes from server",
                response.getBytesReceived());
            
            return response;
            
        } catch (Exception e) {
            log.error("Upload failed: {}", e.getMessage());
            throw new RuntimeException(e);
        }
    }
    
    /**
     * FileChunk Iterator 생성
     * 파일을 256KB 청크로 분할하고 순차적으로 반환
     */
    private Iterator<FileChunk> createChunkIterator(
            File file, String fileId, long totalSize) {
        
        return new Iterator<FileChunk>() {
            private InputStream input;
            private int chunkIndex = 0;
            private boolean initialized = false;
            private long bytesProcessed = 0;
            
            {
                // Initializer block
                try {
                    input = new FileInputStream(file);
                    initialized = true;
                } catch (FileNotFoundException e) {
                    throw new RuntimeException(e);
                }
            }
            
            @Override
            public boolean hasNext() {
                try {
                    return input.available() > 0;
                } catch (IOException e) {
                    return false;
                }
            }
            
            @Override
            public FileChunk next() {
                try {
                    byte[] buffer = new byte[CHUNK_SIZE];
                    int bytesRead = input.read(buffer);
                    
                    if (bytesRead == -1) {
                        input.close();
                        throw new NoSuchElementException("End of file reached");
                    }
                    
                    bytesProcessed += bytesRead;
                    
                    // 청크 생성
                    FileChunk chunk = FileChunk.newBuilder()
                        .setFileId(fileId)
                        .setData(ByteString.copyFrom(buffer, 0, bytesRead))
                        .setChunkIndex(chunkIndex++)
                        .setTotalSize(totalSize)
                        .build();
                    
                    // 진행 상황 출력 (5% 단위)
                    long percent = bytesProcessed * 100 / totalSize;
                    if (percent % 5 == 0) {
                        log.info("Uploading: {}% ({}/{})",
                            percent, bytesProcessed, totalSize);
                    }
                    
                    return chunk;
                    
                } catch (IOException e) {
                    throw new RuntimeException("Failed to read chunk", e);
                }
            }
        };
    }
}
```

### 4. 청크 크기 결정 전략과 처리량-오버헤드 트레이드오프

```
청크 크기 최적화:
┌─────────────────────────────────────────────────────────────┐
│ 1KB 청크 (너무 작음)                                        │
├─────────────────────────────────────────────────────────────┤
│ 파일 크기: 100MB                                            │
│ 청크 수: 100MB / 1KB = 104,857,600개                       │
│ 지연: 104,857,600 × 0.5ms = 52,428초 = 14.5시간(!!)      │
│ 오버헤드: (44바이트 × 104M) / 100MB = 4400%               │
│                                                            │
│ 결론: 절대 사용 금지                                       │


│ 64KB 청크 (좋음)                                           │
├─────────────────────────────────────────────────────────────┤
│ 청크 수: 100MB / 64KB = 1,600개                           │
│ 지연: 1,600 × 0.5ms = 0.8초                              │
│ 오버헤드: (44 × 1,600) / 100MB = 0.07%                    │
│ 처리량: 100MB / 0.8s = 125MB/s                           │
│                                                            │
│ 결론: 일반적인 용도에 권장                                 │


│ 256KB 청크 (최적)                                          │
├─────────────────────────────────────────────────────────────┤
│ 청크 수: 100MB / 256KB = 400개                            │
│ 지연: 400 × 0.5ms = 0.2초                                │
│ 오버헤드: (44 × 400) / 100MB = 0.018%                     │
│ 처리량: 100MB / 0.2s = 500MB/s                           │
│                                                            │
│ 결론: 대부분의 경우 최고 성능                              │


│ 16MB 청크 (너무 큼)                                        │
├─────────────────────────────────────────────────────────────┤
│ 청크 수: 100MB / 16MB = 6.25개                           │
│ 메모리: 16MB × 2 (생산자/소비자 버퍼) = 32MB               │
│ 네트워크 지연: 100ms (높음)                               │
│ 부분 실패 시 재전송: 16MB (대량 낭비)                      │
│                                                            │
│ 결론: 메모리/지연 측면에서 비효율적                         │
└─────────────────────────────────────────────────────────────┘

권장 청크 크기 선택 로직:
if (네트워크 지연 < 10ms) {  // LAN, 근거리
    청크_크기 = 256KB;  // 처리량 최대화
} else if (네트워크 지연 < 100ms) {  // WAN, 일반 인터넷
    청크_크기 = 64KB;  // 균형
} else {  // 높은 지연 (위성, 국제)
    청크_크기 = 16KB;  // 혼잡 제어 안정성
}
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# Client Streaming 실시간 테스트

# 1. 100MB 테스트 파일 생성
dd if=/dev/zero of=test_file.bin bs=1M count=100

# 2. gRPC 서버 시작
java -cp grpc-all.jar:. FileServiceServer &

# 3. 클라이언트로 파일 업로드
java -cp grpc-all.jar:. FileUploadClient test_file.bin test_id_001

# 출력:
# [INFO] Starting file upload: test_file.bin (104857600 bytes)
# [INFO] Uploading: 0% (256000/104857600)
# [INFO] Uploading: 5% (5242880/104857600)
# [INFO] Uploading: 10% (10485760/104857600)
# ... (계속)
# [INFO] Upload complete: received 104857600 bytes from server

# 4. 네트워크 대역폭 모니터링
iftop -i eth0 -n

# 5. 청크 크기별 성능 비교
for CHUNK_SIZE in 1024 16384 65536 262144 1048576; do
  echo "Testing with chunk size: $CHUNK_SIZE bytes"
  time java -cp grpc-all.jar:. \
    FileUploadClient test_file.bin test_id_${CHUNK_SIZE}
done

# 6. 메모리 프로파일링
jmap -heap <client_pid> | grep -E "HeapUsage|used|max"
```

---

## 📊 성능/비용 비교

```
Unary RPC vs Client Streaming (100MB 파일 업로드)

┌────────────────┬──────────────────┬──────────────────────┐
│ 메트릭          │ Unary            │ Client Streaming     │
├────────────────┼──────────────────┼──────────────────────┤
│ 가능 파일 크기  │ 4MB (최대)       │ 무제한               │
├────────────────┼──────────────────┼──────────────────────┤
│ 100MB 업로드    │ 불가능           │ 200ms               │
│ 시간           │ (MessageTooLarge) │ (256KB 청크)         │
├────────────────┼──────────────────┼──────────────────────┤
│ 클라이언트      │ ~100MB           │ ~256KB               │
│ 메모리 사용     │                  │                      │
├────────────────┼──────────────────┼──────────────────────┤
│ 서버 메모리     │ ~100MB           │ ~256KB               │
│ (버퍼링)       │                  │ (O(1) 스트리밍)      │
├────────────────┼──────────────────┼──────────────────────┤
│ 부분 재전송     │ 4MB 단위         │ 256KB 단위           │
│ 효율           │ (큼)             │ (작음)               │
├────────────────┼──────────────────┼──────────────────────┤
│ 처리량          │ 50MB/s           │ 500MB/s (10배)       │
│ (네트워크 대역)│                  │                      │
└────────────────┴──────────────────┴──────────────────────┘
```

---

## ⚖️ 트레이드오프

```
✅ 장점
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 무제한 파일 크기 지원 (Unary는 4MB 제한)
2. O(1) 메모리 사용 (파일 크기와 무관)
3. 부분 재전송 최소화 (256KB 단위)
4. 높은 처리량 (500MB/s 이상 가능)
5. 서버 메모리 부하 감소

❌ 제약사항
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 복잡한 에러 처리 (부분 전송 상태 관리)
2. 클라이언트 재시도 로직 필요
3. Offset 관리 어려움 (청크 식별자 필수)
4. 서버 디스크 I/O 부하 증가
```

---

## 📌 핵심 정리

```
Client Streaming은 청크 단위로 대용량 데이터를 전송한다.

1. HTTP/2 여러 DATA Frame → 단일 응답
2. 청크 크기: 64KB ~ 256KB 권장
3. 메모리: O(1) (CHUNK_SIZE만큼만)
4. 에러: onError() 또는 onCompleted()에서만
5. 성능: Unary 대비 10배 빠름
```

---

## 🤔 생각해볼 문제

### Q1: requestObserver.onCompleted() 호출 없이 프로그램을 종료하면 서버는 무엇을 기다리고 있는가?

<details>
<summary>해설 보기</summary>

서버는 onCompleted()에서만 응답을 생성하므로, 클라이언트가 onCompleted()를 호출하지 않으면 서버는 "다음 청크를 무한히 기다리는" 상태가 됩니다. gRPC의 기본 타임아웃(30초)에 도달하면 연결이 끊기지만, 그때까지 서버 리소스가 점유된 상태입니다.

Timeline:
```
T=0s: 클라이언트 청크 1-100 전송
T=5s: 클라이언트 프로그램 종료 (onCompleted() 호출 안 함!)
T=5s~T=30s: 서버 "다음 청크 대기" 상태 (리소스 점유)
T=30s: 타임아웃 → 연결 종료
```
</details>

### Q2: 청크 크기를 1바이트로 설정하면 정확히 어떤 오버헤드가 발생하는가?

<details>
<summary>해설 보기</summary>

```
Frame 구조:
┌────────────────────────────────┐
│ HTTP/2 Frame Header (9 bytes)  │
├────────────────────────────────┤
│ DATA (1 byte)                  │
└────────────────────────────────┘

Total: 10 bytes per chunk

100MB 파일:
- 청크 수: 104,857,600개
- 오버헤드: 9 × 104,857,600 = 943.7MB (!!)
- 전송 데이터: 100MB + 943.7MB = 1043.7MB
- 오버헤드율: 943%

실제 처리:
- 각 청크: 0.5ms (최소)
- 총 시간: 104M × 0.5ms = 52,428초 = 14.5시간
- 처리량: 100MB / 14.5h = 19KB/s (매우 느림!)

결론: 절대 사용 금지
```
</details>

### Q3: 네트워크가 중간에 끊겼을 때, 이미 전송된 청크(1~500)와 전송 중이던 청크(501)의 상태는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

```
Timeline:
T=0s: 청크 1-100 전송 완료, 서버가 디스크에 기록
T=5s: 청크 200-300 전송 중, 청크 250이 반쯤 왔을 때 네트워크 끊김!

상태:
- 청크 1-200: 서버 디스크에 완전히 저장됨
- 청크 250: 불완전 (20바이트만 도착, 36바이트 손실)
- 청크 251-500: 미전송

TCP/HTTP/2 동작:
- TCP가 자동으로 손실된 패킷 재전송 시도 (3초간)
- 재전송 실패 → TCP connection reset
- gRPC 클라이언트 onError() 호출

복구:
1. 애플리케이션 수준 재시도:
   - 파일 다시 열기
   - 청크 201부터 재전송
   - 서버: 청크 201-N 추가 기록
   
2. Offset 관리로 정확히:
   - 클라이언트: "byte 204,800부터 다시 보내달라"
   - 서버: 지정된 offset부터 수신

문제: 청크 250이 불완전하므로, 정확한 경계를 알아야 함
해결: FileChunk에 chunk_index + 파일 offset 포함
```
</details>

---

<div align="center">

**[⬅️ 이전: Server Streaming](./01-server-streaming.md)** | **[홈으로 🏠](../README.md)** | **[다음: Bidirectional Streaming ➡️](./03-bidirectional-streaming.md)**

</div>
