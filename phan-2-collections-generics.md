# 📕 CẨM NANG SINH TỒN JAVA CORE & OOP THỰC CHIẾN

## PHẦN 2: COLLECTIONS, HASHMAP BÍ ẨN & GENERICS

> *"Bạn có thể viết 1000 dòng code mà không hiểu Collections. Nhưng bạn sẽ viết 1000 dòng code TỒI."*

---

## 2.1. ARRAYLIST vs LINKEDLIST — CUỘC CHIẾN CỦA KIẾN TRÚC BỘ NHỚ

### 2.1.1. Kiến trúc bên trong

| Tiêu chí | `ArrayList` | `LinkedList` |
|----------|-------------|--------------|
| **Cấu trúc** | Mảng động (`Object[]`) liên tục trên Heap | Danh sách liên kết đôi (Doubly Linked List) — node rải rác trên Heap |
| **Truy cập theo index** | **O(1)** — nhảy thẳng đến vị trí | **O(n)** — phải duyệt từ đầu/cuối |
| **Thêm/Xóa ở giữa** | **O(n)** — phải dịch chuyển mảng | **O(1)** — nếu đã có reference đến node |
| **Thêm ở cuối** | **O(1)** amortized (khi cần resize → copy toàn bộ mảng) | **O(1)** |
| **Bộ nhớ** | Ít overhead (chỉ là mảng) | Mỗi node tốn thêm 2 pointer (prev + next) ~32 bytes |

### 2.1.2. CPU Cache Locality — Lý do ArrayList THỐNG TRỊ thực tế

Đây là kiến thức mà **90% dev không biết**: Tại sao `ArrayList` gần như **LUÔN nhanh hơn** `LinkedList`, kể cả khi thêm/xóa?

**Giải thích:** CPU không đọc RAM từng byte. Nó đọc theo **cache line** (thường 64 bytes). Khi CPU đọc phần tử thứ `i` của ArrayList, nó **tự động kéo cả vùng lân cận** vào L1/L2 cache (vì các phần tử nằm **liên tiếp** trên RAM). Khi đọc phần tử `i+1`, dữ liệu **đã có sẵn trong cache** → cực nhanh.

Với `LinkedList`, mỗi node nằm **rải rác khắp nơi** trên Heap → CPU liên tục **cache miss** → phải quay lại đọc RAM → **chậm gấp nhiều lần**.

```
ArrayList trên RAM (liên tiếp → Cache Friendly ✅):
┌──────┬──────┬──────┬──────┬──────┬──────┐
│ E[0] │ E[1] │ E[2] │ E[3] │ E[4] │ E[5] │  ← CPU đọc 1 cache line = được 8 phần tử
└──────┴──────┴──────┴──────┴──────┴──────┘

LinkedList trên RAM (rải rác → Cache MISS ❌):
┌──────┐            ┌──────┐     ┌──────┐
│Node 0│──→  ...    │Node 1│──→  │Node 2│──→  ...
│@0x1A │            │@0x8F │     │@0xC3 │
└──────┘            └──────┘     └──────┘
   ↑ Mỗi lần nhảy đến node tiếp = 1 lần cache miss
```

```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;

public class ListComparisonDemo {
    public static void main(String[] args) {
        int size = 100_000;

        // === BENCHMARK: Truy cập theo index ===
        List<String> arrayList = new ArrayList<>();
        List<String> linkedList = new LinkedList<>();

        for (int i = 0; i < size; i++) {
            String orderId = "ORD-" + i;
            arrayList.add(orderId);
            linkedList.add(orderId);
        }

        // ArrayList: get(i) = O(1) → gần như instant
        long start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            arrayList.get(i); // Nhảy thẳng bằng index → O(1)
        }
        long arrayTime = System.nanoTime() - start;

        // LinkedList: get(i) = O(n) → duyệt từ đầu mỗi lần
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            linkedList.get(i); // Phải duyệt từ head → O(n) mỗi lần → tổng O(n²)!
        }
        long linkedTime = System.nanoTime() - start;

        System.out.println("=== Random Access " + size + " phần tử ===");
        System.out.println("ArrayList : " + (arrayTime / 1_000_000) + " ms");
        System.out.println("LinkedList: " + (linkedTime / 1_000_000) + " ms");
        System.out.println("LinkedList chậm hơn ~" + (linkedTime / Math.max(arrayTime, 1)) + " lần");

        // === KẾT LUẬN THỰC TẾ ===
        System.out.println("\n=== KẾT LUẬN ===");
        System.out.println("• 99% trường hợp → Dùng ArrayList");
        System.out.println("• LinkedList chỉ ưu việt khi: thêm/xóa ĐẦU danh sách + KHÔNG bao giờ truy cập index");
        System.out.println("• Khi cần Queue/Deque → dùng ArrayDeque (nhanh hơn LinkedList)");
    }
}
// Output (ví dụ, thời gian thay đổi theo máy):
// === Random Access 100000 phần tử ===
// ArrayList : 2 ms
// LinkedList: 4812 ms
// LinkedList chậm hơn ~2406 lần
//
// === KẾT LUẬN ===
// • 99% trường hợp → Dùng ArrayList
// • LinkedList chỉ ưu việt khi: thêm/xóa ĐẦU danh sách + KHÔNG bao giờ truy cập index
// • Khi cần Queue/Deque → dùng ArrayDeque (nhanh hơn LinkedList)
```

> **Quy tắc thực chiến**: Joshua Bloch (tác giả Java Collections Framework) nói: *"LinkedList gần như không bao giờ là lựa chọn đúng."* Trong Spring Boot, mọi nơi bạn thấy `List<>` — hầu hết đều là `ArrayList`.

---

## 2.2. ĐI SÂU VÀO HASHMAP — CẤU TRÚC DỮ LIỆU BÍ ẨN NHẤT CỦA JAVA

### 2.2.1. Cơ chế băm (Hashing) — HashMap hoạt động thế nào?

`HashMap<K, V>` bên trong là một **mảng Node[]** (gọi là **table**), mặc định có **16 buckets** (slots).

**Khi gọi `map.put(key, value)`:**

1. Tính `hashCode()` của key → được một số nguyên (ví dụ: `1742810335`).
2. Áp dụng công thức: `index = hash(key) & (n - 1)` (trong đó `n` = kích thước mảng) → ra vị trí bucket (ví dụ: bucket `7`).
3. Nếu bucket **trống** → tạo Node mới, đặt vào.
4. Nếu bucket **đã có Node** (Hash Collision!) → so sánh `equals()`:
   - Nếu `equals()` trả về `true` → **ghi đè** value cũ.
   - Nếu `equals()` trả về `false` → **thêm Node mới vào linked list** tại bucket đó.

**Từ Java 8**: Nếu linked list tại 1 bucket dài quá **8 node** (và tổng capacity ≥ 64) → **chuyển thành Red-Black Tree** (O(log n) thay vì O(n) khi tìm kiếm).

```
HashMap internal (Java 8+):
┌─────────────────────────────────────────────────────────────┐
│  Node[] table (mặc định 16 buckets)                         │
│                                                             │
│  [0] → null                                                 │
│  [1] → null                                                 │
│  [2] → Node("ORD-001", Order) → null                       │
│  [3] → null                                                 │
│  [4] → Node("ORD-007", Order) → Node("ORD-023", Order)     │ ← Collision!
│         ↑ Linked List (< 8 nodes)                           │
│  [5] → null                                                 │
│  [6] → RedBlackTree { ... }                                 │ ← ≥ 8 collisions
│  [7] → Node("ORD-042", Order) → null                       │
│  ...                                                        │
│  [15]→ null                                                 │
└─────────────────────────────────────────────────────────────┘
```

### 2.2.2. Tại sao Java 8 dùng Red-Black Tree?

Trước Java 8, khi nhiều key hash vào cùng bucket → linked list dài → tìm kiếm là **O(n)** — tình huống **worst case DoS attack**. Kẻ tấn công cố tình gửi request với key có cùng hashCode, khiến HashMap degrade thành linked list.

Red-Black Tree giữ **O(log n)** ngay cả worst case → bảo mật hơn, hiệu năng ổn định hơn.

```java
import java.util.HashMap;
import java.util.Map;

public class HashMapInternalDemo {
    public static void main(String[] args) {
        Map<String, Double> orderAmounts = new HashMap<>();

        // HashMap tính hashCode của key để xác định bucket
        String key1 = "ORD-001";
        String key2 = "ORD-002";
        String key3 = "ORD-001"; // Cùng nội dung với key1

        System.out.println("=== hashCode() của các key ===");
        System.out.println("key1 (\"ORD-001\") hashCode: " + key1.hashCode());
        System.out.println("key2 (\"ORD-002\") hashCode: " + key2.hashCode());
        System.out.println("key3 (\"ORD-001\") hashCode: " + key3.hashCode());
        System.out.println("key1 hashCode == key3 hashCode: " + (key1.hashCode() == key3.hashCode()));

        // put: Tính bucket → đặt vào
        orderAmounts.put(key1, 500_000.0);
        orderAmounts.put(key2, 1_200_000.0);

        // put key3 ("ORD-001"): cùng hashCode → cùng bucket → equals() trả true → GHI ĐÈ
        orderAmounts.put(key3, 750_000.0); // Ghi đè value của key1!

        System.out.println("\n=== HashMap sau khi put ===");
        System.out.println("ORD-001: " + orderAmounts.get("ORD-001")); // 750000.0 (đã bị ghi đè)
        System.out.println("ORD-002: " + orderAmounts.get("ORD-002")); // 1200000.0
        System.out.println("Size: " + orderAmounts.size()); // 2 (không phải 3!)

        // === Load Factor & Resize ===
        // Default: capacity = 16, loadFactor = 0.75
        // Khi size > 16 * 0.75 = 12 → HashMap TỰ ĐỘNG resize (capacity * 2 = 32)
        // Resize = tạo mảng mới + rehash TẤT CẢ entry → TỐN KÉM O(n)
        System.out.println("\n=== Resize Demo ===");
        Map<String, Integer> bigMap = new HashMap<>();
        for (int i = 0; i < 20; i++) {
            bigMap.put("KEY-" + i, i);
        }
        System.out.println("Size: " + bigMap.size());
        // HashMap đã tự resize từ 16 → 32 buckets khi size > 12
        // TIP: Nếu biết trước số lượng phần tử, khởi tạo capacity để tránh resize
        // Map<String, Integer> optimized = new HashMap<>(32); // tránh resize không cần thiết
    }
}
// Output:
// === hashCode() của các key ===
// key1 ("ORD-001") hashCode: -1979525907
// key2 ("ORD-002") hashCode: -1979525906
// key3 ("ORD-001") hashCode: -1979525907
// key1 hashCode == key3 hashCode: true
//
// === HashMap sau khi put ===
// ORD-001: 750000.0
// ORD-002: 1200000.0
// Size: 2
//
// === Resize Demo ===
// Size: 20
```

### 2.2.3. ⚠️ THẢM HỌA: Mất dữ liệu khi dùng Custom Object làm Key mà KHÔNG override `equals()` và `hashCode()`

Đây là **bug kinh điển** mà dev chuyên nghiệp vẫn mắc. Khi bạn dùng object làm key trong HashMap mà không override đúng hai method này → **dữ liệu sẽ biến mất một cách bí ẩn**.

```java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class HashMapKeyDisasterDemo {
    public static void main(String[] args) {
        System.out.println("===== BUG: KHÔNG override equals/hashCode =====");
        demonstrateBug();

        System.out.println("\n===== FIX: ĐÃ override equals/hashCode =====");
        demonstrateFix();
    }

    static void demonstrateBug() {
        // OrderKey KHÔNG override equals/hashCode
        Map<OrderKeyBuggy, String> orderStatus = new HashMap<>();

        // Tạo key và put vào map
        OrderKeyBuggy key1 = new OrderKeyBuggy("ORD-001", "USER-A");
        orderStatus.put(key1, "PAID");
        System.out.println("Sau put → size: " + orderStatus.size());

        // Tạo key MỚI với CÙNG nội dung → cố get lại
        OrderKeyBuggy key2 = new OrderKeyBuggy("ORD-001", "USER-A");
        String status = orderStatus.get(key2);

        System.out.println("key1 == key2: " + (key1 == key2));         // false (khác object)
        System.out.println("key1.equals(key2): " + key1.equals(key2)); // false (dùng Object.equals = ==)
        System.out.println("key1.hashCode(): " + key1.hashCode());     // Mỗi object 1 hashCode khác nhau
        System.out.println("key2.hashCode(): " + key2.hashCode());
        System.out.println("Kết quả get: " + status);                  // null → MẤT DỮ LIỆU!

        // GIẢI THÍCH:
        // key2 có hashCode KHÁC key1 → HashMap tìm ở BUCKET KHÁC → không thấy → trả null
        // Ngay cả nếu trùng bucket, equals() trả false → vẫn không match
        System.out.println("⚠ DỮ LIỆU BIẾN MẤT dù key có cùng orderId + userId!");
    }

    static void demonstrateFix() {
        // OrderKeyFixed ĐÃ override equals/hashCode đúng cách
        Map<OrderKeyFixed, String> orderStatus = new HashMap<>();

        OrderKeyFixed key1 = new OrderKeyFixed("ORD-001", "USER-A");
        orderStatus.put(key1, "PAID");

        OrderKeyFixed key2 = new OrderKeyFixed("ORD-001", "USER-A");
        String status = orderStatus.get(key2);

        System.out.println("key1.equals(key2): " + key1.equals(key2)); // true
        System.out.println("key1.hashCode(): " + key1.hashCode());
        System.out.println("key2.hashCode(): " + key2.hashCode());     // CÙNG hashCode
        System.out.println("Kết quả get: " + status);                  // "PAID" → ĐÚNG!
        System.out.println("✅ Dữ liệu được tìm thấy chính xác!");
    }
}

// ❌ Class BUGGY: KHÔNG override equals/hashCode
class OrderKeyBuggy {
    String orderId;
    String userId;

    OrderKeyBuggy(String orderId, String userId) {
        this.orderId = orderId;
        this.userId = userId;
    }
    // Mặc định: equals() dùng == (so sánh địa chỉ)
    // Mặc định: hashCode() dùng identity hash (mỗi object 1 số khác nhau)
}

// ✅ Class FIXED: override equals + hashCode đúng contract
class OrderKeyFixed {
    String orderId;
    String userId;

    OrderKeyFixed(String orderId, String userId) {
        this.orderId = orderId;
        this.userId = userId;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        OrderKeyFixed that = (OrderKeyFixed) o;
        // Hai key "bằng nhau" nếu cùng orderId VÀ cùng userId
        return Objects.equals(orderId, that.orderId)
                && Objects.equals(userId, that.userId);
    }

    @Override
    public int hashCode() {
        // HỢP ĐỒNG: Nếu a.equals(b) == true → a.hashCode() == b.hashCode()
        // (Chiều ngược lại KHÔNG bắt buộc — 2 object khác nhau CÓ THỂ cùng hashCode → collision)
        return Objects.hash(orderId, userId);
    }
}
// Output:
// ===== BUG: KHÔNG override equals/hashCode =====
// Sau put → size: 1
// key1 == key2: false
// key1.equals(key2): false
// key1.hashCode(): 1836019240     (số cụ thể sẽ khác mỗi lần chạy)
// key2.hashCode(): 325040804
// Kết quả get: null
// ⚠ DỮ LIỆU BIẾN MẤT dù key có cùng orderId + userId!
//
// ===== FIX: ĐÃ override equals/hashCode =====
// key1.equals(key2): true
// key1.hashCode(): -1487964334
// key2.hashCode(): -1487964334
// Kết quả get: PAID
// ✅ Dữ liệu được tìm thấy chính xác!
```

> **LUẬT SẮT của HashMap Key:**
> 1. Nếu override `equals()` → **BẮT BUỘC** override `hashCode()`.
> 2. Nếu `a.equals(b) == true` → `a.hashCode() == b.hashCode()` (**BẮT BUỘC**).
> 3. Nếu `a.hashCode() == b.hashCode()` → `a.equals(b)` **KHÔNG nhất thiết true** (collision hợp lệ).
> 4. Key nên là **Immutable** (String, Integer...) vì nếu field thay đổi sau khi put → hashCode thay đổi → **mất key vĩnh viễn**.

---

## 2.3. GENERICS — XÓA KIỂU (TYPE ERASURE) & LUẬT PECS

### 2.3.1. Generics là gì và tại sao cần?

Trước Java 5 (không có Generics), Collection lưu `Object` → phải ép kiểu (cast) thủ công → dễ gặp `ClassCastException` lúc runtime.

```java
import java.util.ArrayList;
import java.util.List;

public class WhyGenericsDemo {
    public static void main(String[] args) {
        // ===== CÁCH CŨ (Java 4): Raw Type — KHÔNG Generics =====
        List rawList = new ArrayList();
        rawList.add("ORD-001");       // String
        rawList.add(42);              // Integer — Compiler KHÔNG cảnh báo!
        rawList.add(new User("X", "x@mail.com")); // User — vẫn OK!

        // Khi lấy ra → phải ép kiểu THỦ CÔNG
        // String orderId = (String) rawList.get(1); // ClassCastException lúc RUNTIME!
        System.out.println("Raw list size: " + rawList.size());
        System.out.println("⚠ Raw list không an toàn kiểu — bug chỉ phát hiện lúc runtime!");

        // ===== CÁCH MỚI (Java 5+): Generics — AN TOÀN KIỂU =====
        List<String> typedList = new ArrayList<>();
        typedList.add("ORD-001");
        typedList.add("ORD-002");
        // typedList.add(42);         // ❌ COMPILE ERROR! Compiler bắt lỗi ngay
        // typedList.add(new User()); // ❌ COMPILE ERROR!

        String orderId = typedList.get(0); // KHÔNG cần ép kiểu → an toàn
        System.out.println("Typed list get(0): " + orderId);
        System.out.println("✅ Generics bắt lỗi tại COMPILE TIME — an toàn hơn vạn lần");
    }
}
// Output:
// Raw list size: 3
// ⚠ Raw list không an toàn kiểu — bug chỉ phát hiện lúc runtime!
// Typed list get(0): ORD-001
// ✅ Generics bắt lỗi tại COMPILE TIME — an toàn hơn vạn lần
```

### 2.3.2. Type Erasure — Sự thật tàn khốc về Generics

**Bản chất**: Generics chỉ tồn tại lúc **compile time**. Khi bytecode được tạo ra, JVM **xóa sạch** (erase) mọi thông tin generic type. `List<String>` và `List<Integer>` trở thành **cùng một class** `List` trong bytecode.

```java
import java.util.ArrayList;
import java.util.List;

public class TypeErasureDemo {
    public static void main(String[] args) {
        List<String> stringList = new ArrayList<>();
        List<Integer> intList = new ArrayList<>();

        // Lúc RUNTIME, JVM chỉ thấy "ArrayList" — KHÔNG phân biệt type parameter
        System.out.println("stringList class: " + stringList.getClass().getName());
        System.out.println("intList class: " + intList.getClass().getName());
        System.out.println("Cùng class? " + (stringList.getClass() == intList.getClass()));

        // HỆ QUẢ 1: Không thể dùng instanceof với generic type
        // if (stringList instanceof List<String>) {} // ❌ Compile Error!
        // Chỉ có thể: if (stringList instanceof List<?>) {} // ✅

        // HỆ QUẢ 2: Không thể tạo mảng generic
        // T[] array = new T[10]; // ❌ Compile Error!

        // HỆ QUẢ 3: Không thể dùng primitive type với Generics
        // List<int> list = new ArrayList<>(); // ❌ phải dùng Integer wrapper

        // CHỨNG MINH Type Erasure bằng Reflection
        System.out.println("\n=== Chứng minh Type Erasure ===");
        try {
            // Ép thêm Integer vào List<String> qua Reflection → JVM KHÔNG chặn!
            List<String> secureList = new ArrayList<>();
            secureList.add("Hello");

            // Dùng Reflection bypass Generics → vì runtime không có type info
            secureList.getClass()
                      .getMethod("add", Object.class)
                      .invoke(secureList, 12345); // Thêm Integer vào List<String>!

            System.out.println("secureList: " + secureList);
            System.out.println("⚠ Reflection bypass Generics vì Type Erasure!");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
// Output:
// stringList class: java.util.ArrayList
// intList class: java.util.ArrayList
// Cùng class? true
//
// === Chứng minh Type Erasure ===
// secureList: [Hello, 12345]
// ⚠ Reflection bypass Generics vì Type Erasure!
```

### 2.3.3. PECS — Producer Extends, Consumer Super

Đây là quy tắc **khiến 95% dev bối rối** nhưng lại cực kỳ quan trọng khi thiết kế API. PECS quyết định khi nào dùng `? extends T` và khi nào dùng `? super T`.

| Wildcard | Ý nghĩa | Cho phép | KHÔNG cho phép |
|----------|---------|---------|----------------|
| `? extends T` | "Kiểu T hoặc con của T" | **ĐỌC** (get) từ collection | GHI (add) vào collection (trừ null) |
| `? super T` | "Kiểu T hoặc cha của T" | **GHI** (add) vào collection | ĐỌC ra kiểu cụ thể (chỉ đọc được Object) |

**Nhớ đơn giản:**
- **PE** = **P**roducer **E**xtends → Collection **sản xuất** dữ liệu cho bạn → dùng `extends` → bạn chỉ **get** từ nó.
- **CS** = **C**onsumer **S**uper → Collection **tiêu thụ** dữ liệu bạn đưa vào → dùng `super` → bạn chỉ **add** vào nó.

```java
import java.util.ArrayList;
import java.util.List;

public class PecsDemo {
    public static void main(String[] args) {
        // ===== Hệ thống Payment: Có cấu trúc kế thừa =====
        // BasePayment (cha)
        //   ├── MomoPayment (con)
        //   └── VNPayPayment (con)

        List<MomoPayment> momoPayments = new ArrayList<>();
        momoPayments.add(new MomoPayment("TXN-M1", 100_000));
        momoPayments.add(new MomoPayment("TXN-M2", 200_000));

        List<VNPayPayment> vnpayPayments = new ArrayList<>();
        vnpayPayments.add(new VNPayPayment("TXN-V1", 300_000));

        // ===== PRODUCER EXTENDS: Đọc dữ liệu từ collection =====
        System.out.println("=== PECS: Producer Extends (ĐỌC) ===");
        double momoTotal = calculateTotal(momoPayments);   // List<MomoPayment> → OK
        double vnpayTotal = calculateTotal(vnpayPayments); // List<VNPayPayment> → OK
        System.out.println("Tổng Momo: " + momoTotal + "đ");
        System.out.println("Tổng VNPay: " + vnpayTotal + "đ");

        // ===== CONSUMER SUPER: Ghi dữ liệu vào collection =====
        System.out.println("\n=== PECS: Consumer Super (GHI) ===");
        List<BasePayment> allPayments = new ArrayList<>();
        addSamplePayments(allPayments); // List<BasePayment> → OK (BasePayment super MomoPayment)

        List<Object> objectList = new ArrayList<>();
        addSamplePayments(objectList);  // List<Object> → cũng OK! (Object super MomoPayment)

        System.out.println("allPayments size: " + allPayments.size());
        for (BasePayment p : allPayments) {
            System.out.println("  → " + p.getTransactionId() + ": " + p.getAmount() + "đ");
        }
    }

    // PRODUCER EXTENDS: Collection là NGUỒN DỮ LIỆU (producer) → dùng "extends"
    // Bạn chỉ ĐỌC (get) từ source, không ghi vào
    static double calculateTotal(List<? extends BasePayment> payments) {
        double total = 0;
        for (BasePayment payment : payments) { // Đọc ra kiểu BasePayment → OK
            total += payment.getAmount();
        }
        // payments.add(new MomoPayment(...)); // ❌ COMPILE ERROR! Không thể add
        // Lý do: Compiler không biết List chứa Momo hay VNPay → từ chối để đảm bảo type safety
        return total;
    }

    // CONSUMER SUPER: Collection là ĐÍCH ĐẾN (consumer) → dùng "super"
    // Bạn chỉ GHI (add) vào destination, không đọc kiểu cụ thể
    static void addSamplePayments(List<? super MomoPayment> destination) {
        destination.add(new MomoPayment("TXN-SAMPLE-1", 50_000));  // ✅ OK
        destination.add(new MomoPayment("TXN-SAMPLE-2", 75_000));  // ✅ OK
        // destination.add(new BasePayment("TXN-X", 0));            // ❌ Compile Error!
        // Lý do: List<? super MomoPayment> đảm bảo nhận MomoPayment, nhưng BasePayment chưa chắc

        // Khi đọc ra chỉ được kiểu Object (không biết cụ thể là kiểu gì)
        Object item = destination.get(0); // Chỉ đọc được Object
        System.out.println("  Đọc ra từ consumer: " + item.getClass().getSimpleName());
    }
}

class BasePayment {
    private String transactionId;
    private double amount;

    public BasePayment(String transactionId, double amount) {
        this.transactionId = transactionId;
        this.amount = amount;
    }

    public String getTransactionId() { return transactionId; }
    public double getAmount() { return amount; }
}

class MomoPayment extends BasePayment {
    public MomoPayment(String txnId, double amount) {
        super(txnId, amount);
    }
}

class VNPayPayment extends BasePayment {
    public VNPayPayment(String txnId, double amount) {
        super(txnId, amount);
    }
}
// Output:
// === PECS: Producer Extends (ĐỌC) ===
// Tổng Momo: 300000.0đ
// Tổng VNPay: 300000.0đ
//
// === PECS: Consumer Super (GHI) ===
//   Đọc ra từ consumer: MomoPayment
//   Đọc ra từ consumer: MomoPayment
// allPayments size: 2
//   → TXN-SAMPLE-1: 50000.0đ
//   → TXN-SAMPLE-2: 75000.0đ
```

### 2.3.4. Ứng dụng PECS trong thực tế: Viết Generic Repository

```java
import java.util.*;
import java.util.stream.Collectors;

public class GenericRepositoryDemo {
    public static void main(String[] args) {
        // Repository generic: dùng được cho BẤT KỲ entity nào
        InMemoryRepository<User> userRepo = new InMemoryRepository<>();
        userRepo.save(new User("Triet", "triet@gmail.com"));
        userRepo.save(new User("Minh", "minh@gmail.com"));
        userRepo.save(new User("An", "an@gmail.com"));

        // Tìm kiếm bằng predicate
        List<User> gmailUsers = userRepo.findAll(u -> u.getEmail().endsWith("@gmail.com"));
        System.out.println("Gmail users: " + gmailUsers.size());

        // Copy kết quả vào List<Object> → dùng PECS (Consumer Super)
        List<Object> destination = new ArrayList<>();
        userRepo.copyAllTo(destination);
        System.out.println("Copied to Object list, size: " + destination.size());
    }
}

// Generic Repository — T là type parameter, bị XÓA lúc runtime (Type Erasure)
class InMemoryRepository<T> {
    private final List<T> storage = new ArrayList<>();

    public void save(T entity) {
        storage.add(entity);
        System.out.println("💾 Saved: " + entity);
    }

    // PECS → Producer Extends (không cần ở đây vì T đã rõ ràng)
    public List<T> findAll(java.util.function.Predicate<T> filter) {
        return storage.stream().filter(filter).collect(Collectors.toList());
    }

    // PECS → Consumer Super: destination nhận T hoặc cha của T
    public void copyAllTo(List<? super T> destination) {
        destination.addAll(storage);
    }

    public int count() { return storage.size(); }
}

class User {
    private String name;
    private String email;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public String getName() { return name; }
    public String getEmail() { return email; }

    @Override
    public String toString() {
        return "User{" + name + ", " + email + "}";
    }
}
// Output:
// 💾 Saved: User{Triet, triet@gmail.com}
// 💾 Saved: User{Minh, minh@gmail.com}
// 💾 Saved: User{An, an@gmail.com}
// Gmail users: 3
// Copied to Object list, size: 3
```

---

## 🔥 2.4. GÓC PHỎNG VẤN — PHẦN 2

### ❓ Câu 1: "Tại sao nói HashMap `get()` là O(1)? Nó có thực sự là O(1) không?"

**Đáp án sắc bén:**

> *"O(1) của HashMap là **amortized average case**, KHÔNG PHẢI absolute guarantee.*
>
> *Trong trường hợp lý tưởng (hash phân bố đều, ít collision), `get()` chỉ cần: (1) tính hashCode → O(1), (2) tìm bucket bằng index → O(1), (3) so sánh equals() → O(1). Tổng = O(1).*
>
> *Nhưng trong **worst case** (tất cả key hash vào cùng 1 bucket):*
> - *Trước Java 8: linked list → O(n)*
> - *Từ Java 8: Red-Black Tree khi ≥ 8 collisions → O(log n)*
>
> *Ngoài ra, khi HashMap resize (size > capacity × loadFactor), toàn bộ entry bị **rehash** → O(n) cho lần put đó. Nhưng amortized ra vẫn là O(1).*
>
> *Tip: Dùng `String` hoặc `Integer` làm key vì chúng có hashCode() chất lượng cao và immutable."*

---

### ❓ Câu 2: "Hash Collision là gì? Cho ví dụ 2 object khác nhau nhưng cùng hashCode."

**Đáp án sắc bén:**

> *"Hash Collision xảy ra khi 2 object KHÁC NHAU (equals() trả false) nhưng có CÙNG hashCode(). Điều này hoàn toàn hợp lệ — vì hashCode() trả về `int` (khoảng 4.2 tỷ giá trị), nhưng số lượng object có thể là vô hạn.*
>
> *Ví dụ kinh điển: `"Aa".hashCode() == "BB".hashCode()` → cả hai đều bằng `2112` (trong Java). Đây là collision tự nhiên.*
>
> *HashMap xử lý collision bằng: (1) Đặt cùng bucket, (2) Duyệt linked list/tree trong bucket, (3) Dùng equals() để tìm đúng entry."*

```java
public class HashCollisionProof {
    public static void main(String[] args) {
        String s1 = "Aa";
        String s2 = "BB";

        System.out.println("\"Aa\".hashCode() = " + s1.hashCode());
        System.out.println("\"BB\".hashCode() = " + s2.hashCode());
        System.out.println("Cùng hashCode? " + (s1.hashCode() == s2.hashCode()));
        System.out.println("equals()? " + s1.equals(s2));
        System.out.println("→ Đây là Hash Collision hợp lệ!");

        // HashMap vẫn hoạt động đúng vì dùng equals() để phân biệt
        java.util.HashMap<String, String> map = new java.util.HashMap<>();
        map.put("Aa", "Value-Aa");
        map.put("BB", "Value-BB");
        System.out.println("\nmap.get(\"Aa\"): " + map.get("Aa"));
        System.out.println("map.get(\"BB\"): " + map.get("BB"));
        System.out.println("Size: " + map.size()); // 2 — không bị ghi đè
    }
}
// Output:
// "Aa".hashCode() = 2112
// "BB".hashCode() = 2112
// Cùng hashCode? true
// equals()? false
// → Đây là Hash Collision hợp lệ!
//
// map.get("Aa"): Value-Aa
// map.get("BB"): Value-BB
// Size: 2
```

---

### ❓ Câu 3: "Phân biệt List, Set, Map — Khi nào dùng cái nào trong hệ thống Backend?"

**Đáp án sắc bén:**

> | Collection | Đặc điểm | Use case thực tế |
> |-----------|----------|-----------------|
> | **List** (`ArrayList`) | Có thứ tự, cho phép trùng lặp, truy cập bằng index | Danh sách đơn hàng của user, lịch sử giao dịch, kết quả tìm kiếm (cần phân trang) |
> | **Set** (`HashSet`, `LinkedHashSet`) | KHÔNG trùng lặp, KHÔNG đảm bảo thứ tự (HashSet) | Danh sách permission/role của user (không muốn trùng), danh sách email unique, blacklist IP |
> | **Map** (`HashMap`, `LinkedHashMap`) | Key-Value, key KHÔNG trùng | Cache (userId → User object), config (key → value), grouping (category → list of products) |
>
> *"Nguyên tắc: Cần thứ tự + trùng lặp OK → **List**. Cần loại bỏ trùng lặp → **Set**. Cần ánh xạ key-value → **Map**. Trong Spring Boot, `List` dùng nhiều nhất (trả về từ Repository), `Map` dùng cho cache và DTO mapping, `Set` dùng cho quan hệ Many-to-Many (ví dụ: User → Set\<Role\>)."*

```java
import java.util.*;

public class CollectionUseCaseDemo {
    public static void main(String[] args) {
        // LIST: Danh sách đơn hàng (có thứ tự, có thể trùng user)
        List<String> recentOrders = new ArrayList<>();
        recentOrders.add("ORD-001");
        recentOrders.add("ORD-002");
        recentOrders.add("ORD-001"); // Trùng → OK trong List
        System.out.println("Recent Orders (List): " + recentOrders);
        // [ORD-001, ORD-002, ORD-001]

        // SET: Danh sách quyền (KHÔNG cho trùng)
        Set<String> userRoles = new LinkedHashSet<>(); // Giữ thứ tự insert
        userRoles.add("ROLE_USER");
        userRoles.add("ROLE_ADMIN");
        userRoles.add("ROLE_USER"); // Trùng → BỊ BỎ QUA
        System.out.println("User Roles (Set): " + userRoles);
        // [ROLE_USER, ROLE_ADMIN] → chỉ 2 phần tử

        // MAP: Cache user (userId → User)
        Map<String, String> userCache = new HashMap<>();
        userCache.put("USR-001", "Triet");
        userCache.put("USR-002", "Minh");
        userCache.put("USR-001", "Triet Updated"); // Cùng key → GHI ĐÈ
        System.out.println("User Cache (Map): " + userCache);
        // {USR-002=Minh, USR-001=Triet Updated}

        // TREEMAP: Tự động sắp xếp theo key
        Map<String, Double> sortedRevenue = new TreeMap<>();
        sortedRevenue.put("2024-03", 15_000_000.0);
        sortedRevenue.put("2024-01", 10_000_000.0);
        sortedRevenue.put("2024-02", 12_000_000.0);
        System.out.println("Revenue (TreeMap, sorted): " + sortedRevenue);
        // {2024-01=1.0E7, 2024-02=1.2E7, 2024-03=1.5E7}
    }
}
// Output:
// Recent Orders (List): [ORD-001, ORD-002, ORD-001]
// User Roles (Set): [ROLE_USER, ROLE_ADMIN]
// User Cache (Map): {USR-002=Minh, USR-001=Triet Updated}
// Revenue (TreeMap, sorted): {2024-01=1.0E7, 2024-02=1.2E7, 2024-03=1.5E7}
```

---

> **KẾT THÚC PHẦN 2** — Bạn đã nắm được: ArrayList vs LinkedList (bao gồm Cache Locality), HashMap sâu đến tận Red-Black Tree, thảm họa mất dữ liệu khi dùng Custom Key, Type Erasure, và quy luật PECS.
