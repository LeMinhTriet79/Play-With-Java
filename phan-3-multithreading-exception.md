# 📕 CẨM NANG SINH TỒN JAVA CORE & OOP THỰC CHIẾN

## PHẦN 3: MULTITHREADING (ĐA LUỒNG) & XỬ LÝ NGOẠI LỆ

> *"Nếu bạn không hiểu thread, bạn không hiểu Spring Boot. Mỗi HTTP request vào Tomcat chính là một thread."*

---

## 3.1. EXCEPTION HANDLING — KIẾN TRÚC XỬ LÝ LỖI CỦA JAVA

### 3.1.1. Cây phân cấp Exception — Nhìn từ trên xuống

```
                      java.lang.Object
                           │
                    java.lang.Throwable
                     ┌─────┴──────┐
                     │            │
               Exception        Error
              ┌────┴────┐        │
              │         │        ├── OutOfMemoryError
       Checked      RuntimeException  ├── StackOverflowError
       Exception    (Unchecked)       └── ... (KHÔNG bắt, KHÔNG xử lý)
              │         │
    ├── IOException     ├── NullPointerException
    ├── SQLException    ├── IllegalArgumentException
    └── ...             ├── IndexOutOfBoundsException
                        ├── ClassCastException
                        └── ...
```

| Loại | Khi nào phát hiện? | BẮT BUỘC xử lý? | Ví dụ |
|------|-------------------|-----------------|-------|
| **Checked Exception** | **Compile time** — Compiler ép bạn phải `try-catch` hoặc `throws` | ✅ Có | `IOException`, `SQLException` |
| **Unchecked Exception** (RuntimeException) | **Runtime** — Compiler KHÔNG ép | ❌ Không bắt buộc | `NullPointerException`, `IllegalArgumentException` |
| **Error** | **Runtime** — Lỗi hệ thống JVM | ❌ KHÔNG nên bắt | `OutOfMemoryError`, `StackOverflowError` |

### 3.1.2. Checked vs Unchecked — Code demo rõ ràng

```java
import java.io.*;

public class ExceptionTypesDemo {
    public static void main(String[] args) {
        // ===== CHECKED EXCEPTION: Compiler BẮT BUỘC bạn xử lý =====
        System.out.println("=== Checked Exception ===");
        try {
            readConfigFile("/etc/app/config.properties");
        } catch (IOException e) {
            // Nếu không có try-catch hoặc throws → COMPILE ERROR
            System.out.println("❌ Checked: " + e.getMessage());
        }

        // ===== UNCHECKED EXCEPTION: Compiler KHÔNG ép =====
        System.out.println("\n=== Unchecked Exception ===");
        try {
            processOrder(null, -500);
        } catch (IllegalArgumentException e) {
            System.out.println("❌ Unchecked: " + e.getMessage());
        } catch (NullPointerException e) {
            System.out.println("❌ Unchecked: " + e.getMessage());
        }

        // ===== CUSTOM EXCEPTION thực tế =====
        System.out.println("\n=== Custom Business Exception ===");
        try {
            withdraw("ACC-001", 10_000_000, 500_000);
        } catch (InsufficientBalanceException e) {
            System.out.println("❌ Business Error: " + e.getMessage());
            System.out.println("   Account: " + e.getAccountId());
            System.out.println("   Thiếu: " + e.getDeficit() + "đ");
        }
    }

    // CHECKED: Phải khai báo "throws IOException" — compiler ép
    static String readConfigFile(String path) throws IOException {
        File file = new File(path);
        if (!file.exists()) {
            throw new IOException("File không tồn tại: " + path);
        }
        return "config content";
    }

    // UNCHECKED: Không cần "throws" — nhưng NÊN validate input
    static void processOrder(String orderId, double amount) {
        if (orderId == null) {
            throw new NullPointerException("orderId không được null");
        }
        if (amount <= 0) {
            throw new IllegalArgumentException("amount phải > 0, nhận được: " + amount);
        }
        System.out.println("Processing order: " + orderId);
    }

    // Method sử dụng Custom Exception
    static void withdraw(String accountId, double balance, double amount) {
        if (amount > balance) {
            throw new InsufficientBalanceException(accountId, amount - balance);
        }
        System.out.println("Rút tiền thành công");
    }
}

// CUSTOM UNCHECKED EXCEPTION — Chuẩn Spring Boot style
// Kế thừa RuntimeException → Unchecked → không ép caller xử lý
class InsufficientBalanceException extends RuntimeException {
    private final String accountId;
    private final double deficit;

    public InsufficientBalanceException(String accountId, double deficit) {
        // Gọi constructor cha để set message
        super("Số dư không đủ cho tài khoản: " + accountId);
        this.accountId = accountId;
        this.deficit = deficit;
    }

    public String getAccountId() { return accountId; }
    public double getDeficit() { return deficit; }
}
// Output:
// === Checked Exception ===
// ❌ Checked: File không tồn tại: /etc/app/config.properties
//
// === Unchecked Exception ===
// ❌ Unchecked: amount phải > 0, nhận được: -500.0
//
// === Custom Business Exception ===
// ❌ Business Error: Số dư không đủ cho tài khoản: ACC-001
//    Account: ACC-001
//    Thiếu: 9500000.0đ
```

### 3.1.3. Tại sao Spring Boot ưu tiên Unchecked Exception?

| Tiêu chí | Checked Exception | Unchecked Exception |
|----------|-------------------|---------------------|
| **Code pollution** | Ép mọi caller phải try-catch hoặc throws → **lan truyền khắp nơi** | Caller tự quyết có bắt hay không |
| **Layer leakage** | `SQLException` từ DAO lộ ra Controller → **vi phạm Abstraction** | Wrap thành `DataAccessException` (Unchecked) → sạch sẽ |
| **Global Handling** | Khó tập trung xử lý | Dùng `@ControllerAdvice` + `@ExceptionHandler` xử lý **1 chỗ duy nhất** |
| **Thực tế** | Java cũ (JDBC, IO) dùng nhiều | Spring Boot, Hibernate, modern Java đều dùng Unchecked |

```java
// ===== CÁCH SPRING BOOT XỬ LÝ EXCEPTION (Mô phỏng) =====
public class SpringStyleExceptionDemo {
    public static void main(String[] args) {
        // Giả lập Controller nhận request
        UserController controller = new UserController(new UserServiceImpl());

        // Case 1: User hợp lệ
        System.out.println("=== Case 1: User hợp lệ ===");
        String response1 = simulateRequest(() -> controller.getUser("USR-001"));
        System.out.println("Response: " + response1);

        // Case 2: User không tồn tại → Business Exception
        System.out.println("\n=== Case 2: User không tồn tại ===");
        String response2 = simulateRequest(() -> controller.getUser("USR-999"));
        System.out.println("Response: " + response2);

        // Case 3: Input null → System Exception
        System.out.println("\n=== Case 3: Input null ===");
        String response3 = simulateRequest(() -> controller.getUser(null));
        System.out.println("Response: " + response3);
    }

    // Giả lập @ControllerAdvice — xử lý exception TẬP TRUNG 1 CHỖ
    static String simulateRequest(java.util.function.Supplier<String> action) {
        try {
            return action.get();
        } catch (ResourceNotFoundException e) {
            // HTTP 404
            return "404 NOT FOUND: " + e.getMessage();
        } catch (IllegalArgumentException e) {
            // HTTP 400
            return "400 BAD REQUEST: " + e.getMessage();
        } catch (Exception e) {
            // HTTP 500
            return "500 INTERNAL ERROR: " + e.getMessage();
        }
    }
}

// Custom Exception cho nghiệp vụ "không tìm thấy"
class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, String id) {
        super(resource + " không tồn tại với ID: " + id);
    }
}

// Service layer
interface UserService {
    String findById(String id);
}

class UserServiceImpl implements UserService {
    @Override
    public String findById(String id) {
        if (id == null) throw new IllegalArgumentException("ID không được null");
        if (id.equals("USR-001")) return "User{Triet}";
        // Không tìm thấy → throw Unchecked (KHÔNG dùng return null!)
        throw new ResourceNotFoundException("User", id);
    }
}

// Controller layer — SẠCH SẼ, không có try-catch rải rác
class UserController {
    private final UserService userService;

    UserController(UserService userService) {
        this.userService = userService;
    }

    // Không cần try-catch! @ControllerAdvice bắt hết
    String getUser(String id) {
        return userService.findById(id);
    }
}
// Output:
// === Case 1: User hợp lệ ===
// Response: User{Triet}
//
// === Case 2: User không tồn tại ===
// Response: 404 NOT FOUND: User không tồn tại với ID: USR-999
//
// === Case 3: Input null ===
// Response: 400 BAD REQUEST: ID không được null
```

> **Nguyên tắc Spring Boot**: Controller KHÔNG chứa try-catch. Service throw Unchecked Exception. `@ControllerAdvice` tập trung xử lý tất cả → code sạch, dễ maintain.

---

## 3.2. MULTITHREADING — ĐA LUỒNG THỰC CHIẾN

### 3.2.1. Thread là gì? Tại sao cần biết?

Mỗi **HTTP request** đến Spring Boot (Tomcat) được xử lý bởi **một thread riêng** (thread pool mặc định 200 thread). Nếu 2 request cùng truy cập **shared resource** (database row, biến static, cache) → có thể xảy ra **Race Condition**.

### 3.2.2. 🔴 Demo Race Condition — 2 thread cùng rút tiền

Đây là bài toán kinh điển: 2 luồng giao dịch đồng thời rút tiền từ cùng 1 tài khoản. Kết quả **sai be bét** nếu không đồng bộ.

```java
public class RaceConditionDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("===== DEMO RACE CONDITION — KHÔNG ĐỒNG BỘ =====\n");

        BankAccountUnsafe unsafeAccount = new BankAccountUnsafe("ACC-001", 1_000_000);

        // Thread 1: ATM Hà Nội rút 200 lần x 3000đ = 600,000đ
        Thread atm1 = new Thread(() -> {
            for (int i = 0; i < 200; i++) {
                unsafeAccount.withdraw(3_000);
            }
        }, "ATM-HaNoi");

        // Thread 2: ATM Sài Gòn rút 200 lần x 3000đ = 600,000đ
        Thread atm2 = new Thread(() -> {
            for (int i = 0; i < 200; i++) {
                unsafeAccount.withdraw(3_000);
            }
        }, "ATM-SaiGon");

        atm1.start();
        atm2.start();

        // Chờ cả 2 thread kết thúc
        atm1.join();
        atm2.join();

        // Tổng rút: 400 lần × 3000 = 1,200,000đ
        // Nhưng tài khoản chỉ có 1,000,000đ → phải âm 200,000đ
        // THỰC TẾ: Kết quả KHÔNG NHẤT QUÁN mỗi lần chạy!
        System.out.println("\n[KẾT QUẢ] Số dư cuối: " + unsafeAccount.getBalance() + "đ");
        System.out.println("[EXPECTED] Nếu đúng (có kiểm tra): nên từ chối khi hết tiền");
        System.out.println("⚠ Mỗi lần chạy cho kết quả KHÁC NHAU → RACE CONDITION!");
    }
}

class BankAccountUnsafe {
    private String accountId;
    private long balance; // SHARED MUTABLE STATE — điểm chết!

    BankAccountUnsafe(String accountId, long balance) {
        this.accountId = accountId;
        this.balance = balance;
    }

    // KHÔNG synchronized → RACE CONDITION
    public void withdraw(long amount) {
        // BƯỚC 1: Đọc balance (ví dụ: balance = 3000)
        // BƯỚC 2: Kiểm tra đủ tiền
        if (balance >= amount) {
            // ⚡ NGAY Ở ĐÂY: Thread khác có thể đã rút xong, balance = 0
            // Nhưng thread hiện tại vẫn nghĩ balance = 3000 (đọc giá trị cũ)

            // Giả lập delay nhỏ để tăng khả năng xảy ra race condition
            // Thread.yield(); // Nhường CPU cho thread khác

            // BƯỚC 3: Trừ tiền — có thể trừ sai vì balance đã bị thread khác thay đổi
            balance -= amount;
        }
    }

    public long getBalance() { return balance; }
}
// Output (KHÔNG ổn định — mỗi lần khác nhau!):
// ===== DEMO RACE CONDITION — KHÔNG ĐỒNG BỘ =====
//
// [KẾT QUẢ] Số dư cuối: -47000đ    ← Có thể âm! Bug nghiêm trọng!
// [EXPECTED] Nếu đúng (có kiểm tra): nên từ chối khi hết tiền
// ⚠ Mỗi lần chạy cho kết quả KHÁC NHAU → RACE CONDITION!
```

**Phân tích Race Condition trên vùng nhớ:**

```
Thời điểm     Thread ATM-HN          Bộ nhớ (Heap)      Thread ATM-SG
─────────     ─────────────          ─────────────       ─────────────
T1            Đọc balance=6000                           (đang chạy)
T2            Kiểm tra: 6000≥3000✓  balance = 6000      Đọc balance=6000
T3            (tính toán...)                             Kiểm tra: 6000≥3000✓
T4            balance = 6000-3000   balance = 3000       (tính toán...)
T5            (xong)                                     balance = 6000-3000
T6                                  balance = 3000 ← SAI! Phải là 0
                                    ↑ Thread SG ghi đè với giá trị cũ
```

### 3.2.3. ✅ Sửa Race Condition bằng `synchronized`

```java
public class SynchronizedFixDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("===== FIX RACE CONDITION VỚI synchronized =====\n");

        BankAccountSafe safeAccount = new BankAccountSafe("ACC-001", 1_000_000);

        Thread atm1 = new Thread(() -> {
            for (int i = 0; i < 200; i++) {
                safeAccount.withdraw(3_000, "ATM-HaNoi");
            }
        }, "ATM-HaNoi");

        Thread atm2 = new Thread(() -> {
            for (int i = 0; i < 200; i++) {
                safeAccount.withdraw(3_000, "ATM-SaiGon");
            }
        }, "ATM-SaiGon");

        atm1.start();
        atm2.start();

        atm1.join();
        atm2.join();

        System.out.println("\n[KẾT QUẢ] Số dư cuối: " + safeAccount.getBalance() + "đ");
        System.out.println("[GIAO DỊCH] Thành công: " + safeAccount.getSuccessCount());
        System.out.println("[GIAO DỊCH] Từ chối:    " + safeAccount.getRejectedCount());
        System.out.println("✅ Kết quả NHẤT QUÁN mỗi lần chạy!");
    }
}

class BankAccountSafe {
    private String accountId;
    private long balance;
    private int successCount = 0;
    private int rejectedCount = 0;

    BankAccountSafe(String accountId, long balance) {
        this.accountId = accountId;
        this.balance = balance;
    }

    // synchronized = Chỉ CHO PHÉP 1 THREAD vào method này tại 1 thời điểm
    // Cơ chế: JVM dùng "intrinsic lock" (monitor) gắn với object "this"
    // Thread khác phải ĐỢI cho đến khi thread hiện tại thoát method
    public synchronized void withdraw(long amount, String source) {
        if (balance >= amount) {
            balance -= amount;
            successCount++;
            // Không in mỗi giao dịch để output gọn
        } else {
            rejectedCount++;
        }
    }

    // Đọc cũng cần synchronized nếu có thread khác đang ghi
    public synchronized long getBalance() { return balance; }
    public synchronized int getSuccessCount() { return successCount; }
    public synchronized int getRejectedCount() { return rejectedCount; }
}
// Output (LUÔN NHẤT QUÁN):
// ===== FIX RACE CONDITION VỚI synchronized =====
//
// [KẾT QUẢ] Số dư cuối: 1000đ
// [GIAO DỊCH] Thành công: 333
// [GIAO DỊCH] Từ chối:    67
// ✅ Kết quả NHẤT QUÁN mỗi lần chạy!
```

> **Giải thích `synchronized` ở tầng JVM:**
> - Mỗi object Java có một **intrinsic lock** (hay **monitor**).
> - Khi thread A vào `synchronized` method → nó **acquire lock** của object `this`.
> - Thread B cố vào method cùng object → thấy lock đã bị chiếm → **BLOCKED** (chờ).
> - Thread A thoát method → **release lock** → Thread B được vào.
> - Đây là **Mutual Exclusion** (loại trừ lẫn nhau) — chỉ 1 thread vào Critical Section.

### 3.2.4. `AtomicInteger` — Giải pháp nhẹ hơn synchronized

Khi chỉ cần bảo vệ **một biến đơn lẻ** (counter, balance), `java.util.concurrent.atomic` nhanh hơn `synchronized` vì dùng **CAS** (Compare-And-Swap) — thao tác phần cứng CPU, không cần lock.

```java
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("===== ATOMIC — Lock-Free Concurrency =====\n");

        // Bộ đếm request — nhiều thread tăng đồng thời
        RequestCounter counter = new RequestCounter();

        // 10 thread, mỗi thread tăng 10,000 lần = 100,000 tổng
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10_000; j++) {
                    counter.increment();
                }
            });
            threads[i].start();
        }

        for (Thread t : threads) t.join();

        System.out.println("Expected: 100,000");
        System.out.println("Actual:   " + counter.getCount());
        System.out.println("✅ AtomicLong đảm bảo chính xác — không cần synchronized");

        // So sánh: KHÔNG atomic → sai
        System.out.println("\n--- Unsafe counter ---");
        UnsafeCounter unsafeCounter = new UnsafeCounter();
        Thread[] threads2 = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads2[i] = new Thread(() -> {
                for (int j = 0; j < 10_000; j++) {
                    unsafeCounter.increment();
                }
            });
            threads2[i].start();
        }
        for (Thread t : threads2) t.join();
        System.out.println("Expected: 100,000");
        System.out.println("Actual:   " + unsafeCounter.getCount());
        System.out.println("⚠ Kết quả có thể < 100,000 vì race condition!");
    }
}

class RequestCounter {
    // AtomicLong: Mọi thao tác (increment, get) đều THREAD-SAFE
    // Dùng CAS (Compare-And-Swap) ở tầng CPU — không lock
    private final AtomicLong count = new AtomicLong(0);

    public void increment() {
        count.incrementAndGet(); // Atomic: Đọc + Tăng + Ghi = 1 thao tác nguyên tử
    }

    public long getCount() {
        return count.get();
    }
}

class UnsafeCounter {
    private long count = 0; // KHÔNG atomic — KHÔNG thread-safe

    public void increment() {
        count++; // count++ = 3 bước: đọc → tăng → ghi → RACE CONDITION
    }

    public long getCount() { return count; }
}
// Output:
// ===== ATOMIC — Lock-Free Concurrency =====
//
// Expected: 100,000
// Actual:   100000
// ✅ AtomicLong đảm bảo chính xác — không cần synchronized
//
// --- Unsafe counter ---
// Expected: 100,000
// Actual:   87342    ← Sai! (Số cụ thể thay đổi mỗi lần chạy)
// ⚠ Kết quả có thể < 100,000 vì race condition!
```

### 3.2.5. ExecutorService — Tại sao TUYỆT ĐỐI KHÔNG `new Thread()` trong production?

| Tiêu chí | `new Thread()` | `ExecutorService` (Thread Pool) |
|----------|---------------|-------------------------------|
| **Tạo thread** | Tạo MỚI mỗi lần → tốn tài nguyên OS (~1MB stack/thread) | Tái sử dụng thread có sẵn trong pool |
| **Kiểm soát** | Không giới hạn số thread → có thể tạo hàng ngàn → **OOM** | Giới hạn pool size → kiểm soát tài nguyên |
| **Quản lý lifecycle** | Phải quản lý thủ công: start, interrupt, join | Pool tự quản lý: submit → nhận Future → done |
| **Exception handling** | Exception trong thread "nuốt" im lặng | Future.get() bắt được exception |
| **Thực tế** | ❌ KHÔNG BAO GIỜ dùng trong Spring Boot | ✅ Spring dùng `@Async` + `ThreadPoolTaskExecutor` bên dưới |

```java
import java.util.concurrent.*;
import java.util.List;
import java.util.ArrayList;

public class ExecutorServiceDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("===== ExecutorService — Thread Pool thực chiến =====\n");

        // Tạo Thread Pool với TỐI ĐA 3 thread
        // Dù submit 10 task, chỉ có 3 thread chạy đồng thời
        ExecutorService executor = Executors.newFixedThreadPool(3);

        // Giả lập: Xử lý 5 đơn hàng song song
        List<Future<String>> futures = new ArrayList<>();

        for (int i = 1; i <= 5; i++) {
            final String orderId = "ORD-" + String.format("%03d", i);

            // Callable<T>: Giống Runnable nhưng TRẢ VỀ kết quả + throw Exception
            Future<String> future = executor.submit(() -> {
                String threadName = Thread.currentThread().getName();
                System.out.println("[" + threadName + "] 🔄 Đang xử lý " + orderId + "...");

                // Giả lập thời gian xử lý
                Thread.sleep(1000);

                // Giả lập lỗi cho đơn ORD-003
                if (orderId.equals("ORD-003")) {
                    throw new RuntimeException("Payment gateway timeout cho " + orderId);
                }

                System.out.println("[" + threadName + "] ✅ Hoàn thành " + orderId);
                return orderId + " → PAID";
            });

            futures.add(future);
        }

        // Thu thập kết quả — Future.get() BLOCK cho đến khi task xong
        System.out.println("\n=== Thu thập kết quả ===");
        for (Future<String> future : futures) {
            try {
                String result = future.get(5, TimeUnit.SECONDS); // Timeout 5s
                System.out.println("📦 " + result);
            } catch (ExecutionException e) {
                // Exception trong Callable được wrap trong ExecutionException
                System.out.println("❌ Lỗi: " + e.getCause().getMessage());
            } catch (TimeoutException e) {
                System.out.println("⏱ Timeout — task chạy quá lâu");
                future.cancel(true); // Hủy task
            }
        }

        // BẮT BUỘC shutdown — nếu không, JVM sẽ không thoát!
        executor.shutdown();
        boolean terminated = executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("\nPool đã shutdown: " + terminated);

        // ===== DEMO CÁC LOẠI THREAD POOL =====
        System.out.println("\n=== Các loại Thread Pool ===");
        System.out.println("• newFixedThreadPool(n)   → Pool cố định n thread (dùng cho CPU-bound task)");
        System.out.println("• newCachedThreadPool()   → Pool co giãn, tạo/hủy thread tự động (dùng cho I/O-bound)");
        System.out.println("• newSingleThreadExecutor → Pool 1 thread (xử lý tuần tự, đảm bảo thứ tự)");
        System.out.println("• newScheduledThreadPool  → Pool hỗ trợ lập lịch (thay thế Timer)");
        System.out.println("• Spring Boot: @Async + ThreadPoolTaskExecutor (customize pool size, queue)");
    }
}
// Output (thứ tự thread có thể thay đổi):
// ===== ExecutorService — Thread Pool thực chiến =====
//
// [pool-1-thread-1] 🔄 Đang xử lý ORD-001...
// [pool-1-thread-2] 🔄 Đang xử lý ORD-002...
// [pool-1-thread-3] 🔄 Đang xử lý ORD-003...
// [pool-1-thread-1] ✅ Hoàn thành ORD-001
// [pool-1-thread-2] ✅ Hoàn thành ORD-002
// [pool-1-thread-1] 🔄 Đang xử lý ORD-004...
// [pool-1-thread-2] 🔄 Đang xử lý ORD-005...
// [pool-1-thread-1] ✅ Hoàn thành ORD-004
// [pool-1-thread-2] ✅ Hoàn thành ORD-005
//
// === Thu thập kết quả ===
// 📦 ORD-001 → PAID
// 📦 ORD-002 → PAID
// ❌ Lỗi: Payment gateway timeout cho ORD-003
// 📦 ORD-004 → PAID
// 📦 ORD-005 → PAID
//
// Pool đã shutdown: true
//
// === Các loại Thread Pool ===
// • newFixedThreadPool(n)   → Pool cố định n thread (dùng cho CPU-bound task)
// • newCachedThreadPool()   → Pool co giãn, tạo/hủy thread tự động (dùng cho I/O-bound)
// • newSingleThreadExecutor → Pool 1 thread (xử lý tuần tự, đảm bảo thứ tự)
// • newScheduledThreadPool  → Pool hỗ trợ lập lịch (thay thế Timer)
// • Spring Boot: @Async + ThreadPoolTaskExecutor (customize pool size, queue)
```

### 3.2.6. CompletableFuture — Xử lý bất đồng bộ chuỗi (Java 8+)

Trong thực tế, các task thường **phụ thuộc nhau**: Lấy user → kiểm tra quyền → tạo order → gửi email. `CompletableFuture` cho phép xâu chuỗi (chaining) mà không block thread.

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CompletableFutureDemo {
    static ExecutorService pool = Executors.newFixedThreadPool(3);

    public static void main(String[] args) throws Exception {
        System.out.println("===== CompletableFuture — Async Pipeline =====\n");

        // Xâu chuỗi: Tìm user → Validate → Tạo order → Gửi notification
        CompletableFuture<String> pipeline = CompletableFuture
                // BƯỚC 1: Tìm user (chạy trên thread pool)
                .supplyAsync(() -> {
                    System.out.println("[" + threadName() + "] Bước 1: Tìm user USR-001...");
                    simulateDelay(500);
                    return "User{Triet, ROLE_ADMIN}";
                }, pool)
                // BƯỚC 2: Validate user (chạy SAU bước 1, dùng kết quả bước 1)
                .thenApplyAsync(user -> {
                    System.out.println("[" + threadName() + "] Bước 2: Validate " + user);
                    simulateDelay(300);
                    return "ORD-NEW-001";
                }, pool)
                // BƯỚC 3: Tạo order
                .thenApplyAsync(orderId -> {
                    System.out.println("[" + threadName() + "] Bước 3: Tạo order " + orderId);
                    simulateDelay(400);
                    return orderId + " → CREATED";
                }, pool)
                // BƯỚC 4: Gửi notification (không cần kết quả → thenAccept)
                .thenApply(result -> {
                    System.out.println("[" + threadName() + "] Bước 4: Gửi email cho " + result);
                    return "✅ Pipeline hoàn thành: " + result;
                })
                // Xử lý exception ở BẤT KỲ bước nào
                .exceptionally(ex -> {
                    return "❌ Pipeline thất bại: " + ex.getMessage();
                });

        // Chờ kết quả cuối cùng
        String finalResult = pipeline.get();
        System.out.println("\n[FINAL] " + finalResult);

        // ===== DEMO: 2 task song song rồi combine =====
        System.out.println("\n=== Combine 2 async tasks ===");
        CompletableFuture<String> getUserTask = CompletableFuture.supplyAsync(() -> {
            simulateDelay(600);
            return "User{Triet}";
        }, pool);

        CompletableFuture<String> getOrderTask = CompletableFuture.supplyAsync(() -> {
            simulateDelay(400);
            return "Order{ORD-001, 500k}";
        }, pool);

        // Khi CẢ HAI xong → combine kết quả
        String combined = getUserTask.thenCombine(getOrderTask,
                (user, order) -> "Matched: " + user + " ↔ " + order
        ).get();

        System.out.println(combined);

        pool.shutdown();
    }

    static String threadName() {
        return Thread.currentThread().getName();
    }

    static void simulateDelay(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
// Output:
// ===== CompletableFuture — Async Pipeline =====
//
// [pool-1-thread-1] Bước 1: Tìm user USR-001...
// [pool-1-thread-2] Bước 2: Validate User{Triet, ROLE_ADMIN}
// [pool-1-thread-3] Bước 3: Tạo order ORD-NEW-001
// [pool-1-thread-3] Bước 4: Gửi email cho ORD-NEW-001 → CREATED
//
// [FINAL] ✅ Pipeline hoàn thành: ORD-NEW-001 → CREATED
//
// === Combine 2 async tasks ===
// Matched: User{Triet} ↔ Order{ORD-001, 500k}
```

---

## 🔥 3.3. GÓC PHỎNG VẤN — PHẦN 3

### ❓ Câu 1: "Deadlock là gì? Cho ví dụ code gây Deadlock và cách phòng tránh?"

**Đáp án sắc bén:**

> *"**Deadlock** xảy ra khi 2 thread đợi nhau giải phóng lock — cả hai đều bị kẹt vĩnh viễn.*
>
> *Điều kiện (4 điều kiện Coffman): (1) Mutual Exclusion, (2) Hold and Wait, (3) No Preemption, (4) Circular Wait. Phá BẤT KỲ điều kiện nào → phá deadlock.*
>
> *Cách phòng: (1) **Lock ordering** — luôn lock theo thứ tự cố định. (2) Dùng `tryLock()` với timeout thay vì `synchronized`. (3) Giảm scope của lock."*

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class DeadlockDemo {
    // ===== DEADLOCK: 2 tài khoản chuyển tiền cho nhau đồng thời =====
    static final Object lockA = new Object(); // Lock cho Account A
    static final Object lockB = new Object(); // Lock cho Account B

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== DEADLOCK DEMO (sẽ treo nếu uncomment) ===");

        // ĐỂ THẤY DEADLOCK, uncomment khối bên dưới:
        /*
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {           // T1 chiếm lockA
                System.out.println("T1: giữ lockA, đợi lockB...");
                try { Thread.sleep(100); } catch (Exception e) {}
                synchronized (lockB) {       // T1 đợi lockB ← T2 đang giữ → DEADLOCK
                    System.out.println("T1: chuyển tiền A→B");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (lockB) {           // T2 chiếm lockB
                System.out.println("T2: giữ lockB, đợi lockA...");
                try { Thread.sleep(100); } catch (Exception e) {}
                synchronized (lockA) {       // T2 đợi lockA ← T1 đang giữ → DEADLOCK
                    System.out.println("T2: chuyển tiền B→A");
                }
            }
        });

        t1.start(); t2.start();
        // Chương trình TREO VĨNH VIỄN tại đây!
        */

        // ===== FIX: Lock Ordering — luôn lock A trước B =====
        System.out.println("\n=== FIX VỚI LOCK ORDERING ===");
        Thread t1Safe = new Thread(() -> {
            synchronized (lockA) {       // Luôn lock A trước
                synchronized (lockB) {
                    System.out.println("T1: chuyển tiền A→B ✅");
                }
            }
        });

        Thread t2Safe = new Thread(() -> {
            synchronized (lockA) {       // Cũng lock A trước (thay vì B)
                synchronized (lockB) {
                    System.out.println("T2: chuyển tiền B→A ✅");
                }
            }
        });

        t1Safe.start(); t2Safe.start();
        t1Safe.join(); t2Safe.join();

        // ===== FIX 2: tryLock với timeout =====
        System.out.println("\n=== FIX VỚI tryLock + timeout ===");
        ReentrantLock lock1 = new ReentrantLock();
        ReentrantLock lock2 = new ReentrantLock();

        Thread t3 = new Thread(() -> {
            try {
                if (lock1.tryLock(1, TimeUnit.SECONDS)) {
                    try {
                        if (lock2.tryLock(1, TimeUnit.SECONDS)) {
                            try {
                                System.out.println("T3: chuyển tiền thành công ✅");
                            } finally { lock2.unlock(); }
                        } else {
                            System.out.println("T3: không lấy được lock2 → rollback");
                        }
                    } finally { lock1.unlock(); }
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        t3.start(); t3.join();
        System.out.println("Không deadlock — tryLock thoát ra sau timeout!");
    }
}
// Output:
// === DEADLOCK DEMO (sẽ treo nếu uncomment) ===
//
// === FIX VỚI LOCK ORDERING ===
// T1: chuyển tiền A→B ✅
// T2: chuyển tiền B→A ✅
//
// === FIX VỚI tryLock + timeout ===
// T3: chuyển tiền thành công ✅
// Không deadlock — tryLock thoát ra sau timeout!
```

---

### ❓ Câu 2: "`volatile` khác `synchronized` thế nào? Khi nào dùng cái nào?"

**Đáp án sắc bén:**

> | Tiêu chí | `volatile` | `synchronized` |
> |----------|-----------|----------------|
> | **Giải quyết** | **Visibility** — đảm bảo thread đọc giá trị MỚI NHẤT từ main memory | **Visibility + Atomicity** — đảm bảo chỉ 1 thread vào critical section |
> | **Lock** | KHÔNG lock | CÓ lock (intrinsic lock) |
> | **Hiệu năng** | Nhanh hơn (không context switch) | Chậm hơn (thread blocked phải đợi) |
> | **Dùng khi** | Biến flag (boolean), biến trạng thái đơn lẻ, chỉ 1 thread ghi + nhiều thread đọc | Nhiều thread cùng ghi, thao tác phức hợp (check-then-act) |
>
> *"`volatile` giải quyết vấn đề **CPU cache**: Mỗi thread có thể cache biến trong register/L1 cache. Không có volatile, thread B có thể đọc giá trị CŨ mà thread A đã sửa. `volatile` buộc mọi đọc/ghi đi qua main memory."*
>
> *"Nhưng `volatile` **KHÔNG giải quyết** race condition cho `count++` vì đó là 3 bước (đọc→tăng→ghi). Cần `synchronized` hoặc `AtomicInteger` cho trường hợp này."*

---

### ❓ Câu 3: "ConcurrentHashMap khác Collections.synchronizedMap khác Hashtable thế nào?"

**Đáp án sắc bén:**

> | Collection | Lock mechanism | Hiệu năng | Null key/value |
> |-----------|---------------|-----------|----------------|
> | `Hashtable` | Lock **toàn bộ** map (1 lock duy nhất) | ❌ Tệ — mọi thread đều chờ | ❌ Không cho null |
> | `Collections.synchronizedMap` | Lock **toàn bộ** map (wrapper) | ❌ Tệ — giống Hashtable | ✅ Cho phép 1 null key |
> | `ConcurrentHashMap` | Lock **từng segment/bucket** (Java 8: CAS + synchronized per bucket) | ✅ Tốt — nhiều thread đọc/ghi đồng thời ở các bucket khác nhau | ❌ Không cho null |
>
> *"Trong thực tế **LUÔN dùng `ConcurrentHashMap`** cho shared map. `Hashtable` là class cổ từ Java 1.0, **KHÔNG BAO GIỜ dùng** trong code mới. `ConcurrentHashMap` cho phép concurrent reads mà KHÔNG lock, và chỉ lock ở bucket cụ thể khi write → throughput cực cao."*

---

> **KẾT THÚC PHẦN 3** — Bạn đã nắm: Checked vs Unchecked Exception (và cách Spring Boot xử lý), Race Condition + synchronized + Atomic, ExecutorService (Thread Pool), CompletableFuture, Deadlock, volatile vs synchronized, ConcurrentHashMap.
