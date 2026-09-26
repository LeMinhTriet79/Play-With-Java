# 📕 CẨM NANG SINH TỒN JAVA CORE & OOP THỰC CHIẾN

## PHẦN 1: BẢN CHẤT VÙNG NHỚ, GARBAGE COLLECTION & TƯ DUY OOP THỰC CHIẾN

> *"Nếu bạn không hiểu JVM quản lý bộ nhớ thế nào, bạn chỉ đang viết code — chứ không phải đang lập trình."*

---

## 1.1. BỨC TRANH JVM: STACK, HEAP VÀ SỰ THẬT VỀ THAM CHIẾU

### 1.1.1. Kiến trúc vùng nhớ JVM — Nhìn từ góc phẫu thuật

Khi bạn gõ `java Main`, JVM khởi động và chia bộ nhớ thành **hai vùng chính** mà mọi dòng code của bạn đều phụ thuộc:

| Vùng nhớ | Lưu gì? | Vòng đời | Đặc điểm |
|-----------|----------|-----------|-----------|
| **Stack** | Biến local, tham số method, địa chỉ tham chiếu (reference) | Tự hủy khi method kết thúc (LIFO) | Mỗi thread có 1 Stack riêng, cực nhanh |
| **Heap** | Mọi **object** được tạo bằng `new` | Tồn tại đến khi GC dọn | Dùng chung giữa tất cả thread, chậm hơn Stack |

**Quy tắc vàng**: Biến primitive (`int`, `double`, `boolean`...) khai báo trong method → nằm trên **Stack**. Object (`new User()`, `new String()`) → nằm trên **Heap**. Biến tham chiếu (reference variable) trên Stack **chỉ lưu địa chỉ** trỏ đến object trên Heap.

```java
public class JvmMemoryDemo {
    public static void main(String[] args) {
        // === STACK FRAME của main() ===
        int orderId = 1001;            // primitive → nằm trọn trên Stack
        double totalAmount = 250_000;  // primitive → nằm trọn trên Stack

        // "user" là biến tham chiếu → nằm trên Stack
        // nhưng object User thật sự → nằm trên Heap
        User user = new User("Triet", "triet@gmail.com");

        System.out.println("Order ID: " + orderId);
        System.out.println("User: " + user.getName());

        // Gọi method → JVM tạo Stack Frame MỚI cho processOrder()
        processOrder(user, totalAmount);

        // Khi processOrder() kết thúc → Stack Frame của nó bị POP ra
        // Biến "discount" bên trong processOrder() BIẾN MẤT
        // Nhưng object User trên Heap VẪN CÒN (vì "user" ở main vẫn trỏ đến)
    }

    static void processOrder(User buyer, double amount) {
        // === STACK FRAME MỚI của processOrder() ===
        // "buyer" là BẢN SAO của tham chiếu "user" → cùng trỏ đến 1 object trên Heap
        // "amount" là BẢN SAO của giá trị 250_000
        double discount = amount * 0.1; // biến local → Stack
        System.out.println("Giảm giá cho " + buyer.getName() + ": " + discount + "đ");
        // Output: Giảm giá cho Triet: 25000.0đ
    }
}

class User {
    private String name;   // instance field → nằm trên Heap (cùng object)
    private String email;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public String getName() { return name; }
    public String getEmail() { return email; }
}
// Output:
// Order ID: 1001
// User: Triet
// Giảm giá cho Triet: 25000.0đ
```

**Phân tích vùng nhớ từng dòng:**

```
┌─────────────────────────────────┐     ┌──────────────────────────────────┐
│           STACK                 │     │             HEAP                 │
│                                 │     │                                  │
│  ┌─── main() Frame ──────┐     │     │   ┌─────────────────────┐        │
│  │ orderId = 1001         │     │     │   │   User Object       │        │
│  │ totalAmount = 250000.0 │     │     │   │ name ──→ "Triet"    │        │
│  │ user = 0xABC ──────────│─────│────→│   │ email──→ "triet@.." │        │
│  └────────────────────────┘     │     │   └─────────────────────┘        │
│                                 │     │                                  │
│  ┌─ processOrder() Frame ─┐    │     │   ┌─────────────────────┐        │
│  │ buyer = 0xABC ─────────│────│────→│   │ (cùng object User   │        │
│  │ amount = 250000.0      │    │     │   │  ở trên — KHÔNG tạo │        │
│  │ discount = 25000.0     │    │     │   │  object mới!)        │        │
│  └────────────────────────┘    │     │   └─────────────────────┘        │
└─────────────────────────────────┘     └──────────────────────────────────┘
```

> **Sự thật sắc lạnh**: `buyer` trong `processOrder()` và `user` trong `main()` là **hai biến khác nhau trên Stack**, nhưng chứa **cùng một địa chỉ** (0xABC). Cả hai đều trỏ đến **duy nhất một** object `User` trên Heap. Đây chính là bản chất của **Pass-by-Value of Reference** trong Java.

---

### 1.1.2. Tham chiếu (Reference) thực chất là gì?

Nhiều sách viết: *"Biến object lưu object"*. **SAI.** Biến object **chỉ lưu một con số** — đó là **địa chỉ bộ nhớ** (memory address) trỏ đến vùng Heap nơi object thực sự sống.

```java
public class ReferenceDemo {
    public static void main(String[] args) {
        // ref1 lưu địa chỉ Heap, ví dụ: 0x7F3A
        User ref1 = new User("Admin", "admin@system.com");

        // ref2 COPY địa chỉ từ ref1 → cả hai trỏ đến CÙNG object
        User ref2 = ref1;

        // Thay đổi qua ref2 → ảnh hưởng ref1 (vì cùng 1 object!)
        ref2.setEmail("newadmin@system.com");

        System.out.println("ref1 email: " + ref1.getEmail());
        System.out.println("ref2 email: " + ref2.getEmail());
        System.out.println("ref1 == ref2: " + (ref1 == ref2)); // so sánh ĐỊA CHỈ

        // Tạo object MỚI cho ref2
        ref2 = new User("Moderator", "mod@system.com");
        System.out.println("\n--- Sau khi ref2 = new User() ---");
        System.out.println("ref1 email: " + ref1.getEmail()); // KHÔNG đổi!
        System.out.println("ref2 email: " + ref2.getEmail());
        System.out.println("ref1 == ref2: " + (ref1 == ref2)); // false → khác địa chỉ
    }
}

class User {
    private String name;
    private String email;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
// Output:
// ref1 email: newadmin@system.com
// ref2 email: newadmin@system.com
// ref1 == ref2: true
//
// --- Sau khi ref2 = new User() ---
// ref1 email: newadmin@system.com
// ref2 email: mod@system.com
// ref1 == ref2: false
```

---

### 1.1.3. `==` vs `.equals()` — Phẫu thuật sự khác biệt

| Toán tử | So sánh cái gì? | Dùng khi nào? |
|---------|-----------------|---------------|
| `==` | So sánh **địa chỉ vùng nhớ** (reference identity) | Kiểm tra 2 biến có trỏ cùng 1 object không |
| `.equals()` | So sánh **nội dung logic** (nếu đã override) | Kiểm tra 2 object có "bằng nhau về mặt nghiệp vụ" không |

```java
public class EqualsDemo {
    public static void main(String[] args) {
        // ===== VỚI STRING =====
        String s1 = "Java";         // String Pool trên Heap
        String s2 = "Java";         // Trỏ đến CÙNG object trong String Pool
        String s3 = new String("Java"); // Tạo object MỚI trên Heap (ngoài Pool)

        System.out.println("--- String ---");
        System.out.println("s1 == s2: " + (s1 == s2));         // true  → cùng địa chỉ (String Pool)
        System.out.println("s1 == s3: " + (s1 == s3));         // false → khác địa chỉ
        System.out.println("s1.equals(s3): " + s1.equals(s3)); // true  → cùng nội dung "Java"

        // ===== VỚI CUSTOM OBJECT (CHƯA OVERRIDE equals) =====
        Order order1 = new Order("ORD-001", 500_000);
        Order order2 = new Order("ORD-001", 500_000);

        System.out.println("\n--- Order (CHƯA override equals) ---");
        System.out.println("order1 == order2: " + (order1 == order2));
        // false → 2 object khác nhau trên Heap
        System.out.println("order1.equals(order2): " + order1.equals(order2));
        // false → Mặc định equals() kế thừa từ Object, hoạt động GIỐNG ==

        // ===== VỚI CUSTOM OBJECT (ĐÃ OVERRIDE equals) =====
        OrderV2 orderA = new OrderV2("ORD-001", 500_000);
        OrderV2 orderB = new OrderV2("ORD-001", 500_000);

        System.out.println("\n--- OrderV2 (ĐÃ override equals) ---");
        System.out.println("orderA == orderB: " + (orderA == orderB));
        // false → vẫn khác địa chỉ
        System.out.println("orderA.equals(orderB): " + orderA.equals(orderB));
        // true → vì ta đã định nghĩa: cùng orderId = cùng đơn hàng
    }
}

// Class CHƯA override equals
class Order {
    String orderId;
    double amount;

    Order(String orderId, double amount) {
        this.orderId = orderId;
        this.amount = amount;
    }
}

// Class ĐÃ override equals + hashCode (luôn đi cặp!)
class OrderV2 {
    String orderId;
    double amount;

    OrderV2(String orderId, double amount) {
        this.orderId = orderId;
        this.amount = amount;
    }

    @Override
    public boolean equals(Object o) {
        // Bước 1: Cùng địa chỉ → chắc chắn bằng
        if (this == o) return true;
        // Bước 2: Null hoặc khác class → không bằng
        if (o == null || getClass() != o.getClass()) return false;
        // Bước 3: Ép kiểu và so sánh theo logic nghiệp vụ
        OrderV2 that = (OrderV2) o;
        return this.orderId.equals(that.orderId); // Cùng mã đơn = cùng đơn hàng
    }

    @Override
    public int hashCode() {
        // LUẬT SẮT: Nếu override equals() → BẮT BUỘC override hashCode()
        // Lý do: HashMap, HashSet dùng hashCode() để xác định bucket
        return orderId.hashCode();
    }
}
// Output:
// --- String ---
// s1 == s2: true
// s1 == s3: false
// s1.equals(s3): true
//
// --- Order (CHƯA override equals) ---
// order1 == order2: false
// order1.equals(order2): false
//
// --- OrderV2 (ĐÃ override equals) ---
// orderA == orderB: false
// orderA.equals(orderB): true
```

> **Khắc cốt ghi tâm**: Class `Object` (cha của mọi class) có `equals()` mặc định dùng `==`. Nếu bạn không override, gọi `.equals()` cũng chỉ so sánh địa chỉ — **vô nghĩa về mặt nghiệp vụ.**

---

## 1.2. GARBAGE COLLECTION — KẺ DỌN RÁC THẦM LẶNG

### 1.2.1. Khi nào object bị coi là "rác"?

Khi **KHÔNG CÒN bất kỳ tham chiếu nào** trỏ đến nó. GC sẽ tự động thu hồi vùng nhớ Heap.

```java
public class GarbageCollectionDemo {
    public static void main(String[] args) {
        // Object User #1 được tạo, ref "user" trỏ đến nó
        User user = new User("OldUser", "old@mail.com");

        // ref "user" giờ trỏ đến Object User #2
        // → Object User #1 KHÔNG CÒN AI TRỎ ĐẾN → trở thành RÁC
        user = new User("NewUser", "new@mail.com");

        // Gợi ý GC chạy (KHÔNG đảm bảo GC sẽ chạy ngay!)
        System.gc();

        System.out.println("Current user: " + user.getName());

        // Ví dụ 2: Gán null
        User tempUser = new User("TempUser", "temp@mail.com");
        tempUser = null; // Object TempUser → RÁC

        // Ví dụ 3: Object tạo trong scope method
        createAndForget();
        // Sau khi method kết thúc, object bên trong không còn ai trỏ → RÁC
    }

    static void createAndForget() {
        // localUser nằm trên Stack của method này
        User localUser = new User("Forgotten", "forgotten@mail.com");
        System.out.println("Inside method: " + localUser.getName());
        // Method kết thúc → biến localUser bị pop khỏi Stack
        // → Object "Forgotten" trên Heap mất tham chiếu → RÁC
    }
}
// Output:
// Inside method: Forgotten
// Current user: NewUser
```

### 1.2.2. Cấu trúc Heap: Young Generation & Old Generation

```
┌──────────────────────── HEAP ────────────────────────────┐
│                                                          │
│  ┌──────────── Young Generation ──────────────┐          │
│  │  ┌──────┐  ┌──────────┐  ┌──────────┐     │          │
│  │  │ Eden │  │Survivor 0│  │Survivor 1│     │          │
│  │  │      │  │  (S0)    │  │  (S1)    │     │          │
│  │  └──────┘  └──────────┘  └──────────┘     │          │
│  │  Object mới sinh ra ở đây (Minor GC)      │          │
│  └────────────────────────────────────────────┘          │
│                                                          │
│  ┌──────────── Old Generation ────────────────┐          │
│  │                                            │          │
│  │  Object "sống sót" nhiều lần GC sẽ         │          │
│  │  được promote lên đây (Major GC / Full GC) │          │
│  └────────────────────────────────────────────┘          │
│                                                          │
│  ┌──────────── Metaspace (Java 8+) ──────────┐          │
│  │  Class metadata, method bytecode           │          │
│  │  (Thay thế PermGen cũ, dùng native memory)│          │
│  └────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────┘
```

**Luồng đời object trên Heap:**

1. **Object vừa `new`** → đặt vào **Eden** (trong Young Gen).
2. Khi Eden đầy → **Minor GC** chạy. Object còn tham chiếu → chuyển sang **Survivor (S0/S1)**. Object rác → bị dọn.
3. Object sống sót qua **nhiều lần Minor GC** (mặc định ~15 lần, tùy JVM) → **promote** lên **Old Generation**.
4. Khi Old Gen đầy → **Major GC (Full GC)** chạy. Đây là event **tốn kém nhất**, có thể gây **Stop-the-World** (tạm dừng toàn bộ thread).

### 1.2.3. Các thuật toán GC cơ bản

| Thuật toán | Cơ chế | Ưu/Nhược |
|-----------|--------|----------|
| **Mark-and-Sweep** | (1) Đánh dấu (mark) tất cả object còn tham chiếu. (2) Quét (sweep) và xóa object không được đánh dấu. | Đơn giản nhưng gây phân mảnh (fragmentation) |
| **Mark-and-Compact** | Như Mark-and-Sweep nhưng thêm bước nén (compact) — dồn object sống sát nhau | Khắc phục phân mảnh, nhưng tốn CPU hơn |
| **Copying (Young Gen)** | Copy object sống sót từ S0 sang S1, rồi xóa sạch S0 | Rất nhanh cho Young Gen (phần lớn object chết sớm) |

> **Tại sao chia Young/Old?** Vì thống kê cho thấy **~90% object chết trẻ** (weak generational hypothesis). Tách riêng giúp GC chỉ cần quét vùng nhỏ (Young Gen) thường xuyên, thay vì quét toàn bộ Heap.

```java
public class GcGenerationDemo {
    public static void main(String[] args) {
        System.out.println("=== Demo GC Generations ===");

        // Các object tạm này sẽ chết trong Young Gen (Minor GC sẽ dọn)
        for (int i = 0; i < 10_000; i++) {
            // Object Order được tạo rồi không ai trỏ đến → RÁC ngay ở Eden
            new Order("TEMP-" + i, i * 100.0);
        }

        // Object này tồn tại lâu → có thể được promote sang Old Gen
        User longLivedUser = new User("PermanentAdmin", "admin@enterprise.com");

        // Gợi ý GC
        System.gc();

        System.out.println("Long-lived user vẫn còn: " + longLivedUser.getName());
        System.out.println("10,000 Order tạm đã bị GC dọn sạch trên Heap");
    }
}
// Output:
// === Demo GC Generations ===
// Long-lived user vẫn còn: PermanentAdmin
// 10,000 Order tạm đã bị GC dọn sạch trên Heap
```

---

## 1.3. ĐẬP TAN LÝ THUYẾT SUÔNG VỀ 4 TÍNH CHẤT OOP

Chúng ta sẽ thiết kế một **hệ thống Thanh toán (Payment)** hỗ trợ Momo và VNPay. Mỗi tính chất OOP sẽ được **chứng minh bằng code chạy được**.

### 1.3.1. TÍNH ĐÓNG GÓI (Encapsulation)

**Bản chất**: Che giấu trạng thái nội bộ, chỉ cho phép tương tác qua **interface công khai** (public methods). Bảo vệ dữ liệu khỏi sự can thiệp trái phép.

```java
public class EncapsulationDemo {
    public static void main(String[] args) {
        PaymentTransaction tx = new PaymentTransaction("TXN-001", 500_000);

        // KHÔNG THỂ truy cập trực tiếp: tx.amount = -1; → Compile Error!
        // Phải đi qua method công khai (có validation bên trong)
        tx.applyDiscount(10); // Giảm 10%
        System.out.println("Số tiền sau giảm giá: " + tx.getAmount() + "đ");

        tx.applyDiscount(110); // Cố giảm 110% → bị chặn
        System.out.println("Số tiền (sau cố giảm 110%): " + tx.getAmount() + "đ");

        System.out.println("Trạng thái: " + tx.getStatus());
    }
}

class PaymentTransaction {
    // ĐÓ là ĐÓNG GÓI: private = che giấu hoàn toàn
    private String transactionId;
    private double amount;
    private String status;

    public PaymentTransaction(String transactionId, double amount) {
        this.transactionId = transactionId;
        this.amount = amount;
        this.status = "PENDING";
    }

    // Validation logic được gói gọn bên trong method
    public void applyDiscount(double percent) {
        if (percent < 0 || percent > 100) {
            System.out.println("⚠ Phần trăm giảm giá không hợp lệ: " + percent + "%");
            return; // BẢO VỆ dữ liệu — không cho phá trạng thái
        }
        this.amount -= this.amount * percent / 100;
    }

    // Chỉ cho phép ĐỌC, không cho phép GHI trực tiếp
    public double getAmount() { return amount; }
    public String getStatus() { return status; }
}
// Output:
// Số tiền sau giảm giá: 450000.0đ
// ⚠ Phần trăm giảm giá không hợp lệ: 110.0%
// Số tiền (sau cố giảm 110%): 450000.0đ
// Trạng thái: PENDING
```

### 1.3.2. TÍNH KẾ THỪA (Inheritance)

**Bản chất**: Tái sử dụng code từ class cha. Class con **kế thừa** mọi field/method non-private và có thể **mở rộng** thêm.

```java
public class InheritanceDemo {
    public static void main(String[] args) {
        MomoPayment momo = new MomoPayment("0901234567", "TXN-M01", 200_000);
        momo.displayInfo(); // Gọi method kế thừa từ cha
        momo.sendOtp();     // Method riêng của MomoPayment

        System.out.println("---");

        VNPayPayment vnpay = new VNPayPayment("VNPAY-BANK-001", "TXN-V01", 1_500_000);
        vnpay.displayInfo();
        vnpay.redirectToBank();
    }
}

// Class CHA: chứa logic dùng chung cho MỌI phương thức thanh toán
class BasePayment {
    protected String transactionId;  // protected → con truy cập được
    protected double amount;
    private String createdAt;        // private → con KHÔNG truy cập trực tiếp

    public BasePayment(String transactionId, double amount) {
        this.transactionId = transactionId;
        this.amount = amount;
        this.createdAt = java.time.LocalDateTime.now().toString();
    }

    // Method dùng chung
    public void displayInfo() {
        System.out.println("[" + transactionId + "] Số tiền: " + amount + "đ");
    }
}

// Class CON 1: Kế thừa từ BasePayment, thêm logic riêng của Momo
class MomoPayment extends BasePayment {
    private String phoneNumber; // Field riêng

    public MomoPayment(String phoneNumber, String txnId, double amount) {
        super(txnId, amount); // Gọi constructor cha — BẮT BUỘC là dòng đầu tiên
        this.phoneNumber = phoneNumber;
    }

    public void sendOtp() {
        System.out.println("📱 Gửi OTP đến SĐT: " + phoneNumber);
    }
}

// Class CON 2: Kế thừa từ BasePayment, thêm logic riêng của VNPay
class VNPayPayment extends BasePayment {
    private String bankCode;

    public VNPayPayment(String bankCode, String txnId, double amount) {
        super(txnId, amount);
        this.bankCode = bankCode;
    }

    public void redirectToBank() {
        System.out.println("🏦 Chuyển hướng đến cổng ngân hàng: " + bankCode);
    }
}
// Output:
// [TXN-M01] Số tiền: 200000.0đ
// 📱 Gửi OTP đến SĐT: 0901234567
// ---
// [TXN-V01] Số tiền: 1500000.0đ
// 🏦 Chuyển hướng đến cổng ngân hàng: VNPAY-BANK-001
```

### 1.3.3. TÍNH ĐA HÌNH (Polymorphism) — Sức mạnh cốt lõi

**Bản chất**: Cùng một lời gọi method, nhưng **hành vi khác nhau** tùy thuộc vào object thực tế lúc runtime. Đây là nền tảng của **mọi Design Pattern** và cách Spring Boot hoạt động.

Có **2 loại** đa hình:
- **Compile-time (Static)**: Method Overloading — cùng tên, khác tham số.
- **Runtime (Dynamic)**: Method Overriding — class con ghi đè method cha. **ĐÂY là loại quan trọng nhất.**

```java
public class PolymorphismDemo {
    public static void main(String[] args) {
        // ĐA HÌNH RUNTIME: Biến kiểu CHA, object kiểu CON
        PaymentProcessor momoProcessor = new MomoProcessor();
        PaymentProcessor vnpayProcessor = new VNPayProcessor();

        // Cùng gọi .pay() → nhưng hành vi KHÁC NHAU
        processPayment(momoProcessor, 300_000);
        processPayment(vnpayProcessor, 1_200_000);

        // ===== SỨC MẠNH: Thêm phương thức mới mà KHÔNG SỬA code cũ =====
        // Nếu ngày mai thêm ZaloPayProcessor,
        // chỉ cần tạo class mới implements PaymentProcessor
        // Method processPayment() KHÔNG CẦN SỬA 1 DÒNG NÀO
    }

    // Method này KHÔNG CẦN BIẾT đang xử lý Momo hay VNPay
    // Nó chỉ biết PaymentProcessor có method .pay()
    // → Đây là sức mạnh của Đa hình + Interface
    static void processPayment(PaymentProcessor processor, double amount) {
        System.out.println("Bắt đầu xử lý thanh toán: " + amount + "đ");
        boolean result = processor.pay(amount);
        System.out.println("Kết quả: " + (result ? "✅ Thành công" : "❌ Thất bại"));
        System.out.println();
    }
}

// INTERFACE: Hợp đồng (contract) mà mọi processor PHẢI tuân thủ
interface PaymentProcessor {
    boolean pay(double amount);
    String getProviderName();
}

class MomoProcessor implements PaymentProcessor {
    @Override
    public boolean pay(double amount) {
        System.out.println("🟣 [MOMO] Xử lý thanh toán qua ví Momo...");
        System.out.println("   → Gửi request đến api.momo.vn/v2/gateway");
        return amount <= 5_000_000; // Momo giới hạn 5 triệu/lần
    }

    @Override
    public String getProviderName() { return "Momo"; }
}

class VNPayProcessor implements PaymentProcessor {
    @Override
    public boolean pay(double amount) {
        System.out.println("🔵 [VNPAY] Xử lý thanh toán qua cổng VNPay...");
        System.out.println("   → Redirect đến sandbox.vnpayment.vn/paymentv2");
        return amount <= 50_000_000; // VNPay giới hạn 50 triệu/lần
    }

    @Override
    public String getProviderName() { return "VNPay"; }
}
// Output:
// Bắt đầu xử lý thanh toán: 300000.0đ
// 🟣 [MOMO] Xử lý thanh toán qua ví Momo...
//    → Gửi request đến api.momo.vn/v2/gateway
// Kết quả: ✅ Thành công
//
// Bắt đầu xử lý thanh toán: 1200000.0đ
// 🔵 [VNPAY] Xử lý thanh toán qua cổng VNPay...
//    → Redirect đến sandbox.vnpayment.vn/paymentv2
// Kết quả: ✅ Thành công
```

### 1.3.4. TÍNH TRỪU TƯỢNG (Abstraction)

**Bản chất**: Ẩn chi tiết triển khai phức tạp, chỉ lộ ra **cái gì** object làm, không lộ **làm như thế nào**.

```java
public class AbstractionDemo {
    public static void main(String[] args) {
        // Client code chỉ biết: "Tôi cần gửi thông báo thanh toán"
        // Client KHÔNG BIẾT và KHÔNG CẦN BIẾT bên trong dùng HTTP hay SDK
        NotificationService emailService = new EmailNotification();
        NotificationService smsService = new SmsNotification();

        PaymentResult result = new PaymentResult("TXN-999", 750_000, true);

        emailService.notifyPaymentResult(result);
        System.out.println("---");
        smsService.notifyPaymentResult(result);
    }
}

class PaymentResult {
    String txnId;
    double amount;
    boolean success;

    PaymentResult(String txnId, double amount, boolean success) {
        this.txnId = txnId;
        this.amount = amount;
        this.success = success;
    }
}

// ABSTRACT CLASS: Định nghĩa khung xử lý chung (Template Method Pattern)
abstract class NotificationService {
    // Method cụ thể — logic dùng chung, con KHÔNG CẦN override
    public void notifyPaymentResult(PaymentResult result) {
        String message = buildMessage(result);  // Gọi method trừu tượng
        send(message);                          // Gọi method trừu tượng
        logNotification(result.txnId);          // Logic chung
    }

    // Method trừu tượng — con BẮT BUỘC phải implement
    protected abstract String buildMessage(PaymentResult result);
    protected abstract void send(String message);

    // Method cụ thể — logic chung cho mọi loại notification
    private void logNotification(String txnId) {
        System.out.println("   📝 Đã log thông báo cho giao dịch: " + txnId);
    }
}

class EmailNotification extends NotificationService {
    @Override
    protected String buildMessage(PaymentResult result) {
        return String.format("Subject: Thanh toán %s | Số tiền: %.0fđ | Trạng thái: %s",
                result.txnId, result.amount, result.success ? "Thành công" : "Thất bại");
    }

    @Override
    protected void send(String message) {
        System.out.println("📧 [EMAIL] Gửi qua SMTP server...");
        System.out.println("   → " + message);
    }
}

class SmsNotification extends NotificationService {
    @Override
    protected String buildMessage(PaymentResult result) {
        // SMS ngắn gọn hơn email
        return String.format("[%s] %.0fđ - %s",
                result.txnId, result.amount, result.success ? "OK" : "FAIL");
    }

    @Override
    protected void send(String message) {
        System.out.println("📱 [SMS] Gửi qua Twilio API...");
        System.out.println("   → " + message);
    }
}
// Output:
// 📧 [EMAIL] Gửi qua SMTP server...
//    → Subject: Thanh toán TXN-999 | Số tiền: 750000đ | Trạng thái: Thành công
//    📝 Đã log thông báo cho giao dịch: TXN-999
// ---
// 📱 [SMS] Gửi qua Twilio API...
//    → [TXN-999] 750000đ - OK
//    📝 Đã log thông báo cho giao dịch: TXN-999
```

---

## 1.4. TẠI SAO SPRING BOOT DÙNG INTERFACE Ở KHẮP NƠI?

Đây là câu hỏi phỏng vấn **khiến 80% Fresher bối rối**. Câu trả lời nằm ở sự kết hợp của **Đa hình + Dependency Inversion + Testability**.

```java
public class WhyInterfaceDemo {
    public static void main(String[] args) {
        // ===== CÁCH SAI: Phụ thuộc trực tiếp vào implementation =====
        // UserController controller = new UserController(new UserServiceMySQL());
        // Nếu muốn đổi sang MongoDB → phải SỬA code UserController!

        // ===== CÁCH ĐÚNG (Spring Boot style): Phụ thuộc vào INTERFACE =====
        // Chỉ cần đổi object truyền vào, KHÔNG SỬA UserController
        UserService mysqlService = new UserServiceMySQL();
        UserService mongoService = new UserServiceMongoDB();

        UserController controllerV1 = new UserController(mysqlService);
        controllerV1.registerUser("Triet", "triet@gmail.com");

        System.out.println();

        UserController controllerV2 = new UserController(mongoService);
        controllerV2.registerUser("Triet", "triet@gmail.com");

        // ===== BONUS: Unit Test dễ dàng =====
        System.out.println("\n--- Unit Test với Mock ---");
        UserService mockService = new MockUserService(); // Fake implementation
        UserController testController = new UserController(mockService);
        testController.registerUser("TestUser", "test@test.com");
    }
}

// INTERFACE: Hợp đồng nghiệp vụ — KHÔNG quan tâm lưu vào đâu
interface UserService {
    void save(User user);
    User findByEmail(String email);
}

// Implementation 1: MySQL
class UserServiceMySQL implements UserService {
    @Override
    public void save(User user) {
        System.out.println("💾 [MySQL] INSERT INTO users VALUES ('" + user.getName() + "')");
    }

    @Override
    public User findByEmail(String email) {
        System.out.println("🔍 [MySQL] SELECT * FROM users WHERE email = '" + email + "'");
        return new User("FromMySQL", email);
    }
}

// Implementation 2: MongoDB
class UserServiceMongoDB implements UserService {
    @Override
    public void save(User user) {
        System.out.println("💾 [MongoDB] db.users.insertOne({ name: '" + user.getName() + "' })");
    }

    @Override
    public User findByEmail(String email) {
        System.out.println("🔍 [MongoDB] db.users.find({ email: '" + email + "' })");
        return new User("FromMongo", email);
    }
}

// Mock Implementation: Dùng cho Unit Test (không cần database thật!)
class MockUserService implements UserService {
    @Override
    public void save(User user) {
        System.out.println("🧪 [MOCK] Giả lập save thành công (không gọi DB)");
    }

    @Override
    public User findByEmail(String email) {
        return new User("MockUser", email);
    }
}

// Controller: CHỈ BIẾT Interface, KHÔNG BIẾT implementation cụ thể
class UserController {
    private final UserService userService; // Kiểu INTERFACE, không phải class cụ thể

    // Constructor Injection (cách Spring Boot inject Bean)
    public UserController(UserService userService) {
        this.userService = userService;
    }

    public void registerUser(String name, String email) {
        User user = new User(name, email);
        userService.save(user); // Gọi interface → Đa hình quyết định runtime
        System.out.println("   ✅ Đã đăng ký user: " + name);
    }
}
// Output:
// 💾 [MySQL] INSERT INTO users VALUES ('Triet')
//    ✅ Đã đăng ký user: Triet
//
// 💾 [MongoDB] db.users.insertOne({ name: 'Triet' })
//    ✅ Đã đăng ký user: Triet
//
// --- Unit Test với Mock ---
// 🧪 [MOCK] Giả lập save thành công (không gọi DB)
//    ✅ Đã đăng ký user: TestUser
```

**3 lý do Spring Boot ưa chuộng Interface:**

| # | Lý do | Giải thích |
|---|-------|-----------|
| 1 | **Loose Coupling** | Controller không cần biết dùng MySQL hay MongoDB. Đổi DB = đổi Bean config, không sửa logic |
| 2 | **Testability** | Mock dễ dàng, không cần chạy database thật khi unit test |
| 3 | **Open-Closed Principle** | Thêm tính năng mới (ví dụ: PostgreSQL) bằng cách TẠO class mới, không SỬA code cũ |

---

## 🔥 1.5. GÓC PHỎNG VẤN — PHẦN 1

### ❓ Câu 1: "Java truyền tham trị (Pass-by-Value) hay tham chiếu (Pass-by-Reference)?"

**Đáp án sắc bén:**

> *"Java **LUÔN LUÔN** là Pass-by-Value. Không có ngoại lệ.*
>
> *Với **primitive** (int, double...): Bản sao giá trị được truyền vào method. Thay đổi trong method không ảnh hưởng biến gốc.*
>
> *Với **object**: Bản sao **địa chỉ tham chiếu** được truyền vào. Method nhận được một copy của reference, trỏ đến cùng object trên Heap. Nên nếu gọi setter → ảnh hưởng object gốc. Nhưng nếu gán `ref = new Object()` bên trong method → chỉ thay đổi bản copy, biến gốc bên ngoài KHÔNG đổi."*

```java
public class PassByValueProof {
    public static void main(String[] args) {
        // === Primitive: Copy giá trị ===
        int balance = 1_000_000;
        tryModifyPrimitive(balance);
        System.out.println("balance sau method: " + balance); // KHÔNG ĐỔI

        // === Object: Copy reference ===
        User user = new User("OriginalName", "original@mail.com");
        tryModifyObject(user);
        System.out.println("user.name sau method: " + user.getName()); // ĐÃ ĐỔI

        tryReassignObject(user);
        System.out.println("user.name sau reassign: " + user.getName()); // KHÔNG ĐỔI
    }

    static void tryModifyPrimitive(int value) {
        value = 0; // Thay đổi BẢN SAO → gốc không ảnh hưởng
    }

    static void tryModifyObject(User u) {
        // "u" là BẢN SAO reference, nhưng trỏ cùng object
        u.setName("ModifiedName"); // Thay đổi OBJECT thật trên Heap
    }

    static void tryReassignObject(User u) {
        // "u" là BẢN SAO reference → gán lại chỉ ảnh hưởng bản sao
        u = new User("BrandNewUser", "new@mail.com"); // Gốc KHÔNG ĐỔI
        System.out.println("Bên trong method: " + u.getName()); // BrandNewUser
    }
}
// Output:
// balance sau method: 1000000
// user.name sau method: ModifiedName
// Bên trong method: BrandNewUser
// user.name sau reassign: ModifiedName
```

---

### ❓ Câu 2: "String Pool là gì? Tại sao `String s = "hello"` khác `new String("hello")`?"

**Đáp án sắc bén:**

> *"**String Pool** là một vùng đặc biệt trên Heap nơi JVM cache các String literal để tái sử dụng, tiết kiệm bộ nhớ.*
>
> *`String s1 = "hello"` → JVM kiểm tra Pool: nếu `"hello"` đã tồn tại → trả về reference cũ. Nếu chưa → tạo mới trong Pool.*
>
> *`String s2 = new String("hello")` → **LUÔN** tạo object mới trên Heap (ngoài Pool), dù nội dung giống hệt. Đó là lý do `s1 == s2` trả về `false`, nhưng `s1.equals(s2)` trả về `true`.*
>
> *Có thể dùng `s2.intern()` để đưa String về Pool."*

```java
public class StringPoolDemo {
    public static void main(String[] args) {
        // Literal → String Pool
        String s1 = "Java";
        String s2 = "Java";

        // new → Heap (ngoài Pool)
        String s3 = new String("Java");

        // intern() → đưa về Pool, trả reference của Pool
        String s4 = s3.intern();

        System.out.println("s1 == s2: " + (s1 == s2));         // true  (cùng ref trong Pool)
        System.out.println("s1 == s3: " + (s1 == s3));         // false (s3 ngoài Pool)
        System.out.println("s1 == s4: " + (s1 == s4));         // true  (s4 = intern → trỏ về Pool)
        System.out.println("s1.equals(s3): " + s1.equals(s3)); // true  (cùng nội dung)

        // TẠI SAO STRING LÀ IMMUTABLE?
        // → An toàn cho String Pool (nếu mutable, thay đổi 1 chỗ → hỏng tất cả ref cùng trỏ)
        // → An toàn cho HashMap key (hashCode không đổi)
        // → Thread-safe mà không cần synchronization
    }
}
// Output:
// s1 == s2: true
// s1 == s3: false
// s1 == s4: true
// s1.equals(s3): true
```

---

### ❓ Câu 3: "Giải thích cơ chế Đa hình Runtime. JVM quyết định gọi method nào lúc nào?"

**Đáp án sắc bén:**

> *"Đa hình Runtime dựa trên cơ chế **Dynamic Method Dispatch** của JVM. Khi gọi method trên biến kiểu cha, JVM KHÔNG nhìn vào kiểu khai báo (declared type) mà nhìn vào kiểu thực tế của object (actual type) lúc runtime để quyết định method nào được gọi.*
>
> *JVM sử dụng **vtable** (virtual method table) — một bảng ánh xạ method cho mỗi class. Khi gọi `ref.method()`, JVM tra vtable của object thực tế trên Heap, không phải vtable của kiểu khai báo."*

```java
public class DynamicDispatchDemo {
    public static void main(String[] args) {
        // Kiểu khai báo: PaymentProcessor (cha/interface)
        // Kiểu thực tế trên Heap: MomoProcessor
        PaymentProcessor processor = new MomoProcessor();

        // JVM nhìn vào OBJECT THỰC TẾ (MomoProcessor) trên Heap
        // → gọi MomoProcessor.pay(), KHÔNG phải PaymentProcessor.pay()
        processor.pay(100_000);

        // Thay đổi object thực tế lúc runtime
        processor = new VNPayProcessor();
        processor.pay(200_000); // Giờ gọi VNPayProcessor.pay()

        // CHỨNG MINH: getClass() trả về kiểu THỰC TẾ trên Heap
        PaymentProcessor p = new MomoProcessor();
        System.out.println("Declared type: PaymentProcessor");
        System.out.println("Actual type: " + p.getClass().getSimpleName());
    }
}
// Output:
// 🟣 [MOMO] Thanh toán 100000.0đ qua ví Momo
// 🔵 [VNPAY] Thanh toán 200000.0đ qua cổng VNPay
// Declared type: PaymentProcessor
// Actual type: MomoProcessor
```

---

> **KẾT THÚC PHẦN 1** — Bạn đã nắm được: Bức tranh vùng nhớ Stack/Heap, cơ chế GC, 4 tính chất OOP triển khai bằng hệ thống thực chiến, và lý do Spring Boot sống bằng Interface.
