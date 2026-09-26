# 📕 CẨM NANG SINH TỒN JAVA CORE & OOP THỰC CHIẾN

## PHẦN 5: BÀI TEST TƯ DUY TỔNG HỢP — MOCK INTERVIEW

> *"Phỏng vấn không kiểm tra bạn biết bao nhiêu. Nó kiểm tra bạn TƯ DUY thế nào khi đối mặt với bài toán chưa từng gặp."*

---

## 📋 HƯỚNG DẪN CHUẨN STAR CHO CÂU TRẢ LỜI KỸ THUẬT

| Bước | Ý nghĩa | Áp dụng vào code |
|------|---------|------------------|
| **S** — Situation | Bối cảnh, yêu cầu bài toán | Đọc đề, xác định input/output, edge case |
| **T** — Task | Nhiệm vụ cụ thể của bạn | Chọn cấu trúc dữ liệu, design pattern |
| **A** — Action | Hành động chi tiết, giải thích TẠI SAO | Code, giải thích từng quyết định thiết kế |
| **R** — Result | Kết quả, đánh giá độ phức tạp | Phân tích O(n), mở rộng, clean code |

---

## 🧪 BÀI 1: THIẾT KẾ HỆ THỐNG RETRY TỰ ĐỘNG CHO PAYMENT GATEWAY

### 📝 ĐỀ BÀI

> *"Bạn đang xây dựng hệ thống thanh toán. Payment Gateway bên thứ 3 (Momo, VNPay) thỉnh thoảng trả về lỗi tạm thời (timeout, 503 Service Unavailable). Hãy thiết kế cơ chế retry thông minh:*
> 1. *Retry tối đa N lần*
> 2. *Mỗi lần retry chờ lâu hơn (Exponential Backoff)*
> 3. *Phân biệt lỗi retryable (timeout) vs non-retryable (invalid card)*
> 4. *Code phải tuân thủ OCP — thêm chiến lược retry mới không sửa code cũ"*

### 🔍 PHÂN TÍCH THEO STAR

**Situation**: Payment Gateway không ổn định, cần retry thông minh để tránh giao dịch thất bại không cần thiết. Nhưng không phải lỗi nào cũng nên retry (ví dụ: thẻ hết hạn → retry bao nhiêu lần cũng thất bại).

**Task**: Thiết kế hệ thống retry có:
- Chiến lược backoff (chờ lâu dần)
- Phân loại exception (retryable vs non-retryable)
- Tách biệt logic retry khỏi logic nghiệp vụ (SRP)
- Mở rộng dễ dàng (OCP)

**Action**: Áp dụng Strategy Pattern cho retry policy + Template Method cho retry executor.

```java
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public class RetrySystemDemo {
    public static void main(String[] args) {
        System.out.println("╔══════════════════════════════════════════╗");
        System.out.println("║  BÀI 1: PAYMENT RETRY SYSTEM            ║");
        System.out.println("╚══════════════════════════════════════════╝\n");

        // Cấu hình retry: 3 lần, exponential backoff bắt đầu từ 500ms
        RetryPolicy policy = new ExponentialBackoffPolicy(3, 500);
        RetryExecutor retryExecutor = new RetryExecutor(policy);

        // ===== Case 1: Lỗi tạm thời → retry thành công =====
        System.out.println("=== CASE 1: Lỗi tạm thời (retry thành công lần 3) ===");
        SimulatedPaymentGateway gateway1 = new SimulatedPaymentGateway(2); // Fail 2 lần đầu

        String result1 = retryExecutor.executeWithRetry(
            () -> gateway1.charge("TXN-001", 500_000),
            "Payment TXN-001"
        );
        System.out.println("→ Kết quả: " + result1 + "\n");

        // ===== Case 2: Lỗi vĩnh viễn (non-retryable) → dừng ngay =====
        System.out.println("=== CASE 2: Lỗi non-retryable (thẻ hết hạn) ===");
        SimulatedPaymentGateway gateway2 = new SimulatedPaymentGateway(-1); // Luôn lỗi card

        String result2 = retryExecutor.executeWithRetry(
            () -> gateway2.chargeWithCardError("TXN-002", 200_000),
            "Payment TXN-002"
        );
        System.out.println("→ Kết quả: " + result2 + "\n");

        // ===== Case 3: Vượt quá số lần retry =====
        System.out.println("=== CASE 3: Vượt quá max retry ===");
        SimulatedPaymentGateway gateway3 = new SimulatedPaymentGateway(10); // Fail 10 lần

        String result3 = retryExecutor.executeWithRetry(
            () -> gateway3.charge("TXN-003", 1_000_000),
            "Payment TXN-003"
        );
        System.out.println("→ Kết quả: " + result3);
    }
}

// ===== EXCEPTION HIERARCHY: Phân loại lỗi =====

// Lỗi CÓ THỂ retry (timeout, 503, network)
class RetryableException extends RuntimeException {
    RetryableException(String message) { super(message); }
}

// Lỗi KHÔNG retry được (thẻ hết hạn, số tiền không hợp lệ)
class NonRetryableException extends RuntimeException {
    NonRetryableException(String message) { super(message); }
}

// ===== STRATEGY PATTERN: Retry Policy =====

interface RetryPolicy {
    int getMaxRetries();
    long getDelayMs(int attemptNumber); // Thời gian chờ cho lần retry thứ n
}

// Chiến lược Exponential Backoff: chờ 500ms, 1000ms, 2000ms, 4000ms...
class ExponentialBackoffPolicy implements RetryPolicy {
    private final int maxRetries;
    private final long baseDelayMs;

    ExponentialBackoffPolicy(int maxRetries, long baseDelayMs) {
        this.maxRetries = maxRetries;
        this.baseDelayMs = baseDelayMs;
    }

    @Override
    public int getMaxRetries() { return maxRetries; }

    @Override
    public long getDelayMs(int attemptNumber) {
        // 500 * 2^0 = 500ms, 500 * 2^1 = 1000ms, 500 * 2^2 = 2000ms
        return baseDelayMs * (long) Math.pow(2, attemptNumber);
    }
}

// ===== RETRY EXECUTOR: Logic retry tách biệt khỏi nghiệp vụ (SRP) =====

class RetryExecutor {
    private final RetryPolicy policy;

    RetryExecutor(RetryPolicy policy) {
        this.policy = policy;
    }

    public <T> T executeWithRetry(Supplier<T> task, String taskName) {
        int attempt = 0;

        while (true) {
            try {
                attempt++;
                System.out.println("   [Attempt " + attempt + "] Executing " + taskName + "...");
                T result = task.get(); // Thực thi task
                System.out.println("   ✅ Success on attempt " + attempt);
                return result;

            } catch (NonRetryableException e) {
                // Lỗi không thể retry → DỪNG NGAY, không thử lại
                System.out.println("   🚫 Non-retryable error: " + e.getMessage());
                System.out.println("   → Dừng ngay, không retry");
                return "FAILED_NON_RETRYABLE: " + e.getMessage();

            } catch (RetryableException e) {
                // Lỗi có thể retry → kiểm tra còn lượt không
                if (attempt >= policy.getMaxRetries()) {
                    System.out.println("   ❌ Max retries (" + policy.getMaxRetries()
                                     + ") reached. Last error: " + e.getMessage());
                    return "FAILED_MAX_RETRIES: " + e.getMessage();
                }

                long delay = policy.getDelayMs(attempt - 1);
                System.out.println("   ⚠ Retryable error: " + e.getMessage());
                System.out.println("   ⏳ Waiting " + delay + "ms before retry...");

                // Thực tế dùng Thread.sleep(delay), ở đây giả lập để demo nhanh
                simulateWait(delay);
            }
        }
    }

    private void simulateWait(long ms) {
        // Giả lập chờ (in ra thay vì sleep thật để demo nhanh)
        System.out.println("   ... (chờ " + ms + "ms)");
    }
}

// ===== SIMULATED GATEWAY =====

class SimulatedPaymentGateway {
    private int failuresBeforeSuccess;
    private int callCount = 0;

    SimulatedPaymentGateway(int failuresBeforeSuccess) {
        this.failuresBeforeSuccess = failuresBeforeSuccess;
    }

    // Lỗi tạm thời (retryable)
    public String charge(String txnId, double amount) {
        callCount++;
        if (callCount <= failuresBeforeSuccess) {
            throw new RetryableException("Gateway timeout (503) for " + txnId);
        }
        return txnId + " → CHARGED " + String.format("%,.0f", amount) + "đ";
    }

    // Lỗi vĩnh viễn (non-retryable)
    public String chargeWithCardError(String txnId, double amount) {
        throw new NonRetryableException("Card expired for " + txnId);
    }
}
// Output:
// ╔══════════════════════════════════════════╗
// ║  BÀI 1: PAYMENT RETRY SYSTEM            ║
// ╚══════════════════════════════════════════╝
//
// === CASE 1: Lỗi tạm thời (retry thành công lần 3) ===
//    [Attempt 1] Executing Payment TXN-001...
//    ⚠ Retryable error: Gateway timeout (503) for TXN-001
//    ⏳ Waiting 500ms before retry...
//    ... (chờ 500ms)
//    [Attempt 2] Executing Payment TXN-001...
//    ⚠ Retryable error: Gateway timeout (503) for TXN-001
//    ⏳ Waiting 1000ms before retry...
//    ... (chờ 1000ms)
//    [Attempt 3] Executing Payment TXN-001...
//    ✅ Success on attempt 3
// → Kết quả: TXN-001 → CHARGED 500,000đ
//
// === CASE 2: Lỗi non-retryable (thẻ hết hạn) ===
//    [Attempt 1] Executing Payment TXN-002...
//    🚫 Non-retryable error: Card expired for TXN-002
//    → Dừng ngay, không retry
// → Kết quả: FAILED_NON_RETRYABLE: Card expired for TXN-002
//
// === CASE 3: Vượt quá max retry ===
//    [Attempt 1] Executing Payment TXN-003...
//    ⚠ Retryable error: Gateway timeout (503) for TXN-003
//    ⏳ Waiting 500ms before retry...
//    ... (chờ 500ms)
//    [Attempt 2] Executing Payment TXN-003...
//    ⚠ Retryable error: Gateway timeout (503) for TXN-003
//    ⏳ Waiting 1000ms before retry...
//    ... (chờ 1000ms)
//    [Attempt 3] Executing Payment TXN-003...
//    ❌ Max retries (3) reached. Last error: Gateway timeout (503) for TXN-003
// → Kết quả: FAILED_MAX_RETRIES: Gateway timeout (503) for TXN-003
```

**Result — Điểm đánh giá của Interviewer:**

| Tiêu chí | Đạt? | Ghi chú |
|----------|------|---------|
| Clean Code | ✅ | Tách riêng retry logic, policy, exception |
| OCP | ✅ | Thêm `FixedDelayPolicy`, `LinearBackoffPolicy` → tạo class mới, không sửa `RetryExecutor` |
| SRP | ✅ | `RetryExecutor` chỉ lo retry, `PaymentGateway` chỉ lo gọi API |
| Exception Design | ✅ | Phân loại retryable vs non-retryable rõ ràng |
| Edge Case | ✅ | Xử lý max retry, lỗi vĩnh viễn, thành công |

---

## 🧪 BÀI 2: THIẾT KẾ RATE LIMITER BẢO VỆ API

### 📝 ĐỀ BÀI

> *"API hệ thống đang bị spam request. Hãy thiết kế Rate Limiter đơn giản:*
> 1. *Mỗi user chỉ được gọi tối đa N request trong khoảng thời gian T giây*
> 2. *Request vượt giới hạn → trả về HTTP 429 (Too Many Requests)*
> 3. *Hỗ trợ nhiều user đồng thời (thread-safe)*
> 4. *Thuật toán: Sliding Window Counter"*

### 🔍 PHÂN TÍCH THEO STAR

**Situation**: API bị spam → server quá tải, ảnh hưởng user hợp lệ. Cần giới hạn tần suất request theo từng user.

**Task**: Implement Rate Limiter dùng Sliding Window — theo dõi timestamp của mỗi request, chỉ đếm request trong cửa sổ thời gian gần nhất.

**Action**: Dùng `ConcurrentHashMap` + `LinkedList<Long>` (timestamp queue) cho mỗi user. Thread-safe bằng `synchronized` trên per-user level.

```java
import java.util.*;
import java.util.concurrent.*;

public class RateLimiterDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("╔══════════════════════════════════════════╗");
        System.out.println("║  BÀI 2: API RATE LIMITER                ║");
        System.out.println("╚══════════════════════════════════════════╝\n");

        // Config: Mỗi user tối đa 5 requests trong 3 giây
        SlidingWindowRateLimiter limiter = new SlidingWindowRateLimiter(5, 3_000);

        ApiGateway gateway = new ApiGateway(limiter);

        // ===== CASE 1: 1 user gửi liên tục =====
        System.out.println("=== CASE 1: User USR-001 gửi 8 requests liên tục ===\n");
        for (int i = 1; i <= 8; i++) {
            String response = gateway.handleRequest("USR-001", "/api/orders");
            System.out.println("  Request " + i + ": " + response);
        }

        // ===== CASE 2: User khác không bị ảnh hưởng =====
        System.out.println("\n=== CASE 2: User USR-002 (user khác, quota riêng) ===\n");
        for (int i = 1; i <= 3; i++) {
            String response = gateway.handleRequest("USR-002", "/api/users");
            System.out.println("  Request " + i + ": " + response);
        }

        // ===== CASE 3: Chờ window hết hạn → gửi lại được =====
        System.out.println("\n=== CASE 3: Chờ 3s (window reset) → gửi lại ===\n");
        System.out.println("  ⏳ Waiting 3 seconds...");
        Thread.sleep(3_100); // Chờ window trôi qua

        for (int i = 1; i <= 3; i++) {
            String response = gateway.handleRequest("USR-001", "/api/orders");
            System.out.println("  Request " + i + ": " + response);
        }

        // ===== CASE 4: Multi-thread test =====
        System.out.println("\n=== CASE 4: Multi-thread stress test ===\n");
        SlidingWindowRateLimiter strictLimiter = new SlidingWindowRateLimiter(3, 2_000);
        ApiGateway strictGateway = new ApiGateway(strictLimiter);

        ExecutorService pool = Executors.newFixedThreadPool(5);
        List<Future<String>> results = new ArrayList<>();

        for (int i = 0; i < 6; i++) {
            final int reqNum = i + 1;
            results.add(pool.submit(() -> {
                String resp = strictGateway.handleRequest("USR-MT", "/api/data");
                return "  Thread-Request " + reqNum + ": " + resp;
            }));
        }

        for (Future<String> f : results) {
            System.out.println(f.get());
        }

        pool.shutdown();
    }
}

// ===== SLIDING WINDOW RATE LIMITER =====

class SlidingWindowRateLimiter {
    private final int maxRequests;      // Số request tối đa trong window
    private final long windowSizeMs;    // Kích thước window (ms)

    // Mỗi user có 1 queue chứa timestamp của các request gần đây
    // ConcurrentHashMap: thread-safe ở level put/get
    private final ConcurrentHashMap<String, LinkedList<Long>> requestLog = new ConcurrentHashMap<>();

    SlidingWindowRateLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
    }

    public boolean isAllowed(String userId) {
        long now = System.currentTimeMillis();

        // computeIfAbsent: Thread-safe tạo list nếu chưa có
        LinkedList<Long> timestamps = requestLog.computeIfAbsent(userId, k -> new LinkedList<>());

        // synchronized PER-USER: Chỉ lock queue của user đó, không ảnh hưởng user khác
        synchronized (timestamps) {
            // Loại bỏ timestamp cũ nằm NGOÀI window
            long windowStart = now - windowSizeMs;
            while (!timestamps.isEmpty() && timestamps.peekFirst() < windowStart) {
                timestamps.pollFirst(); // Xóa request hết hạn
            }

            // Kiểm tra quota
            if (timestamps.size() < maxRequests) {
                timestamps.addLast(now); // Ghi nhận request mới
                return true;  // ✅ Cho phép
            }

            return false; // ❌ Vượt giới hạn
        }
    }

    public int getRemainingQuota(String userId) {
        LinkedList<Long> timestamps = requestLog.getOrDefault(userId, new LinkedList<>());
        synchronized (timestamps) {
            long windowStart = System.currentTimeMillis() - windowSizeMs;
            while (!timestamps.isEmpty() && timestamps.peekFirst() < windowStart) {
                timestamps.pollFirst();
            }
            return Math.max(0, maxRequests - timestamps.size());
        }
    }
}

// ===== API GATEWAY — Sử dụng Rate Limiter =====

class ApiGateway {
    private final SlidingWindowRateLimiter rateLimiter;

    ApiGateway(SlidingWindowRateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }

    public String handleRequest(String userId, String endpoint) {
        if (!rateLimiter.isAllowed(userId)) {
            // HTTP 429 Too Many Requests
            return "❌ 429 TOO MANY REQUESTS (quota: " + rateLimiter.getRemainingQuota(userId) + " remaining)";
        }

        // Xử lý request bình thường
        return "✅ 200 OK → " + endpoint + " (quota: " + rateLimiter.getRemainingQuota(userId) + " remaining)";
    }
}
// Output:
// ╔══════════════════════════════════════════╗
// ║  BÀI 2: API RATE LIMITER                ║
// ╚══════════════════════════════════════════╝
//
// === CASE 1: User USR-001 gửi 8 requests liên tục ===
//
//   Request 1: ✅ 200 OK → /api/orders (quota: 4 remaining)
//   Request 2: ✅ 200 OK → /api/orders (quota: 3 remaining)
//   Request 3: ✅ 200 OK → /api/orders (quota: 2 remaining)
//   Request 4: ✅ 200 OK → /api/orders (quota: 1 remaining)
//   Request 5: ✅ 200 OK → /api/orders (quota: 0 remaining)
//   Request 6: ❌ 429 TOO MANY REQUESTS (quota: 0 remaining)
//   Request 7: ❌ 429 TOO MANY REQUESTS (quota: 0 remaining)
//   Request 8: ❌ 429 TOO MANY REQUESTS (quota: 0 remaining)
//
// === CASE 2: User USR-002 (user khác, quota riêng) ===
//
//   Request 1: ✅ 200 OK → /api/users (quota: 4 remaining)
//   Request 2: ✅ 200 OK → /api/users (quota: 3 remaining)
//   Request 3: ✅ 200 OK → /api/users (quota: 2 remaining)
//
// === CASE 3: Chờ 3s (window reset) → gửi lại ===
//
//   ⏳ Waiting 3 seconds...
//   Request 1: ✅ 200 OK → /api/orders (quota: 4 remaining)
//   Request 2: ✅ 200 OK → /api/orders (quota: 3 remaining)
//   Request 3: ✅ 200 OK → /api/orders (quota: 2 remaining)
//
// === CASE 4: Multi-thread stress test ===
//
//   Thread-Request 1: ✅ 200 OK → /api/data (quota: 2 remaining)
//   Thread-Request 2: ✅ 200 OK → /api/data (quota: 1 remaining)
//   Thread-Request 3: ✅ 200 OK → /api/data (quota: 0 remaining)
//   Thread-Request 4: ❌ 429 TOO MANY REQUESTS (quota: 0 remaining)
//   Thread-Request 5: ❌ 429 TOO MANY REQUESTS (quota: 0 remaining)
//   Thread-Request 6: ❌ 429 TOO MANY REQUESTS (quota: 0 remaining)
```

**Result — Điểm đánh giá:**

| Tiêu chí | Đạt? | Ghi chú |
|----------|------|---------|
| Thread Safety | ✅ | `ConcurrentHashMap` + `synchronized` per-user |
| Algorithm | ✅ | Sliding Window — chính xác hơn Fixed Window |
| Isolation | ✅ | Mỗi user có quota riêng, không ảnh hưởng nhau |
| Clean Code | ✅ | Tách ApiGateway ↔ RateLimiter (SRP, DIP) |
| Performance | ✅ | Per-user lock (không lock toàn bộ map) |

---

## 🧪 BÀI 3: THIẾT KẾ HỆ THỐNG ĐƠN HÀNG VỚI STATE MACHINE

### 📝 ĐỀ BÀI

> *"Đơn hàng trong hệ thống e-commerce có các trạng thái: CREATED → PAID → SHIPPED → DELIVERED / CANCELLED. Hãy thiết kế:*
> 1. *Mỗi trạng thái chỉ được chuyển sang trạng thái hợp lệ (VD: CREATED chỉ → PAID hoặc CANCELLED, KHÔNG thể → SHIPPED)*
> 2. *Mỗi lần chuyển trạng thái, ghi log audit trail*
> 3. *Code phải tuân thủ OCP — thêm trạng thái mới không sửa switch/case cũ*
> 4. *Xử lý edge case: chuyển trạng thái không hợp lệ → throw exception rõ ràng"*

### 🔍 PHÂN TÍCH THEO STAR

**Situation**: Đơn hàng phải tuân thủ luồng trạng thái nghiêm ngặt. Chuyển trạng thái sai → dữ liệu inconsistent, khiếu nại khách hàng.

**Task**: Implement State Machine pattern — mỗi State là một object, tự quyết định State nào là transition hợp lệ tiếp theo.

**Action**: Dùng Enum + EnumSet cho state transitions, kết hợp Event-Driven cho audit logging.

```java
import java.util.*;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class OrderStateMachineDemo {
    public static void main(String[] args) {
        System.out.println("╔══════════════════════════════════════════╗");
        System.out.println("║  BÀI 3: ORDER STATE MACHINE             ║");
        System.out.println("╚══════════════════════════════════════════╝\n");

        OrderStateMachine order = new OrderStateMachine("ORD-2024-001", "USR-TRIET");

        // ===== CASE 1: Luồng hạnh phúc (Happy Path) =====
        System.out.println("=== CASE 1: Happy Path ===\n");

        order.transition(OrderEvent.PAY, "Thanh toán qua Momo");
        order.transition(OrderEvent.SHIP, "Giao hàng GHN Express");
        order.transition(OrderEvent.DELIVER, "Khách đã nhận hàng");

        System.out.println();
        order.printAuditTrail();

        // ===== CASE 2: Chuyển trạng thái KHÔNG HỢP LỆ =====
        System.out.println("\n=== CASE 2: Invalid Transition ===\n");
        OrderStateMachine order2 = new OrderStateMachine("ORD-2024-002", "USR-MINH");

        try {
            // CREATED → SHIPPED: Không hợp lệ (phải PAY trước)
            order2.transition(OrderEvent.SHIP, "Cố ship mà chưa thanh toán");
        } catch (InvalidStateTransitionException e) {
            System.out.println("❌ " + e.getMessage());
        }

        // ===== CASE 3: Hủy đơn =====
        System.out.println("\n=== CASE 3: Hủy đơn (CREATED → CANCELLED) ===\n");
        OrderStateMachine order3 = new OrderStateMachine("ORD-2024-003", "USR-AN");
        order3.transition(OrderEvent.CANCEL, "Khách đổi ý");

        order3.printAuditTrail();

        // ===== CASE 4: Hủy đơn đã giao =====
        System.out.println("\n=== CASE 4: Cố hủy đơn đã giao ===\n");
        try {
            order.transition(OrderEvent.CANCEL, "Khách muốn trả hàng");
        } catch (InvalidStateTransitionException e) {
            System.out.println("❌ " + e.getMessage());
        }
    }
}

// ===== ORDER STATE: Enum định nghĩa state + valid transitions =====

enum OrderState {
    CREATED {
        @Override
        public Set<OrderEvent> allowedEvents() {
            return EnumSet.of(OrderEvent.PAY, OrderEvent.CANCEL);
        }
    },
    PAID {
        @Override
        public Set<OrderEvent> allowedEvents() {
            return EnumSet.of(OrderEvent.SHIP, OrderEvent.CANCEL); // Có thể hủy khi đã trả tiền
        }
    },
    SHIPPED {
        @Override
        public Set<OrderEvent> allowedEvents() {
            return EnumSet.of(OrderEvent.DELIVER); // Đang ship → chỉ có thể giao
        }
    },
    DELIVERED {
        @Override
        public Set<OrderEvent> allowedEvents() {
            return EnumSet.noneOf(OrderEvent.class); // Trạng thái cuối → không chuyển tiếp
        }
    },
    CANCELLED {
        @Override
        public Set<OrderEvent> allowedEvents() {
            return EnumSet.noneOf(OrderEvent.class); // Trạng thái cuối
        }
    };

    // Mỗi state tự biết event nào được phép → OCP: thêm state mới = thêm enum constant
    public abstract Set<OrderEvent> allowedEvents();
}

// ===== ORDER EVENT: Sự kiện gây chuyển trạng thái =====

enum OrderEvent {
    PAY     (OrderState.PAID),
    SHIP    (OrderState.SHIPPED),
    DELIVER (OrderState.DELIVERED),
    CANCEL  (OrderState.CANCELLED);

    private final OrderState targetState;

    OrderEvent(OrderState targetState) {
        this.targetState = targetState;
    }

    public OrderState getTargetState() { return targetState; }
}

// ===== CUSTOM EXCEPTION =====

class InvalidStateTransitionException extends RuntimeException {
    InvalidStateTransitionException(String orderId, OrderState current, OrderEvent event) {
        super(String.format("Đơn hàng %s không thể thực hiện [%s] khi đang ở trạng thái [%s]. " +
              "Các sự kiện hợp lệ: %s", orderId, event, current, current.allowedEvents()));
    }
}

// ===== AUDIT LOG ENTRY =====

class AuditEntry {
    final LocalDateTime timestamp;
    final OrderState fromState;
    final OrderState toState;
    final OrderEvent event;
    final String note;

    AuditEntry(OrderState from, OrderState to, OrderEvent event, String note) {
        this.timestamp = LocalDateTime.now();
        this.fromState = from;
        this.toState = to;
        this.event = event;
        this.note = note;
    }

    @Override
    public String toString() {
        return String.format("  [%s] %s → %s (event: %s) | %s",
            timestamp.format(DateTimeFormatter.ofPattern("HH:mm:ss.SSS")),
            fromState, toState, event, note);
    }
}

// ===== STATE MACHINE: Quản lý luồng trạng thái =====

class OrderStateMachine {
    private final String orderId;
    private final String userId;
    private OrderState currentState;
    private final List<AuditEntry> auditTrail = new ArrayList<>();

    OrderStateMachine(String orderId, String userId) {
        this.orderId = orderId;
        this.userId = userId;
        this.currentState = OrderState.CREATED;
        System.out.println("📦 Đơn hàng " + orderId + " được tạo (state: CREATED)");
    }

    public void transition(OrderEvent event, String note) {
        // BƯỚC 1: Kiểm tra event có hợp lệ cho state hiện tại không
        if (!currentState.allowedEvents().contains(event)) {
            throw new InvalidStateTransitionException(orderId, currentState, event);
        }

        // BƯỚC 2: Thực hiện chuyển trạng thái
        OrderState previousState = currentState;
        currentState = event.getTargetState();

        // BƯỚC 3: Ghi audit log
        AuditEntry entry = new AuditEntry(previousState, currentState, event, note);
        auditTrail.add(entry);

        System.out.println("  ✅ [" + orderId + "] " + previousState + " → " + currentState
                         + " | " + note);
    }

    public void printAuditTrail() {
        System.out.println("📋 Audit Trail cho " + orderId + " (user: " + userId + "):");
        auditTrail.forEach(System.out::println);
        System.out.println("  → Trạng thái hiện tại: " + currentState);
    }

    public OrderState getCurrentState() { return currentState; }
}
// Output:
// ╔══════════════════════════════════════════╗
// ║  BÀI 3: ORDER STATE MACHINE             ║
// ╚══════════════════════════════════════════╝
//
// === CASE 1: Happy Path ===
//
// 📦 Đơn hàng ORD-2024-001 được tạo (state: CREATED)
//   ✅ [ORD-2024-001] CREATED → PAID | Thanh toán qua Momo
//   ✅ [ORD-2024-001] PAID → SHIPPED | Giao hàng GHN Express
//   ✅ [ORD-2024-001] SHIPPED → DELIVERED | Khách đã nhận hàng
//
// 📋 Audit Trail cho ORD-2024-001 (user: USR-TRIET):
//   [18:30:15.123] CREATED → PAID (event: PAY) | Thanh toán qua Momo
//   [18:30:15.124] PAID → SHIPPED (event: SHIP) | Giao hàng GHN Express
//   [18:30:15.124] SHIPPED → DELIVERED (event: DELIVER) | Khách đã nhận hàng
//   → Trạng thái hiện tại: DELIVERED
//
// === CASE 2: Invalid Transition ===
//
// 📦 Đơn hàng ORD-2024-002 được tạo (state: CREATED)
// ❌ Đơn hàng ORD-2024-002 không thể thực hiện [SHIP] khi đang ở trạng thái [CREATED]. Các sự kiện hợp lệ: [PAY, CANCEL]
//
// === CASE 3: Hủy đơn (CREATED → CANCELLED) ===
//
// 📦 Đơn hàng ORD-2024-003 được tạo (state: CREATED)
//   ✅ [ORD-2024-003] CREATED → CANCELLED | Khách đổi ý
// 📋 Audit Trail cho ORD-2024-003 (user: USR-AN):
//   [18:30:15.125] CREATED → CANCELLED (event: CANCEL) | Khách đổi ý
//   → Trạng thái hiện tại: CANCELLED
//
// === CASE 4: Cố hủy đơn đã giao ===
//
// ❌ Đơn hàng ORD-2024-001 không thể thực hiện [CANCEL] khi đang ở trạng thái [DELIVERED]. Các sự kiện hợp lệ: []
```

**Result — Điểm đánh giá:**

| Tiêu chí | Đạt? | Ghi chú |
|----------|------|---------|
| State Machine | ✅ | Mỗi State tự biết transitions hợp lệ |
| OCP | ✅ | Thêm state REFUNDING = thêm enum constant + update allowedEvents |
| Error Handling | ✅ | Exception message chi tiết: state hiện tại, event cố gọi, events hợp lệ |
| Audit Trail | ✅ | Ghi log đầy đủ: timestamp, from/to state, event, note |
| Clean Code | ✅ | Enum thay switch/case, SRP rõ ràng |
| Mở rộng | ✅ | Thêm event REFUND, state REFUNDING chỉ cần thêm enum constants |

---

## 📊 BẢNG TỔNG KẾT TOÀN BỘ 5 PHẦN

| Phần | Chủ đề | Kiến thức cốt lõi |
|------|--------|-------------------|
| **1** | JVM & OOP | Stack/Heap, Reference, GC Generations, 4 tính chất OOP, Interface design |
| **2** | Collections | ArrayList vs LinkedList (Cache Locality), HashMap (hashing, Red-Black Tree), Generics (Type Erasure, PECS) |
| **3** | Threading & Exception | Race Condition, synchronized, Atomic, ExecutorService, CompletableFuture, Checked/Unchecked |
| **4** | Java 8 & SOLID | Stream API, Lambda (invokedynamic), Optional, 5 nguyên tắc SOLID với Notification system |
| **5** | Mock Interview | Retry System (Strategy Pattern), Rate Limiter (Sliding Window), State Machine (Enum-based) |

---

> **LỜI KẾT**: Cuốn cẩm nang này không dạy bạn "pass phỏng vấn". Nó dạy bạn **TƯ DUY NHƯ MỘT KỸ SƯ** — hiểu WHY trước khi viết HOW. Mỗi đoạn code ở đây đều có thể copy vào IDE, chạy, và kiểm chứng output. Hãy làm điều đó. Rồi sửa. Rồi phá hỏng. Rồi sửa lại. Đó là cách duy nhất để HIỂU CHẶT CHẼ.
>
> *"The only way to learn a new programming language is by writing programs in it."* — **Dennis Ritchie**
