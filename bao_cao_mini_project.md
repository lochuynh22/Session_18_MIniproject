# BÁO CÁO GIẢI PHÁP KỸ THUẬT & TRIỂN KHAI DỰ ÁN

## HỆ THỐNG BACKEND NGÂN HÀNG SỐ "RIKKEIBANK API" (MICROSERVICES ARCHITECTURE)

---

## PHẦN I: TỔNG QUAN DỰ ÁN & PHÂN TÍCH YÊU CẦU NGHIỆP VỤ

### 1. Bối cảnh và Lý do chuyển dịch kiến trúc (Monolith sang Microservices)

- **Thực trạng Monolith của RikkeiBank:** Hệ thống tập trung gặp tắc nghẽn khi khối lượng giao dịch tăng cao, rủi ro sập toàn bộ hệ thống khi một module gặp sự cố (Single Point of Failure), khó mở rộng riêng lẻ các tác vụ nặng (như truy vấn biến động số dư hoặc xử lý giao dịch chuyển khoản cao điểm).

- Lý do lựa chọn Microservices Architecture (MSA):

- **Tách rời (Decoupling):** Các nghiệp vụ quản lý tài khoản, khách hàng, giao dịch tiền tệ được cô lập, tránh việc một lỗi logic ở module danh mục làm treo module thanh toán.

- **Mở rộng độc lập (Independent Scalability):** Dịch vụ giao dịch (`transaction-service`) và tài khoản (`account-service`) có thể scale-out (tăng số lượng instance) độc lập mà không cần nhân bản toàn bộ hệ thống.

- **Độ tin cậy & Chịu lỗi (Fault Isolation):** Kết hợp Circuit Breaker đảm bảo lỗi từ một dịch vụ phụ thuộc không dẫn tới sập dây chuyền (Cascading Failure).

### 2. Ma trận Phân quyền & Trải nghiệm Người dùng (RBAC & UX Matrix)

| Vai trò (Role)       | Phạm vi Quyền hạn (Permissions)                          | Trải nghiệm & Cơ chế Bảo mật |
| -------------------- | -------------------------------------------------------- | ---------------------------- |
| **GUEST (Vãng lai)** | Chỉ có quyền Đăng nhập, Đăng ký, xem API docs công khai. |

| Bị chặn toàn bộ dữ liệu tài chính (HTTP 401 Unauthorized).

|
| **CUSTOMER (Khách hàng)** | Xem thông tin hồ sơ của chính mình; xem số dư, tài khoản sở hữu; xem lịch sử giao dịch của mình; tạo lệnh chuyển khoản.

| **Liền mạch (Seamless UX):** Dùng cặp Access Token (ngắn hạn: 15 phút) và Refresh Token (dài hạn: 30–90 ngày) để duy trì phiên làm việc không bị gián đoạn khi mở app.

|
| **TELLER (Giao dịch viên)** | Xem danh sách các giao dịch được thực hiện trong ngày thuộc phạm vi phân công; duyệt/xử lý hồ sơ khách hàng.

| Không được can thiệp vào giao dịch của giao dịch viên khác (phân tách theo `tellerId` / `branchId`).

|
| **ADMIN (Quản trị viên)** | Toàn quyền CRUD Khách hàng, Nhân viên, Danh mục Loại tài khoản; Giám sát hệ thống; **Cưỡng chế đăng xuất (Force Logout)** tài khoản bất kỳ.

| Khi thực hiện Force Logout, hệ thống đưa JWT của tài khoản đó vào Redis Blacklist, hủy hiệu lực token ngay tức thì.

|

---

## PHẦN II: THIẾT KẾ KIẾN TRÚC HỆ THỐNG TỔNG THỂ (SYSTEM ARCHITECTURE)

```
                            [ Mobile Client / Web Client / Postman ]
                                               │
                                               ▼
                              [ Spring Cloud Gateway (Port 8080) ]
                    (JWT Validation Filter, Rate Limiting, Route Balancing)
                                               │
                ┌──────────────────────────────┼──────────────────────────────┐
                │                              │                              │
                ▼                              ▼                              ▼
      [ Identity Service ]           [ Customer Service ]           [ Account Service ]
         (Port: 8081)                   (Port: 8082)                   (Port: 8083)
         DB: identity_db                DB: customer_db                DB: account_db
                │                              │                              │
                └───────────────┬──────────────┴──────────────┬───────────────┘
                                │                             │
                                ▼                             ▼
                    [ Transaction Service ]        [ Notification Service ]
                         (Port: 8084)                   (Port: 8085)
                       DB: transaction_db            (WebFlux + Kafka)
                                │                             ▲
                                └───────── [ Apache Kafka ] ──┘
                                      (Event-Driven Topics)

   Hạ tầng bổ trợ:
   - Spring Cloud Config Server (Port 8888)[cite: 1]
   - Spring Cloud Eureka Discovery Server (Port 8761)[cite: 1]
   - Redis Cluster/Single Node (Port 6379): Cache-Aside & Token Blacklist[cite: 1]

```

### 1. Phân định Bounded Context và Database-per-Service

Mỗi microservice sở hữu cơ sở dữ liệu riêng biệt, cấm hoàn toàn hành vi query chéo database (Cross-DB Joins):

1. **`identity-service` (`identity_db`):** Quản lý User Credentials, Role, Permission, Auth Refresh Tokens.

2. **`customer-service` (`customer_db`):** Quản lý hồ sơ định danh cá nhân (Customer KYC: CCCD, Ngày sinh, Địa chỉ, SĐT) và danh sách Nhân viên (Staff/Teller).

3. **`account-service` (`account_db`):** Quản lý danh mục loại tài khoản (Account Type: Tiết kiệm, Thanh toán), thông tin tài khoản ngân hàng, số dư khả dụng (Balance), trạng thái khóa/mở.

4. **`transaction-service` (`transaction_db`):** Quản lý điều phối giao dịch chuyển khoản (Saga Orchestrator), lưu vết lịch sử giao dịch (Transaction Logs, Debit/Credit ledger).

5. **`notification-service` (`notification_db` / in-memory reactive):** Nhận sự kiện từ Kafka và gửi cảnh báo biến động số dư, email thông báo.

---

## PHẦN III: THIẾT KẾ CƠ SỞ DỮ LIỆU & ĐẶC TẢ API ENDPOINTS

### 1. Thiết kế Lược đồ Dữ liệu (Database Schema)

#### a. `identity_db` (Service: `identity-service`)

- **`users`:** `id` (BIGINT, PK), `username` (VARCHAR, Unique), `password_hash` (VARCHAR), `role` (ENUM: 'ADMIN', 'TELLER', 'CUSTOMER'), `status` (VARCHAR: 'ACTIVE', 'BLOCKED'), `created_at` (TIMESTAMP).
- **`refresh_tokens`:** `id` (BIGINT, PK), `user_id` (BIGINT), `token` (VARCHAR, Unique), `expiry_date` (TIMESTAMP), `revoked` (BOOLEAN).

#### b. `customer_db` (Service: `customer-service`)

- **`customers`:** `id` (BIGINT, PK), `user_id` (BIGINT, Unique - liên kết logic với Identity), `full_name` (VARCHAR), `identity_card_number` (VARCHAR, Unique), `phone` (VARCHAR), `email` (VARCHAR), `address` (TEXT), `status` (VARCHAR).
- **`staffs`:** `id` (BIGINT, PK), `user_id` (BIGINT, Unique), `staff_code` (VARCHAR, Unique), `full_name` (VARCHAR), `branch_id` (VARCHAR), `department` (VARCHAR).

#### c. `account_db` (Service: `account-service`)

- **`account_types`:** `id` (BIGINT, PK), `type_code` (VARCHAR, Unique: 'SAVING', 'CHECKING'), `name` (VARCHAR), `description` (TEXT), `interest_rate` (DECIMAL), `status` (VARCHAR).
- **`accounts`:** `id` (BIGINT, PK), `account_number` (VARCHAR, Unique - 10-12 số), `customer_id` (BIGINT - logic ID), `account_type_id` (BIGINT, FK), `balance` (DECIMAL(19, 4)), `currency` (VARCHAR: 'VND'), `status` (VARCHAR: 'ACTIVE', 'LOCKED'), `created_at` (TIMESTAMP).

#### d. `transaction_db` (Service: `transaction-service`)

- **`transactions`:** `id` (VARCHAR/UUID, PK), `source_account_number` (VARCHAR), `destination_account_number` (VARCHAR), `amount` (DECIMAL(19, 4)), `content` (TEXT), `status` (ENUM: 'PENDING', 'SUCCESS', 'FAILED', 'COMPENSATED'), `failure_reason` (TEXT), `teller_id` (BIGINT, Nullable - ghi nhận nếu do Teller thực hiện), `created_at` (TIMESTAMP).
- **`saga_transaction_events`:** `event_id` (UUID, PK), `transaction_id` (UUID), `step` (VARCHAR), `status` (VARCHAR), `payload` (JSON), `updated_at` (TIMESTAMP).

---

### 2. Danh mục RESTful API Endpoints Chi tiết

| Service      | Method | URI Pattern          | Mô tả                                    | Phân quyền (@PreAuthorize) | HTTP Statuses |
| ------------ | ------ | -------------------- | ---------------------------------------- | -------------------------- | ------------- |
| **Identity** | `POST` | `/api/v1/auth/login` | Đăng nhập lấy cặp Access & Refresh Token |

| Public | 200 OK, 401 Unauthorized |
| | `POST` | `/api/v1/auth/refresh-token` | Làm mới Access Token

| Public (kèm Refresh Token) | 200 OK, 403 Forbidden |
| | `POST` | `/api/v1/admin/users/{id}/force-logout` | Cưỡng chế đăng xuất (thu hồi token)

| `hasRole('ADMIN')`<br> | 200 OK, 404 Not Found |
| **Customer** | `POST` | `/api/v1/customers` | Tạo mới hồ sơ khách hàng

| `hasRole('ADMIN')`<br> | 201 Created, 400 Bad Request |
| | `GET` | `/api/v1/customers/me` | Xem hồ sơ cá nhân hiện tại

| `hasRole('CUSTOMER')`<br> | 200 OK, 403 Forbidden |
| | `GET` | `/api/v1/customers` | Quản lý danh sách khách hàng

| `hasRole('ADMIN')`<br> | 200 OK, 403 Forbidden |
| | `POST` | `/api/v1/staffs` | Thêm nhân viên/Giao dịch viên

| `hasRole('ADMIN')`<br> | 201 Created |
| **Account** | `GET` | `/api/v1/account-types` | Xem danh mục loại tài khoản (có Redis Cache)

| `isAuthenticated()` | 200 OK |
| | `POST` | `/api/v1/account-types` | Thêm mới loại tài khoản

| `hasRole('ADMIN')`<br> | 201 Created |
| | `GET` | `/api/v1/accounts/my-accounts` | Xem danh sách tài khoản của chính mình

| `hasRole('CUSTOMER')`<br> | 200 OK |
| | `POST` | `/api/v1/internal/accounts/debit` | Trừ tiền tài khoản (API nội bộ Saga)

| Service-to-Service | 200 OK, 400 Insufficient Funds |
| | `POST` | `/api/v1/internal/accounts/credit` | Cộng tiền tài khoản (API nội bộ Saga)

| Service-to-Service | 200 OK, 404 Not Found |
| **Transaction** | `POST` | `/api/v1/transfers` | Khởi tạo lệnh chuyển khoản liên tài khoản

| `hasRole('CUSTOMER')`<br> | 202 Accepted / 200 OK |
| | `GET` | `/api/v1/transfers/my-history` | Xem lịch sử giao dịch cá nhân

| `hasRole('CUSTOMER')`<br> | 200 OK |
| | `GET` | `/api/v1/transfers/daily-reports` | Xem danh sách giao dịch trong ngày theo Teller

| `hasRole('TELLER')`<br> | 200 OK, 403 Forbidden |

---

## PHẦN IV: THIẾT KẾ CÁC GIẢI PHÁP KỸ THUẬT CỐT LÕI

### 1. Phân quyền, JWT & Trải nghiệm Đăng nhập (Seamless Session & Force Logout)

#### a. Cơ chế Duy trì Phiên Liền mạch (Seamless Login Experience)

- **Vấn đề:** Khách hàng không muốn nhập mật khẩu liên tục mỗi khi mở ứng dụng.

- **Giải pháp:** Sử dụng mô hình **Dual Token (Access Token + Refresh Token)**:

- `AccessToken`: Chứa Claims (`userId`, `username`, `roles`), thời hạn ngắn (15 phút), được ký số (HMAC-SHA256 hoặc RSA).

- `RefreshToken`: Chuỗi định danh ngẫu nhiên mã hóa lưu tại `identity_db`, thời hạn dài (60 ngày). Khi mở ứng dụng, nếu `AccessToken` hết hạn, app ngầm gọi `POST /api/v1/auth/refresh-token` để cấp mới `AccessToken` mà người dùng không nhận biết.

#### b. Cơ chế Cưỡng chế Đăng xuất Ngay lập tức (Admin Forced Logout via Redis)

- **Vấn đề:** JWT là stateless, thông thường không thể thu hồi trước hạn cho tới khi token tự hết hạn. Khi Admin phát hiện gian lận hoặc nghi ngờ tài khoản bị chiếm đoạt, cần ép đăng xuất ngay.

- Giải pháp: Distributed Token Blacklist bằng Redis:

1. Khi Admin gọi `POST /api/v1/admin/users/{userId}/force-logout`, `identity-service` tìm token hiện tại của user và ghi vào Redis:
   `KEY: "blacklist:token:" + jwt_token`, `VALUE: "revoked"`, với `TTL` = thời gian còn lại của token.

2. Tại **Spring Cloud Gateway (hoặc JwtAuthenticationFilter)**, mỗi request gửi lên sẽ được check nhanh với Redis:

```java
String isRevoked = redisTemplate.opsForValue().get("blacklist:token:" + token);
if (isRevoked != null) {
    // Chặn đứng ngay lập tức, trả về lỗi 401 Unauthorized
    exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
    return exchange.getResponse().setComplete();
}

```

3. Chi phí kiểm tra trên Redis In-memory cực kỳ thấp (< 1ms) và TTL tự động giải phóng RAM khi token hết hạn tự nhiên.

---

### 2. Giao dịch Phân tán với Saga Pattern (Chuyển khoản 2 tài khoản)

Vì áp dụng nguyên tắc _Database-per-service_, nghiệp vụ chuyển khoản từ Tài khoản A (ở `account-service`) sang Tài khoản B và lưu vết vào `transaction-service` không thể dùng `@Transactional` ACID truyền thống của hệ quản trị cơ sở dữ liệu. Chúng ta áp dụng **Saga Orchestrator Pattern** do `transaction-service` làm nhạc trưởng điều phối:

```
[Customer] ---> (1. POST /transfers) ---> [Transaction Service (Orchestrator)]
                                                    │
                 ┌──────────────────────────────────┴──────────────────────────────────┐
                 │                                                                     │
          [Bước 1: Trừ tiền]                                                    [Bước 2: Cộng tiền]
                 ▼                                                                     ▼
      OpenFeign: /accounts/debit                                            OpenFeign: /accounts/credit
      (Tài khoản A: -500.000đ)                                              (Tài khoản B: +500.000đ)
                 │                                                                     │
         [Thành công? YES]                                                     [Thất bại? NO]
                 │                                                                     │
                 ▼                                                                     ▼
        Tiếp tục Bước 2                                                      [KÍCH HOẠT COMPENSATING]
                                                                                       │
                                                                                       ▼
                                                                           OpenFeign: /accounts/credit
                                                                           (Hoàn trả: A: +500.000đ)
                                                                                       │
                                                                                       ▼
                                                                           Cập nhật Transaction: FAILED

```

#### Chi tiết Luồng Happy Path (Thành công)

:

1. Khách hàng gửi yêu cầu chuyển khoản: Gửi `sourceAccount`, `destAccount`, `amount`.

2. `transaction-service` tạo bản ghi giao dịch trạng thái `PENDING`.

3. `transaction-service` gọi đồng bộ qua OpenFeign tới `account-service` endpoint `/api/v1/internal/accounts/debit` để trừ tiền tài khoản nguồn.

4. Trừ tiền thành công, `transaction-service` tiếp tục gọi `account-service` endpoint `/api/v1/internal/accounts/credit` để cộng tiền vào tài khoản đích.

5. Cộng tiền thành công, `transaction-service` cập nhật trạng thái `SUCCESS`.

6. Bắn sự kiện `TransactionSuccessEvent` lên Apache Kafka Topic: `bank.transaction.completed`.

#### Chi tiết Luồng Compensating (Rollback khi gặp sự cố)

:

1. Giả định: Trừ tiền tài khoản A thành công, nhưng khi gọi cộng tiền tài khoản B thì phát sinh lỗi (Tài khoản B bị khóa, sai số tài khoản đích, hoặc `account-service` crash).

2. `transaction-service` bắt được ngoại lệ (Exception).

3. **Thực thi giao dịch bù trừ (Compensating Transaction):** `transaction-service` lập tức gọi lại API hoàn tiền `/api/v1/internal/accounts/credit` để hoàn lại đúng số tiền đã trừ cho tài khoản A.

4. Cập nhật trạng thái giao dịch thành `COMPENSATED` hoặc `FAILED` kèm lý do lỗi.

5. Bắn sự kiện `TransactionFailedEvent` lên Kafka để lưu vết kiểm toán và thông báo hoàn tiền.

---

### 3. Kiến trúc Hướng Sự kiện (Event-Driven với Apache Kafka & Spring WebFlux)

- **Producer:** `transaction-service` sau khi chốt trạng thái giao dịch thành công sẽ xuất bản (publish) message vào Kafka Topic `bank.transaction.notifications`.

- **Consumer:** `notification-service` được xây dựng bằng **Spring WebFlux (Reactive Framework)** lắng nghe từ Topic thông qua Kafka Reactive Receiver / Spring Kafka Consumer.

- **Payload của Kafka Event:**

```json
{
  "eventId": "f9a4e8d2-1c2b-4e89-9a8d-7a6b5c4d3e2f",
  "eventType": "TRANSACTION_SUCCESS",
  "transactionId": "TXN_99887766",
  "sourceAccount": "1000000001",
  "destAccount": "2000000002",
  "amount": 500000.0,
  "timestamp": "2026-09-25T10:30:00Z",
  "recipientEmail": "customer@rikkeibank.vn",
  "message": "Tài khoản 1000000001 biến động -500,000 VND. Số dư hiện tại: 4,500,000 VND."
}
```

- **Lợi ích:** Đảm bảo tính liên kết lỏng (Loose Coupling). Nếu `notification-service` bị tạm dừng hoặc khởi động lại, các message trong Kafka Topic không bị mất mà sẽ được xử lý bù ngay khi service hoạt động trở lại, không làm chậm hoặc ảnh hưởng tới luồng chuyển tiền chính của khách hàng.

---

### 4. Kháng lỗi Hệ thống (Fault Tolerance với Resilience4j Circuit Breaker)

Để ngăn chặn lỗi dây chuyền (Cascading Failure) khi `transaction-service` gọi `account-service` hoặc `customer-service`:

- **Cấu hình Circuit Breaker (`application.yml`):**

```yaml
resilience4j.circuitbreaker:
  instances:
    accountServiceCB:
      sliding-window-type: COUNT_BASED
      sliding-window-size: 10
      minimum-number-of-calls: 5
      failure-rate-threshold: 50
      wait-duration-in-open-state: 10s
      permitted-number-of-calls-in-half-open-state: 3
      automatic-transition-from-open-to-half-open-enabled: true
```

- **3 Trạng thái hoạt động:**
- **CLOSED:** Trạng thái bình thường, các request được thông qua tự do.

- **OPEN:** Khi tỷ lệ lỗi vượt ngưỡng 50%, Circuit Breaker ngắt kết nối ngay lập tức, từ chối gửi request đến `account-service` và lập tức kích hoạt phương thức Fallback để tránh treo luồng hệ thống.

- **HALF-OPEN:** Sau 10 giây chờ, cho phép thử nghiệm 3 request. Nếu thành công -> chuyển về CLOSED; nếu tiếp tục lỗi -> quay lại OPEN.

- **Cơ chế Fallback:**

```java
@CircuitBreaker(name = "accountServiceCB", fallbackMethod = "handleAccountServiceFallback")
public AccountDto getAccountDetails(String accountNumber) {
    return accountFeignClient.getAccount(accountNumber);
}

public AccountDto handleAccountServiceFallback(String accountNumber, Throwable t) {
    log.error("Account-service đang gặp sự cố. Kích hoạt Fallback: {}", t.getMessage());
    return new AccountDto(accountNumber, BigDecimal.ZERO, "SERVICE_UNAVAILABLE_FALLBACK");
}

```

---

### 5. Bộ nhớ đệm Phân tán (Distributed Caching với Redis - Cache-Aside Pattern)

Áp dụng cho danh mục Loại tài khoản (`Account Type`) và Hồ sơ khách hàng thường xuyên tra cứu nhưng ít thay đổi:

- Chiến lược Cache-Aside:

- **Đọc (`@Cacheable`):** Kiểm tra dữ liệu trong Redis. Nếu có (Cache Hit) -> trả về ngay. Nếu chưa có (Cache Miss) -> query DB, nạp kết quả vào Redis với TTL (ví dụ: 60 phút) rồi trả về cho client.

- **Cập nhật (`@CachePut` / `@CacheEvict`):** Khi Admin sửa hoặc xóa loại tài khoản, sử dụng `@CacheEvict(value = "accountTypes", key = "#id")` hoặc `@CacheEvict(allEntries = true)` để xóa cache cũ, đảm bảo dữ liệu không bị sai lệch (Stale Data).

---

### 6. Kiểm soát Luồng lỗi Tập trung (AOP Global Exception Handling)

Xây dựng `@RestControllerAdvice` trong module dùng chung (`common-library`) để chuẩn hóa định dạng phản hồi lỗi:

```json
{
  "timestamp": "2026-09-25T10:35:12Z",
  "status": 400,
  "errorCode": "INSUFFICIENT_BALANCE",
  "message": "Số dư tài khoản nguồn không đủ để thực hiện giao dịch",
  "path": "/api/v1/transfers"
}
```

---

## PHẦN V: KỊCH BẢN KIỂM THỬ, MINH CHỨNG & DEMO (DELIVERABLES VERIFICATION)

Dưới đây là 7 kịch bản cụ thể để thực hiện và đưa vào bản báo cáo kết quả (kèm hình ảnh log / Postman):

### Kịch bản 1: Khởi động Hạ tầng & Service Discovery

- **Các bước:**

1. Khởi chạy `config-server` (Port 8888).

2. Khởi chạy `discovery-server` (Eureka Dashboard tại `http://localhost:8761`).

3. Khởi chạy `api-gateway`, `identity-service`, `customer-service`, `account-service`, `transaction-service`, `notification-service`.

- **Kết quả cần chụp minh chứng:** Dashboard Eureka hiển thị đầy đủ tên các service với trạng thái `UP (n)`.

### Kịch bản 2: Luồng Chuyển khoản Thành công (Saga Success Flow)

- **Dữ liệu chuẩn bị:** Tài khoản A (Số dư 2.000.000đ), Tài khoản B (Số dư 500.000đ).
- **Thao tác:** Customer A gọi `POST /api/v1/transfers` chuyển 500.000đ cho Tài khoản B.

- **Kết quả:**
- Tài khoản A còn 1.500.000đ.

- Tài khoản B tăng lên 1.000.000đ.

- Bảng `transactions` ghi nhận bản ghi với trạng thái `SUCCESS`.

### Kịch bản 3: Kịch bản Bù trừ Thất bại (Saga Rollback / Compensating Flow)

- **Thao tác chủ động gây lỗi:** Tạo lệnh chuyển tiền từ Tài khoản A tới một Tài khoản B không tồn tại hoặc cố tình mock lỗi 500 tại bước Credit.

- **Kết quả:**
- Bước 1: Tài khoản A bị trừ 500.000đ.

- Bước 2: Gọi credit lỗi -> Saga Orchestrator kích hoạt Compensating call.

- Bước 3: Tài khoản A được hoàn lại 500.000đ (Số dư bảo toàn đúng 2.000.000đ, không bị thất thoát).

- Bản ghi transaction có trạng thái `COMPENSATED` / `FAILED`.

### Kịch bản 4: Xử lý Bất đồng bộ với Kafka (Event-Driven Notification)

- **Thao tác:** Thực hiện giao dịch chuyển tiền thành công.

- **Kết quả:**
- Quan sát console của `notification-service`: Nhận được event từ Kafka Topic ngay lập tức và in log: `"Đã gửi thông báo biến động số dư tới email..."`.

### Kịch bản 5: Kháng lỗi Dây chuyền (Circuit Breaker Tripping)

- **Thao tác:** Chủ động dừng (Stop) instance của `account-service`, sau đó liên tục gửi 10 request từ client tới `transaction-service`.

- **Kết quả:**
- 5 request đầu ghi nhận lỗi kết nối.

- Từ request thứ 6 trở đi, Circuit Breaker chuyển trạng thái từ `CLOSED` sang `OPEN`. Request lập tức nhận được Fallback message mà không bị treo connection (tránh nghẽn thread pool).

### Kịch bản 6: Tối ưu Tốc độ với Redis Caching

- **Thao tác:** Gọi `GET /api/v1/account-types` lần 1 (Cache Miss - query DB, mất khoảng ~60-100ms); gọi lần 2 (Cache Hit từ Redis, mất < 10ms).

- **Thao tác Evict:** Admin gọi update thông tin loại tài khoản -> Kiểm tra key trong Redis bị xóa (`@CacheEvict`) và được làm mới ở lần đọc tiếp theo.

### Kịch bản 7: Cưỡng chế Đăng xuất (Admin Force Logout)

- **Thao tác:**

1. Customer đăng nhập, nhận JWT token, gọi API `/api/v1/accounts/my-accounts` thành công.

2. Admin gọi `POST /api/v1/admin/users/{customerId}/force-logout`.

3. Customer dùng lại đúng token cũ gọi lại API -> Nhận ngay mã lỗi `401 Unauthorized` kèm thông báo `"Token has been revoked"`.

---

## PHẦN VI: TỔ CHỨC SOURCE CODE & CẤU TRÚC BÀN GIAO (DELIVERABLES)

### 1. Cấu trúc Thư mục Dự án Đề xuất (Multi-Module Maven)

```text
rikkeibank-microservices/
├── pom.xml                               # Root POM quản lý dependency versions
├── common-library/                       # DTOs, Base Exceptions, Security Utils
├── config-server/                        # Spring Cloud Config Server (Port 8888)
├── discovery-server/                     # Spring Cloud Netflix Eureka (Port 8761)
├── api-gateway/                          # Spring Cloud Gateway (Port 8080)
├── identity-service/                     # Auth & Token Blacklist (Port 8081)
├── customer-service/                     # Customer & Staff Management (Port 8082)
├── account-service/                      # Bank Accounts & Redis Cache (Port 8083)
├── transaction-service/                  # Transfers & Saga Orchestrator (Port 8084)
├── notification-service/                 # Kafka WebFlux Consumer (Port 8085)
└── docker-compose.yml                    # Khởi động PostgreSQL, Kafka, Zookeeper, Redis

```

### 2. Kiểm thử Chất lượng (Unit Test & JaCoCo Coverage)

- Viết đầy đủ Unit Test cho tầng Service và Controller sử dụng **JUnit 5** và **Mockito**.

- Tích hợp plugin `jacoco-maven-plugin` vào file `pom.xml`, cấu hình ngưỡng bao phủ (coverage threshold) tối thiểu từ 70% trở lên. Chạy lệnh `mvn clean verify` và trích xuất báo cáo từ thư mục `target/site/jacoco/index.html` để nộp kèm.
