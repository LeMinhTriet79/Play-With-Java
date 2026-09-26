# 📕 CẨM NANG SINH TỒN JAVA CORE & OOP THỰC CHIẾN

## PHẦN 4: JAVA 8+ FEATURES & SOLID PRINCIPLES

> *"Java 8 không chỉ là 'thêm Lambda'. Nó thay đổi cách bạn TƯ DUY về xử lý dữ liệu — từ 'làm thế nào' (imperative) sang 'làm gì' (declarative)."*

---

## 4.1. STREAM API — TUYÊN NGÔN KHAI TỬ VÒNG LẶP FOR

### 4.1.1. Tại sao phải học Stream API?

Vòng lặp `for` bắt bạn phải **mô tả từng bước** (imperative): khởi tạo biến, duyệt, kiểm tra điều kiện, thêm vào list kết quả. Stream API cho phép bạn **khai báo ý muốn** (declarative): lọc → biến đổi → thu thập.

**Ưu điểm Stream:**
- Code **ngắn hơn 3-5 lần**, dễ đọc hơn
- Hỗ trợ **Lazy Evaluation** — không tính toán cho đến khi gọi terminal operation
- Dễ dàng chuyển sang **Parallel Stream** để xử lý song song
- Phù hợp với tư duy **Functional Programming**

### 4.1.2. Demo so sánh: For loop cổ điển vs Stream API

```java
import java.util.*;
import java.util.stream.*;

public class StreamVsForDemo {
    public static void main(String[] args) {
        // Dữ liệu đơn hàng mẫu
        List<Order> orders = Arrays.asList(
            new Order("ORD-001", "Electronics", 2_500_000, "PAID"),
            new Order("ORD-002", "Books",       150_000,   "PAID"),
            new Order("ORD-003", "Electronics", 8_000_000, "PENDING"),
            new Order("ORD-004", "Fashion",     3_200_000, "PAID"),
            new Order("ORD-005", "Books",       450_000,   "CANCELLED"),
            new Order("ORD-006", "Electronics", 1_200_000, "PAID"),
            new Order("ORD-007", "Fashion",     750_000,   "PAID"),
            new Order("ORD-008", "Books",       980_000,   "PAID"),
            new Order("ORD-009", "Electronics", 15_000_000,"PAID"),
            new Order("ORD-010", "Fashion",     200_000,   "PENDING")
        );

        // ============================================================
        // BÀI TOÁN 1: Lọc đơn hàng đã thanh toán có giá > 1,000,000đ
        //             → lấy danh sách mã đơn
        // ============================================================

        System.out.println("=== BÀI TOÁN 1: Lọc đơn > 1 triệu, đã thanh toán ===\n");

        // --- CÁCH CŨ: For loop ---
        List<String> resultOld = new ArrayList<>();
        for (Order order : orders) {
            if (order.getStatus().equals("PAID") && order.getAmount() > 1_000_000) {
                resultOld.add(order.getOrderId());
            }
        }
        System.out.println("[FOR LOOP] " + resultOld);

        // --- CÁCH MỚI: Stream API ---
        List<String> resultNew = orders.stream()
            .filter(o -> o.getStatus().equals("PAID"))   // Lọc: chỉ lấy PAID
            .filter(o -> o.getAmount() > 1_000_000)       // Lọc: giá > 1 triệu
            .map(Order::getOrderId)                        // Biến đổi: Order → orderId
            .collect(Collectors.toList());                 // Thu thập: thành List
        System.out.println("[STREAM]   " + resultNew);

        // ============================================================
        // BÀI TOÁN 2: Gom nhóm đơn hàng theo danh mục (category)
        // ============================================================

        System.out.println("\n=== BÀI TOÁN 2: Gom nhóm theo danh mục ===\n");

        // --- CÁCH CŨ: For loop ---
        Map<String, List<Order>> groupedOld = new HashMap<>();
        for (Order order : orders) {
            String category = order.getCategory();
            if (!groupedOld.containsKey(category)) {
                groupedOld.put(category, new ArrayList<>());
            }
            groupedOld.get(category).add(order);
        }
        System.out.println("[FOR LOOP] Số nhóm: " + groupedOld.size());
        groupedOld.forEach((cat, list) ->
            System.out.println("  " + cat + ": " + list.size() + " đơn"));

        // --- CÁCH MỚI: Stream API — 1 dòng! ---
        Map<String, List<Order>> groupedNew = orders.stream()
            .collect(Collectors.groupingBy(Order::getCategory));
        System.out.println("\n[STREAM] Số nhóm: " + groupedNew.size());
        groupedNew.forEach((cat, list) ->
            System.out.println("  " + cat + ": " + list.size() + " đơn"));

        // ============================================================
        // BÀI TOÁN 3: Thống kê — Tổng doanh thu theo danh mục (chỉ PAID)
        // ============================================================

        System.out.println("\n=== BÀI TOÁN 3: Tổng doanh thu theo danh mục (PAID) ===\n");

        Map<String, Double> revenueByCategory = orders.stream()
            .filter(o -> o.getStatus().equals("PAID"))
            .collect(Collectors.groupingBy(
                Order::getCategory,
                Collectors.summingDouble(Order::getAmount)
            ));
        revenueByCategory.forEach((cat, revenue) ->
            System.out.println("  " + cat + ": " + String.format("%,.0f", revenue) + "đ"));

        // ============================================================
        // BÀI TOÁN 4: Tìm đơn hàng có giá trị cao nhất
        // ============================================================

        System.out.println("\n=== BÀI TOÁN 4: Đơn hàng giá trị cao nhất ===\n");

        Optional<Order> maxOrder = orders.stream()
            .filter(o -> o.getStatus().equals("PAID"))
            .max(Comparator.comparingDouble(Order::getAmount));

        // Optional: Xử lý trường hợp KHÔNG CÓ kết quả (thay vì return null!)
        maxOrder.ifPresentOrElse(
            o -> System.out.println("Đơn lớn nhất: " + o.getOrderId()
                                  + " → " + String.format("%,.0f", o.getAmount()) + "đ"),
            () -> System.out.println("Không có đơn hàng nào")
        );

        // ============================================================
        // BÀI TOÁN 5: Pipeline phức tạp — Top 3 đơn hàng Electronics, sắp xếp giảm dần
        // ============================================================

        System.out.println("\n=== BÀI TOÁN 5: Top 3 đơn Electronics ===\n");

        orders.stream()
            .filter(o -> o.getCategory().equals("Electronics"))
            .filter(o -> o.getStatus().equals("PAID"))
            .sorted(Comparator.comparingDouble(Order::getAmount).reversed())
            .limit(3)
            .forEach(o -> System.out.println("  " + o.getOrderId()
                        + " → " + String.format("%,.0f", o.getAmount()) + "đ"));
    }
}

class Order {
    private String orderId;
    private String category;
    private double amount;
    private String status;

    Order(String orderId, String category, double amount, String status) {
        this.orderId = orderId;
        this.category = category;
        this.amount = amount;
        this.status = status;
    }

    public String getOrderId() { return orderId; }
    public String getCategory() { return category; }
    public double getAmount() { return amount; }
    public String getStatus() { return status; }
}
// Output:
// === BÀI TOÁN 1: Lọc đơn > 1 triệu, đã thanh toán ===
//
// [FOR LOOP] [ORD-001, ORD-004, ORD-006, ORD-009]
// [STREAM]   [ORD-001, ORD-004, ORD-006, ORD-009]
//
// === BÀI TOÁN 2: Gom nhóm theo danh mục ===
//
// [FOR LOOP] Số nhóm: 3
//   Electronics: 4 đơn
//   Books: 3 đơn
//   Fashion: 3 đơn
//
// [STREAM] Số nhóm: 3
//   Electronics: 4 đơn
//   Books: 3 đơn
//   Fashion: 3 đơn
//
// === BÀI TOÁN 3: Tổng doanh thu theo danh mục (PAID) ===
//
//   Electronics: 18,700,000đ
//   Books: 1,130,000đ
//   Fashion: 3,950,000đ
//
// === BÀI TOÁN 4: Đơn hàng giá trị cao nhất ===
//
// Đơn lớn nhất: ORD-009 → 15,000,000đ
//
// === BÀI TOÁN 5: Top 3 đơn Electronics ===
//
//   ORD-009 → 15,000,000đ
//   ORD-001 → 2,500,000đ
//   ORD-006 → 1,200,000đ
```

---

### 4.1.3. Lambda Expression — Bản chất sâu xa

Lambda **KHÔNG PHẢI** là "cú pháp viết tắt". Nó là **instance của Functional Interface** — một interface chỉ có **đúng 1 abstract method**.

```java
import java.util.function.*;
import java.util.Arrays;
import java.util.List;

public class LambdaEssenceDemo {
    public static void main(String[] args) {
        // ===== TRƯỚC JAVA 8: Anonymous Inner Class =====
        System.out.println("=== Anonymous Inner Class (cách cũ) ===");
        Runnable oldWay = new Runnable() {
            @Override
            public void run() {
                System.out.println("Task chạy bằng Anonymous Class");
            }
        };
        oldWay.run();

        // ===== JAVA 8: Lambda Expression =====
        System.out.println("\n=== Lambda (cách mới) ===");
        Runnable newWay = () -> System.out.println("Task chạy bằng Lambda");
        newWay.run();

        // BẢN CHẤT: Lambda được compile thành invokedynamic bytecode
        // JVM tạo ra 1 class ẩn (hidden class) implements Functional Interface
        // → KHÔNG tạo .class file riêng như Anonymous Class → nhẹ hơn

        // ===== CÁC FUNCTIONAL INTERFACE CÓ SẴN =====
        System.out.println("\n=== Built-in Functional Interfaces ===");

        // Predicate<T>: T → boolean (dùng cho filter)
        Predicate<Double> isExpensive = amount -> amount > 1_000_000;
        System.out.println("5tr đắt? " + isExpensive.test(5_000_000.0));

        // Function<T, R>: T → R (dùng cho map/transform)
        Function<String, String> maskEmail =
            email -> email.replaceAll("(.)(.+)(@.+)", "$1***$3");
        System.out.println("Mask: " + maskEmail.apply("triet@gmail.com"));

        // Consumer<T>: T → void (dùng cho forEach)
        Consumer<String> logger = msg -> System.out.println("[LOG] " + msg);
        logger.accept("User logged in");

        // Supplier<T>: () → T (dùng cho factory/lazy init)
        Supplier<String> txnIdGenerator =
            () -> "TXN-" + System.currentTimeMillis();
        System.out.println("Generated: " + txnIdGenerator.get());

        // BiFunction<T, U, R>: (T, U) → R
        BiFunction<Double, Double, Double> calculateTax =
            (amount, rate) -> amount * rate / 100;
        System.out.println("Tax: " + calculateTax.apply(1_000_000.0, 10.0) + "đ");

        // ===== METHOD REFERENCE — Viết gọn hơn Lambda =====
        System.out.println("\n=== Method Reference ===");
        List<String> emails = Arrays.asList("a@mail.com", "b@mail.com", "c@mail.com");

        // Lambda
        emails.forEach(e -> System.out.println(e));
        // Method Reference (tương đương, ngắn hơn)
        emails.forEach(System.out::println);

        // Static method reference
        List<String> amounts = Arrays.asList("100000", "250000", "500000");
        // Lambda: amounts.stream().map(s -> Double.parseDouble(s))
        amounts.stream()
            .map(Double::parseDouble)          // Method ref: Class::staticMethod
            .map(a -> String.format("%,.0f", a))
            .forEach(a -> System.out.println("  Amount: " + a + "đ"));
    }
}
// Output:
// === Anonymous Inner Class (cách cũ) ===
// Task chạy bằng Anonymous Class
//
// === Lambda (cách mới) ===
// Task chạy bằng Lambda
//
// === Built-in Functional Interfaces ===
// 5tr đắt? true
// Mask: t***@gmail.com
// [LOG] User logged in
// Generated: TXN-1695729600000
// Tax: 100000.0đ
//
// === Method Reference ===
// a@mail.com
// b@mail.com
// c@mail.com
//   Amount: 100,000đ
//   Amount: 250,000đ
//   Amount: 500,000đ
```

### 4.1.4. Optional — Khai tử NullPointerException

```java
import java.util.Optional;

public class OptionalDemo {
    public static void main(String[] args) {
        // ===== CÁCH CŨ: null check lồng nhau (Pyramid of Doom) =====
        System.out.println("=== Cách cũ: null check ===");
        UserProfile user = findUserOld("USR-001");
        String city;
        if (user != null) {
            Address address = user.getAddress();
            if (address != null) {
                city = address.getCity();
                if (city != null) {
                    System.out.println("City: " + city);
                }
            }
        }

        // ===== CÁCH MỚI: Optional chain =====
        System.out.println("\n=== Cách mới: Optional ===");
        String result = findUser("USR-001")
            .flatMap(UserProfile::getOptionalAddress)    // Optional<Address>
            .map(Address::getCity)                       // Optional<String>
            .orElse("Không có thông tin thành phố");     // Default value

        System.out.println("City: " + result);

        // User không tồn tại
        String result2 = findUser("USR-999")
            .flatMap(UserProfile::getOptionalAddress)
            .map(Address::getCity)
            .orElse("Không có thông tin thành phố");

        System.out.println("City (USR-999): " + result2);

        // ===== orElseThrow — Ném exception nếu không có giá trị =====
        System.out.println("\n=== orElseThrow ===");
        try {
            UserProfile mustExist = findUser("USR-999")
                .orElseThrow(() -> new RuntimeException("User không tồn tại!"));
        } catch (RuntimeException e) {
            System.out.println("❌ " + e.getMessage());
        }

        // ===== ANTI-PATTERN: KHÔNG LÀM NHƯ NÀY =====
        // ❌ Optional.get() → ném NoSuchElementException nếu empty
        // ❌ if (optional.isPresent()) optional.get() → quay lại null check kiểu cũ
        // ❌ Dùng Optional cho field/parameter → chỉ dùng cho RETURN TYPE
    }

    // Cách cũ: return null nếu không tìm thấy → NGUY HIỂM
    static UserProfile findUserOld(String id) {
        if (id.equals("USR-001"))
            return new UserProfile("Triet", new Address("HCMC"));
        return null;
    }

    // Cách mới: return Optional → BUỘC caller xử lý trường hợp rỗng
    static Optional<UserProfile> findUser(String id) {
        if (id.equals("USR-001"))
            return Optional.of(new UserProfile("Triet", new Address("HCMC")));
        return Optional.empty(); // Thay vì return null
    }
}

class Address {
    private String city;
    Address(String city) { this.city = city; }
    public String getCity() { return city; }
}

class UserProfile {
    private String name;
    private Address address;

    UserProfile(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public Address getAddress() { return address; }
    public Optional<Address> getOptionalAddress() { return Optional.ofNullable(address); }
}
// Output:
// === Cách cũ: null check ===
// City: HCMC
//
// === Cách mới: Optional ===
// City: HCMC
// City (USR-999): Không có thông tin thành phố
//
// === orElseThrow ===
// ❌ User không tồn tại!
```

---

## 4.2. 5 NGUYÊN TẮC SOLID — TÁI CẤU TRÚC HỆ THỐNG GỬI THÔNG BÁO

Chúng ta sẽ bắt đầu với **code VI PHẠM** SOLID, sau đó **refactor** từng bước theo đúng 5 nguyên tắc.

### 4.2.0. CODE BAN ĐẦU — VI PHẠM TẤT CẢ SOLID

```java
// ❌ CODE VI PHẠM — Class "thần thánh" làm tất cả mọi thứ
public class NotificationManagerBad {
    // VI PHẠM SRP: 1 class làm 3 việc (email, sms, format)
    // VI PHẠM OCP: Thêm Slack → phải SỬA class này
    // VI PHẠM DIP: Phụ thuộc trực tiếp vào implementation

    public void sendNotification(String type, String recipient, String message) {
        // VI PHẠM OCP: if-else chain → mỗi khi thêm loại mới → sửa code
        if (type.equals("EMAIL")) {
            // Logic gửi email
            System.out.println("Connecting to SMTP server...");
            System.out.println("Sending email to " + recipient + ": " + message);
        } else if (type.equals("SMS")) {
            // Logic gửi SMS
            System.out.println("Calling Twilio API...");
            System.out.println("Sending SMS to " + recipient + ": " + message);
        } else if (type.equals("SLACK")) {
            // Mỗi lần thêm kênh → phải SỬA file này → VI PHẠM OCP
            System.out.println("Posting to Slack channel " + recipient + ": " + message);
        }
        // Thêm Telegram? → lại sửa class... vĩnh viễn bành trướng
    }
}
```

### 4.2.1. S — Single Responsibility Principle (SRP)

> **"Một class chỉ nên có MỘT lý do để thay đổi."**

```java
// ✅ SRP: Mỗi class chỉ làm 1 việc duy nhất

// Class 1: Chỉ chịu trách nhiệm FORMAT nội dung thông báo
class NotificationFormatter {
    public String formatOrderConfirmation(String orderId, double amount) {
        return String.format("[Xác nhận] Đơn hàng %s - Số tiền: %,.0fđ đã được xử lý.",
                orderId, amount);
    }

    public String formatPasswordReset(String userName) {
        return String.format("Xin chào %s, click link bên dưới để đặt lại mật khẩu.", userName);
    }
}

// Class 2: Chỉ chịu trách nhiệm GỬI email
class EmailSender {
    public void send(String to, String content) {
        System.out.println("📧 [SMTP] → " + to + ": " + content);
    }
}

// Class 3: Chỉ chịu trách nhiệm GỬI SMS
class SmsSender {
    public void send(String phoneNumber, String content) {
        System.out.println("📱 [Twilio] → " + phoneNumber + ": " + content);
    }
}
// → Thay đổi cách format? Sửa NotificationFormatter. Không ảnh hưởng sender.
// → Đổi provider SMS? Sửa SmsSender. Không ảnh hưởng formatter.
```

### 4.2.2. O — Open/Closed Principle (OCP)

> **"Mở cho mở rộng (extension), đóng cho sửa đổi (modification)."**
> Thêm tính năng mới bằng cách TẠO code mới, không SỬA code cũ.

```java
import java.util.*;

public class OcpDemo {
    public static void main(String[] args) {
        // Thêm kênh mới = thêm class → KHÔNG SỬA code cũ
        List<NotificationChannel> channels = Arrays.asList(
            new EmailChannel(),
            new SmsChannel(),
            new SlackChannel()   // Mới thêm → KHÔNG sửa NotificationDispatcher
        );

        NotificationDispatcher dispatcher = new NotificationDispatcher(channels);
        dispatcher.broadcast("Hệ thống bảo trì lúc 2:00 AM");
    }
}

// INTERFACE: Hợp đồng mở rộng
interface NotificationChannel {
    void send(String message);
    String getChannelName();
}

// Implementation 1
class EmailChannel implements NotificationChannel {
    @Override
    public void send(String message) {
        System.out.println("📧 [Email] " + message);
    }
    @Override
    public String getChannelName() { return "Email"; }
}

// Implementation 2
class SmsChannel implements NotificationChannel {
    @Override
    public void send(String message) {
        System.out.println("📱 [SMS] " + message);
    }
    @Override
    public String getChannelName() { return "SMS"; }
}

// Implementation 3: Thêm MỚI — không sửa bất kỳ class nào ở trên
class SlackChannel implements NotificationChannel {
    @Override
    public void send(String message) {
        System.out.println("💬 [Slack] " + message);
    }
    @Override
    public String getChannelName() { return "Slack"; }
}

// Dispatcher: ĐÓNG cho sửa đổi — code này KHÔNG BAO GIỜ cần sửa
class NotificationDispatcher {
    private final List<NotificationChannel> channels;

    NotificationDispatcher(List<NotificationChannel> channels) {
        this.channels = channels;
    }

    // Dùng Đa hình → không có if-else chain
    public void broadcast(String message) {
        System.out.println("=== Broadcasting to " + channels.size() + " channels ===");
        for (NotificationChannel channel : channels) {
            channel.send(message); // Đa hình quyết định runtime
        }
    }
}
// Output:
// === Broadcasting to 3 channels ===
// 📧 [Email] Hệ thống bảo trì lúc 2:00 AM
// 📱 [SMS] Hệ thống bảo trì lúc 2:00 AM
// 💬 [Slack] Hệ thống bảo trì lúc 2:00 AM
```

### 4.2.3. L — Liskov Substitution Principle (LSP)

> **"Class con phải thay thế được class cha mà KHÔNG phá vỡ logic chương trình."**

```java
public class LspDemo {
    public static void main(String[] args) {
        // ===== VI PHẠM LSP =====
        System.out.println("=== VI PHẠM LSP ===");
        NotificationSender emailSender = new EmailNotificationSender();
        NotificationSender readOnlySender = new ReadOnlyAuditLogger();

        sendSafely(emailSender, "Hello from email");     // OK
        sendSafely(readOnlySender, "Hello from audit");  // BÙM! UnsupportedOperationException
        // → ReadOnlyAuditLogger KHÔNG THỂ thay thế NotificationSender → VI PHẠM LSP

        // ===== TUÂN THỦ LSP =====
        System.out.println("\n=== TUÂN THỦ LSP ===");
        // Tách interface: Gửi thông báo vs Ghi log là 2 hợp đồng KHÁC NHAU
        Sendable email = new EmailSendable();
        Loggable audit = new AuditLogger();

        email.send("Notification content");   // ✅ Đúng hợp đồng
        audit.log("Audit record");            // ✅ Đúng hợp đồng
    }

    static void sendSafely(NotificationSender sender, String msg) {
        try {
            sender.send(msg);
        } catch (UnsupportedOperationException e) {
            System.out.println("❌ LSP vi phạm: " + e.getMessage());
        }
    }
}

// --- VI PHẠM LSP ---
class NotificationSender {
    public void send(String message) {
        System.out.println("Sending: " + message);
    }
}

class EmailNotificationSender extends NotificationSender {
    @Override
    public void send(String message) {
        System.out.println("📧 Email sent: " + message);
    }
}

// ❌ VI PHẠM: Class con từ chối thực hiện hợp đồng của cha
class ReadOnlyAuditLogger extends NotificationSender {
    @Override
    public void send(String message) {
        throw new UnsupportedOperationException("Audit logger không gửi thông báo!");
    }
}

// --- TUÂN THỦ LSP: Tách thành 2 interface riêng ---
interface Sendable {
    void send(String message);
}

interface Loggable {
    void log(String message);
}

class EmailSendable implements Sendable {
    @Override
    public void send(String message) {
        System.out.println("📧 [LSP-OK] Sent: " + message);
    }
}

class AuditLogger implements Loggable {
    @Override
    public void log(String message) {
        System.out.println("📝 [LSP-OK] Logged: " + message);
    }
}
// Output:
// === VI PHẠM LSP ===
// 📧 Email sent: Hello from email
// ❌ LSP vi phạm: Audit logger không gửi thông báo!
//
// === TUÂN THỦ LSP ===
// 📧 [LSP-OK] Sent: Notification content
// 📝 [LSP-OK] Logged: Audit record
```

### 4.2.4. I — Interface Segregation Principle (ISP)

> **"Client không nên bị ép phụ thuộc vào method mà nó KHÔNG dùng."**

```java
public class IspDemo {
    public static void main(String[] args) {
        // ===== VI PHẠM ISP: Interface quá béo =====
        // interface MonsterNotification {
        //     void sendEmail(String to, String msg);
        //     void sendSms(String phone, String msg);
        //     void sendSlack(String channel, String msg);
        //     void sendPush(String deviceToken, String msg);
        //     String formatHtml(String content);
        //     String formatPlainText(String content);
        //     void logToDatabase(String logEntry);
        // }
        // → 1 class chỉ cần gửi Email vẫn bị ÉP implement 7 methods!

        // ===== TUÂN THỦ ISP: Tách thành interface nhỏ, gọn =====
        System.out.println("=== ISP: Interface gọn, tập trung ===");

        MessageSender emailSender = new EmailMessageSender();
        MessageSender smsSender = new SmsMessageSender();

        // Mỗi class chỉ implement interface mà nó CẦN
        emailSender.send("admin@app.com", "Server health check passed");
        smsSender.send("0901234567", "OTP: 123456");

        // Class cần cả send + format → implement 2 interface
        FullEmailService fullService = new FullEmailService();
        String formatted = fullService.format("Welcome to our platform!");
        fullService.send("user@app.com", formatted);
    }
}

// Interface NHỎ, tập trung → client chỉ phụ thuộc vào những gì nó CẦN
interface MessageSender {
    void send(String to, String content);
}

interface MessageFormatter {
    String format(String rawContent);
}

// Class chỉ cần GỬI → implement MessageSender
class EmailMessageSender implements MessageSender {
    @Override
    public void send(String to, String content) {
        System.out.println("📧 [ISP] → " + to + ": " + content);
    }
}

class SmsMessageSender implements MessageSender {
    @Override
    public void send(String to, String content) {
        System.out.println("📱 [ISP] → " + to + ": " + content);
    }
}

// Class cần CẢ HAI → implement 2 interface (composition thay vì 1 interface béo)
class FullEmailService implements MessageSender, MessageFormatter {
    @Override
    public void send(String to, String content) {
        System.out.println("📧 [ISP Full] → " + to + ": " + content);
    }

    @Override
    public String format(String rawContent) {
        return "<html><body><h1>" + rawContent + "</h1></body></html>";
    }
}
// Output:
// === ISP: Interface gọn, tập trung ===
// 📧 [ISP] → admin@app.com: Server health check passed
// 📱 [ISP] → 0901234567: OTP: 123456
// 📧 [ISP Full] → user@app.com: <html><body><h1>Welcome to our platform!</h1></body></html>
```

### 4.2.5. D — Dependency Inversion Principle (DIP)

> **"Module bậc cao KHÔNG phụ thuộc module bậc thấp. Cả hai đều phụ thuộc vào ABSTRACTION (interface)."**
> Đây chính là nền tảng của **Dependency Injection** trong Spring Boot.

```java
import java.util.*;

public class DipDemo {
    public static void main(String[] args) {
        // ===== VI PHẠM DIP =====
        // class OrderService {
        //     private MySqlOrderRepository repo = new MySqlOrderRepository(); // ← Phụ thuộc TRỰC TIẾP
        //     private SmtpEmailSender sender = new SmtpEmailSender();        // ← vào implementation
        // }
        // → Đổi database? Đổi email provider? → SỬA OrderService!

        // ===== TUÂN THỦ DIP: Phụ thuộc vào INTERFACE =====
        System.out.println("=== DIP: Dependency Injection ===\n");

        // "Wiring" — Trong Spring Boot, framework làm việc này tự động (@Autowired)
        OrderRepository repo = new InMemoryOrderRepository();
        NotificationPort notifier = new ConsoleNotifier();

        // OrderService KHÔNG BIẾT implementation cụ thể
        OrderService orderService = new OrderService(repo, notifier);

        orderService.placeOrder("USR-001", 2_500_000);
        orderService.placeOrder("USR-002", 750_000);

        System.out.println("\n--- Đổi sang implementation khác (không sửa OrderService) ---\n");

        // Đổi cách thông báo → CHỈ ĐỔI config, KHÔNG sửa OrderService
        NotificationPort slackNotifier = new SlackNotifier();
        OrderService orderServiceV2 = new OrderService(repo, slackNotifier);
        orderServiceV2.placeOrder("USR-003", 1_200_000);
    }
}

// ABSTRACTION (Interface) — Module bậc cao VÀ bậc thấp đều phụ thuộc vào đây
interface OrderRepository {
    void save(String orderId, double amount);
    int count();
}

interface NotificationPort {
    void notify(String userId, String message);
}

// MODULE BẬC CAO: OrderService — chứa business logic
// Chỉ biết INTERFACE, không biết implementation
class OrderService {
    private final OrderRepository repository;    // Interface, không phải class cụ thể
    private final NotificationPort notifier;      // Interface, không phải class cụ thể

    // Constructor Injection — cách Spring Boot inject dependencies
    OrderService(OrderRepository repository, NotificationPort notifier) {
        this.repository = repository;
        this.notifier = notifier;
    }

    public void placeOrder(String userId, double amount) {
        String orderId = "ORD-" + System.currentTimeMillis() % 10000;
        repository.save(orderId, amount);
        notifier.notify(userId, "Đơn hàng " + orderId + " (" +
                String.format("%,.0f", amount) + "đ) đã được tạo!");
        System.out.println("   Total orders in DB: " + repository.count());
    }
}

// MODULE BẬC THẤP 1: Lưu vào memory
class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Double> store = new HashMap<>();

    @Override
    public void save(String orderId, double amount) {
        store.put(orderId, amount);
        System.out.println("💾 [InMemory] Saved " + orderId);
    }

    @Override
    public int count() { return store.size(); }
}

// MODULE BẬC THẤP 2: Thông báo qua console
class ConsoleNotifier implements NotificationPort {
    @Override
    public void notify(String userId, String message) {
        System.out.println("🔔 [Console] → " + userId + ": " + message);
    }
}

// MODULE BẬC THẤP 3: Thông báo qua Slack (thêm MỚI — không sửa gì)
class SlackNotifier implements NotificationPort {
    @Override
    public void notify(String userId, String message) {
        System.out.println("💬 [Slack] → #channel-" + userId + ": " + message);
    }
}
// Output:
// === DIP: Dependency Injection ===
//
// 💾 [InMemory] Saved ORD-3847
// 🔔 [Console] → USR-001: Đơn hàng ORD-3847 (2,500,000đ) đã được tạo!
//    Total orders in DB: 1
// 💾 [InMemory] Saved ORD-3848
// 🔔 [Console] → USR-002: Đơn hàng ORD-3848 (750,000đ) đã được tạo!
//    Total orders in DB: 2
//
// --- Đổi sang implementation khác (không sửa OrderService) ---
//
// 💾 [InMemory] Saved ORD-3849
// 💬 [Slack] → #channel-USR-003: Đơn hàng ORD-3849 (1,200,000đ) đã được tạo!
//    Total orders in DB: 3
```

### 4.2.6. Bảng tổng kết SOLID

| Nguyên tắc | Tóm tắt 1 câu | Vi phạm? | Tuân thủ? |
|-----------|---------------|----------|----------|
| **S**ingle Responsibility | 1 class = 1 lý do thay đổi | God class làm tất cả | Tách EmailSender, Formatter, Logger |
| **O**pen/Closed | Mở rộng = TẠO mới, không SỬA cũ | if-else chain cho loại mới | Interface + Đa hình |
| **L**iskov Substitution | Class con thay thế cha không phá vỡ | Class con throw UnsupportedOp | Tách interface phù hợp |
| **I**nterface Segregation | Interface nhỏ, tập trung | Interface 10 methods, client chỉ dùng 2 | Nhiều interface nhỏ |
| **D**ependency Inversion | Phụ thuộc abstraction, không implementation | `new MySqlRepo()` trong service | Constructor Injection |

---

## 🔥 4.3. GÓC PHỎNG VẤN — PHẦN 4

### ❓ Câu 1: "Stream API có phải luôn nhanh hơn for loop? Khi nào KHÔNG nên dùng Stream?"

**Đáp án sắc bén:**

> *"**KHÔNG.** Stream thường **chậm hơn** for loop thuần với dữ liệu nhỏ vì có overhead tạo pipeline (tạo object Stream, Spliterator, lambda capture).*
>
> *Không nên dùng Stream khi:*
> 1. *Logic phức tạp cần `break`, `continue`, hoặc modify biến ngoài → for loop rõ ràng hơn*
> 2. *Cần hiệu năng tuyệt đối với collection nhỏ (< 1000 phần tử) → overhead không đáng*
> 3. *Code trở nên khó đọc khi chain quá dài (> 5-6 operations)*
>
> *Nên dùng Stream khi: Cần filter/map/reduce trên dữ liệu lớn, cần parallel processing, hoặc cần code declarative dễ maintain. Trong Spring Boot, Stream dùng ở Service layer xử lý business logic rất phổ biến."*

---

### ❓ Câu 2: "Lambda Expression bản chất là gì? Nó khác Anonymous Inner Class thế nào ở tầng bytecode?"

**Đáp án sắc bén:**

> | Tiêu chí | Anonymous Inner Class | Lambda Expression |
> |----------|----------------------|-------------------|
> | **Bytecode** | Compiler tạo file `.class` riêng (VD: `Main$1.class`) | Compiler dùng `invokedynamic` → JVM tạo class ẩn lúc runtime |
> | **Object tạo ra** | Luôn tạo object MỚI mỗi lần | JVM có thể **cache** và tái sử dụng (nếu non-capturing) |
> | **`this` keyword** | `this` = instance của Anonymous class | `this` = instance của class bao ngoài (enclosing class) |
> | **Hiệu năng** | Chậm hơn (class loading, object creation) | Nhanh hơn (invokedynamic, có thể inline) |
> | **Hạn chế** | Có thể implement interface có nhiều method | Chỉ dùng cho **Functional Interface** (1 abstract method) |
>
> *"Lambda = syntactic sugar, nhưng ở tầng JVM nó được tối ưu hơn Anonymous Class nhờ `invokedynamic` (JSR 292). JVM có quyền quyết định cách tạo instance tối ưu nhất lúc runtime."*

---

### ❓ Câu 3: "Giải thích nguyên tắc Dependency Inversion. Nó liên quan gì đến @Autowired trong Spring Boot?"

**Đáp án sắc bén:**

> *"DIP nói: Module bậc cao (Service) KHÔNG phụ thuộc module bậc thấp (Repository). Cả hai phụ thuộc vào Abstraction (Interface).*
>
> *`@Autowired` là cơ chế **Dependency Injection** (DI) của Spring — cách TRIỂN KHAI nguyên tắc DIP:*
> 1. *Bạn khai báo field kiểu **Interface**: `private UserService userService;`*
> 2. *Spring IoC Container quét classpath, tìm class nào `@Component`/`@Service` implement interface đó*
> 3. *Spring **inject** (tiêm) instance cụ thể vào lúc runtime*
>
> *Lợi ích: (1) Đổi implementation chỉ cần đổi `@Primary` hoặc `@Profile`, không sửa business logic. (2) Unit test dễ dàng — inject Mock thay vì real implementation. (3) Code tuân thủ OCP — mở rộng bằng class mới.*
>
> *Best practice: Dùng **Constructor Injection** (không dùng field injection @Autowired trực tiếp) vì rõ ràng, testable, và bắt buộc dependency ngay lúc khởi tạo."*

---

> **KẾT THÚC PHẦN 4** — Bạn đã nắm: Stream API (5 bài toán thực chiến), Lambda (bản chất invokedynamic), Optional (khai tử null), Method Reference, và 5 nguyên tắc SOLID áp dụng hoàn chỉnh vào hệ thống Notification.
