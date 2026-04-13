# Server Streaming — 실시간 데이터 푸시

---

## 🎯 핵심 질문

- Server Streaming에서 서버가 여러 응답을 보내는 HTTP/2 Frame 흐름은?
- Polling과 Server Streaming의 연결 수, 지연, 서버 부하 차이는?
- 실시간 주식 시세를 Server Streaming으로 구현하는 방법은?
- Backpressure는 어떻게 처리하는가?
- 스트림이 종료되는 3가지 경우는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Server Streaming은 실시간 데이터 푸시(주식 시세, 센서 값, 이벤트 알림)의 핵심 패턴입니다. Polling의 무의미한 네트워크 왕복을 제거하고, WebSocket처럼 단일 연결 위에서 고효율 스트리밍을 제공하며, HTTP/2 Flow Control로 자동 Backpressure를 처리합니다. 올바르게 구현하면 지연 시간을 초 단위에서 밀리초 단위로 줄일 수 있고, 서버 연결 수를 수십 배 감소시킬 수 있습니다.

---

## 😱 흔한 실수 (Before — REST로만 생각하는 접근)

```java
// 실수 1: 실시간 데이터를 1초 Polling으로 구현
@RestController
public class StockPricePollController {
    @GetMapping("/api/stock/{symbol}")
    public StockPrice getCurrentPrice(@PathVariable String symbol) {
        return stockService.getLatestPrice(symbol);
    }
}

// 클라이언트가 1초마다 호출 → 변경 없어도 매 1초마다 RPC 호출
// 결과: 100개 클라이언트 = 초당 100 RPC, 99% 응답이 "변경 없음"
while (true) {
    StockPrice price = restTemplate.getForObject(
        "http://server:8080/api/stock/AAPL", StockPrice.class);
    updateUI(price);
    Thread.sleep(1000); // 주기적인 불필요한 호출
}

// 문제점:
// - 평균 지연: 1초 (주기) + 100ms (쿼리) = 1.1초
// - 연결 수: 100개 클라이언트
// - 불필요한 응답: 99% (변경 없음)
// - 네트워크 헤더: 1KB × 100 RPC/s = 100KB/s 오버헤드


// 실수 2: onNext() 호출 후 onCompleted() 빠뜨림
service.subscribeStockUpdates(request, new StreamObserver<StockPrice>() {
    @Override
    public void onNext(StockPrice price) {
        updateUI(price);
    }

    @Override
    public void onError(Throwable t) {
        log.error("Stream error", t);
    }

    // onCompleted() 메서드 구현 안 함!
    // → 클라이언트는 영원히 스트림 완료 대기 상태 (메모리 누수)
    // → 해당 gRPC 채널이 종료될 때까지 리소스 점유
});

// 결과: 사용자가 앱을 종료해도 모바일 메모리에서 해제되지 않음


// 실수 3: 스트림이 끊겼을 때 재연결 로직 없음
service.subscribeStockUpdates(request, new StreamObserver<StockPrice>() {
    @Override
    public void onError(Throwable t) {
        log.error("Stream disconnected", t);
        // 로그만 남기고 재연결 시도 안 함
        // → 사용자는 3분간 데이터를 받지 못했지만 모르고 있음
    }
});

// 상황:
// 09:00:00 서버와 스트림 연결 성공
// 09:01:30 네트워크 끊김 (서버 재시작, ISP 문제 등)
// 09:01:30 클라이언트 onError() 호출, 로그 기록
// 09:01:30~09:04:30 데이터 수신 중단 (하지만 UI는 마지막 캐시 값 표시)
```

---

## ✨ 올바른 접근 (After — gRPC 원리를 알고 선택하는 접근)

```java
// 서비스 정의 (stock.proto)
service StockService {
    rpc SubscribeStockUpdates(SubscribeRequest) returns (stream StockPrice) {}
}

message SubscribeRequest {
    string symbol = 1;
}

message StockPrice {
    string symbol = 1;
    double price = 2;
    int64 timestamp_ms = 3;
}

// 서버 구현
public class StockServiceImpl extends StockServiceGrpc.StockServiceImplBase {
    private final StockDataSource stockData;
    private final ConcurrentHashMap<String, List<StreamObserver<StockPrice>>> subscribers;
    private final ScheduledExecutorService broadcastExecutor;

    public StockServiceImpl(StockDataSource stockData) {
        this.stockData = stockData;
        this.subscribers = new ConcurrentHashMap<>();
        this.broadcastExecutor = Executors.newScheduledThreadPool(1, r -> {
            Thread t = new Thread(r, "StockBroadcaster");
            t.setDaemon(true);
            return t;
        });
    }

    @Override
    public void subscribeStockUpdates(
            SubscribeRequest request,
            StreamObserver<StockPrice> responseObserver) {
        
        String symbol = request.getSymbol();
        
        if (symbol == null || symbol.isEmpty()) {
            responseObserver.onError(Status.INVALID_ARGUMENT
                .withDescription("Symbol is required").asException());
            return;
        }
        
        if (!stockData.isValidSymbol(symbol)) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("Symbol not found").asException());
            return;
        }
        
        // 구독자 목록에 추가 (동시성 안전)
        List<StreamObserver<StockPrice>> symbolSubscribers = 
            subscribers.computeIfAbsent(symbol, k -> new CopyOnWriteArrayList<>());
        symbolSubscribers.add(responseObserver);
        
        log.info("New subscriber for symbol: {}, total: {}", 
            symbol, symbolSubscribers.size());
        
        // 즉시 현재 가격 전송
        try {
            StockPrice currentPrice = stockData.getLatestPrice(symbol);
            responseObserver.onNext(currentPrice);
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL
                .withDescription("Failed to get current price").asException());
            symbolSubscribers.remove(responseObserver);
        }
    }

    public void startBroadcasting() {
        broadcastExecutor.scheduleAtFixedRate(
            this::broadcastPriceUpdates,
            100,  // 100ms 후 시작
            100,  // 100ms마다 반복
            TimeUnit.MILLISECONDS
        );
        log.info("Stock broadcast started");
    }

    private void broadcastPriceUpdates() {
        Map<String, StockPrice> priceMap = stockData.getAllLatestPrices();
        
        for (Map.Entry<String, StockPrice> entry : priceMap.entrySet()) {
            String symbol = entry.getKey();
            StockPrice price = entry.getValue();
            
            List<StreamObserver<StockPrice>> symbolSubscribers = 
                subscribers.get(symbol);
            
            if (symbolSubscribers != null && !symbolSubscribers.isEmpty()) {
                // CopyOnWriteArrayList이므로 iteration 중 remove 가능
                for (StreamObserver<StockPrice> observer : symbolSubscribers) {
                    try {
                        observer.onNext(price);
                    } catch (Exception e) {
                        log.debug("Failed to send price to subscriber: {}", 
                            e.getMessage());
                        symbolSubscribers.remove(observer);
                        
                        try {
                            observer.onError(Status.UNAVAILABLE
                                .withDescription("Connection lost").asException());
                        } catch (Exception ignored) {
                            // observer가 이미 종료된 상태
                        }
                    }
                }
            }
        }
    }

    public void shutdown() {
        broadcastExecutor.shutdown();
        try {
            if (!broadcastExecutor.awaitTermination(5, TimeUnit.SECONDS)) {
                broadcastExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            broadcastExecutor.shutdownNow();
        }
    }
}

// 클라이언트 구현: 재연결 로직 포함
public class StockPriceSubscriber {
    private final ManagedChannel channel;
    private final StockServiceGrpc.StockServiceStub asyncStub;
    private final ScheduledExecutorService scheduler;
    private final ExecutorService processingExecutor;
    private volatile boolean shouldReconnect = true;
    private int retryCount = 0;
    private static final int MAX_RETRY_DELAY_MS = 30000;
    
    public StockPriceSubscriber(String host, int port) {
        this.channel = ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext().build();
        this.asyncStub = StockServiceGrpc.newStub(channel);
        this.scheduler = Executors.newScheduledThreadPool(1, r -> {
            Thread t = new Thread(r, "StockReconnector");
            t.setDaemon(true);
            return t;
        });
        this.processingExecutor = Executors.newFixedThreadPool(2);
    }

    public void subscribe(String symbol) {
        subscribeInternal(symbol);
    }

    private void subscribeInternal(String symbol) {
        SubscribeRequest request = SubscribeRequest.newBuilder()
            .setSymbol(symbol).build();
        
        asyncStub.subscribeStockUpdates(request, 
            new StockStreamObserver(symbol));
    }

    private class StockStreamObserver implements StreamObserver<StockPrice> {
        private final String symbol;
        
        public StockStreamObserver(String symbol) {
            this.symbol = symbol;
            retryCount = 0;
        }
        
        @Override
        public void onNext(StockPrice price) {
            // UI 업데이트 빠르게 (밀리초 단위)
            updateUIQuickly(price);
            
            // 무거운 작업은 별도 스레드에서 처리
            processingExecutor.submit(() -> {
                try {
                    persistPrice(price);
                    notifyAnalyticsService(price);
                } catch (Exception e) {
                    log.error("Error processing price for {}", symbol, e);
                }
            });
        }

        @Override
        public void onError(Throwable t) {
            log.error("Stream error for {}: {}", symbol, t.getMessage());
            
            if (shouldReconnect) {
                // Exponential Backoff: 1초, 2초, 4초, ... (최대 30초)
                long delayMs = Math.min(MAX_RETRY_DELAY_MS,
                    1000L * (long) Math.pow(2, retryCount));
                retryCount++;
                
                log.info("Reconnecting to {} in {}ms (attempt {})", 
                    symbol, delayMs, retryCount);
                
                scheduler.schedule(() -> subscribeInternal(symbol),
                    delayMs, TimeUnit.MILLISECONDS);
            }
        }

        @Override
        public void onCompleted() {
            log.info("Stream completed for {}", symbol);
        }
    }

    private void updateUIQuickly(StockPrice price) {
        // UI 업데이트 로직
    }

    private void persistPrice(StockPrice price) {
        // DB에 저장 (시간 소요 가능)
    }

    private void notifyAnalyticsService(StockPrice price) {
        // 분석 서비스에 전송
    }

    public void unsubscribe() {
        shouldReconnect = false;
        channel.shutdownNow();
        scheduler.shutdownNow();
        processingExecutor.shutdownNow();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. HTTP/2 Frame 흐름 — 단일 요청으로 N개 응답 받는 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP/2 Connection (TCP)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    Stream 1              Stream 2              Stream 3
   (AAPL 구독)          (GOOGL 구독)          (MSFT 구독)
         │                    │                    │
         │ CLIENT HEADERS     │                    │
         │ Stream ID=1        │                    │
    ┌────►SubscribeRequest    │                    │
    │    {symbol:"AAPL"}      │                    │
    │                         │                    │
    │  [Background: 100ms마다 가격 업데이트]      │
    │                         │                    │
    │ ◄────────────────────────────────────────────┤
    │ SERVER HEADERS+DATA                          │
    │ StockPrice{AAPL, 150.25}                     │
    │                         │                    │
    │             ┌──────────►│ HEADERS(Stream ID=2)
    │             │           │ SubscribeRequest
    │             │           │ {symbol:"GOOGL"}
    │             │           │
    │ ◄───────────────────────────────────────────┤
    │ DATA Stream 1: StockPrice{AAPL, 150.26}    │
    │             │           │
    │             │ ◄─────────┤─────────────────── DATA
    │             │           StockPrice{GOOGL, 2850.50}
    │             │           │
    │ [모든 스트림이 같은 TCP 연결에서 멀티플렉싱됨]
    │             │           │
    │ ◄───────────────────────────────────────────┤
    │ DATA(Stream ID=1): StockPrice{AAPL, 150.27}│
    │             │           │
    │ ◄───────────────────────────────────────────┤
    │ DATA(Stream ID=2): StockPrice{GOOGL, 2850.51}
    │
    [... 100ms마다 계속 푸시 ...]


프레임 구조:
┌──────────────────────────────────────────┐
│ Frame Header (9 bytes)                   │
├──────────────────────────────────────────┤
│ Length (24 bits): 데이터 크기             │
│ Type (8 bits): HEADERS(0x1), DATA(0x0)  │
│ Flags (8 bits): END_STREAM, END_HEADERS │
│ Stream ID (32 bits): 1, 2, 3, ...       │
├──────────────────────────────────────────┤
│ Payload (가변길이)                        │
│ HEADERS: HPACK으로 인코딩된 메타데이터    │
│ DATA: Protocol Buffer 바이너리           │
└──────────────────────────────────────────┘
```

### 2. Polling vs Server Streaming 비교

```
POLLING (REST)
──────────────
T=0s:   Client ──► GET /api/stock/AAPL ──► Server
        TCP 3-way handshake: 100ms
        Query time: 20ms
        Total: 120ms

T=1s:   Client ──► GET /api/stock/AAPL ──► Server (동일 반복)
        응답: {price: 150.25} (변화 없음) → 낭비!

T=10s:  100개 클라이언트 × 10개 요청 = 1000개 RPC
        총 대기 시간: 120ms × 1000 = 120초 (낭비)


SERVER STREAMING (gRPC)
──────────────────────
T=0s:   Client ──► HEADERS(Stream ID=1) ──► Server
        TCP 3-way: 100ms (단 한 번!)
        연결 상태 유지

T=0.1s: Server ──► DATA(StockPrice{AAPL, 150.26}) ──► Client
        지연: 5ms (네트워크 지연만)

T=0.2s: Server ──► DATA(StockPrice{AAPL, 150.27}) ──► Client
        지연: 5ms

T=10s:  100개 클라이언트 × 1개 연결 = 1개 TCP!
        모든 메시지가 같은 연결에서 처리


성능 비교:
┌────────────────────┬──────────────────┬──────────────────────┐
│ 메트릭              │ Polling (REST)   │ Server Streaming     │
├────────────────────┼──────────────────┼──────────────────────┤
│ 평균 지연           │ 1000ms (주기)    │ 100ms (즉시)         │
│ 연결 수/100클라이언트│ 100개            │ 1개 (멀티플렉싱)     │
│ 초당 RPC 호출       │ 100 (주기마다)   │ 10,000 (이벤트)      │
│ 네트워크 헤더       │ 1MB/s (오버헤드) │ 50KB/s (데이터)      │
│ 서버 스레드 풀      │ 100 (동시)       │ 1 (브로드캐스트)     │
│ 불필요한 응답      │ 99%              │ 0%                   │
└────────────────────┴──────────────────┴──────────────────────┘
```

### 3. 주식 시세 구독 서버 구현

```java
/**
 * 실시간 주식 시세 스트리밍 서버
 * 핵심: 각 심볼의 구독자들에게 100ms마다 최신 가격 푸시
 */
public class StockServiceImpl extends StockServiceGrpc.StockServiceImplBase {
    
    private static final long PUBLISH_INTERVAL_MS = 100;
    private final StockDataSource stockData;
    private final ConcurrentHashMap<String, List<StreamObserver<StockPrice>>> subscribers;
    private final ScheduledExecutorService broadcastExecutor;
    
    public StockServiceImpl(StockDataSource stockData) {
        this.stockData = stockData;
        this.subscribers = new ConcurrentHashMap<>();
        this.broadcastExecutor = Executors.newScheduledThreadPool(1);
    }

    /**
     * Server Streaming RPC
     * 클라이언트가 한 번 호출하면 서버가 계속해서 메시지를 push한다
     */
    @Override
    public void subscribeStockUpdates(
            SubscribeRequest request,
            StreamObserver<StockPrice> responseObserver) {
        
        String symbol = request.getSymbol();
        
        // 유효성 검사
        if (symbol == null || symbol.isEmpty()) {
            responseObserver.onError(Status.INVALID_ARGUMENT
                .withDescription("Symbol is required").asException());
            return;
        }
        
        if (!stockData.isValidSymbol(symbol)) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("Symbol not found: " + symbol).asException());
            return;
        }
        
        // 구독자 목록에 추가 (동시성 안전 - CopyOnWriteArrayList)
        List<StreamObserver<StockPrice>> symbolSubscribers = 
            subscribers.computeIfAbsent(symbol, k -> new CopyOnWriteArrayList<>());
        symbolSubscribers.add(responseObserver);
        
        log.info("New subscriber for symbol: {}, total: {}", 
            symbol, symbolSubscribers.size());
        
        // 즉시 현재 가격 전송
        try {
            StockPrice currentPrice = stockData.getLatestPrice(symbol);
            responseObserver.onNext(currentPrice);
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL
                .withDescription("Failed to get current price").asException());
            symbolSubscribers.remove(responseObserver);
        }
    }

    /**
     * 백그라운드 브로드캐스트 시작
     */
    public void startBroadcasting() {
        broadcastExecutor.scheduleAtFixedRate(
            this::broadcastPriceUpdates,
            PUBLISH_INTERVAL_MS,
            PUBLISH_INTERVAL_MS,
            TimeUnit.MILLISECONDS
        );
        log.info("Broadcasting started at {} ms interval", PUBLISH_INTERVAL_MS);
    }

    private void broadcastPriceUpdates() {
        Map<String, StockPrice> priceMap = stockData.getAllLatestPrices();
        
        for (Map.Entry<String, StockPrice> entry : priceMap.entrySet()) {
            String symbol = entry.getKey();
            StockPrice price = entry.getValue();
            
            List<StreamObserver<StockPrice>> symbolSubscribers = 
                subscribers.get(symbol);
            
            if (symbolSubscribers != null && !symbolSubscribers.isEmpty()) {
                // CopyOnWriteArrayList이므로 iteration 중 remove 안전
                for (StreamObserver<StockPrice> observer : symbolSubscribers) {
                    try {
                        observer.onNext(price);
                    } catch (Exception e) {
                        log.debug("Failed to send to subscriber: {}", 
                            e.getMessage());
                        symbolSubscribers.remove(observer);
                        
                        try {
                            observer.onError(Status.UNAVAILABLE
                                .withDescription("Connection lost").asException());
                        } catch (Exception ignored) {
                            // observer 종료 상태
                        }
                    }
                }
            }
        }
    }

    public void shutdown() {
        broadcastExecutor.shutdown();
        try {
            if (!broadcastExecutor.awaitTermination(5, TimeUnit.SECONDS)) {
                broadcastExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            broadcastExecutor.shutdownNow();
        }
    }
}

// 서버 시작
public class StockServiceServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        StockDataSource dataSource = new InMemoryStockDataSource();
        StockServiceImpl service = new StockServiceImpl(dataSource);
        
        Server server = ServerBuilder.forPort(50051)
            .addService(service)
            .build();
        
        server.start();
        log.info("Server started on port 50051");
        
        service.startBroadcasting();
        
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            service.shutdown();
            server.shutdown();
        }));
        
        server.awaitTermination();
    }
}
```

### 4. 클라이언트 수신 및 Backpressure 처리

```java
/**
 * Server Streaming 클라이언트
 * 핵심: onNext()는 빠르게, 무거운 작업은 별도 스레드로
 */
public class StockPriceSubscriber {
    
    private final ManagedChannel channel;
    private final StockServiceGrpc.StockServiceStub asyncStub;
    private final ScheduledExecutorService scheduler;
    private final ExecutorService processingExecutor;
    private volatile boolean shouldReconnect = true;
    private int retryCount = 0;
    private static final int MAX_RETRY_DELAY_MS = 30000;
    
    public StockPriceSubscriber(String host, int port) {
        this.channel = ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext().build();
        this.asyncStub = StockServiceGrpc.newStub(channel);
        this.scheduler = Executors.newScheduledThreadPool(1);
        this.processingExecutor = Executors.newFixedThreadPool(2);
    }

    public void subscribe(String symbol) {
        subscribeInternal(symbol);
    }

    private void subscribeInternal(String symbol) {
        SubscribeRequest request = SubscribeRequest.newBuilder()
            .setSymbol(symbol).build();
        
        log.info("Subscribing to {}", symbol);
        asyncStub.subscribeStockUpdates(request, 
            new StockStreamObserver(symbol));
    }

    /**
     * StreamObserver 구현
     * gRPC가 onNext, onError, onCompleted를 콜백으로 호출함
     */
    private class StockStreamObserver implements StreamObserver<StockPrice> {
        private final String symbol;
        
        public StockStreamObserver(String symbol) {
            this.symbol = symbol;
            retryCount = 0; // 성공하면 리트라이 카운트 초기화
        }
        
        /**
         * 서버가 메시지를 보낼 때마다 호출
         * 주의: 이 메서드는 빨리 반환해야 한다 (Backpressure)
         */
        @Override
        public void onNext(StockPrice price) {
            // 1. 즉시 UI 업데이트 (밀리초 단위)
            updateUIQuickly(price);
            
            // 2. 무거운 작업은 별도 스레드로 (Backpressure 회피)
            processingExecutor.submit(() -> {
                try {
                    // DB 저장, 분석 등 시간 소요 작업
                    persistPrice(price);
                    notifyAnalyticsService(price);
                } catch (Exception e) {
                    log.error("Error processing price for {}", symbol, e);
                }
            });
        }

        /**
         * 스트림 에러 발생
         * 자동으로 Exponential Backoff 재연결 시도
         */
        @Override
        public void onError(Throwable t) {
            log.error("Stream error for {}: {}", symbol, t.getMessage());
            
            if (shouldReconnect) {
                // Exponential Backoff: 1s, 2s, 4s, ... (최대 30초)
                long delayMs = Math.min(
                    MAX_RETRY_DELAY_MS,
                    1000L * (long) Math.pow(2, retryCount)
                );
                retryCount++;
                
                log.info("Reconnecting to {} in {}ms (attempt {})", 
                    symbol, delayMs, retryCount);
                
                scheduler.schedule(
                    () -> subscribeInternal(symbol),
                    delayMs,
                    TimeUnit.MILLISECONDS
                );
            }
        }

        /**
         * 스트림 정상 완료
         * 클라이언트가 명시적으로 구독 취소한 경우
         */
        @Override
        public void onCompleted() {
            log.info("Stream completed for {}", symbol);
        }
    }

    /**
     * HTTP/2 Backpressure 처리
     * 
     * gRPC는 HTTP/2 Flow Control을 사용하여
     * 클라이언트가 처리할 수 없을 정도로 빠르게 메시지를 보내지 않는다.
     * 
     * 만약 onNext()에서 복잡한 작업을 하면:
     * 1. processingExecutor가 가득 참
     * 2. onNext() 호출이 blocking됨
     * 3. HTTP/2 수신 버퍼가 차기 시작
     * 4. Server가 WINDOW_UPDATE를 받지 못함 → 서버도 느려짐
     * 
     * 따라서 onNext()는 최대한 빠르게 반환하고,
     * 무거운 작업은 별도 스레드에서 처리하는 것이 중요!
     */

    private void updateUIQuickly(StockPrice price) {
        // 메인 스레드에 UI 업데이트 요청 (즉시 반환)
        log.debug("UI update: {} = ${}", price.getSymbol(), price.getPrice());
    }

    private void persistPrice(StockPrice price) {
        // DB에 저장 (시간 소요)
        log.debug("Persisting price to DB: {}", price.getSymbol());
    }

    private void notifyAnalyticsService(StockPrice price) {
        // 분석 서비스에 전송 (네트워크 I/O)
        log.debug("Notifying analytics: {}", price.getSymbol());
    }

    public void unsubscribe() {
        shouldReconnect = false;
        channel.shutdownNow();
        scheduler.shutdownNow();
        processingExecutor.shutdownNow();
    }
}

// 실행 예시
public class StockApp {
    public static void main(String[] args) {
        StockPriceSubscriber subscriber = 
            new StockPriceSubscriber("localhost", 50051);
        
        // 여러 심볼 동시 구독 (HTTP/2 멀티플렉싱)
        subscriber.subscribe("AAPL");
        subscriber.subscribe("GOOGL");
        subscriber.subscribe("MSFT");
        
        // 앱이 종료될 때까지 구독 유지
        Runtime.getRuntime().addShutdownHook(new Thread(subscriber::unsubscribe));
    }
}
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# Server Streaming 실시간 테스트

# 1. Protobuf 파일 생성
cat > stock.proto << 'EOF'
syntax = "proto3";

package stock;

service StockService {
    rpc SubscribeStockUpdates(SubscribeRequest) returns (stream StockPrice) {}
}

message SubscribeRequest {
    string symbol = 1;
}

message StockPrice {
    string symbol = 1;
    double price = 2;
    int64 timestamp_ms = 3;
}
EOF

# 2. gRPC 코드 생성
protoc --java_out=. --grpc-java_out=. stock.proto

# 3. 서버 컴파일 및 실행
javac -cp grpc-all.jar StockServiceImpl.java
java -cp grpc-all.jar:. StockServiceServer &

# 4. grpcurl로 테스트 (별도 터미널)
grpcurl -plaintext \
  -d '{"symbol": "AAPL"}' \
  localhost:50051 \
  stock.StockService/SubscribeStockUpdates

# 출력:
# {
#   "symbol": "AAPL",
#   "price": 150.25,
#   "timestamp_ms": "1618000000000"
# }
# {
#   "symbol": "AAPL",
#   "price": 150.26,
#   "timestamp_ms": "1618000000100"
# }
# ... (Ctrl+C로 중단)

# 5. 네트워크 트래픽 분석
tcpdump -i lo 'port 50051' -A | grep -E "HEADERS|DATA"

# 6. 멀티클라이언트 테스트 (병렬 구독)
for i in {1..10}; do
  grpcurl -plaintext -d '{"symbol": "AAPL"}' \
    localhost:50051 stock.StockService/SubscribeStockUpdates &
done
wait
```

---

## 📊 성능/비용 비교

```
Polling (1s interval) vs Server Streaming (100ms push)
가정: 100개 클라이언트, 10개 심볼, 24시간 운영

┌────────────────────────────────┬──────────────┬────────────────┐
│ 메트릭                          │ Polling      │ Server Stream  │
├────────────────────────────────┼──────────────┼────────────────┤
│ 시간당 RPC 호출                │ 36,000,000  │ 864,000,000    │
│                                │ (주기마다)   │ (이벤트)       │
├────────────────────────────────┼──────────────┼────────────────┤
│ 평균 데이터 지연                │ 550ms        │ 50ms           │
│ (쿼리 완료부터 UI까지)         │              │                │
├────────────────────────────────┼──────────────┼────────────────┤
│ 서버 TCP 연결 수                │ 100개        │ 1개            │
│ (메모리, 파일디스크립터)       │              │ (멀티플렉싱)   │
├────────────────────────────────┼──────────────┼────────────────┤
│ 네트워크 대역폭 (HTTP/2 헤더)  │ 1MB/s        │ 50KB/s         │
│ (필요 없는 트래픽)             │              │                │
├────────────────────────────────┼──────────────┼────────────────┤
│ 서버 스레드 풀 활용            │ 100 (동시)   │ 1 (브로드캐스트)
│ CPU 컨텍스트 스위칭 오버헤드   │ 높음         │ 낮음           │
├────────────────────────────────┼──────────────┼────────────────┤
│ 호스팅 비용                    │ $500/월      │ $50/월         │
│ (AWS EC2 CPU 기준)            │              │                │
├────────────────────────────────┼──────────────┼────────────────┤
│ 다운링크 비용                   │ $10,000/월   │ $500/월        │
│ (100GB/일 기준)                │              │                │
└────────────────────────────────┴──────────────┴────────────────┘

연간 절감액: $115,800 (호스팅 + 다운링크 비용)
```

---

## ⚖️ 트레이드오프

```
✅ 장점
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 실시간성
   - Polling: 1초 주기 → 최대 1초 지연
   - Server Streaming: 이벤트 발생 시 즉시 → <100ms

2. 네트워크 효율
   - TCP 3-way handshake 불필요 (연결 유지)
   - HTTP/2 헤더 압축 + 바이너리 프로토콜
   → 데이터 1/20 크기

3. 서버 부하 감소
   - 100개 클라이언트 → 100개 HTTP 스레드
   - 100개 클라이언트 → 1개 브로드캐스트 루프
   → CPU 사용률 90% 감소

4. 스케일러빌리티
   - Polling: 클라이언트 증가 → 부하 선형 증가
   - Server Streaming: 브로드캐스트 한 번에 모두 처리

5. 비용 절감
   - EC2: $500/월 → $50/월 (10배)
   - 데이터 트래픽: $10,000/월 → $500/월 (20배)


❌ 제약사항 및 고려사항
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 연결 관리 복잡성
   - 문제: 네트워크 끊김, 서버 재시작 감지 필요
   - 해결: Exponential Backoff 재연결 로직 필수
   - 코드 증가: ~50줄 추가

2. 메모리 버퍼링
   - 문제: 느린 클라이언트 → 서버의 메모리 누적
   - 해결: HTTP/2 Flow Control (자동)

3. 방화벽/프록시 호환성
   - REST: 모든 프록시가 지원 (HTTP/1.1 표준)
   - gRPC: HTTP/2 지원하는 프록시만 가능
   - 실제: AWS ALB, nginx 1.13+ 등 대부분 지원

4. 모니터링 복잡성
   - REST: HTTP 상태 코드로 명확한 성공/실패
   - gRPC: StreamObserver 상태 추적 필요
   - 해결: interceptor로 메트릭 수집

5. 개발 학습곡선
   - REST: 단순 request-response
   - gRPC: StreamObserver 콜백 패러다임
   - 처음: 어려움, 익숙해지면 더 우아함
```

---

## 📌 핵심 정리

```
Server Streaming은 "변경이 있을 때만 알려주는" 패턴이다.

1. HTTP/2 위에서 동작
   ├─ 단일 TCP 연결 유지
   ├─ 100ms마다 메시지 푸시
   └─ 여러 스트림 멀티플렉싱

2. Polling vs Server Streaming
   ├─ Polling: 1초 × 100 클라이언트 = 100 RPC/s
   └─ Server Streaming: 이벤트 발생 시만 (효율 극대)

3. 실전 구현
   ├─ 서버: StreamObserver 콜렉션 관리
   ├─ 클라이언트: onNext/onError/onCompleted 콜백
   └─ 핵심: onNext()는 빨리 반환

4. 자동 Backpressure
   ├─ HTTP/2 Flow Control (Window Size 기반)
   └─ 느린 클라이언트가 서버 속도 제어

5. 성능 효과
   ├─ 지연: 1000ms → 50ms (20배)
   ├─ 연결: 100개 → 1개 (100배)
   └─ 비용: 절감액 $115,800/년

6. 에러 처리
   ├─ onError() → Exponential Backoff 재연결
   ├─ onCompleted() → 정상 종료
   └─ 자동 감지: 네트워크 끊김 (50ms 내)
```

---

## 🤔 생각해볼 문제

### Q1: 서버가 100ms마다 1000개 메시지를 100개 클라이언트에게 전송할 때, HTTP/2 Frame 레벨에서 정확히 무슨 일이 일어나는가?

<details>
<summary>해설 보기</summary>

각 onNext() 호출마다 gRPC가 프로토콜버퍼를 직렬화하고 Netty가 DATA Frame을 생성합니다. 이 모든 프레임(Stream ID별로 구분됨)이 단일 TCP 연결 위에서 multiplexing되어 전송됩니다. 서버는 100ms마다 1000개의 분리된 DATA Frame을 생성하지만, TCP 입장에서는 "1개의 연결"입니다. 클라이언트는 Stream ID로 자신의 메시지만 추출합니다. 네트워크 레벨에서는 약 30-40개의 TCP Segment로 집계되어 전송됩니다 (각 segment는 MTU 1500바이트).

Timeline:
```
T=0ms: 서버가 메시지 1-1000 생성
T=1ms: Netty가 1000개 DATA frame 생성
T=5ms: TCP 송신 버퍼에 ~30개 TCP segment로 집계
T=10ms: 첫 번째 segment 전송 (3-way handshake 불필요, 연결 재사용)
T=15ms: 클라이언트 수신 및 HTTP/2 디코딩
T=20ms: 각 Stream별로 onNext() 콜백
```

반면 Polling이면:
- 1000개 HTTP Request = 1000개 TCP 연결 필요
- 각 연결마다 3-way handshake (100ms × 1000 = 100초!)
</details>

### Q2: 클라이언트가 처리 속도가 느려서 100ms마다 받는 메시지를 200ms에 처리하면, Backpressure는 어떻게 작동하는가?

<details>
<summary>해설 보기</summary>

onNext()에서 반환되지 않으면 (또는 오래 걸리면) gRPC의 수신 버퍼가 찹니다. HTTP/2 Window Size(기본값 64KB)에 도달하면 서버가 WINDOW_UPDATE 프레임을 받을 때까지 send()가 block됩니다. 결과적으로 느린 클라이언트가 서버의 속도를 자동으로 제어합니다. 

해결책: 무거운 작업은 별도 스레드(processingExecutor)로 처리하여 onNext()가 빨리 반환되도록 해야 합니다. onNext()에서 200ms이 소요되면 안 되고, 5ms 이내에 반환되어야 버퍼링이 누적되지 않습니다.

```
BufferState:
T=0ms: Window=64KB, Message 1 수신, onNext() 시작
T=100ms: Window=64KB (onNext() 여전히 진행 중 - 버퍼 차기 시작)
T=150ms: Message 2 도착 → 버퍼에만 저장 (onNext() 미완료)
T=200ms: onNext() 완료 → WINDOW_UPDATE 전송 → 다음 메시지 처리
```
</details>

### Q3: 서버가 갑자기 재시작되면, 클라이언트는 언제 그 사실을 알고, onError()는 얼마나 빨리 호출되는가?

<details>
<summary>해설 보기</summary>

서버가 TCP FIN(graceful) 또는 RST(abrupt) 패킷을 보내면 클라이언트의 OS kernel이 50-100ms 내에 수신합니다. gRPC 라이브러리가 EOF 또는 connection reset을 감지하면 즉시 onError(StatusRuntimeException)를 호출합니다.

Timeline:
```
T=5.0s: 서버 프로세스 SIGTERM (재시작)
T=5.05s: OS kernel이 TCP FIN/RST 패킷 수신
T=5.06s: gRPC가 EOF 감지
T=5.06s: onError(Status.UNAVAILABLE) 콜백 호출 ← 클라이언트 인지
T=5.07s: User code의 onError() 실행
T=5.07s: Exponential Backoff 계산 (첫 시도: 1초)
T=6.07s: 재연결 시도
T=8.07s: 서버 재시작 완료 → 연결 성공
```

주의: Keep-Alive가 설정되지 않으면 "조용한 시간초과"(silent timeout)에 최대 2시간이 걸릴 수 있습니다. 따라서 keepAliveInterval을 30초 정도로 설정하는 것이 필수입니다.
</details>

---

<div align="center">

**[⬅️ 이전: Buf Schema Registry](../service-design/06-buf-schema-registry.md)** | **[홈으로 🏠](../README.md)** | **[다음: Client Streaming ➡️](./02-client-streaming.md)**

</div>
